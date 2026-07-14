# DHCP

## DHCPv4 server (on a router)

``` linenums="1"
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp pool LAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8 1.1.1.1
 domain-name example.com
 lease 2 12 30
```

- `ip dhcp excluded-address 192.168.10.1 192.168.10.10` — reserve addresses the pool must **not** hand out (gateways, servers)
- `ip dhcp pool LAN10` — create/enter a DHCP pool
- `network 192.168.10.0 255.255.255.0` — the scope: hand out addresses from this subnet
- `default-router 192.168.10.1` — default gateway given to clients
- `dns-server 8.8.8.8 1.1.1.1` — DNS servers given to clients
- `domain-name example.com` — domain suffix
- `lease 2 12 30` — lease time: 2 days, 12 hours, 30 minutes (default 1 day)

## DHCP relay & client

| Command | Explanation |
|---|---|
| `ip helper-address 10.1.1.5` | (Interface mode) Relay clients' DHCP broadcasts to a server on another subnet |
| `ip address dhcp` | (Interface mode) Make this router interface a DHCP **client** (e.g., ISP-facing) |

## DHCPv4 verification

| Command | Explanation |
|---|---|
| `show ip dhcp binding` | Leased addresses and client MACs |
| `show ip dhcp pool` | Pool usage statistics |
| `show ip dhcp server statistics` | Message counters |
| `show ip dhcp conflict` | Addresses detected in use by two hosts |
| `clear ip dhcp binding *` | Release all leases |

## DHCPv6 — stateless (SLAAC address + DHCP options)

``` linenums="1"
ipv6 unicast-routing
ipv6 dhcp pool STATELESS
 dns-server 2001:db8::8
 domain-name example.com
interface g0/0
 ipv6 nd other-config-flag
 ipv6 dhcp server STATELESS
```

- `ipv6 dhcp pool STATELESS` — pool carrying only *options*, no addresses
- `dns-server 2001:db8::8` / `domain-name example.com` — the options handed out
- `ipv6 nd other-config-flag` — tell hosts: SLAAC your address, fetch options via DHCPv6
- `ipv6 dhcp server STATELESS` — attach the pool to the interface

## DHCPv6 — stateful (server assigns addresses)

``` linenums="1"
ipv6 dhcp pool STATEFUL
 address prefix 2001:db8:acad:1::/64
 dns-server 2001:db8::8
interface g0/0
 ipv6 nd managed-config-flag
 ipv6 nd prefix default no-autoconfig
 ipv6 dhcp server STATEFUL
```

- `address prefix 2001:db8:acad:1::/64` — the server assigns addresses from this prefix
- `ipv6 nd managed-config-flag` — tell hosts: get your address from DHCPv6
- `ipv6 nd prefix default no-autoconfig` — suppress SLAAC so DHCPv6 is the only source
- `ipv6 dhcp server STATEFUL` — attach the pool to the interface

Verify with `show ipv6 dhcp pool` and `show ipv6 dhcp binding`.

---