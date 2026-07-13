# OSPF (Open Shortest Path First)

## OSPFv2 (IPv4) basic setup

```
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

```
interface g0/1
 ip ospf message-digest-key 1 md5 SecretKey
router ospf 1
 area 0 authentication message-digest
```

- `ip ospf message-digest-key 1 md5 SecretKey` — configure the MD5 key on the link (key ID must match the neighbor)
- `area 0 authentication message-digest` — require MD5 authentication for the whole area

## OSPFv3 (IPv6)

```
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