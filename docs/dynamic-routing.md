# OSPF (Open Shortest Path First)

## OSPF theory — link-state fundamentals

Distance-vector protocols (RIP) learn *rumors*: "network X is 3 hops that way, trust me." Link-state protocols work like navigation apps: every router obtains **the full map** and computes its own routes:

1. **Meet the neighbors** — Hello packets discover adjacent routers and keep the relationship alive.
2. **Flood the topology** — each router describes its links in **LSAs** (Link-State Advertisements); LSAs are flooded reliably until **every router in the area holds an identical LSDB** (Link-State Database). This is why `show ip ospf database` looks the same on every router in area 0.
3. **Compute independently** — each router runs **Dijkstra's SPF algorithm** on the LSDB, building a shortest-path tree with *itself* as the root; the best (lowest total cost) path per destination goes to the routing table.

Consequences worth internalizing: OSPF routers don't exchange *routes*, they exchange *topology* — so a lying/misconfigured router corrupts the shared map (the argument for OSPF authentication); and any link change forces a partial or full SPF re-run on every router in the area — the argument for **areas** (contain the flooding/recalculation) on large networks.

**Router ID selection** (evaluated once, at process start): explicit `router-id` command → else highest **loopback** IP → else highest active physical interface IP. Always set it manually — and remember a new RID takes effect only after `clear ip ospf process`.

### The five OSPF packet types

All ride directly on IP **protocol 89** (no TCP/UDP) — reliability is built in via acknowledgments:

| # | Packet | Role |
|---|---|---|
| 1 | **Hello** | Discover/keepalive neighbors; carries the parameters that must match (see below). Multicast to `224.0.0.5` (all OSPF routers) |
| 2 | **DBD** (Database Description) | A *table of contents* of the sender's LSDB — headers only — exchanged when synchronizing |
| 3 | **LSR** (Link-State Request) | "Send me the full copy of these LSAs I'm missing/have stale" |
| 4 | **LSU** (Link-State Update) | The actual LSAs — both the answer to an LSR and the flooding vehicle for changes |
| 5 | **LSAck** | Acknowledges LSUs — makes flooding reliable |

(`224.0.0.6` is the *DR/BDR-only* group — DROthers send their LSUs there; the DR re-floods to `224.0.0.5`.)

### The neighbor state machine — what each state means

The states from `show ip ospf neighbor` in order, with what's happening (the troubleshooting map in §18.7 tells you which *stuck* state implies which fault — this is the theory behind it):

| State | Meaning |
|---|---|
| **Down** | No Hellos heard from this neighbor |
| **Init** | I hear your Hellos, but they don't list *my* RID yet (one-way) |
| **2-Way** | Both Hellos list each other — bidirectional confirmed. **DR/BDR election happens here.** Non-DR pairs on a LAN *stay* here by design |
| **ExStart** | Master/slave negotiation (higher RID = master) for the DBD exchange; MTU mismatch traps adjacencies here |
| **Exchange** | DBDs (LSDB summaries) flow |
| **Loading** | LSRs sent for missing entries; LSUs arriving |
| **Full** | LSDBs identical — true adjacency. Routes can now be computed through this neighbor |

**Hello requirements — the mismatch checklist.** Two routers become neighbors only if these agree: same **subnet** and mask, same **area** ID, same **Hello/Dead timers**, same **authentication**, same area type (stub flags), unique **RIDs**, and matching **MTU** (checked at ExStart, not in the Hello). Timers by network type: Ethernet/broadcast **10/40 s**, point-to-point 10/40, NBMA 30/120 — and the dead interval is simply 4× hello unless set.

### DR/BDR — why and how

On a multi-access segment (Ethernet), *n* routers forming full adjacencies with each other would mean **n(n−1)/2** adjacency pairs all flooding to all — a mess. Instead the segment elects a **Designated Router** (and a **Backup DR** that maintains full adjacencies but stays quiet): everyone forms Full only with the DR/BDR, DROthers stay at 2-Way with each other, and all flooding goes *through* the DR. Adjacencies drop from n(n−1)/2 to 2(n−2)+1.

**Election rules (the exam favorites):**

- Highest **interface priority** wins (default 1; `ip ospf priority 0` = refuses the role; range 0–255)
- Tie → highest **Router ID**
- **No preemption:** a "better" router joining later does *not* take over — the DR keeps the job until it dies (then the BDR is promoted and a new BDR is elected). To force a re-election after changing priorities: `clear ip ospf process` on the segment's routers, or bounce the interfaces
- Point-to-point links skip the election entirely — no DR/BDR (`FULL/ -` in the neighbor table), which is why `ip ospf network point-to-point` on router-to-router Ethernet links is a common optimization (faster adjacency, one less thing to elect)

### Cost calculation — worked end-to-end example

The formula and reference-bandwidth mechanics are in the tuning subsection above; here is how the *route metric* is actually assembled. Cost is counted on **outgoing interfaces only, along the direction of travel**:

```
R1 ----GigE---- R2 ----FastE---- R3 ----GigE---- 192.168.20.0/24
     cost 1          cost 10           cost 1     (ref-bw 1000 Mb/s everywhere)

R1's metric to 192.168.20.0/24:
  R1's outgoing GigE (1) + R2's outgoing FastE (10) + R3's outgoing GigE (1) = 12
  → show ip route on R1:  O 192.168.20.0/24 [110/12] via ...
```

- The *inbound* interfaces contribute nothing; costs are asymmetric if link speeds differ per direction of the path — R3's metric back toward R1's LAN can legitimately differ.
- Equal total cost to the same prefix → **ECMP**: OSPF installs up to 4 equal-cost routes by default (`maximum-paths` raises it) and CEF load-shares per-destination across them. To *prefer* one path, break the tie with `ip ospf cost` on an interface.
- Practical debugging order: `show ip route ospf` (the total) → `show ip ospf interface` at each hop along the path (the per-hop terms) — the sum must reconcile, and the hop where it doesn't is where someone tuned `bandwidth`/`cost`/reference inconsistently.

### Areas, LSA types, and router roles (multi-area concepts)

CCNA configures single-area OSPF but expects the multi-area vocabulary:

- **Why areas:** bound the LSDB size and the SPF blast radius — a flapping link in area 1 doesn't force SPF runs in area 2. All areas must touch **area 0** (the backbone); inter-area traffic transits it.
- **Router roles:**

| Role | Definition |
|---|---|
| **Internal router** | All interfaces in one area |
| **Backbone router** | At least one interface in area 0 |
| **ABR** (Area Border Router) | Interfaces in ≥2 areas — holds an LSDB *per area*, summarizes between them |
| **ASBR** (Autonomous System Boundary Router) | Injects external routes into OSPF (redistribution; `default-information originate` makes you one) |

- **LSA types to recognize** (deep detail is CCNP, the mapping is CCNA-adjacent): **Type 1** Router-LSA (every router describes its own links, flooded within the area), **Type 2** Network-LSA (generated by the DR for its segment), **Type 3** Summary-LSA (ABR advertises one area's prefixes into another — shows as `O IA` in the routing table), **Type 5** External-LSA (ASBR's redistributed routes — `O E2` by default: metric does *not* accumulate internal cost; `O E1` adds it).
- Route preference when several OSPF flavors offer the same prefix: `O` (intra-area) > `O IA` > `O E1` > `O E2` — intra-area wins even at a higher metric.

### OSPF — data-structure & state details 
**The three OSPF data structures → the three tables:**

| Structure | Builds the… | View with |
|---|---|---|
| **Adjacency database** | **Neighbor table** (routers with bidirectional comms) | `show ip ospf neighbor` |
| **Link-state database (LSDB)** | **Topology table** — identical on all routers in an area | `show ip ospf database` |
| **Forwarding database** | **Routing table** — unique per router | `show ip route ospf` |

!!!note
    **Single-area OSPF fact:** all routers are in the **backbone (area 0)**; they share an **identical LSDB**, but their **neighbor and routing tables are unique** to each router. 

!!!warning
    A common trap is "all routers have the same routing table" — false.

**The generic link-state process, in order:**
1. Select the **Router ID**
2. Discover neighbors (Hello) → build the **adjacency/neighbor table**
3. Build the **LSDB** (topology table) from received **LSAs**
4. **Run SPF (Dijkstra)** → build the SPF tree → install best routes

**Neighbor state transitions:**
- **Down → Init:** happens when an **OSPF-enabled interface becomes active** and the router starts sending Hellos.
- **Init → Two-Way:** a router sees its **own RID** in a neighbor's Hello (bidirectional confirmed).
- **DR/BDR election occurs in the Two-Way state** (before ExStart).
- A **DROTHER reaches FULL only with the DR and BDR**; two DROTHERs stay at **Two-Way** with each other.

**Advertising a default route into OSPF:** on the **ASBR/edge router** (the one facing the ISP), configure a default static route **and** `default-information originate`. It must be entered on the **edge router itself**, not the ISP.

**Wildcard-mask shortcut for `network` statements:** subtract the subnet mask from `255.255.255.255`.

| Subnet mask | Wildcard |
|---|---|
| 255.255.255.0 (/24) | 0.0.0.255 |
| 255.255.254.0 (/23) | 0.0.1.255 |
| 255.255.252.0 (/22) | 0.0.3.255 |
| 255.255.255.192 (/26) | 0.0.0.63 |
| 255.255.255.252 (/30) | 0.0.0.3 |
| 0.0.0.0 (exact interface match) | 0.0.0.0 |

## OSPFv2 (IPv4) basic setup

``` linenums="1"
router ospf 1
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0
 network 10.0.0.1 0.0.0.0 area 0
 passive-interface default
 no passive-interface s0/0/0
 default-information originate
 auto-cost reference-bandwidth 10000
```

- `router ospf 1` — start OSPF; `1` is the local process ID (doesn't need to match neighbors)
- `router-id 1.1.1.1` — manually set the router ID (best practice; otherwise the highest loopback/interface IP is used)
- `network 192.168.10.0 0.0.0.255 area 0` — enable OSPF on interfaces in this range; note the **wildcard** mask (inverse of the subnet mask)
- `network 10.0.0.1 0.0.0.0 area 0` — a /32 wildcard matches exactly one interface
- `passive-interface default` — advertise subnets but send hellos nowhere... (`passive-interface g0/0` does the same for a single LAN-facing port)
- `no passive-interface s0/0/0` — ...then re-enable hellos on actual neighbor links
- `default-information originate` — advertise your default route to the rest of the OSPF domain
- `auto-cost reference-bandwidth 10000` — reference bandwidth in Mb/s so fast links get distinct costs (details next section) — set the **same** value on all routers

### Passive Interface
By default, OSPF messages are forwarded out all OSPF-enabled interfaces. 
However, these messages really only need to be sent out interfaces that are connecting to other OSPF-enabled routers. 
Sending out unneeded messages on a LAN affects the network in three ways, as follows:
- Inefficient Use of Bandwidth: Available bandwidth is consumed transporting unnecessary messages.
- Inefficient Use of Resources: All devices on the LAN must process and eventually discard the message.
- Increased Security Risk: Without additional OSPF security configurations, OSPF messages can be intercepted with packet sniffing software. 
Routing updates can be modified and sent back to the router, corrupting the routing table with false metrics that misdirect traffic.
Use the `passive-interface router` configuration mode command to prevent the transmission of routing messages through a router interface, but still allow that network to be advertised to other routers. 

!!! note

    In production networks, loopback interfaces do not require to be passive.

The network of this interface is still included as a route entry in OSPFv2 updates that are sent to neighbors.

As an alternative, all interfaces can be made passive using the `passive-interface default` command. 
Interfaces that should not be passive can be re-enabled using the `no passive-interface` command.

## OSPF cost & reference bandwidth (metric tuning)

OSPF picks paths by **lowest total cost**, and each interface's cost is calculated as:

```
cost = reference bandwidth ÷ interface bandwidth
```

The **default reference bandwidth is 100 Mb/s** — a value from the 1990s. Any result below 1 is rounded **up to 1**, and cost is an integer, which creates the classic problem:

| Link speed | Cost (default ref = 100 Mb/s) | Cost (ref = 10000 Mb/s) | Cost (ref = 100000 Mb/s) |
|---|---|---|---|
| 10 Mb/s Ethernet | 10 | 1000 | 10000 |
| 100 Mb/s FastEthernet | 1 | 100 | 1000 |
| 1 Gb/s | **1** | 10 | 100 |
| 10 Gb/s | **1** | **1** | 10 |
| 100 Gb/s | **1** | **1** | 1 |

With the default, **FastEthernet, 1G, 10G, and 100G all cost 1** — OSPF literally cannot tell them apart and may load-share or prefer a slow path. Fix it by raising the reference under `router ospf 1`:

| Command | Explanation |
|---|---|
| `auto-cost reference-bandwidth 10000` | Value is in Mb/s: `10000` = 10 Gbps reference. Pick a reference ≥ your fastest link (use `100000` if you have 100G) |

IOS prints a warning when you enter this: *"Please ensure reference bandwidth is consistent across all routers"* — heed it. **Set the identical value on EVERY router in the OSPF domain**, or costs become inconsistent and routers disagree about the best path (asymmetric/suboptimal routing).

Three ways to influence an interface's cost, from bluntest to most surgical:

| Command | Mode | Explanation |
|---|---|---|
| `ip ospf cost 50` | Interface | Hard-set the cost directly — overrides the formula entirely. Most explicit and predictable; preferred for traffic engineering |
| `bandwidth 500000` | Interface | Change the bandwidth **value** (in Kb/s!) the formula uses. Does **not** change real link speed — and beware: EIGRP metrics and QoS percentages read this value too (side effects) |
| `auto-cost reference-bandwidth 10000` | Router | Change the reference — affects **all** interfaces at once. The domain-wide baseline; combine with `ip ospf cost` for exceptions |

The same commands exist for OSPFv3: `ipv6 ospf cost 50` on the interface and `auto-cost reference-bandwidth 10000` under `ipv6 router ospf 1`.

Verification:

| Command | Explanation |
|---|---|
| `show ip ospf interface g0/1` | Look for `Cost: 10` — the value actually in use |
| `show ip ospf` | Shows the configured reference bandwidth for the process |
| `show ip route ospf` | The metric in `[110/metric]` is the **sum** of costs along the path |

**Reading the path cost:** a route shown as `O 192.168.20.0/24 [110/30]` means the costs of every *outgoing* interface along the path (calculated at each hop toward the destination) add up to 30. To trace where the number comes from, check the `show ip ospf interface` cost at each hop.

## Interface-level OSPF (modern alternative to `network`)

| Command | Explanation |
|---|---|
| `ip ospf 1 area 0` | Enable OSPF directly on the interface |
| `ip ospf cost 15` | Manually set cost (lower preferred); cost = reference-bw ÷ interface-bw |
| `ip ospf priority 100` | DR election priority (`0` = never DR; higher wins; default 1) |
| `ip ospf hello-interval 5` | Hello timer (must match the neighbor; default 10 s on Ethernet) |
| `ip ospf dead-interval 20` | Dead timer (must match; default 4× hello) |
| `ip ospf network point-to-point` | Skip DR/BDR election on point-to-point Ethernet links |

## OSPF authentication (hardening)

``` linenums="1"
interface g0/1
 ip ospf message-digest-key 1 md5 SecretKey
router ospf 1
 area 0 authentication message-digest
```

- `ip ospf message-digest-key 1 md5 SecretKey` — configure the MD5 key on the link (key ID must match the neighbor)
- `area 0 authentication message-digest` — require MD5 authentication for the whole area

## OSPFv3 (IPv6)

``` linenums="1"
ipv6 unicast-routing
ipv6 router ospf 1
 router-id 1.1.1.1
interface g0/0
 ipv6 ospf 1 area 0
```

- `ipv6 router ospf 1` — start the OSPFv3 process
- `router-id 1.1.1.1` — still a 32-bit dotted value, even for IPv6
- `ipv6 ospf 1 area 0` — OSPFv3 is always enabled per-interface (there is no `network` command)

## Verification

| Command | Explanation |
|---|---|
| `show ip ospf neighbor` | Adjacencies — look for FULL state: `FULL/DR`, `FULL/BDR`, `FULL/-` (see §18.7) |
| `show ip ospf interface [brief]` | Timers, cost, DR/BDR, network type per interface |
| `show ip ospf database` | The link-state database (LSAs) |
| `show ip route ospf` | Only OSPF-learned routes (marked `O`) |
| `show ip protocols` | Process ID, router ID, networks, passive interfaces |
| `clear ip ospf process` | Restart OSPF (needed for a new `router-id` to take effect) |
| `show ipv6 ospf neighbor` | OSPFv3 equivalent |

**Neighbors stuck / not forming?** Check: mismatched hello/dead timers, different areas, different subnets, `passive-interface`, mismatched authentication, or MTU mismatch (stuck in EXSTART).

---


