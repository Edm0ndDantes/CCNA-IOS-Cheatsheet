# How Switches & Routers Actually Forward — Frame Switching and Packet Routing Internals

## F.1 The division of labor

| Device | Looks at | Decision table | Scope |
|---|---|---|---|
| **Switch (L2)** | Destination **MAC** in the frame | MAC address table (CAM) | Within one VLAN/broadcast domain |
| **Router (L3)** | Destination **IP** in the packet | Routing table → FIB | Between networks |

The core contract: **L2 addresses change every hop, L3 addresses survive end-to-end.** A packet from Host A to a server across three routers keeps `src/dst IP` intact the whole way, while its frame is stripped and rebuilt with new `src/dst MAC` at every router.

## F.2 Switch: the three frame decisions

For every arriving frame, a switch does two things with the **source** MAC and one of three things with the **destination** MAC:

**Learning (source MAC):** record `source MAC → arrival port (+VLAN)` in the MAC table. Entries age out (default **300 s**) unless refreshed. If a MAC appears on a new port (device moved — or is being spoofed), the entry moves.

**Forwarding decision (destination MAC):**

| Case | Action | Name |
|---|---|---|
| Destination MAC in table, **different** port | Send out that one port only | **Forwarding** |
| Destination MAC in table, **same** port it arrived on | Drop (destination is already on that segment) | **Filtering** |
| Destination MAC **not in table** | Send out **all** ports in the VLAN except the arrival port | **Flooding** (unknown unicast) |
| Destination = broadcast `ffff.ffff.ffff` or (most) multicast | Send out all ports in the VLAN except arrival | Flooding by design |

- Flooding is why an idle listener still eventually "hears" traffic for it — its reply teaches the switch its port and stops the flood. Persistent unknown-unicast flooding (asymmetric paths, aging mismatches) is a real performance problem.
- **This is also the attack surface:** MAC flooding attacks overflow the CAM table so *everything* floods (making the switch a hub for eavesdropping) — the reason port security limits MACs per port.

**Frame handling modes** (how much of the frame is received before forwarding):

| Mode | Behavior |
|---|---|
| **Store-and-forward** | Receive the whole frame, verify the FCS, then forward — errors are dropped here. The modern default |
| **Cut-through** | Forward as soon as the destination MAC (first 6 bytes after preamble) is read — lowest latency, but propagates corrupt frames |
| Fragment-free | Cut-through after 64 bytes (past the collision window) — legacy compromise |

Switching happens in hardware (**ASICs** consulting **CAM/TCAM** memory) at line rate; the CPU sees only control-plane traffic (STP BPDUs, CDP, management).

## F.3 Memory buffering — where frames wait

Forwarding decisions are instant; **transmission is not**. Frames must be held whenever the egress port can't send *right now*:

- **Speed mismatch:** a burst arrives on a 10 Gb/s uplink destined for a 1 Gb/s access port — nine-tenths of it must wait somewhere.
- **Many-to-one (incast):** several ports send to the same destination port simultaneously — only one frame transmits at a time.
- **Store-and-forward itself:** the whole frame must be received and FCS-checked *somewhere* before forwarding.

The buffer memory that absorbs this is organized in one of two architectures:

| Architecture | How it works | Consequence |
|---|---|---|
| **Port-based buffering** | Each port owns a fixed, dedicated slice of memory with its own queues | Simple, but wasteful and fragile: a frame can be delayed by unrelated frames queued ahead of it, one congested port can fill *its* buffer and drop while other ports' buffers sit empty — memory can't be lent where it's needed |
| **Shared memory buffering** | All ports draw from one **common pool**, allocated dynamically to whichever port is congested *right now*. The frame is written to shared memory **once**; only a *pointer* to it moves from the ingress queue to the egress port's queue — no copying | Modern default. A single busy port can borrow a large fraction of the total buffer, absorbing bursts that would overflow a fixed per-port slice; ports also don't need to be the same speed, which is exactly the uplink-to-access-port case |

Two further details worth knowing:

- **Ingress vs egress queueing:** frames can queue at the input (before the forwarding decision/fabric) and at the output (waiting for the wire). Output queues dominate in practice — congestion is almost always an *egress* phenomenon (the decision is fast; the transmit is the bottleneck). Pure ingress queueing suffers **head-of-line blocking**: a frame stuck waiting for a busy output port blocks frames behind it that are bound for idle ports — one reason shared-memory/pointer designs and virtual output queues won.
- **When the buffer is full, frames drop** — this is a *tail drop*, and it's normal behavior under sustained oversubscription, not a fault. Buffers absorb **bursts**; they cannot fix a link that is simply too small (a bigger buffer there just adds delay — bufferbloat). Persistent drops are a capacity or QoS problem: QoS is precisely the toolset that decides *which* frames enter *which* queue and *what* drops first.

Where you see all of this in IOS:

| Command | Explanation |
|---|---|
| `show interfaces g0/1` | The queueing story per port: `Input queue: 0/75/0/0 (size/max/drops/flushes)`, `Total output drops`, and `output buffer failures` — output drops climbing = egress congestion on this port |
| `show interfaces g0/1 counters errors` | (Switches) Per-port drop/error counters in table form |
| `show buffers` | The router's shared buffer pools (small/middle/big…): `misses` and `failures` climbing = the pool can't keep up |
| `show platform hardware ... qos queue stats` | (Platform-dependent, e.g. Cat 9k) Hardware queue depths and drops per port — the ASIC-level truth |
| `clear counters g0/1` | Zero the counters, then watch whether drops are *currently* accumulating |

**Reading `Input queue: 0/75/0/0`:** current depth 0, max 75 packets, 0 drops, 0 flushes. A nonzero third number = packets arrived faster than the CPU/fabric drained them. On routers, chronic input-queue drops often mean traffic is being punted to the CPU (see `show cef not-cef-switched` in §F.6) rather than genuine line-rate congestion.

## F.4 Router: the routing decision

For every packet, conceptually:

1. **Sanity:** verify the IP header checksum; **decrement TTL** — if it hits 0, drop and send ICMP *Time Exceeded* (this is what traceroute exploits).
2. **Lookup:** find the **longest-prefix match** for the destination in the forwarding table. `/32` host route beats `/24` beats `/16` beats `0.0.0.0/0` — *specificity always wins*; AD and metric only break ties **between candidates for the same prefix** (AD picks the source, metric picks the path — that contest happens when the routing table is *built*, not per packet).
3. **No match at all** (and no default route): drop, send ICMP *Destination Unreachable*.
4. **Resolve the exit:** the route gives a next-hop IP and/or exit interface; recursive lookups resolve a next-hop to its own exit interface if needed.
5. **Rewrite the frame:** new L2 header — `src MAC` = router's exit interface, `dst MAC` = the next-hop's MAC (learned via **ARP** for IPv4 / **NDP** for IPv6). The IP packet inside is untouched except TTL and checksum.
6. If the next-hop MAC is unknown: hold/drop the packet, ARP for it, forward when resolved — the reason the *first* ping across a fresh path often times out while the rest succeed.

## F.5 The three switching paths (how the lookup is implemented)

"Routing" (building the table from static/OSPF/BGP — control plane) and "switching" a packet (moving it — data plane) are separate jobs. IOS has had three generations of the data-plane mechanism:

| Method | How it works | Cost |
|---|---|---|
| **Process switching** | Every packet is punted to the CPU; the full routing table is consulted per packet | Brutally slow — today only for packets that *require* the CPU (options, TTL expiry, destined-to-router) |
| **Fast switching** | First packet of a flow is process-switched; the result is cached ("route once, switch many"); subsequent packets hit the cache | Better, but the first packet is slow and cache invalidation is messy. Legacy |
| **CEF** (Cisco Express Forwarding) | Tables are **pre-built before any traffic arrives**: the **FIB** (routing table distilled to prefix → next-hop, optimized for longest-match lookup) + the **adjacency table** (next-hop → ready-made L2 rewrite string from ARP). Forwarding = one FIB lookup + paste the rewrite | Line rate, deterministic. The default everywhere; MPLS and most features require it |

- CEF's two tables mirror the two things step 5 above needs: *where* (FIB, fed by the routing table) and *how to dress the frame* (adjacency table, fed by ARP). When ARP hasn't resolved yet, the FIB entry points to a **glean** adjacency = "punt to CPU to ARP first".
- On multilayer switches the same FIB/adjacency logic is burned into **TCAM/ASIC** hardware — "routing at switching speed" is literally CEF in silicon.
- **Load sharing:** with equal routes, CEF balances **per-destination** (technically per src-dst hash) by default — per-packet balancing exists but reorders TCP and is avoided.

## F.6 Commands & verification

| Command | Explanation |
|---|---|
| `show mac address-table` | The switch's learned MACs: VLAN, MAC, type (`DYNAMIC`/`STATIC`), port |
| `show mac address-table address 000c.1234.abcd` | Find one host's port — the everyday "where is this device plugged in" |
| `show mac address-table aging-time` | The aging timer (default 300 s) |
| `clear mac address-table dynamic` | Flush learned MACs (watch relearning during loop hunts) |
| `show ip route <dest>` | The control-plane answer: which route wins for this destination |
| `show ip cef <dest> detail` | The data-plane answer: the exact FIB entry + next-hop + exit interface actually used — when routing "should" work but doesn't, compare this against `show ip route` |
| `show adjacency detail` | The pre-built L2 rewrite strings (you can read the future frame's MAC header in the hex) |
| `show ip cef summary` | FIB size, health |
| `show cef not-cef-switched` | Counters of punted packets — a high `punt`/`receive` count explains high CPU |
| `show processes cpu sorted` | If `IP Input` is a top process, significant traffic is being **process-switched** — something is defeating CEF |
| `show buffers` | Buffer pool health (misses/failures — see §F.3) |
| `show arp` | The ARP cache feeding the adjacency table |

## F.7 Annotated example — `show ip cef` and the adjacency

```
R1# show ip cef 192.168.20.5 detail
192.168.20.0/24, epoch 2
  nexthop 10.1.1.2 GigabitEthernet0/1

R1# show adjacency g0/1 detail
Protocol Interface                 Address
IP       GigabitEthernet0/1        10.1.1.2(11)
                                   0C1A2B3C4D5E0C1A2B3C4D000800
                                   ARP        03:12:44
```

**How to read it:**
- The FIB entry answers step 4 of §F.4 in one line: packets to `192.168.20.5` matched `/24`, exit `g0/1`, next-hop `10.1.1.2`. If this disagrees with `show ip route`, the FIB is stale — `clear ip route *` or investigate.
- The adjacency hex is the **pre-built frame header**: destination MAC `0c1a.2b3c.4d5e` (next-hop), source MAC `0c1a.2b3c.4d00` (this router's `g0/1`), EtherType `0800` (IPv4). Forwarding a packet = FIB lookup + prepend these 14 bytes + fix TTL/checksum. That's the whole trick.
- `ARP 03:12:44` — the rewrite was learned from ARP and ages with it. A missing adjacency (`glean`) here + working route = ARP problem, not routing problem.
