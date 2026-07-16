# First-Hop Redundancy 

## FHRP.1 FHRP overview — the protocol family

All First-Hop Redundancy Protocols solve the same problem: hosts get exactly **one** default gateway, so that IP must survive a router failure. The family:

| Protocol | Standard? | Terminology | Load balancing | Virtual MAC |
|---|---|---|---|---|
| **HSRP** | Cisco proprietary | Active / Standby | Per-VLAN only (different groups per VLAN) | `0000.0c07.acXX` (v1, XX = group) / `0000.0c9f.fXXX` (v2) |
| **VRRP** | Open standard (RFC 5798) | Master / Backup | Per-VLAN only | `0000.5e00.01XX` (XX = group) |
| **GLBP** | Cisco proprietary | AVG / AVF | **Per-host**, within one group (the AVG hands out different virtual MACs round-robin) | `0007.b400.XXYY` |
| **CARP** | Open (BSD world) | Master / Backup | Per-VLAN (or ARP-balancing tricks) | `00:00:5e:00:01:XX` (borrows the VRRP range) |

Concepts transfer 1:1 between them: a shared **virtual IP + virtual MAC**, hello messages, a priority election, and optional preemption. Learn HSRP and you know 90% of the others.

## FHRP.2 Hot Standby Router Protocol (HSRP)

Two routers share a virtual gateway IP so hosts survive a router failure.

``` linenums="1"
interface g0/0
 ip address 192.168.10.2 255.255.255.0
 standby version 2
 standby 1 ip 192.168.10.1
 standby 1 priority 150
 standby 1 preempt
```

- `standby version 2` — HSRPv2 (supports more groups and IPv6)
- `standby 1 ip 192.168.10.1` — the virtual gateway IP that hosts point at
- `standby 1 priority 150` — higher priority becomes Active (default 100)
- `standby 1 preempt` — reclaim the Active role when this router comes back up

Verify with `show standby [brief]` — states: Active / Standby (see §18.12).

---

## FHRP.3 VRRP

### Theory — differences from HSRP that matter

- **Open standard** — the reason to choose it: mixed-vendor gateway pairs (Cisco + anything).
- **Master/Backup** instead of Active/Standby; multicast `224.0.0.18` (IP protocol 112) instead of HSRP's UDP/1985.
- **Preemption is ON by default** (HSRP: off). The higher-priority router always takes over — disable with `no preempt` if you don't want flapping hardware yanking the role back.
- **The virtual IP can be a real interface address**: if the VIP equals the Master's own interface IP, that router is the "owner", runs priority 255, and always wins. HSRP forbids this — the VIP must be a spare address. (Using a spare address works in VRRP too and is the common design.)
- **Timers are faster by default**: advertisements every **1 s** (HSRP hellos: 3 s), master-down interval ≈ 3× — so VRRP fails over quicker out of the box.
- Priority range 1–254 (default 100), highest wins; tie-break = highest interface IP.

#### Configuration (mirrors the HSRP recipe)

**R1 — intended Master:**

```text linenums="1"
interface g0/0
 ip address 192.168.10.2 255.255.255.0
 vrrp 1 ip 192.168.10.1
 vrrp 1 priority 150
 vrrp 1 preempt delay minimum 60
 vrrp 1 authentication md5 key-string VrrpKey
```

- `vrrp 1 ip 192.168.10.1` — the virtual gateway IP for group 1 (a spare address on the subnet, same on both routers)
- `vrrp 1 priority 150` — higher becomes Master (default 100; the peer stays at default)
- `vrrp 1 preempt delay minimum 60` — preemption is already on by default; the `delay minimum 60` makes a recovering router wait 60 s (until its routing protocol converges) before grabbing Master — takes over the gateway with an empty routing table otherwise
- `vrrp 1 authentication md5 key-string VrrpKey` — reject rogue VRRP speakers on the LAN (plaintext auth also exists; MD5 where supported)

**R2** is identical minus the priority line (its own IP `.3`, default priority 100).

Object tracking — fail over when the *uplink* dies, not just the router:

```text linenums="1"
track 10 interface g0/1 line-protocol
interface g0/0
 vrrp 1 track 10 decrement 60
```

- `track 10 interface g0/1 line-protocol` — a tracking object watching the WAN interface
- `vrrp 1 track 10 decrement 60` — if it goes down, drop this router's priority by 60 (150 → 90), so the peer (100) preempts and becomes Master. Size the decrement so it actually crosses the peer's priority

(On newer IOS-XE the syntax is VRRPv3: `fhrp version vrrp v3`, then `vrrp 1 address-family ipv4` — same concepts, adds IPv6 support and millisecond timers.)

### Verification

| Command | Explanation |
|---|---|
| `show vrrp brief` | One line per group: priority, `P` (preempt), state, master address, VIP |
| `show vrrp` | Full detail: timers, tracking objects and their current effect on priority |
| `show vrrp interface g0/0` | Groups on one interface |
| `debug vrrp state` | Watch Master/Backup transitions live (lab use) |

Annotated example:

```text linenums="1"
R1# show vrrp brief
Interface   Grp Pri Time  Own Pre State   Master addr     Group addr
Gi0/0       1   150 3414        Y   Master  192.168.10.2    192.168.10.1
```

- `State Master` + `Master addr` = its **own** interface IP → this router currently owns the VIP. On the Backup, `Master addr` shows the *peer's* address — a quick way to confirm both agree on who's Master.
- `Pre Y` — preemption on (the VRRP default; seeing `N` here means someone disabled it).
- `Own` blank — the VIP is a spare address, not the router's interface address (an owner would show priority 255 here).
- **Both routers claiming `Master`** = split brain: they can't hear each other's advertisements (multicast `224.0.0.18` blocked, VLAN broken between them) — same diagnosis as dual-Active HSRP.

## FHRP.4 CARP (Common Address Redundancy Protocol)

Theory only — CARP is **not available on Cisco IOS**. It lives in the BSD world (OpenBSD, FreeBSD, and every pfSense/OPNsense firewall pair), and it's included here because those firewalls sit as the default gateway in plenty of real networks, doing exactly the job HSRP/VRRP do on routers.

- **What it is:** OpenBSD's patent-free answer to VRRP (born from the HSRP/VRRP patent disputes), deliberately incompatible but conceptually identical: virtual IP + virtual MAC (it squats on the VRRP MAC range `00:00:5e:00:01:XX`), Master/Backup election, multicast advertisements over **IP protocol 112** — the same protocol number as VRRP, which is why the two **conflict if mixed on one VLAN with the same group/VHID number**. Rule: never reuse a VHID that a VRRP group already uses on that segment.
- **Election is inverted vs Cisco:** CARP's knobs are `advbase`/`advskew` (advertisement interval and skew) and the **lowest** effective value wins — i.e., the router advertising *most frequently* becomes Master, rather than a highest-priority number.
- **Security by default:** advertisements are authenticated (SHA-1 HMAC over the VHID + password) — what HSRP/VRRP achieve only with explicit `authentication` config.
- **The pfSense/OPNsense pattern** (the practical reason to know CARP): a firewall pair runs one CARP VIP per interface (LAN VIP = hosts' gateway, WAN VIP = the address NAT uses), **pfsync** replicates the state table so established connections survive failover, and XML-RPC sync copies rules from primary to secondary. Preemption ("swallow" the master role back) is a checkbox; demotion on interface failure is automatic.
- **Interop rule of thumb:** CARP peers only with CARP. A Cisco router and a pfSense box cannot share one virtual gateway group — pick one family per subnet. They coexist fine on the same VLAN as *separate* gateways for separate subnets/VIPs, VHID collisions aside.

| Concept | HSRP | VRRP | CARP |
|---|---|---|---|
| Winner | Highest priority, Active | Highest priority, Master | **Lowest** advskew (most frequent advertiser), Master |
| Preemption default | Off | **On** | Off (opt-in) |
| Auth | Optional MD5 | Optional MD5/plaintext | **Always** (SHA-1 HMAC) |
| Group ID name | Group | Group (VRID) | **VHID** |
| State sync of connections | — | — | **pfsync** (firewall state survives failover) |
