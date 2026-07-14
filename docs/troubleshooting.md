# Reading Command Output — Annotated Examples

Realistic output from the most important verification commands, with explanations of what to look for.

## `show ip route`

``` linenums="1"
R1# show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       * - candidate default, U - per-user static route

Gateway of last resort is 203.0.113.1 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 203.0.113.1
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.1.0/30 is directly connected, GigabitEthernet0/1
L        10.1.1.1/32 is directly connected, GigabitEthernet0/1
O     192.168.20.0/24 [110/2] via 10.1.1.2, 00:12:45, GigabitEthernet0/1
S     192.168.30.0/24 [1/0] via 10.1.1.2
      192.168.10.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.10.0/24 is directly connected, GigabitEthernet0/0
L        192.168.10.1/32 is directly connected, GigabitEthernet0/0
```

**How to read it:**
- **The letter code** is the route's source: `C` = directly connected network, `L` = the router's own address on that link (always /32), `S` = static, `O` = OSPF-learned, `S*` = a static route that is also the **default** route.
- `Gateway of last resort is 203.0.113.1` — a default route exists; anything not matching a more specific route goes there. If this says *"is not set"*, unknown destinations are dropped.
- `[110/2]` = **[administrative distance / metric]**. AD 110 identifies OSPF; metric 2 is the OSPF cost of the path. When two sources offer the same prefix, lower AD wins; between routes from the same protocol, lower metric wins.
- `via 10.1.1.2` — the next-hop address; `00:12:45` — how long the route has been stable (a young timer on an "old" route hints at flapping); `GigabitEthernet0/1` — the exit interface.
- **Longest prefix wins:** a packet to 192.168.20.5 matches `192.168.20.0/24` (specific) over `0.0.0.0/0` (default), regardless of AD or metric.

## `show ip interface brief`

``` linenums="1"
R1# show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     192.168.10.1    YES manual up                    up
GigabitEthernet0/1     10.1.1.1        YES manual up                    up
GigabitEthernet0/2     unassigned      YES unset  administratively down down
Serial0/0/0            203.0.113.2     YES DHCP   up                    down
Vlan1                  unassigned      YES unset  down                  down
```

**How to read it — the last two columns tell the story:**

| Status | Protocol | Meaning |
|---|---|---|
| up | up | Working normally |
| administratively down | down | Someone typed `shutdown` — fix with `no shutdown` |
| up | down | Physical layer OK, data-link broken: clock/encapsulation mismatch, keepalives failing, far end misconfigured |
| down | down | No signal at all: cable unplugged/bad, far-end device off, wrong port |

- **Method** shows where the IP came from: `manual` (typed), `DHCP`, `unset` (none).

## `show interfaces g0/0` (key excerpts)

``` linenums="1"
GigabitEthernet0/0 is up, line protocol is up
  Hardware is iGbE, address is 0c1a.2b3c.4d00
  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec,
  Full-duplex, 1000Mb/s, media type is RJ45
  Last input 00:00:02, output 00:00:01
  5 minute input rate 2000 bits/sec, 3 packets/sec
     10233 packets input, 1523340 bytes
     0 input errors, 0 CRC, 0 frame, 0 overrun
     8811 packets output, 923412 bytes
     0 output errors, 0 collisions, 2 interface resets
     0 late collision, 0 deferred
```

**What to look for:**
- `Full-duplex, 1000Mb/s` — verify both ends agree. A **duplex mismatch** classically shows as *CRC/late collisions* on the half-duplex side and runts on the other.
- `input errors` / `CRC` climbing → bad cable, bad SFP, or duplex mismatch. Re-run the command twice; what matters is whether counters are *increasing*, not their absolute value (use `clear counters` first).
- `output drops` (in the queue line, not shown here) → congestion: traffic exceeds the interface's capacity.
- `BW 1000000 Kbit/sec` — the bandwidth value OSPF/EIGRP use for metrics; it can be set with the `bandwidth` command without changing real speed.

## `show vlan brief`

``` linenums="1"
S1# show vlan brief
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi0/3, Gi1/0, Gi1/1, Gi1/2
10   SALES                            active    Fa0/5, Fa0/6
20   ENGINEERING                      active    Fa0/7, Fa0/8
99   MANAGEMENT                       active    
1002 fddi-default                     act/unsup
```

**How to read it:**
- Only **access** ports are listed — trunks don't appear here (they carry all VLANs; see `show interfaces trunk`).
- A host in the wrong VLAN? This is the first place to look: find its port in the wrong row.
- A VLAN that exists but shows no ports (like 99 here) is fine for SVI-only/management VLANs.
- VLANs 1002–1005 are legacy FDDI/Token-Ring placeholders — ignore them; they can't be deleted.

## `show interfaces trunk`

``` linenums="1"
S1# show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Gi0/1       on           802.1q         trunking      99

Port        Vlans allowed on trunk
Gi0/1       10,20,99

Port        Vlans allowed and active in management domain
Gi0/1       10,20,99

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/1       10,20
```

**How to read it:**
- Mode `on` = statically configured trunk; `desirable`/`auto` mean DTP negotiated it (harden this).
- `Native vlan 99` — must match on both ends or you'll see CDP "native VLAN mismatch" log messages and inter-VLAN leakage.
- The three VLAN lists narrow down: *allowed* (your config) → *active* (VLAN actually exists) → *forwarding* (STP isn't blocking it). **A VLAN missing from a lower list than the one above it is your fault line:** allowed but not active = VLAN not created; active but not forwarding = STP blocking or pruned (here VLAN 99 is not forwarding — it has no member ports, which is normal).

## `show spanning-tree vlan 10`

``` linenums="1"
S2# show spanning-tree vlan 10
VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    24586
             Address     0c1a.aaaa.0001
             Cost        4
             Port        25 (GigabitEthernet0/1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32778  (priority 32768 sys-id-ext 10)
             Address     0c1a.bbbb.0002

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- ------------------------
Gi0/1               Root FWD 4         128.25   P2p
Gi0/2               Altn BLK 4         128.26   P2p
Fa0/5               Desg FWD 19        128.7    P2p Edge
```

**How to read it:**
- **Root ID vs Bridge ID:** Root ID is the elected root bridge; Bridge ID is *this* switch. If they showed the same MAC, you'd see `This bridge is the root` instead of a Cost/Port line. Here, this switch is NOT the root.
- Root priority `24586` = 24576 + VLAN 10 → someone configured `spanning-tree vlan 10 root primary`. This switch's `32778` = default 32768 + 10.
- **Port roles:** `Root`/`FWD` = best path toward the root; `Desg`/`FWD` = designated (forwarding) for its segment; `Altn`/`BLK` = the **blocked redundant link** — this is the loop prevention working. If a "missing bandwidth" mystery arises, this blocked port is usually why.
- `P2p Edge` on Fa0/5 = PortFast is active (host port, skips listening/learning).

## `show ip ospf neighbor`

``` linenums="1"
R1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           1   FULL/DR         00:00:35    10.1.1.2        GigabitEthernet0/1
3.3.3.3           1   FULL/BDR        00:00:38    10.1.1.3        GigabitEthernet0/1
4.4.4.4           0   2WAY/DROTHER    00:00:31    10.1.1.4        GigabitEthernet0/1
5.5.5.5           1   FULL/  -        00:00:33    10.2.2.2        Serial0/0/0
```

**How to read it:**
- `FULL` = healthy adjacency, databases synchronized. `FULL/ -` (dash) = point-to-point link, where no DR/BDR is elected — also healthy.
- `2WAY/DROTHER` is **normal, not broken**, on a multi-access segment: two non-DR routers stay at 2WAY with each other and only go FULL with the DR and BDR. (Pri `0` = that router refuses to be DR.)
- **Genuinely broken states:** stuck in `INIT` = your hellos reach them, theirs don't return (ACL? unidirectional link?); stuck in `EXSTART`/`EXCHANGE` = classic **MTU mismatch**; neighbor missing entirely = subnet/area/timer/authentication mismatch or `passive-interface`.
- **Dead Time** counts down from the dead interval and resets with each hello — it should hover, never reach zero.

## `show ip nat translations`

``` linenums="1"
R1# show ip nat translations
Pro  Inside global       Inside local        Outside local       Outside global
tcp  203.0.113.5:34122   192.168.10.11:34122 198.51.100.7:443    198.51.100.7:443
tcp  203.0.113.5:51920   192.168.10.42:51920 93.184.216.34:80    93.184.216.34:80
icmp 203.0.113.5:1       192.168.10.11:1     8.8.8.8:1           8.8.8.8:1
---  203.0.113.9         192.168.10.5        ---                 ---
```

**How to read it (each row left→right is "outside view → inside truth"):**
- **Inside local** = the host's real private IP; **inside global** = what the outside world sees. Rows sharing one inside-global IP with different ports (first three rows) = **PAT/overload** at work.
- The last row with no protocol/ports and no outside addresses = a **static NAT** entry — it exists permanently, even with no traffic.
- Empty table when you expect translations? Check: interfaces marked `ip nat inside`/`ip nat outside`, the ACL actually matches the source, and remember translations only appear when traffic flows.

## `show access-lists`

``` linenums="1"
R1# show access-lists
Extended IP access list BLOCK-WEB
    10 deny tcp 192.168.10.0 0.0.0.255 any eq www (152 matches)
    15 deny tcp 192.168.10.0 0.0.0.255 any eq 443 (89 matches)
    20 permit ip any any (10334 matches)
```

**How to read it:**
- `(152 matches)` — the counters are the whole point: they prove the line is actually catching traffic. A rule you expect to fire showing **no counter at all** = never matched — likely ordered below a broader line that grabs the traffic first, applied to the wrong interface/direction, or the traffic isn't what you think.
- Matches only count on the interface(s) where the ACL is applied, and the implicit deny at the end has **no counter** — add an explicit `deny ip any any log` line to see what's being silently dropped.

## `show port-security interface f0/5`

``` linenums="1"
S1# show port-security interface f0/5
Port Security              : Enabled
Port Status                : Secure-shutdown
Violation Mode             : Shutdown
Maximum MAC Addresses      : 2
Total MAC Addresses        : 2
Sticky MAC Addresses       : 2
Last Source Address:Vlan   : 000c.dead.beef:10
Security Violation Count   : 1
```

**How to read it:**
- `Secure-shutdown` = the port is **err-disabled** right now — a violation occurred. (Healthy states: `Secure-up`; `Secure-down` just means the port is down/idle.)
- `Last Source Address` = the MAC that triggered the violation — your evidence of *what* was plugged in.
- Recover: fix the physical issue, then `shutdown` / `no shutdown` on the interface (the violation count survives; clear with `clear port-security sticky`).

## `show etherchannel summary`

``` linenums="1"
S1# show etherchannel summary
Flags:  D - down        P - bundled in port-channel
        S - Layer2      U - in use      s - suspended
        I - stand-alone

Group  Port-channel  Protocol    Ports
------+-------------+-----------+----------------------------
1      Po1(SU)         LACP      Gi0/1(P)  Gi0/2(P)
2      Po2(SD)         LACP      Gi0/3(I)  Gi0/4(s)
```

**How to read it:**
- `Po1(SU)` = Layer-2 channel, **U**p and in use, with both members `(P)` bundled — healthy.
- `Po2(SD)` is down: `Gi0/3(I)` = operating stand-alone (LACP got no response — is the far side configured?), `Gi0/4(s)` = **suspended**, almost always a **member mismatch** (VLAN list, speed, duplex, or mode differs from the channel). Fix the member config or the far end.

## `show standby brief` (HSRP)

``` linenums="1"
R1# show standby brief
                     P indicates configured to preempt.
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Gi0/0       1    150 P Active  local           192.168.10.3    192.168.10.1
```

**How to read it:**
- State `Active` + Active = `local` → this router currently owns the virtual IP 192.168.10.1. The peer at `.3` is `Standby`, ready to take over.
- `P` = preempt is on — after recovering from a failure, this higher-priority (150) router will reclaim the Active role. Without preempt it would wait as Standby forever.
- Both routers showing `Active` = **split brain**: they can't hear each other's hellos — check the link/VLAN between them.

---

## Quick "is it working?" checklist

| Question | Command |
|---|---|
| Are my interfaces up? | `show ip interface brief` |
| Can I reach it? | `ping` / `traceroute` |
| What path is used? | `show ip route <dest>` |
| L2: right VLAN? | `show vlan brief`, `show interfaces trunk` |
| STP blocking something? | `show spanning-tree` |
| OSPF neighbors up? | `show ip ospf neighbor` |
| DHCP leasing? | `show ip dhcp binding` |
| NAT translating? | `show ip nat translations` |
| ACL dropping traffic? | `show access-lists` (watch counters) |
| Who's plugged in where? | `show mac address-table`, `show cdp neighbors` |
