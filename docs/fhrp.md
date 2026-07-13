

## 15. First-Hop Redundancy — HSRP

Two routers share a virtual gateway IP so hosts survive a router failure.

```
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