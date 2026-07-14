# Access Control Lists (ACLs)

## Numbered standard ACL (filters by SOURCE only; 1–99, 1300–1999)
Global command syntax for standard numbered access lists.
``` linenums="1"
access-list <access-list-number> {deny | permit} <source> [<source-wildcard>] [log]
```

Examples:

``` linenums="1"
access-list 10 permit 192.168.10.0 0.0.0.255
access-list 10 deny host 192.168.20.5
access-list 10 permit any
```

- `access-list 10 permit 192.168.10.0 0.0.0.255` — permit the whole subnet (wildcard mask again)
- `access-list 10 deny host 192.168.20.5` — `host` = wildcard `0.0.0.0` (one address)
- `access-list 10 permit any` — allow everything else (**every ACL ends with an implicit deny all**)

Command syntax that defines a remark to help remember what the ACL is supposed to do:
``` linenums="1"
access-list <access-list-number> remark <text>
```

## Numbered extended ACL (source, destination, protocol, ports; 100–199, 2000–2699)

``` linenums="1"
access-list 100 permit tcp 192.168.10.0 0.0.0.255 any eq 443
access-list 100 permit tcp 192.168.10.0 0.0.0.255 any eq 80
access-list 100 deny   tcp any host 10.1.1.5 eq 23
access-list 100 permit icmp any any echo-reply
access-list 100 permit tcp any any established
access-list 100 deny   ip any any log
```

- `permit tcp 192.168.10.0 0.0.0.255 any eq 443` — allow HTTPS from the LAN to anywhere
- `permit tcp ... any eq 80` — allow HTTP
- `deny tcp any host 10.1.1.5 eq 23` — block Telnet to one server
- `permit icmp any any echo-reply` — allow ping replies back in
- `permit tcp any any established` — allow return traffic of TCP sessions started inside
- `deny ip any any log` — explicit deny with logging (makes drops visible; the implicit deny logs nothing)
- Port operators: `eq` (equals), `neq`, `gt`, `lt`, `range 1000 2000`

## Named ACLs (editable, self-documenting — preferred)

``` linenums="1"
ip access-list extended BLOCK-WEB
 10 deny tcp 192.168.10.0 0.0.0.255 any eq 80
 20 permit ip any any
ip access-list extended BLOCK-WEB
 15 deny tcp 192.168.10.0 0.0.0.255 any eq 443
 no 10
```

- `ip access-list extended BLOCK-WEB` — create/enter a named extended ACL
- `10 deny ...` / `20 permit ...` — sequence numbers let you insert/delete individual lines
- `15 deny ...` — insert a new rule between lines 10 and 20
- `no 10` — delete just line 10 (impossible with classic numbered ACLs, which must be rewritten whole)

## Applying ACLs

| Command | Mode | Explanation |
|---|---|---|
| `ip access-group 100 in` | Interface | Filter traffic entering this interface |
| `ip access-group BLOCK-WEB out` | Interface | Filter traffic leaving this interface |
| `access-class 10 in` | Line (vty) | Restrict who may SSH/Telnet to the device itself |

**Placement rules of thumb:** standard ACLs close to the *destination* (they can't see destinations); extended ACLs close to the *source* (kill unwanted traffic early). One ACL per interface, per direction, per protocol.

## IPv6 ACLs

``` linenums="1"
ipv6 access-list BLOCK-V6
 deny tcp 2001:db8:acad:1::/64 any eq 23
 permit ipv6 any any
interface g0/0
 ipv6 traffic-filter BLOCK-V6 in
```

- `ipv6 access-list BLOCK-V6` — IPv6 ACLs are always named
- `ipv6 traffic-filter BLOCK-V6 in` — note: `traffic-filter`, not `access-group`

## Verification

| Command | Explanation |
|---|---|
| `show access-lists` | All ACLs with per-line match counters (see §18.9) |
| `show ip interface g0/0` | Shows which ACLs are applied where |
| `clear access-list counters` | Reset the hit counters |

---