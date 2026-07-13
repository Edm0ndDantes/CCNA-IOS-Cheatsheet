# IPv4 Static Routing

## Route variants

| Command | Explanation |
|---|---|
| `ip route 10.2.0.0 255.255.255.0 192.168.1.2` | Route to 10.2.0.0/24 via next-hop 192.168.1.2 |
| `ip route 10.2.0.0 255.255.255.0 g0/1` | Exit-interface form (fine on point-to-point links) |
| `ip route 10.2.0.0 255.255.255.0 g0/1 192.168.1.2` | Fully specified (both) — most robust |
| `ip route 0.0.0.0 0.0.0.0 203.0.113.1` | Default route ("gateway of last resort") — matches everything |
| `ip route 10.2.0.0 255.255.255.0 172.16.0.2 90` | Floating static: AD 90 makes it a backup to a routing protocol |
| `ip route 10.2.0.0 255.255.255.128 192.168.1.2` | Static route to a specific subnet (e.g., a /25) |

## Administrative Distance (AD) — trustworthiness of a route source (lower wins)

| Source | AD |
|---|---|
| Connected | 0 |
| Static | 1 |
| eBGP | 20 |
| EIGRP | 90 |
| OSPF | 110 |
| RIP | 120 |

## Verification

| Command | Explanation |
|---|---|
| `show ip route` | The routing table: `C`=connected, `S`=static, `O`=OSPF, `S*`=default static (see §18.1) |
| `show ip route 10.2.0.5` | Which route would be used to reach this exact address |
| `show ip route static` | Only static routes |
| `ping 10.2.0.1` | Basic reachability test |
| `ping 10.2.0.1 source g0/1` | Ping sourced from a specific interface |
| `traceroute 10.2.0.1` | Hop-by-hop path |

---