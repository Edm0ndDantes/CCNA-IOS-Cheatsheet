# IPv6

## Enabling and addressing

```
ipv6 unicast-routing
interface g0/0
 ipv6 address 2001:db8:acad:1::1/64
 ipv6 address fe80::1 link-local
 no shutdown
```

- `ipv6 unicast-routing` — **required** on routers to route IPv6 and send Router Advertisements
- `ipv6 address 2001:db8:acad:1::1/64` — static global unicast address
- `ipv6 address fe80::1 link-local` — set a memorable link-local address (otherwise auto-generated via EUI-64)
- Alternatives: `ipv6 address 2001:db8:acad:1::/64 eui-64` builds the host portion automatically from the MAC; `ipv6 enable` enables IPv6 on an interface with only a link-local address

## IPv6 static routing

| Command | Explanation |
|---|---|
| `ipv6 route 2001:db8:acad:2::/64 2001:db8:acad:12::2` | Via a global next-hop |
| `ipv6 route 2001:db8:acad:2::/64 g0/1 fe80::2` | Via a link-local next-hop — the interface is **mandatory** here |
| `ipv6 route ::/0 2001:db8:acad:12::2` | IPv6 default route |

## SLAAC / RA control (interface mode)

| Command | Explanation |
|---|---|
| `ipv6 nd other-config-flag` | RA: use SLAAC for the address, DHCPv6 for other info (DNS etc.) |
| `ipv6 nd managed-config-flag` | RA: hosts should use stateful DHCPv6 for addressing |
| `ipv6 nd router-preference high` | Make hosts prefer this router |
| `ipv6 nd ra suppress` | Stop sending Router Advertisements on this interface |

## Verification

| Command | Explanation |
|---|---|
| `show ipv6 interface brief` | IPv6 addresses per interface |
| `show ipv6 route` | The IPv6 routing table |
| `show ipv6 neighbors` | IPv6 equivalent of the ARP table (NDP cache) |
| `ping 2001:db8:acad:2::1` | IPv6 ping |

---
