# NAT (Network Address Translation)

**Address terminology (memorize the four):**

| Term | Meaning |
|---|---|
| **Inside local** | Private IP of the internal host (real LAN address) |
| **Inside global** | Public IP representing that host to the outside (often the outside-interface address) |
| **Outside global** | Real public IP of the external host |
| **Outside local** | How the external host appears inside (usually same as outside global) |

- In `ip nat inside source static [inside-local] [inside-global]`, the **second** address is the **inside global**, and it's assigned to/reachable via the **outside interface**.

**Required interface tagging** — NAT does nothing until both are set:
```
interface fa0/0
 ip nat inside               ! LAN-facing
interface s0/0/0
 ip nat outside              ! ISP-facing   <-- forgetting this is a classic broken-NAT cause
```

**Binding the ACL to the pool (dynamic/PAT):**
```
access-list 1 permit 172.19.89.0 0.0.0.255       ! IDENTIFY inside-local addresses to translate
ip nat inside source list 1 pool NAT-POOL2       ! BIND the ACL to the pool (missing = NAT won't work)
ip nat inside source list 1 interface s0/0/0 overload   ! PAT variant onto the outside interface
```

**Verification & maintenance:**
| Command | Use |
|---|---|
| `show ip nat translations` | Active mappings (inside/outside, local/global) |
| `show ip nat statistics` | Config parameters, inside/outside interfaces, **pool size & how many addresses allocated** |
| `clear ip nat translation *` | Remove **dynamic** entries before the 24-h timeout (needed when changing pools) |
| `debug ip nat` | Live per-packet translation (`s=` source, `d=` dest, `->` translated) |

**Common failure signatures:**
- `show ip nat statistics` shows **pool 100% allocated** → **pool exhausted**, new clients get nothing.
- `debug ip nat` shows a translated address on a **different subnet than the ISP interface** → return traffic can't be routed back (inside-global not in the provider's subnet).
- **Static NAT** with the **wrong inside-local** (e.g. pointing at the router's own interface instead of the server) → the server is unreachable from outside.



## Mark the interfaces (required for every NAT type)

``` linenums="1"
interface g0/0
 ip nat inside
interface g0/1
 ip nat outside
```

- `ip nat inside` — this interface faces the private LAN
- `ip nat outside` — this interface faces the Internet/ISP

## Static NAT (one-to-one; for servers that must be reachable from outside)

| Command | Explanation |
|---|---|
| `ip nat inside source static 192.168.10.5 203.0.113.5` | Permanently map private ↔ public |

## Dynamic NAT (pool of public addresses)

``` linenums="1"
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat pool PUBLIC 203.0.113.10 203.0.113.20 netmask 255.255.255.0
ip nat inside source list 1 pool PUBLIC
```

- `access-list 1 permit 192.168.10.0 0.0.0.255` — define **who** gets translated
- `ip nat pool PUBLIC 203.0.113.10 203.0.113.20 netmask 255.255.255.0` — define the public address pool
- `ip nat inside source list 1 pool PUBLIC` — tie the ACL and pool together

## PAT / NAT overload (many-to-one; the typical home/office setup)

``` linenums="1"
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface g0/1 overload
```

- `access-list 1 permit 192.168.10.0 0.0.0.255` — who gets translated
- `ip nat inside source list 1 interface g0/1 overload` — everyone shares the outside interface's IP, distinguished by source port. Alternative: `ip nat inside source list 1 pool PUBLIC overload` overloads onto a pool instead

## Port forwarding (static PAT)

| Command | Explanation |
|---|---|
| `ip nat inside source static tcp 192.168.10.5 80 203.0.113.5 8080` | Outside `:8080` → inside server `:80` |

## Verification

| Command | Explanation |
|---|---|
| `show ip nat translations` | The active translation table (see §18.8) |
| `show ip nat statistics` | Hit/miss counters, interface roles, pool usage |
| `clear ip nat translation *` | Flush dynamic translations (needed before changing NAT config) |
| `debug ip nat` | Watch translations happen live (lab use; `undebug all` to stop) |

---

## NAT64 (IPv6 ↔ IPv4 translation)

### Theory

IPv6 hosts and IPv4 servers can't talk directly — the headers are incompatible. During the (long) transition period there are three coexistence strategies: **dual stack** (run both protocols everywhere — preferred), **tunneling** (carry IPv6 inside IPv4, e.g., GRE), and **translation** — rewriting one protocol into the other. NAT64 is the translation approach; it replaced the deprecated NAT-PT.

- **How it works:** an IPv6-only client sends traffic to a special IPv6 address that *embeds* the IPv4 destination. The NAT64 router extracts the IPv4 address, translates the whole packet (headers, checksums), and sources it from an IPv4 pool — classic NAT bookkeeping, but across protocol families.
- **The NAT64 prefix:** IPv4 destinations are embedded in a /96 prefix — 96 prefix bits + 32 IPv4 bits = 128. The IANA **well-known prefix is `64:ff9b::/96`**, so IPv4 host `198.51.100.10` becomes `64:ff9b::198.51.100.10` (IOS displays it in hex: `64:ff9b::c633:640a`). An organization can use its own /96 (an *NSP*, network-specific prefix) instead.
- **DNS64 — the missing half:** clients don't know to use the prefix. A DNS64 server intercepts AAAA queries: when a name has only an A record (IPv4-only site), it *synthesizes* a fake AAAA answer by embedding the A record into the NAT64 prefix. The client then innocently connects "over IPv6" and lands on the NAT64 router. **NAT64 without DNS64 only works for hand-typed embedded addresses.** (DNS64 runs on the DNS server, not on the router.)
- **Stateful vs stateless:**

| Mode | How | Scale | Use case |
|---|---|---|---|
| **Stateful NAT64** | Per-session translation state, many-to-one onto an IPv4 pool (PAT-style, RFC 6146) | Thousands of IPv6 clients share few IPv4 addresses | The normal deployment: IPv6-only clients → IPv4 Internet |
| **Stateless NAT64** | Algorithmic 1:1 mapping, no state (RFC 7915) | One IPv4 address consumed per IPv6 host | Small, static cases; IPv4 clients → IPv6 servers |

- **Direction matters:** stateful NAT64 is designed for sessions **initiated from the IPv6 side** (like PAT, the return path exists only after state is created). Unsolicited IPv4→IPv6 requires static mappings.
- **Limitations:** anything embedding literal IP addresses in the payload breaks or needs an ALG (old FTP, SIP); end-to-end IPsec through the translator fails (the header rewrite invalidates it); ICMP is translated to/from ICMPv6 but some types don't map.

### Stateful NAT64 configuration

IPv6-only LAN on `g0/0`, IPv4 Internet on `g0/1`, translating clients onto pool `203.0.113.100–110`:

```text linenums="1"
ipv6 unicast-routing
interface g0/0
 ipv6 address 2001:db8:acad:1::1/64
 nat64 enable
interface g0/1
 ip address 203.0.113.2 255.255.255.0
 nat64 enable
ipv6 access-list NAT64-CLIENTS
 permit ipv6 2001:db8:acad:1::/64 any
nat64 prefix stateful 64:ff9b::/96
nat64 v4 pool POOL4 203.0.113.100 203.0.113.110
nat64 v6v4 list NAT64-CLIENTS pool POOL4 overload
```

- `nat64 enable` — mark **both** the IPv6-facing and IPv4-facing interfaces; the NAT64 analogue of `ip nat inside`/`outside` (one command for both sides — direction is inferred from the traffic)
- `permit ipv6 2001:db8:acad:1::/64 any` — the ACL defines **which IPv6 sources** may be translated (the analogue of the NAT source list)
- `nat64 prefix stateful 64:ff9b::/96` — the translation prefix: traffic *destined to* this prefix is what triggers NAT64; the embedded IPv4 address is extracted from the last 32 bits
- `nat64 v4 pool POOL4 203.0.113.100 203.0.113.110` — the IPv4 addresses translations are sourced from
- `nat64 v6v4 list NAT64-CLIENTS pool POOL4 overload` — glue it together; `overload` = PAT-style port multiplexing, so the whole IPv6 LAN shares a few IPv4 addresses (omit it and each IPv6 host consumes a pool address 1:1)
- Routing sanity: the IPv4 side needs a route back to the pool (connected here), and the IPv6 side needs a route for `64:ff9b::/96` pointing at this router — typically the clients' default route already handles it

Static mapping (let IPv4 users reach an IPv6-only server):

| Command | Explanation |
|---|---|
| `nat64 v6v4 static 2001:db8:acad:1::10 203.0.113.50` | Permanently bind the IPv6 server to a public IPv4 address — inbound IPv4-initiated sessions now work for this host |

### Verification

| Command | Explanation |
|---|---|
| `show nat64 translations` | The active translation table — IPv6 client ↔ IPv4 pool bindings per session |
| `show nat64 statistics` | Packets translated per direction, active sessions, drops — the counters prove it's working |
| `show nat64 prefix stateful` | The configured translation prefix and where it applies |
| `show nat64 pools` | Pool usage |
| `clear nat64 statistics` | Reset counters before a test |
| `debug nat64` | Watch translations live (lab use; `undebug all` to stop) |

!!! note
    **Quick test without DNS64:** from an IPv6-only client, ping the embedded form of a known IPv4 address — e.g., `ping 64:ff9b::8.8.8.8`. If that works but browsing by name fails, NAT64 is fine and **DNS64 is what's missing**.
