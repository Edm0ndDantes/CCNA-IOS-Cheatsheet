# VPNs & Tunnels
## VPN Types & Classification

### VP.1 What a VPN is

A VPN creates a **private network across a public network** using **virtual (logical) connections** instead of dedicated physical links — carrying traffic securely over the shared Internet.

### VP.2 The two fundamental types

| Type | Connects | "Always on"? | Client needed? |
|---|---|---|---|
| **Site-to-site** | **Entire networks** (site ↔ site) via VPN gateways | Yes, static | No — transparent to end hosts |
| **Remote-access** | **Individual users** to a network | On demand | Yes — VPN client (or clientless browser) on the user device |

- **Remote-access examples:** a mobile sales agent connecting from a hotel; an employee running VPN client software from home.
- **Site-to-site examples:** a branch office ASA tunneling to HQ; a permanent tunnel to a supplier; connecting two merged companies' networks **without leased-line cost** → site-to-site VPN is the cost-effective answer.

### VP.3 Enterprise-managed vs provider-managed

| Managed by | Site-to-site | Remote-access |
|---|---|---|
| **Enterprise** | **IPsec VPN**, **GRE over IPsec**, **DMVPN**, IPsec VTI | Clientless SSL VPN, client-based IPsec/SSL (AnyConnect) |
| **Service provider** | **Layer 3 MPLS VPN**, Layer 2 VPN | — |

### VP.4 Remote-access technologies

- **SSL/TLS VPN** — connects using **Transport Layer Security**; works through a standard **web browser**. If a user's host lacks the **Cisco AnyConnect** client, they start a **clientless SSL VPN** in a compliant browser and download/install the client from there.
- **IPsec remote-access** — client-based, stronger for full network-layer access.

### VP.5 DMVPN & mGRE

- **DMVPN** builds many site-to-site tunnels **dynamically and scalably** from three components: **mGRE + NHRP + IPsec**.
- **mGRE** (multipoint GRE) provides a **permanent tunnel source at the hub** with **dynamically allocated tunnel destinations at the spokes** — one hub interface serves all spokes, enabling on-demand spoke-to-spoke tunnels.
- **NHRP** is the distributed **address-mapping** database (spoke public IPs); **IPsec** encrypts.

### VP.6 Why GRE-over-IPsec for routing protocols

Pure **IPsec carries unicast only** — it can't transport the **multicast/broadcast** that OSPF/EIGRP rely on. **GRE encapsulates multicast/broadcast**, and IPsec then encrypts the GRE packet → **GRE over IPsec** is the standard for running a routing protocol across an encrypted site-to-site link.
---

## Tunneling & VPN Concepts

A **tunnel** encapsulates one protocol inside another so traffic can cross a network that couldn't otherwise carry it (private addresses across the Internet, IPv6 across IPv4, multicast across a unicast-only core). A **VPN** is a tunnel with privacy expectations — though not every tunnel encrypts (GRE and MPLS do **not**).

| Technology | What it does | Encrypts? | Carries multicast/routing protocols? | Typical use |
|---|---|---|---|---|
| **GRE** | Generic encapsulation, IP protocol 47 | No | **Yes** | Connect sites over the Internet, carry OSPF/EIGRP, wrap non-IP traffic |
| **IPsec** | Authenticated + encrypted transport (ESP, IP protocol 50) | **Yes** | No (pure IPsec is unicast IP only) | Site-to-site and remote-access encryption |
| **GRE over IPsec** | GRE flexibility inside IPsec protection | Yes | Yes | The standard encrypted site-to-site with dynamic routing |
| **MPLS** | Label switching in a provider core; L3VPNs via VRFs + MP-BGP | No (separation, not secrecy) | Yes (within a VPN) | Carrier backbone, enterprise WAN (provider-managed) |
| **DMVPN** | Multipoint GRE + NHRP + IPsec | Yes | Yes | Hub-and-spoke that builds dynamic spoke-to-spoke tunnels |

**Overhead planning:** every layer of encapsulation adds header bytes and eats into the 1500-byte Ethernet MTU. 
- GRE adds 24 bytes; 
- IPsec ESP (tunnel mode, AES) adds roughly 50–73. 

This is why tunnel interfaces almost always carry `ip mtu` and `ip tcp adjust-mss` commands — forgetting them is the #1 cause of "ping works but HTTPS hangs" over tunnels.

### VP.7 Packet anatomy — before & after encapsulation

Reference topology used in all diagrams below (the same one as the config sections):

```text
 Host A                R1                    Internet                 R2                Host B
 192.168.1.10 ---- g0/0: 192.168.1.1                        g0/0: 192.168.2.1 ---- 192.168.2.10
                   g0/1: 203.0.113.1  <==== tunnel0 ====>   g0/1: 198.51.100.2
                      tun0: 172.16.0.1                    172.16.0.2 :tun0
```

Three kinds of IP addresses are in play — keeping them straight is the whole game:

| Address type | Example | Where it appears |
|---|---|---|
| **Original / inner** addresses | `192.168.1.10 → 192.168.2.10` | The hosts' packet — always the **inner** header; never changed by tunneling |
| **Physical / transport** addresses | `203.0.113.1 → 198.51.100.2` | The **outer** header of every tunneled packet. Comes from `tunnel source` / `tunnel destination` (GRE) or the crypto peers (IPsec) — i.e., the **physical** (or loopback) interface, *never* the tunnel interface |
| **Tunnel interface (overlay)** addresses | `172.16.0.1 / 172.16.0.2` | **Not in the headers of transit user traffic at all.** They exist for the routing table: routes point at `172.16.0.2` as a next-hop, which resolves to "encapsulate and send out tunnel0". They show up *inside* packets only when the routers themselves talk across the tunnel (OSPF hellos, `ping 172.16.0.2`) — then they are the *inner* src/dst |

#### The original packet (before any tunnel)

```text
+---------------------+----------+------------------+
|      IP header      | TCP/UDP  |       Data       |
|  src 192.168.1.10   |  header  |                  |
|  dst 192.168.2.10   |          |                  |
|  proto 6 (TCP)      |          |                  |
+---------------------+----------+------------------+
```

This can't cross the Internet as-is: `192.168.x.x` isn't routable there. Every method below solves that differently.

#### GRE encapsulation

```text
Before:                                   After (on the wire, R1 → R2):
+------------------+-----+------+         +--------------------+---------+------------------+-----+------+
| IP header        | TCP | Data |   ==>   | NEW IP header      | GRE hdr | ORIGINAL IP hdr  | TCP | Data |
| s 192.168.1.10   |     |      |         | s 203.0.113.1      | (4 B)   | s 192.168.1.10   |     |      |
| d 192.168.2.10   |     |      |         | d 198.51.100.2     |         | d 192.168.2.10   |     |      |
+------------------+-----+------+         | proto 47 (GRE)     |         | proto 6 (TCP)    |     |      |
                                          +--------------------+---------+------------------+-----+------+
                                            20 B                 4 B       = 24 B total overhead
```

- **New (outer) header:** src = R1's `tunnel source` (physical `g0/1`, `203.0.113.1`); dst = R1's `tunnel destination` (R2's physical, `198.51.100.2`). The Internet routes on these alone.
- **Original (inner) header:** completely untouched — R2 decapsulates and forwards it as if it had arrived natively.
- The tunnel IPs `172.16.0.x` appear nowhere in this packet. They were only the routing-table next-hop that *selected* tunnel0.
- **Not encrypted:** anyone on the path reads the inner packet in cleartext.

#### IPsec transport mode

```text
Before:                                   After:
+------------------+-----+------+         +---------------------+---------+===========+========+  ESP    ESP
| IP header        | TCP | Data |   ==>   | ORIGINAL IP header  | ESP hdr | TCP       | Data   | trailer|auth|
| s 192.168.1.10   |     |      |         |  (REUSED, modified) |         | ENCRYPTED | ENCRYPT|  ENCR  |    |
| d 192.168.2.10   |     |      |         | s 192.168.1.10 (!)  |         |           |        |        |    |
+------------------+-----+------+         | d 192.168.2.10 (!)  |         |           |        |        |    |
                                          | proto 50 (ESP)      |         |           |        |        |    |
                                          +---------------------+---------+===========+========+--------+----+
                                          (=== marks the encrypted span)
```

- **No new header is added.** The original header is *reused* — only its protocol field changes (6 → 50) — and the payload behind it is encrypted.
- **Consequence:** the src/dst on the wire are still the original addresses. If those are private LAN addresses (as here), **the packet cannot cross the Internet** — which is why pure transport mode is *not* used for site-to-site host traffic. It fits only when the communicating endpoints' own addresses are already routable end-to-end: host-to-host on an internal network, or **router-to-router packets like GRE**, whose header already carries the public `203.0.113.1 → 198.51.100.2` (see GRE over IPsec below).
- Saves 20 bytes versus tunnel mode — that's its whole appeal.

#### IPsec tunnel mode

```text
Before:                                   After:
+------------------+-----+------+         +--------------------+---------+==================+=====+======+  ESP    ESP
| IP header        | TCP | Data |   ==>   | NEW IP header      | ESP hdr | ORIGINAL IP hdr  | TCP | Data | trailer|auth|
| s 192.168.1.10   |     |      |         | s 203.0.113.1      |         | s 192.168.1.10   |     |      |  ENCR  |    |
| d 192.168.2.10   |     |      |         | d 198.51.100.2     |         | d 192.168.2.10   |     |      |        |    |
+------------------+-----+------+         | proto 50 (ESP)     |         |    ENCRYPTED     |ENCR | ENCR |        |    |
                                          +--------------------+---------+==================+=====+======+--------+----+
```

- **New (outer) header:** src/dst = the **crypto peers** — R1's physical `203.0.113.1` and the `set peer 198.51.100.2` address. With a crypto map there is no tunnel interface at all; with an SVTI (`tunnel mode ipsec ipv4`), the outer IPs still come from `tunnel source`/`tunnel destination` — physical addresses again.
- **Original header + payload: fully encrypted.** An observer sees only "some ESP between 203.0.113.1 and 198.51.100.2" — the LAN addresses, ports, and protocols are hidden. This is both the privacy win and why tunnel mode is the site-to-site default.
- Overhead: new IP (20) + ESP header/IV (~8–16) + trailer/padding + ICV (~12–16) ≈ **50–73 bytes** depending on cipher.

#### GRE over IPsec — transport mode (the recommended combination)

Two-step encapsulation on R1: **GRE first, then ESP protects the GRE packet.**

```text
Step 1 — GRE (as above):
+--------------------+---------+------------------+-----+------+
| GRE IP header      | GRE hdr | ORIGINAL IP hdr  | TCP | Data |
| s 203.0.113.1      |         | s 192.168.1.10   |     |      |
| d 198.51.100.2     |         | d 192.168.2.10   |     |      |
| proto 47           |         |                  |     |      |
+--------------------+---------+------------------+-----+------+

Step 2 — ESP transport mode wraps it (reuses the GRE packet's header):
+--------------------+---------+=========+==================+=====+======+  ESP    ESP
| GRE IP header      | ESP hdr | GRE hdr | ORIGINAL IP hdr  | TCP | Data | trailer|auth|
| s 203.0.113.1      |         | ENCRYPT | s 192.168.1.10   |     |      |  ENCR  |    |
| d 198.51.100.2     |         |         | d 192.168.2.10   |     |      |        |    |
| proto 50 (ESP)     |         |         |    ENCRYPTED     |ENCR | ENCR |        |    |
+--------------------+---------+=========+==================+=====+======+--------+----+
```

- Transport mode is safe here precisely because the packet being encrypted is **router-to-router**: the GRE header's addresses (`203.0.113.1 → 198.51.100.2`) are already public/routable, so reusing that header works perfectly. This is the textbook use-case for transport mode.
- **One** outer IP header total. Overhead: 24 (GRE) + ~30–50 (ESP without a new IP header).
- The GRE header and everything inside it is encrypted — an observer can't even tell it's GRE.

#### GRE over IPsec — tunnel mode (works, but wasteful)

```text
+--------------------+---------+==================+=========+==================+=====+======+  ESP
| NEW IP header      | ESP hdr | GRE IP header    | GRE hdr | ORIGINAL IP hdr  | TCP | Data | trl |auth|
| s 203.0.113.1      |         | s 203.0.113.1    | ENCRYPT | s 192.168.1.10   |     |      | ENC |    |
| d 198.51.100.2     |         | d 198.51.100.2   |         | d 192.168.2.10   |     |      |     |    |
| proto 50           |         | proto 47, ENCR   |         |    ENCRYPTED     | ENC | ENC  |     |    |
+--------------------+---------+==================+=========+==================+=====+======+-----+----+
```

- Tunnel mode adds a **new** outer header — which ends up carrying the *same* `203.0.113.1 → 198.51.100.2` pair that the (now encrypted) GRE header already contains. Two identical IP headers, 20 bytes of pure waste per packet.
- This is exactly why the recipes in §3.5 recommend `mode transport` under the transform set for GRE over IPsec. (Tunnel mode becomes necessary only in some NAT-traversal and multi-vendor corner cases.)

#### Summary — who fills which header

| Method | Outer src/dst filled from | Inner src/dst | Tunnel interface IPs used for |
|---|---|---|---|
| GRE | `tunnel source` / `tunnel destination` → **physical** (or loopback) IPs | Original hosts, unchanged, cleartext | Routing next-hop only (overlay) |
| IPsec transport | *(no outer header — original reused)* | Original hosts, **visible**, payload encrypted | n/a (no tunnel interface) |
| IPsec tunnel | Crypto peers (`set peer` / `tunnel source`+`destination`) → **physical** IPs | Original hosts, **encrypted** | n/a for crypto map; routing next-hop for SVTI |
| GRE over IPsec (transport) | The GRE header's IPs → **physical** IPs, reused by ESP | Original hosts, encrypted (GRE header too) | Routing next-hop only (overlay) |

**Rule of thumb:** outer headers are always built from **physical/loopback** addresses — the ones the transit network can route. Tunnel-interface addresses never ride the wire in user traffic; they exist so *your* routing table has something to point at.

---

## GRE (Generic Routing Encapsulation)

### VP.8 Theory

- GRE wraps an inner packet (IPv4, IPv6, or even non-IP) in a new IP header (protocol **47**) plus a 4-byte GRE header. Total overhead: **24 bytes** (20 IP + 4 GRE).
- The tunnel appears as a **point-to-point interface** (`tunnel0`). Anything routable over an interface is routable over a GRE tunnel — including **routing protocols and multicast**, which plain IPsec can't carry. This is GRE's main selling point.
- **No security whatsoever**: no encryption, no authentication (an optional key value is a tag, not a secret). Assume anything in a bare GRE tunnel is publicly readable — pair with IPsec for privacy (§3.5).
- Terminology: the **tunnel source/destination** addresses (the *outer* header, usually public IPs) are the transport; the **tunnel interface IPs** (usually a private /30) are the overlay that your routing uses.
- **Recursive routing** — the classic GRE failure: if the router learns the route to the *tunnel destination itself* **through the tunnel**, the tunnel collapses (it would need itself to reach itself). IOS detects it and logs `%TUN-5-RECURDOWN`, flapping the tunnel. Prevention: reach the tunnel destination via a static route or a routing process that never runs inside the tunnel.
- Tunnel state logic: the interface is `up/up` if it has a source, a destination, and a route to the destination — **it does not verify the far end is alive**. Add `keepalive` to detect a dead peer.

### VP.9 Configuration (point-to-point GRE between two sites)

**R1 (site A, public IP 203.0.113.1):**

```text linenums="1"
interface tunnel0
 ip address 172.16.0.1 255.255.255.252
 tunnel source g0/1 !or tunnel source 203.0.113.1
 tunnel destination 198.51.100.2
 keepalive 5 3
 ip mtu 1400
 ip tcp adjust-mss 1360
```

**R2 (site B, public IP 198.51.100.2) — mirror image:**

```text linenums="1"
interface tunnel0
 ip address 172.16.0.2 255.255.255.252
 tunnel source g0/1 !or tunnel source 198.51.100.2
 tunnel destination 203.0.113.1
 keepalive 5 3
 ip mtu 1400
 ip tcp adjust-mss 1360
```

- `interface tunnel0` — create the logical tunnel interface (the number is locally significant)
- `ip address 172.16.0.1 255.255.255.252` — the overlay address; a /30 or /31 per tunnel is conventional
- `tunnel source g0/1` — outer-header source: the physical exit interface (or its IP, or a loopback for stability)
- `tunnel destination 198.51.100.2` — outer-header destination: the far end's **public** IP; must be reachable *outside* the tunnel (see recursive routing above)
- `keepalive 5 3` — send a keepalive every 5 s, declare the tunnel down after 3 misses; without this the tunnel stays `up/up` even if the peer is dead
- `ip mtu 1400` — leave room for the 24-byte GRE header (and future IPsec) so the router fragments sanely
- `ip tcp adjust-mss 1360` — clamp TCP handshakes so sessions never *try* to send oversized segments (MSS = IP MTU − 40)
- GRE is the default tunnel encapsulation; `tunnel mode gre ip` exists but doesn't need to be typed

Then route site-to-site traffic **through the tunnel** — either static:

| Command | Explanation |
|---|---|
| `ip route 192.168.2.0 255.255.255.0 172.16.0.2` | Send site B's LAN via the tunnel's far-end overlay IP |

...or run a routing protocol over it (the point of GRE):

```text linenums="1"
router ospf 1
 network 172.16.0.0 0.0.0.3 area 0
 network 192.168.1.0 0.0.0.255 area 0
```

- `network 172.16.0.0 0.0.0.3 area 0` — form an OSPF adjacency *across the tunnel*
- `network 192.168.1.0 0.0.0.255 area 0` — advertise the local LAN into it
- **Never** advertise the tunnel destination's public network into this process, or recursive routing kills the tunnel

### VP.10 Verification

| Command | Explanation |
|---|---|
| `show interfaces tunnel0` | State, encapsulation, source/destination, keepalive settings |
| `show ip interface brief \| include Tunnel` | Quick up/down check of all tunnels |
| `show ip route <far-LAN>` | Confirm site-to-site routes point into the tunnel |
| `ping 172.16.0.2` | Test the overlay (far end's tunnel IP) |
| `ping 192.168.2.1 source g0/0` | Test end-to-end, sourced from the LAN (proves routing, not just the tunnel) |
| `debug tunnel` | Watch encapsulation events (lab use) |

### VP.11 Annotated example — `show interfaces tunnel0`

```text linenums="1"
R1# show interfaces tunnel0
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Internet address is 172.16.0.1/30
  MTU 17916 bytes, BW 100 Kbit/sec, DLY 50000 usec,
  Encapsulation TUNNEL, loopback not set
  Keepalive set (5 sec), retries 3
  Tunnel source 203.0.113.1 (GigabitEthernet0/1), destination 198.51.100.2
  Tunnel protocol/transport GRE/IP
    Key disabled, sequencing disabled
  Tunnel transport MTU 1476 bytes
```

**How to read it:**
- `up, line protocol is up` — with keepalives set, `up/up` actually means the far end answers. Without keepalives, `up/up` only means "I have a source, a destination, and a route to it".
- `up, line protocol is down` + keepalives configured = the peer isn't responding: far end down, misconfigured (source/destination don't mirror), or GRE (IP protocol 47) blocked by a firewall in the path.
- `Tunnel source 203.0.113.1 ... destination 198.51.100.2` — must be the exact mirror of the far end's config; the most common GRE mistake is a source that doesn't match the peer's configured destination (e.g., traffic leaves a different interface, or NAT rewrites it).
- `Tunnel transport MTU 1476` — physical 1500 − 24 GRE bytes; payload larger than this gets fragmented or dropped.
- `BW 100 Kbit/sec` — tunnels default to a tiny bandwidth value, which **wrecks OSPF/EIGRP metrics**; set `bandwidth 100000` on the tunnel to reflect reality.

---

## IPsec

### VP.12 Theory

IPsec is a framework of protocols that provides 
- **confidentiality** (encryption), 
- **integrity** (hashing), 
- **authentication** (peer identity), and 
- **anti-replay** (sequence numbers) 

at the IP layer.

**The two traffic protocols:**

| Protocol | IP proto | Provides | Notes |
|---|---|---|---|
| **ESP** (Encapsulating Security Payload) | 50 | Encryption + integrity + authentication | What everyone actually uses |
| **AH** (Authentication Header) | 51 | Integrity + authentication only, **no encryption** | Breaks through NAT (it hashes the IP header); essentially legacy |

**The two modes:**

- **Tunnel mode** — the entire original IP packet is encrypted and wrapped in a *new* IP header. Site-to-site default: internal addresses are hidden.
- **Transport mode** — only the payload is encrypted; the original IP header is reused. Less overhead; used host-to-host and for GRE-over-IPsec (the GRE header already provides the outer tunnel).

**The negotiation — IKE (Internet Key Exchange), UDP/500:**

- **Phase 1** builds a secure management channel (the *ISAKMP/IKE SA*) between the peers. Here they agree on encryption, hash, Diffie-Hellman group, authentication method (pre-shared key or certificates), and lifetime — memorable as **HAGLE** (Hash, Authentication, Group, Lifetime, Encryption).
- **Phase 2** negotiates the *IPsec SAs* inside that channel: which traffic to protect (the "interesting traffic" ACL / proxy IDs) and with which **transform set** (ESP algorithms). SAs are unidirectional — there's always a pair.
- **IKEv2** is the modern replacement (fewer messages, built-in NAT-T and keepalives, EAP support). Concepts are identical; commands differ (§3.4).
- **NAT-T**: when a NAT device is detected in the path, ESP is wrapped in **UDP/4500** so port translation can work. Firewalls must allow UDP/500, UDP/4500, and ESP (protocol 50).

**Algorithm quick guidance:** prefer `aes 256` / AES-GCM, `sha256` or better, DH group `14`+ (or 19/20 ECDH); avoid DES/3DES, MD5, SHA-1, and groups 1/2/5 — they appear in old examples and exams but are broken or deprecated.

### VP.13 Site-to-site IPsec, IKEv1 with crypto map

**R1 (203.0.113.1), protecting LAN 192.168.1.0/24 ↔ remote LAN 192.168.2.0/24:**

```text linenums="1"
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key MySecretKey123 address 198.51.100.2
crypto ipsec transform-set TSET esp-aes 256 esp-sha256-hmac
ip access-list extended VPN-TRAFFIC
 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
crypto map CMAP 10 ipsec-isakmp
 set peer 198.51.100.2
 set transform-set TSET
 match address VPN-TRAFFIC
interface g0/1
 crypto map CMAP
```

- `crypto isakmp policy 10` — a Phase 1 proposal; the peers compare policies by priority number until one matches
- `encryption aes 256` / `hash sha256` / `group 14` — the HAGLE parameters: cipher, integrity hash, and Diffie-Hellman group for key exchange
- `authentication pre-share` — peers authenticate with a shared secret (certificates are the scalable alternative)
- `lifetime 86400` — rekey the Phase 1 SA daily (seconds)
- `crypto isakmp key MySecretKey123 address 198.51.100.2` — the pre-shared key, bound to the peer's address; **must match on both sides**
- `crypto ipsec transform-set TSET esp-aes 256 esp-sha256-hmac` — the Phase 2 proposal: ESP with AES-256 encryption and SHA-256 integrity (tunnel mode is the default; add `mode transport` under it for GRE-over-IPsec)
- `permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255` — the **interesting traffic** ACL: what gets encrypted. The far end's ACL must be the exact **mirror image**, or Phase 2 fails
- `crypto map CMAP 10 ipsec-isakmp` — glue: peer + transform set + interesting traffic in one policy entry
- `crypto map CMAP` (on the interface) — activate it on the Internet-facing interface; only **one** crypto map per interface (add more peers as more sequence numbers)

**R2 mirrors everything:** same policy/transform values, `set peer 203.0.113.1`, key bound to `203.0.113.1`, and the ACL reversed (`permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255`).

**Gotcha — NAT vs VPN:** if the same router also does PAT to the Internet, the NAT ACL must **deny** the VPN traffic first (`deny ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255` above the `permit`), otherwise packets are translated before encryption and never match the crypto ACL.

### VP.14 IPsec verification

| Command | Explanation |
|---|---|
| `show crypto isakmp sa` | Phase 1 status — you want `QM_IDLE` / `ACTIVE` (see §3.6) |
| `show crypto ipsec sa` | Phase 2 SAs with encrypt/decrypt packet counters — the ground truth |
| `show crypto map` | The assembled policy: peers, ACLs, transform sets, which interface |
| `show crypto session` | One-line summary of all VPN sessions and their state |
| `show crypto isakmp policy` | Configured Phase 1 proposals (and defaults) |
| `clear crypto isakmp` / `clear crypto sa` | Tear down Phase 1 / Phase 2 to force renegotiation after changes |
| `debug crypto isakmp` | Watch Phase 1 negotiate — the #1 tool for "tunnel won't come up" (lab/maintenance window) |
| `debug crypto ipsec` | Watch Phase 2 / SA installation |

### VP.15 Modern equivalent: IKEv2 with SVTI (route-based VPN)

Instead of a crypto map + ACL ("policy-based"), modern IOS builds the VPN as a **virtual tunnel interface** — anything *routed into the tunnel* gets encrypted ("route-based"). Cleaner, supports routing protocols natively, no mirror-image ACLs.

```text linenums="1"
crypto ikev2 proposal PROP
 encryption aes-cbc-256
 integrity sha256
 group 14
crypto ikev2 policy POL
 proposal PROP
crypto ikev2 keyring KEYS
 peer R2
  address 198.51.100.2
  pre-shared-key MySecretKey123
crypto ikev2 profile PROF
 match identity remote address 198.51.100.2 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS
crypto ipsec profile IPSEC-PROF
 set ikev2-profile PROF
interface tunnel0
 ip address 172.16.0.1 255.255.255.252
 tunnel source g0/1
 tunnel destination 198.51.100.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROF
```

- `crypto ikev2 proposal PROP` — the IKEv2 crypto suite (Phase 1 equivalent); `policy POL` selects which proposals are offered
- `crypto ikev2 keyring KEYS` — pre-shared keys, organized per peer
- `crypto ikev2 profile PROF` — ties identity matching, authentication method, and keyring together (the brain of IKEv2 config)
- `crypto ipsec profile IPSEC-PROF` — the Phase 2 wrapper applied to tunnels (uses a default transform set unless you `set transform-set`)
- `tunnel mode ipsec ipv4` — a pure IPsec tunnel interface (SVTI); no GRE header at all
- `tunnel protection ipsec profile IPSEC-PROF` — encrypt everything entering this interface
- Now simply **route** through `tunnel0` (static or OSPF/EIGRP/BGP) — the routing table decides what's encrypted

### VP.16 GRE over IPsec (encrypted tunnel that carries routing protocols)

Take the GRE tunnel from §2.2 and add protection — two extra blocks:

```text linenums="1"
crypto ipsec profile GRE-PROT
 set transform-set TSET
interface tunnel0
 tunnel protection ipsec profile GRE-PROT
```

- `crypto ipsec profile GRE-PROT` + `set transform-set TSET` — reuse the Phase 2 transform set inside a profile (Phase 1 config from §3.2 or §3.4 is still required)
- `tunnel protection ipsec profile GRE-PROT` — wrap the GRE tunnel in IPsec; **both ends** need it or the tunnel dies
- Best practice: add `mode transport` under the transform set — the GRE header already tunnels, so transport mode saves 20 bytes of double-encapsulation
- Result: OSPF/EIGRP/multicast flow like plain GRE, but everything on the wire is ESP

### VP.17 Annotated example — `show crypto isakmp sa` and `show crypto ipsec sa`

```text linenums="1"
R1# show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
198.51.100.2    203.0.113.1     QM_IDLE           1001 ACTIVE
```

**How to read it:**
- `QM_IDLE` + `ACTIVE` = Phase 1 succeeded and is at rest ("Quick Mode idle") — healthy.
- `MM_NO_STATE` = Phase 1 dying repeatedly: usually a **pre-shared key mismatch** or no matching `isakmp policy` (compare HAGLE values on both ends).
- `MM_KEY_EXCH` stuck = keys exchanging but authentication failing — again, almost always the PSK.
- **Empty output** = Phase 1 never even started: no interesting traffic has been sent yet (send a ping between the LANs first!), the crypto map isn't on the interface, or UDP/500 is blocked.

```text linenums="1"
R1# show crypto ipsec sa
interface: GigabitEthernet0/1
    Crypto map tag: CMAP, local addr 203.0.113.1
   protected vrf: (none)
   local  ident (addr/mask/prot/port): (192.168.1.0/255.255.255.0/0/0)
   remote ident (addr/mask/prot/port): (192.168.2.0/255.255.255.0/0/0)
   current_peer 198.51.100.2 port 500
    #pkts encaps: 4312, #pkts encrypt: 4312, #pkts digest: 4312
    #pkts decaps: 4298, #pkts decrypt: 4298, #pkts verify: 4298
    #send errors 0, #recv errors 0
```

**How to read it:**
- `local ident` / `remote ident` — the negotiated proxy IDs; they must be mirror images of the peer's. A Phase 2 failure with Phase 1 `ACTIVE` is almost always **mismatched crypto ACLs**.
- `#pkts encaps/encrypt` **and** `#pkts decaps/decrypt` both incrementing = traffic flows both ways — the tunnel genuinely works.
- Encrypts climbing but **decrypts stuck at 0** = your packets go out, nothing comes back: far end's return route missing, its crypto ACL doesn't mirror yours, or ESP (protocol 50) is blocked toward you. This asymmetry is the single most diagnostic line in IPsec troubleshooting.
- `#send errors` climbing = local problem (often MTU/fragmentation).

---

## MPLS (Multiprotocol Label Switching)

### VP.18 Theory

MPLS forwards packets by a short **label** instead of the destination IP — routers in the core switch labels without ever looking up the full routing table, and more importantly, labels let providers build **separated customer VPNs** over one shared backbone.

**The label:** a 4-byte "shim" header inserted **between** the layer-2 header and the IP header (why MPLS is called *layer 2.5*). Fields: 20-bit label value, 3-bit EXP/TC (QoS), 1 bottom-of-stack bit (labels can **stack** — crucial for VPNs), 8-bit TTL.

**The roles:**

| Term | Meaning |
|---|---|
| **LSR** (P router) | Label Switch Router — core device that only swaps labels; knows nothing about customer routes |
| **LER / edge LSR** (PE router) | Provider Edge — imposes (pushes) labels on ingress, removes (pops) on egress; holds the customer VRFs |
| **CE router** | Customer Edge — the customer's router; runs ordinary IP toward the PE, no MPLS awareness needed |
| **LSP** | Label Switched Path — the unidirectional label path a packet follows through the core |
| **FEC** | Forwarding Equivalence Class — the set of packets treated identically (typically: same destination prefix) |

**The label operations:** **push** (ingress PE adds a label), **swap** (each core LSR replaces the label), **pop** (label removed). **PHP** (Penultimate Hop Popping): the *second-to-last* router pops the label so the final router does just one lookup instead of two — signaled with the reserved **implicit-null** label (3), which is why `show` output says `Pop Label`.

**How labels are learned — LDP (Label Distribution Protocol):** every router assigns a local label to each prefix in its routing table and advertises the binding to its neighbors (LDP uses UDP/646 hello discovery, then a TCP/646 session). LDP **follows the IGP** — it labels the paths OSPF/IS-IS already chose. Therefore: **no working IGP → no working MPLS**.

**MPLS L3VPN — the flagship application (theory only, config is provider-side):**

- Each customer gets a **VRF** (Virtual Routing and Forwarding instance) on the PE — a private routing table. Customers can all use 10.0.0.0/8 without conflict.
- A **RD** (Route Distinguisher) is prepended to each customer prefix to make it globally unique (`65000:1:10.1.1.0/24`); **RTs** (Route Targets, BGP extended communities) control which VRFs import/export which routes — this is how topologies (any-to-any, hub-and-spoke, extranet) are built.
- **MP-BGP** carries these VPNv4 routes between PEs, along with a **VPN label**. Packets in the core carry a **two-label stack**: outer = transport label (LDP, gets you to the egress PE), inner = VPN label (tells the egress PE which VRF/next-hop). The P routers only ever see the outer label — that's the separation.
- From the enterprise view: you hand your routes to the PE via static/OSPF/eBGP, and the provider's MPLS is invisible — but knowing the theory explains provider terms (RD/RT, VPNv4) and why traceroutes through the core may show `MPLS: Label 24005` hops.

### VP.19 Core commands — enabling MPLS/LDP

Prerequisite: a working IGP (e.g., OSPF) across all MPLS-enabled links, ideally with `/32` loopbacks advertised (LDP sessions and BGP next-hops ride on them).

```text linenums="1"
ip cef
mpls ip
interface g0/1
 mpls ip
interface g0/2
 mpls ip
mpls ldp router-id loopback0 force
```

- `ip cef` — Cisco Express Forwarding must be on (it is by default on anything modern); MPLS is built on CEF
- `mpls ip` (global) — enable MPLS forwarding on the router
- `mpls ip` (interface) — run LDP on each core-facing link; both ends must enable it for a session to form
- `mpls ldp router-id loopback0 force` — pin the LDP router ID to the loopback (stability; `force` resets sessions to apply it)

Optional hardening/tuning:

| Command | Explanation |
|---|---|
| `mpls ldp neighbor 10.0.0.2 password MplsPass` | MD5-protect the LDP TCP session with that neighbor |
| `mpls mtu 1512` | (Interface) Allow labeled packets bigger than the IP MTU — two labels = +8 bytes; prevents core fragmentation |
| `mpls ldp igp sync` | Don't route traffic onto a link until LDP is up on it (prevents blackholing after link recovery) |
| `no mpls ip propagate-ttl` | Hide the MPLS core from customer traceroutes (they see one hop instead of every P router) |

### VP.20 VRF commands (the PE-side building block)

Even outside a provider, **VRF-lite** (VRFs without MPLS) is common in enterprises for tenant/guest separation — same commands, no label stack:

```text linenums="1"
vrf definition CUSTOMER-A
 rd 65000:1
 route-target export 65000:1
 route-target import 65000:1
 address-family ipv4
interface g0/0
 vrf forwarding CUSTOMER-A
 ip address 10.1.1.1 255.255.255.0
```

- `vrf definition CUSTOMER-A` — create the VRF (older IOS: `ip vrf CUSTOMER-A`)
- `rd 65000:1` — the Route Distinguisher that keeps this customer's prefixes globally unique
- `route-target export/import 65000:1` — tag routes leaving this VRF / accept routes carrying this tag (MP-BGP environments)
- `address-family ipv4` — activate IPv4 inside the VRF
- `vrf forwarding CUSTOMER-A` — bind an interface to the VRF — **this wipes the interface's IP address**, so re-enter it afterwards (deliberate IOS behavior)

Working *inside* a VRF requires VRF-aware commands:

| Command | Explanation |
|---|---|
| `show ip route vrf CUSTOMER-A` | That customer's private routing table |
| `ping vrf CUSTOMER-A 10.1.1.2` | Ping from within the VRF (a plain `ping` uses the global table and will fail) |
| `traceroute vrf CUSTOMER-A 10.1.1.2` | Trace within the VRF |
| `ip route vrf CUSTOMER-A 10.2.0.0 255.255.255.0 10.1.1.2` | Static route inside the VRF |

### VP.21 MPLS verification

| Command | Explanation |
|---|---|
| `show mpls interfaces` | Which interfaces run MPLS/LDP — the first sanity check |
| `show mpls ldp neighbor` | LDP sessions — look for `Oper` state and the TCP connection |
| `show mpls ldp discovery` | Hello adjacencies (a neighbor here but not in `ldp neighbor` = TCP/646 blocked or password mismatch) |
| `show mpls ldp bindings` | The LIB: every label learned for every prefix, from every neighbor |
| `show mpls forwarding-table` | The LFIB: the labels actually used to forward (see §4.5) |
| `show ip cef 10.0.0.5 detail` | Exactly how one destination is forwarded, labels included |
| `traceroute 10.0.0.5` | Labeled hops appear as `MPLS: Label 24005 Exp 0` — proof packets ride the LSP |

### VP.22 Annotated example — `show mpls forwarding-table`

```text linenums="1"
P1# show mpls forwarding-table
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
16         Pop Label  10.0.0.3/32      152340        Gi0/2      10.1.23.3
17         18         10.0.0.4/32      98220         Gi0/2      10.1.23.3
18         No Label   192.168.5.0/24   0             Gi0/1      10.1.12.1
```

**How to read it:**
- Each row: "if a packet arrives with **Local Label** X, do **Outgoing Label** action and send it to **Next Hop**".
- Row 1 — `Pop Label`: this router is the **penultimate hop** for 10.0.0.3 (PHP in action): it strips the label and forwards plain IP to the final router.
- Row 2 — `17 → 18`: a classic **swap** in the middle of an LSP.
- Row 3 — `No Label`: the next hop never advertised a label for this prefix — the packet leaves **unlabeled**. Fine at the MPLS edge; a **problem in the core** (it breaks L3VPN, whose inner label needs the transport label to survive). Causes: LDP not enabled on that link, LDP session down, or the prefix missing from the neighbor's IGP.
- `Bytes Label Switched` stuck at 0 on a path that should carry traffic = the LSP exists but nothing uses it — check IGP path selection.

---

## Quick "is the tunnel working?" checklist

| Question | Command |
|---|---|
| Is the tunnel interface up? | `show ip interface brief \| include Tunnel` |
| Does the overlay respond? | `ping <far tunnel IP>` |
| GRE peer really alive? | `keepalive` configured + `show interfaces tunnel0` |
| IPsec Phase 1 up? | `show crypto isakmp sa` → `QM_IDLE`/`ACTIVE` |
| IPsec passing traffic *both* ways? | `show crypto ipsec sa` → encaps **and** decaps rising |
| Why won't Phase 1/2 negotiate? | `debug crypto isakmp` / `debug crypto ipsec` |
| LDP neighbors up? | `show mpls ldp neighbor` |
| Labels actually forwarding? | `show mpls forwarding-table` (beware `No Label` in the core) |
| VRF routing correct? | `show ip route vrf <name>`, `ping vrf <name>` |
| Big packets dying in the tunnel? | Check `ip mtu` / `ip tcp adjust-mss` on the tunnel interface |
