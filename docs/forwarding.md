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
- **This is also the attack surface:** MAC flooding attacks overflow the CAM table so *everything* floods (making the switch a hub for eavesdropping) — the reason port security (§4) limits MACs per port.

**Frame handling modes** (how much of the frame is received before forwarding):

| Mode | Behavior |
|---|---|
| **Store-and-forward** | Receive the whole frame, verify the FCS, then forward — errors are dropped here. The modern default |
| **Cut-through** | Forward as soon as the destination MAC (first 6 bytes after preamble) is read — lowest latency, but propagates corrupt frames |
| Fragment-free | Cut-through after 64 bytes (past the collision window) — legacy compromise |

Switching happens in hardware (**ASICs** consulting **CAM/TCAM** memory) at line rate; the CPU sees only control-plane traffic (STP BPDUs, CDP, management).

## F.3 Router: the routing decision

For every packet, conceptually:

1. **Sanity:** verify the IP header checksum; **decrement TTL** — if it hits 0, drop and send ICMP *Time Exceeded* (this is what traceroute exploits).
2. **Lookup:** find the **longest-prefix match** for the destination in the forwarding table. `/32` host route beats `/24` beats `/16` beats `0.0.0.0/0` — *specificity always wins*; AD and metric only break ties **between candidates for the same prefix** (AD picks the source, metric picks the path — that contest happens when the routing table is *built*, not per packet).
3. **No match at all** (and no default route): drop, send ICMP *Destination Unreachable*.
4. **Resolve the exit:** the route gives a next-hop IP and/or exit interface; recursive lookups resolve a next-hop to its own exit interface if needed.
5. **Rewrite the frame:** new L2 header — `src MAC` = router's exit interface, `dst MAC` = the next-hop's MAC (learned via **ARP** for IPv4 / **NDP** for IPv6). The IP packet inside is untouched except TTL and checksum.
6. If the next-hop MAC is unknown: hold/drop the packet, ARP for it, forward when resolved — the reason the *first* ping across a fresh path often times out while the rest succeed.

## F.4 The three switching paths (how the lookup is implemented)

"Routing" (building the table from static/OSPF/BGP — control plane) and "switching" a packet (moving it — data plane) are separate jobs. IOS has had three generations of the data-plane mechanism:

| Method | How it works | Cost |
|---|---|---|
| **Process switching** | Every packet is punted to the CPU; the full routing table is consulted per packet | Brutally slow — today only for packets that *require* the CPU (options, TTL expiry, destined-to-router) |
| **Fast switching** | First packet of a flow is process-switched; the result is cached ("route once, switch many"); subsequent packets hit the cache | Better, but the first packet is slow and cache invalidation is messy. Legacy |
| **CEF** (Cisco Express Forwarding) | Tables are **pre-built before any traffic arrives**: the **FIB** (routing table distilled to prefix → next-hop, optimized for longest-match lookup) + the **adjacency table** (next-hop → ready-made L2 rewrite string from ARP). Forwarding = one FIB lookup + paste the rewrite | Line rate, deterministic. The default everywhere; MPLS and most features require it |

- CEF's two tables mirror the two things step 5 above needs: *where* (FIB, fed by the routing table) and *how to dress the frame* (adjacency table, fed by ARP). When ARP hasn't resolved yet, the FIB entry points to a **glean** adjacency = "punt to CPU to ARP first".
- On multilayer switches the same FIB/adjacency logic is burned into **TCAM/ASIC** hardware — "routing at switching speed" is literally CEF in silicon.
- **Load sharing:** with equal routes, CEF balances **per-destination** (technically per src-dst hash) by default — per-packet balancing exists but reorders TCP and is avoided.

## F.5 Commands & verification

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
| `show arp` | The ARP cache feeding the adjacency table |

## F.6 Annotated example — `show ip cef` and the adjacency

```text linenums="1"
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
- The FIB entry answers step 4 of §F.3 in one line: packets to `192.168.20.5` matched `/24`, exit `g0/1`, next-hop `10.1.1.2`. If this disagrees with `show ip route`, the FIB is stale — `clear ip route *` or investigate.
- The adjacency hex is the **pre-built frame header**: destination MAC `0c1a.2b3c.4d5e` (next-hop), source MAC `0c1a.2b3c.4d00` (this router's `g0/1`), EtherType `0800` (IPv4). Forwarding a packet = FIB lookup + prepend these 14 bytes + fix TTL/checksum. That's the whole trick.
- `ARP 03:12:44` — the rewrite was learned from ARP and ages with it. A missing adjacency (`glean`) here + working route = ARP problem, not routing problem.
```

Notes: standalone section for the split-file project (headings as `F.x`, renumber to fit). It covers the CCNA theory set — MAC learning/forward/filter/flood, aging, store-and-forward vs cut-through, CAM/TCAM and the port-security tie-in, the per-hop routing algorithm with the "L2 changes per hop, L3 survives" rule, longest-prefix-match vs AD vs metric (and *when* each applies), TTL/ICMP behavior, ARP's role in the first-packet delay — then the process/fast/CEF evolution with the FIB + adjacency model, which is also the practical bridge: `show ip cef` vs `show ip route` disagreements and `IP Input` CPU load are real troubleshooting moves, and the annotated adjacency output shows the actual pre-built frame rewrite so the whole story clicks together.