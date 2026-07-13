# NAT (Network Address Translation)

Terminology: **inside local** = private IP of a host; **inside global** = its public representation. The `ip nat inside`/`ip nat outside` interface marks are mandatory for any NAT to work.

## Mark the interfaces (required for every NAT type)

```
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

```
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat pool PUBLIC 203.0.113.10 203.0.113.20 netmask 255.255.255.0
ip nat inside source list 1 pool PUBLIC
```

- `access-list 1 permit 192.168.10.0 0.0.0.255` — define **who** gets translated
- `ip nat pool PUBLIC 203.0.113.10 203.0.113.20 netmask 255.255.255.0` — define the public address pool
- `ip nat inside source list 1 pool PUBLIC` — tie the ACL and pool together

## PAT / NAT overload (many-to-one; the typical home/office setup)

```
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