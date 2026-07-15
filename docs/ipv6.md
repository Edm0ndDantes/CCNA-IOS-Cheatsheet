# IPv6

<!-- ## Enabling and addressing

``` linenums="1"
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

--- -->

## 1 Address format & writing rules

An IPv6 address is **128 bits**, written as eight 16-bit hextets separated by colons:

```text linenums="1"
2001:0db8:0000:0000:0000:00ff:fe00:0001
```

Two compression rules (CCNA loves testing these):

| Rule | Example |
|---|---|
| **1. Drop leading zeros** in each hextet (trailing zeros stay!) | `0db8` → `db8`, `00ff` → `ff`, but `fe00` stays `fe00` |
| **2. Replace ONE run of consecutive all-zero hextets with `::`** | `2001:db8:0:0:0:ff:fe00:1` → `2001:db8::ff:fe00:1` |

- `::` may appear **only once** per address (twice would be ambiguous — you couldn't tell how many zeros each `::` hides).
- If there are two zero-runs, compress the **longest** one; if equal, compress the **leftmost**.
- Fully applied, the example above becomes `2001:db8::ff:fe00:1`.

**Prefix structure of a typical global unicast address:**

```text
| 48 bits                | 16 bits   | 64 bits                    |
| Global routing prefix  | Subnet ID | Interface ID               |
| (from the ISP/RIR)     | (yours)   | (host portion)             |
  2001:db8:acad          : 0001      : 0000:0000:0000:0001
```

- A site typically receives a **/48**, leaving 16 bits of Subnet ID = **65,536 subnets**, each a `/64`.
- **Subnets are always /64** in normal LAN use — SLAAC mathematically requires it (the interface ID is 64 bits). Smaller prefixes (`/127`) appear only on router point-to-point links.
- There is no broadcast in IPv6 and no NAT requirement — every host can hold a globally unique address (privacy comes from firewalls, not address hiding).

## 2 Address types

| Type | Prefix | Purpose |
|---|---|---|
| **Global Unicast (GUA)** | `2000::/3` (starts `2` or `3`) | Public, Internet-routable — the IPv6 "public IP" |
| **Link-Local (LLA)** | `fe80::/10` | Mandatory on every IPv6 interface, auto-generated, valid **only on that link** — used for next-hops, routing protocol adjacencies, NDP |
| **Unique Local (ULA)** | `fc00::/7` (in practice `fd00::/8`) | Private addressing, not Internet-routable — the RFC1918 analogue |
| **Multicast** | `ff00::/8` | One-to-many; **replaces broadcast entirely** |
| **Anycast** | (any unicast prefix) | Same address on multiple devices; routing delivers to the nearest — no special format, just configured as anycast |
| **Loopback** | `::1/128` | The `127.0.0.1` analogue |
| **Unspecified** | `::/128` | "No address yet" — source address during DAD/DHCP solicitation |

**Multicast groups worth memorizing:**

| Group | Who listens |
|---|---|
| `ff02::1` | All nodes on the link (the closest thing to broadcast) |
| `ff02::2` | All routers on the link |
| `ff02::5` / `ff02::6` | All OSPFv3 routers / OSPFv3 DRs |
| `ff02::a` | All EIGRP routers |
| `ff02::1:2` | All DHCPv6 servers/relays |
| `ff02::1:ffxx:xxxx` | **Solicited-node** multicast — formed from the last 24 bits of a unicast address; how NDP reaches a specific host without broadcasting |

**Key mental shift from IPv4:** an interface holds **multiple addresses simultaneously** — always a link-local, usually one or more GUAs, plus its solicited-node and group multicasts. Routing protocols peer and set next-hops using **link-local** addresses.

## 3 Interface ID generation — EUI-64 and the alternatives

Three ways a host/router fills the low 64 bits:

| Method | How it works |
|---|---|
| **Static** | You type it (`::1` for routers is conventional) |
| **EUI-64** | Built from the interface MAC: split the 48-bit MAC in half, insert `fffe` in the middle, **flip the 7th bit** of the first byte. MAC `0c1a.2b3c.4d5e` → interface ID `0e1a:2bff:fe3c:4d5e` |
| **Random** | Modern host default (privacy — a MAC-derived address tracks the device across networks) |

The `fffe` in the middle of an address is the fingerprint of EUI-64 — recognizing it (and the flipped bit) is a classic exam skill.

## 4 NDP — Neighbor Discovery Protocol (the ARP/DHCP-lite replacement)

NDP is a set of ICMPv6 messages that replaces ARP, router discovery, and parts of DHCP:

| Message | Replaces | Function |
|---|---|---|
| **RS** (Router Solicitation, to `ff02::2`) | — | Host at boot: "any routers here?" |
| **RA** (Router Advertisement, to `ff02::1`) | DHCP's gateway option | Router announces: the prefix, whether to use SLAAC/DHCPv6 (the **A/O/M flags**), and itself as gateway |
| **NS** (Neighbor Solicitation, to solicited-node multicast) | ARP request | "Who has this IPv6 address?" — also used for **DAD** (Duplicate Address Detection: NS for your *own* tentative address; silence = address is yours) |
| **NA** (Neighbor Advertisement) | ARP reply | "I have it, here's my MAC" |

## 5 The four ways a host gets an address

Controlled by flags the router sets in its RA:

| Method | RA flags | Address from | Other info (DNS…) from | Router command |
|---|---|---|---|---|
| SLAAC only | A=1, O=0, M=0 | Host builds it: RA prefix + EUI-64/random | RA (RDNSS, if supported) | *(default behavior)* |
| SLAAC + stateless DHCPv6 | A=1, **O=1**, M=0 | SLAAC | DHCPv6 server | `ipv6 nd other-config-flag` |
| Stateful DHCPv6 | A=0, O=1, **M=1** | DHCPv6 server (tracked lease) | DHCPv6 server | `ipv6 nd managed-config-flag` (+ suppress SLAAC) |
| Static | — | You | You | — |

(Full DHCPv6 server recipes are in the DHCP chapter, §11.)

## 6 Enabling and addressing (configuration)

```text linenums="1"
ipv6 unicast-routing
interface g0/0
 ipv6 address 2001:db8:acad:1::1/64
 ipv6 address fe80::1 link-local
 no shutdown
```

- `ipv6 unicast-routing` — **required** on routers to route IPv6 and send RAs; without it the router is just an IPv6 *host*
- `ipv6 address 2001:db8:acad:1::1/64` — static global unicast address
- `ipv6 address fe80::1 link-local` — set a memorable link-local address (otherwise auto-generated via EUI-64); hosts will use this as their gateway, so `fe80::1` on every LAN interface is a popular convention
- Alternatives: `ipv6 address 2001:db8:acad:1::/64 eui-64` builds the interface ID from the MAC; `ipv6 enable` brings up IPv6 with *only* a link-local address; `ipv6 address dhcp` makes the interface a DHCPv6 client; `ipv6 address autoconfig` makes a router interface use SLAAC like a host

## 7 IPv6 static routing

| Command | Explanation |
|---|---|
| `ipv6 route 2001:db8:acad:2::/64 2001:db8:acad:12::2` | Via a global next-hop |
| `ipv6 route 2001:db8:acad:2::/64 g0/1 fe80::2` | Via a link-local next-hop — the exit interface is **mandatory** here (link-locals are ambiguous without one; this is the everyday IPv6 idiom) |
| `ipv6 route ::/0 2001:db8:acad:12::2` | IPv6 default route |
| `ipv6 route 2001:db8:acad:2::/64 2001:db8:acad:12::2 90` | Floating static (AD 90) as backup to a routing protocol |

## 8 SLAAC / RA control (interface mode)

| Command | Explanation |
|---|---|
| `ipv6 nd other-config-flag` | RA with O=1: use SLAAC for the address, DHCPv6 for other info (DNS etc.) |
| `ipv6 nd managed-config-flag` | RA with M=1: hosts should use stateful DHCPv6 for addressing |
| `ipv6 nd prefix default no-autoconfig` | Advertise the prefix with A=0 — suppress SLAAC so DHCPv6 is the only source |
| `ipv6 nd router-preference high` | Make hosts prefer this router when several send RAs |
| `ipv6 nd ra interval 30` | Send RAs every 30 s (default 200) |
| `ipv6 nd ra suppress` | Stop sending RAs on this interface entirely (e.g., toward an ISP) |

## 9 Verification

| Command | Explanation |
|---|---|
| `show ipv6 interface brief` | All IPv6 addresses per interface (note: multiple per interface is normal) |
| `show ipv6 interface g0/0` | Full detail: LLA, GUAs, joined multicast groups, ND timers (see §10.10) |
| `show ipv6 route` | The IPv6 routing table (`C`onnected, `L`ocal, `S`tatic, `O`SPFv3, `ND` = learned from an RA) |
| `show ipv6 neighbors` | The NDP cache — IPv6's ARP table (states: `REACH`, `STALE`, `INCMP`) |
| `show ipv6 routers` | RAs *received* from other routers |
| `ping 2001:db8:acad:2::1` | IPv6 ping |
| `ping fe80::2` | Ping a link-local — IOS will prompt for the output interface (same ambiguity as static routes) |
| `debug ipv6 nd` | Watch NS/NA/RS/RA live (lab use) |

## 10 Annotated example — `show ipv6 interface g0/0`

```text linenums="1"
R1# show ipv6 interface g0/0
GigabitEthernet0/0 is up, line protocol is up
  IPv6 is enabled, link-local address is FE80::1
  No Virtual link-local address(es):
  Global unicast address(es):
    2001:DB8:ACAD:1::1, subnet is 2001:DB8:ACAD:1::/64
  Joined group address(es):
    FF02::1
    FF02::2
    FF02::1:FF00:1
  MTU is 1500 bytes
  ICMP redirects are enabled
  ND DAD is enabled, number of DAD attempts: 1
  ND reachable time is 30000 milliseconds
  ND advertised default router preference is Medium
  Hosts use stateless autoconfig for addresses.
```

**How to read it:**

- `link-local address is FE80::1` — the manually set LLA; hosts on this LAN use it as their default gateway (it's what the RA advertises as the source).

- `Joined group address(es)` — the multicast memberships prove the interface's roles: `FF02::1` (all nodes), `FF02::2` (**all routers** — appears only because `ipv6 unicast-routing` is on; if it's missing, the router isn't routing!), `FF02::1:FF00:1` (the solicited-node group derived from the last 24 bits of both `::1` addresses).

- `ND DAD is enabled` — duplicate address detection runs on every new address; an address flagged `[DUP]` in `show ipv6 interface brief` lost that check and is unusable.

- `Hosts use stateless autoconfig` — the current A/O/M flag outcome in words; after setting `ipv6 nd managed-config-flag` this line changes to "Hosts use DHCP to obtain routable addresses" — a quick way to confirm your RA flags without a packet capture.

!!! note
    Notes on the addition: it replaces the old Section 10 wholesale and keeps its four original blocks (enabling/addressing, static routing, RA control, verification) as §6–9, expanded — the RA-control table gained `prefix default no-autoconfig` and `ra interval`, the static-route table gained the floating static, and verification gained `show ipv6 routers`, link-local ping, and `debug ipv6 nd`. The new theory (§1–5) covers the CCNA checklist: compression rules, prefix anatomy and why /64 is mandatory for SLAAC, all address types with the multicast groups table, EUI-64 with the bit-flip, NDP message types including DAD, and the A/O/M flag matrix that ties RA flags to the DHCPv6 chapter. The annotated `show ipv6 interface` example (§10) follows your Section 18 style — if you keep annotated outputs centralized there instead, that block can be moved.
