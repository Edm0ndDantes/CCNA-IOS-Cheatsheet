# Spanning Tree Protocol (STP)

STP prevents Layer-2 loops by electing a root bridge and blocking redundant paths.

## S.1 STP theory — why it exists

Ethernet frames have **no TTL**: a frame caught in a physical loop circulates *forever*. Redundant links between switches are deliberately built for failover — STP's job is to keep the redundancy while logically breaking the loop, by blocking just enough ports to leave exactly **one active path** between any two points, and unblocking them when the primary path dies.

Without STP, a loop produces three simultaneous disasters:

| Failure | Mechanism |
|---|---|
| **Broadcast storm** | Every broadcast/multicast/unknown-unicast frame is flooded, loops around, is flooded again — traffic multiplies until links and CPUs saturate (takes seconds to kill a network) |
| **MAC table instability** | The same source MAC arrives on different ports as its frames loop, so the table rewrites constantly ("MAC flapping") — the switch CPU churns and forwarding becomes erratic |
| **Duplicate frames** | A host receives multiple copies of the same unicast frame via different loop paths — breaks protocols that don't expect it |

### How the tree is built — BPDUs and the three elections

Switches exchange **BPDUs** (Bridge Protocol Data Units, sent to multicast `0180.c200.0000` every **hello time = 2 s**). Everything is decided by comparing values, **lowest always wins**:

**1. Elect one Root Bridge per VLAN.** The switch with the lowest **Bridge ID** wins. The BID is:

```
| Priority (4 bits)      | Extended System ID (12 bits) | MAC address (48 bits) |
| default 32768,         | = VLAN number                | the tie-breaker       |
| multiples of 4096      |                              |                       |
```

- Priority + VLAN is why `show spanning-tree` displays values like `32778` (32768 + VLAN 10) and why configured priorities must be multiples of 4096.
- With default priorities everywhere, the **oldest switch wins** (lowest MAC) — usually the *worst* choice. This is the argument for always setting `spanning-tree vlan X root primary` (priority 24576) on the intended core switch and `root secondary` (28672) on its backup.
- All ports on the root bridge are Designated/forwarding — the tree grows outward from it.

**2. Each non-root switch elects one Root Port** — its single best path *toward* the root, chosen by (tie-breakers in strict order):

1. Lowest cumulative **root path cost** (sum of port costs along the path)
2. Lowest **sender BID** (neighbor's bridge ID)
3. Lowest **sender port priority** (default 128, tunable with `spanning-tree port-priority`)
4. Lowest **sender port number**

**3. Each segment elects one Designated Port** — the port that forwards toward that segment, on the switch with the lowest path cost to root (same tie-breakers). Every port that ended up neither Root nor Designated becomes **Alternate (blocked)** — those are the loop-breakers.

**Path costs** (IEEE short values — what Catalyst switches use by default):

| Link speed | STP cost |
|---|---|
| 10 Mb/s | 100 |
| 100 Mb/s | 19 |
| 1 Gb/s | 4 |
| 10 Gb/s | 2 |

Costs only differentiate up to 10G in the short system (the long system, `spanning-tree pathcost method long`, fixes this for faster links). Manipulating `spanning-tree cost` on a port changes which path is preferred — the STP analogue of tuning OSPF cost.

### Port states and timers (classic 802.1D)

A port moving toward forwarding walks through states — the delays exist to make sure the whole tree has agreed before traffic flows (a premature forward = temporary loop):

| State | Duration | Learns MACs? | Forwards? | Purpose |
|---|---|---|---|---|
| **Blocking** | stable | No | No | Receives BPDUs only — the loop-prevention state |
| **Listening** | 15 s (forward delay) | No | No | Sends/receives BPDUs, participates in elections |
| **Learning** | 15 s (forward delay) | **Yes** | No | Pre-populates the MAC table so forwarding doesn't start with a flood |
| **Forwarding** | stable | Yes | Yes | Normal operation |
| Disabled | — | No | No | Administratively `shutdown` |

**The three timers** (set by the root bridge, propagated in BPDUs):

| Timer | Default | Role |
|---|---|---|
| Hello | 2 s | BPDU transmission interval |
| Max Age | 20 s | How long a stored BPDU stays valid — a blocked port that stops hearing BPDUs waits this long before acting |
| Forward Delay | 15 s | Length of each of Listening and Learning |

Worst-case classic convergence: **Max Age + 2× Forward Delay = 50 seconds** of outage. That number is the whole justification for RSTP — and for PortFast, which skips Listening/Learning on host ports (a PC boots and DHCPs immediately instead of staring at a dead port for 30 s).

### RSTP (802.1w) — what actually changed

Rapid PVST+ (`spanning-tree mode rapid-pvst`) keeps the same election logic and tie-breakers but converges in **milliseconds-to-seconds** instead of 30–50 s:

- **States collapse to three:** Discarding (absorbs old Blocking/Listening/Disabled), Learning, Forwarding.
- **New port roles:** the blocked ports get jobs. An **Alternate** port is a pre-computed backup *Root Port* (path to root via another switch); a **Backup** port is a backup *Designated Port* (second link to the same segment — rare, needs a hub). When the root port dies, the alternate goes forwarding **immediately** — no timers.
- **Proposal/Agreement handshake:** instead of waiting out timers, point-to-point neighbors negotiate explicitly ("I propose to forward" → "agreed, I've blocked my other ports") — this is what makes it *rapid*, and it only works on **link-type point-to-point** (full-duplex, auto-detected). Half-duplex links fall back to timer-based convergence.
- **Edge ports** = PortFast formalized: a port declared edge goes straight to forwarding and generates no topology changes; it loses edge status the instant a BPDU arrives.
- **Every switch originates BPDUs** every hello (in 802.1D only relayed from the root); a neighbor is declared dead after **3 missed hellos = 6 s**, not Max Age.
- Backward compatible: a port hearing legacy 802.1D BPDUs reverts to classic behavior on that link.

### PVST+ — per-VLAN trees

Cisco's PVST+/Rapid-PVST+ runs **one independent STP instance per VLAN** (the standards run one for all). Cost: CPU/BPDUs scale with VLAN count. Benefit: **per-VLAN load sharing** — make one switch root for VLANs 10/30 and another root for VLANs 20/40, and both uplinks carry traffic (each blocked for the *other* VLAN set) instead of one link idling. This is done purely with per-VLAN `root primary`/`priority` — the topology does the rest.

### The protection toolkit — what each guard is *for*

| Feature | Deploy on | Protects against |
|---|---|---|
| **PortFast** | Access/host ports | 30 s of dead port at boot (DHCP timeouts); also stops hosts from generating topology-change events |
| **BPDU Guard** | PortFast ports | A switch (or bridging host/rogue AP) plugged into a host port — err-disables the port on the first BPDU. PortFast without BPDU Guard is a standing invitation for a loop |
| **Root Guard** | Designated ports toward access/customer switches | A downstream switch with a better BID hijacking the root role — the port goes `root-inconsistent` (blocked) while superior BPDUs arrive, recovers alone when they stop |
| **Loop Guard** | Root/alternate ports on trunk links | A **unidirectional** link failure (BPDUs stop arriving but the link shows up) silently unblocking a port → loop. The port goes `loop-inconsistent` instead of forwarding |
| **BPDU Filter** | (rarely) | Suppresses BPDUs entirely — effectively disables STP on the port; dangerous, know it exists, avoid it |

Root Guard vs Loop Guard in one line: Root Guard fires when *superior BPDUs arrive* where they shouldn't; Loop Guard fires when *BPDUs stop arriving* where they should.


## S.2 Global configuration

| Command | Explanation |
|---|---|
| `spanning-tree mode rapid-pvst` | Use Rapid PVST+ (fast convergence; recommended). Others: `pvst`, `mst` |
| `spanning-tree vlan 10 root primary` | Make this switch the root bridge for VLAN 10 (sets priority to 24576 or lower) |
| `spanning-tree vlan 10 root secondary` | Backup root (priority 28672) |
| `spanning-tree vlan 10 priority 4096` | Or set the priority manually (multiple of 4096; lower wins) |
| `spanning-tree portfast default` | PortFast on all access ports at once |
| `spanning-tree portfast bpduguard default` | BPDU Guard on all PortFast ports |

## S.3 Per-interface features

| Command | Explanation |
|---|---|
| `spanning-tree portfast` | Skip listening/learning on an access port — host ports come up instantly |
| `spanning-tree bpduguard enable` | Err-disable the port if a BPDU arrives (a switch was plugged into a host port) |
| `spanning-tree guard root` | Root Guard: block a downstream switch from becoming root via this port |
| `spanning-tree cost 10` | Manipulate path cost (lower preferred) |
| `spanning-tree port-priority 64` | Tie-breaker between equal-cost ports (lower wins, multiple of 32) |

## S.4 Verification

| Command | Explanation |
|---|---|
| `show spanning-tree` | Root bridge, port roles/states for every VLAN (see §18.6) |
| `show spanning-tree vlan 10` | Same, for one VLAN |
| `show spanning-tree summary` | Modes and feature status (PortFast, BPDU Guard, etc.) |

**Reading the output:** the root bridge shows `This bridge is the root`. Port roles: Root (best path to root), Designated (forwarding), Alternate/Blocked.

---