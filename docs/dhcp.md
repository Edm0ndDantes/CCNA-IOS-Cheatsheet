# DHCP

## DHCPv4

### DHCPv4 server (on a router)

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

### DHCP relay & client

| Command | Explanation |
|---|---|
| `ip helper-address 10.1.1.5` | (Interface mode) Relay clients' DHCP broadcasts to a server on another subnet |
| `ip address dhcp` | (Interface mode) Make this router interface a DHCP **client** (e.g., ISP-facing) |

### DHCPv4 verification

| Command | Explanation |
|---|---|
| `show ip dhcp binding` | Leased addresses and client MACs |
| `show ip dhcp pool` | Pool usage statistics |
| `show ip dhcp server statistics` | Message counters |
| `show ip dhcp conflict` | Addresses detected in use by two hosts |
| `clear ip dhcp binding *` | Release all leases |

## DHCPv6

### DHCPv6 — stateless (SLAAC address + DHCP options)

```text linenums="1"
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

### DHCPv6 — stateful (server assigns addresses)

```text linenums="1"
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

## Address assignment methods — dynamic vs automatic vs static, and APIPA

The word "automatic" is overloaded here — two different things hide behind it, and the CCNA expects you to keep them apart:

### The three DHCP allocation methods (RFC 2131):

| Method | How it works | Lease |
|---|---|---|
| **Dynamic allocation** | Server hands out an address from the pool **for a limited lease time**; the client must renew (at 50% of the lease, unicast to the server) or the address returns to the pool for reuse | Temporary — the normal mode, and what the IOS `lease` command controls |
| **Automatic allocation** | Same pool mechanism, but the assignment is **permanent** — once a client gets an address, the server remembers the MAC↔IP binding forever (an infinite lease: `lease infinite`) | Permanent, server-chosen |
| **Static/manual allocation** | The administrator pre-binds a specific IP to a specific client MAC on the server; DHCP merely *delivers* the admin's choice (a **DHCP reservation**) | Permanent, admin-chosen |

Static allocation via an IOS host pool:

```text linenums="1"
ip dhcp pool PRINTER-1
 host 192.168.10.50 255.255.255.0
 client-identifier 01aa.bbcc.ddee.ff
 default-router 192.168.10.1
```

- `host 192.168.10.50 255.255.255.0` — a one-address pool: this exact IP, always
- `client-identifier 01aa.bbcc.ddee.ff` — who gets it: `01` (Ethernet type) prepended to the client's MAC (`hardware-address aa.bbcc.ddee.ff` is the alternative keyword on many platforms)
- Result: the printer always receives `.50` but stays a zero-touch DHCP client — reservations beat hand-typed static IPs for anything you may ever re-address

### APIPA (Automatic Private IP Addressing)
**The failure fallback, not a DHCP method:**

- When a host is set to DHCP but **gets no answer** (four DISCOVERs go unanswered), Windows/macOS self-assigns a random address from **`169.254.0.0/16`** (link-local, RFC 3927), after ARP-probing that it's unused. Hosts keep DISCOVERing in the background and abandon the APIPA address the moment a real server responds.
- The address is **link-local only**: no default gateway, never routed, valid for talking to other APIPA hosts on the same segment at best.
- **Diagnostic value — this is the exam point:** a client sitting on `169.254.x.x` is a symptom, and it means *the DHCP process failed end-to-end*. Work the chain: is the port up and in the right VLAN? Is the DHCP server/pool alive (`show ip dhcp pool` — pool exhausted?)? If the server is on another subnet, is `ip helper-address` present on the client's gateway interface? Is DHCP snooping blocking an untrusted port?

| See on the client | Conclusion |
|---|---|
| Address from your pool's range | Dynamic/automatic allocation worked |
| The exact reserved address, every time | Static allocation (reservation) — by design |
| `169.254.x.x` | **APIPA: DHCP failed** — no server reachable, pool empty, relay missing, or snooping drop |
| `0.0.0.0` / none | Interface never completed DHCP and the OS doesn't do APIPA (many Linux configs) — same troubleshooting chain |
