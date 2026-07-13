# Cisco IOS Command Reference Cheatsheet

A practical reference for configuring Cisco routers and switches. Single commands are presented in tables; multi-step setup processes are shown as code blocks with each command explained below.

Commands are entered from these prompt levels:

| Prompt | Mode | How to get there |
|---|---|---|
| `Router>` | User EXEC | Default on login |
| `Router#` | Privileged EXEC | `enable` |
| `Router(config)#` | Global configuration | `configure terminal` |
| `Router(config-if)#` | Interface configuration | `interface g0/0` |
| `Router(config-line)#` | Line configuration | `line console 0` / `line vty 0 4` |
| `Router(config-router)#` | Router (routing protocol) config | `router ospf 1` |



## 3. Interface Configuration

### Router interface (routed port)

```
interface GigabitEthernet0/0
 description LINK-TO-LAN
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 duplex full
 speed 1000
```

- `interface GigabitEthernet0/0` — enter interface configuration mode
- `description LINK-TO-LAN` — free-text label (best practice: always document links)
- `ip address 192.168.1.1 255.255.255.0` — assign IPv4 address and subnet mask
- `no shutdown` — enable the interface (router ports are shut down by default)
- `duplex full` — force duplex (default: `auto`)
- `speed 1000` — force speed (default: `auto`)

### Switch management IP (SVI)

Switches are layer 2 — they get a management address on a VLAN interface, not a physical port:

```
interface vlan 1
 ip address 192.168.1.2 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.1.1
```

- `interface vlan 1` — enter the SVI for VLAN 1 (use a dedicated management VLAN in production)
- `ip address 192.168.1.2 255.255.255.0` — the management IP
- `no shutdown` — bring the SVI up
- `ip default-gateway 192.168.1.1` — gateway so the switch can be managed from other subnets

### Loopback interface

```
interface loopback 0
 ip address 10.0.0.1 255.255.255.255
```

- `interface loopback 0` — virtual interface, always up; used for router IDs, testing, management
- `ip address 10.0.0.1 255.255.255.255` — typically a /32 host route

---

## 4. Security Hardening

### Passwords & encryption

| Command | Explanation |
|---|---|
| `enable secret class` | Password for privileged EXEC — stored as a hash (**always use this**) |
| `enable password cisco` | Legacy plaintext version — avoid; `secret` overrides it |
| `service password-encryption` | Weakly encrypts (type 7) all plaintext passwords in the config |
| `security passwords min-length 10` | Enforce a minimum password length |
| `username admin secret StrongPass123` | Create a local user with a hashed password |
| `username admin privilege 15 secret X` | Local user that lands directly in privileged EXEC |

**Why `secret` over `password`:** `secret` stores an MD5/scrypt hash that can't be trivially reversed; `password` is plaintext (type 7 "encryption" is decodable in seconds).

### SSH setup (complete recipe)

```
hostname R1
ip domain-name example.com
crypto key generate rsa
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 2
username admin secret StrongPass123
line vty 0 4
 transport input ssh
 login local
```

- `hostname R1` — a non-default hostname is required for key generation
- `ip domain-name example.com` — a domain name is also required for key generation
- `crypto key generate rsa` — generate the RSA keypair; choose modulus **2048** when prompted
- `ip ssh version 2` — enforce SSHv2 only (v1 is insecure)
- `ip ssh time-out 60` — drop unauthenticated sessions after 60 seconds
- `ip ssh authentication-retries 2` — maximum login attempts per connection
- `username admin secret StrongPass123` — SSH requires local (or AAA) user accounts
- `transport input ssh` — disallow Telnet entirely on the VTY lines
- `login local` — authenticate against the local username database

Verify with `show ip ssh` (settings) and `show ssh` (active sessions).

### AAA — Authentication, Authorization, Accounting

AAA replaces per-line passwords with a centralized policy framework. Authentication = who you are, Authorization = what you may do, Accounting = what you did. Sources can be the **local** user database or external **RADIUS**/**TACACS+** servers.

#### Enable AAA with local authentication

```
username admin privilege 15 secret StrongPass123
aaa new-model
aaa authentication login default local
aaa authentication login CONSOLE none
line console 0
 login authentication CONSOLE
```

- `username admin privilege 15 secret StrongPass123` — **always create a local user BEFORE enabling AAA**, or you can lock yourself out
- `aaa new-model` — turn on AAA (this immediately changes how all lines authenticate!)
- `aaa authentication login default local` — default policy: authenticate all logins against the local user database
- `aaa authentication login CONSOLE none` — optional named list: no authentication (lab consoles only)
- `login authentication CONSOLE` — apply a named list to a specific line (lines without one use `default`)

#### TACACS+ server (Cisco-preferred for device administration)

```
tacacs server TAC1
 address ipv4 10.1.1.50
 single-connection
 key TacacsSharedKey
aaa group server tacacs+ TAC-GROUP
 server name TAC1
```

- `tacacs server TAC1` — define a TACACS+ server object
- `address ipv4 10.1.1.50` — the server's IP address
- `single-connection` — reuse one TCP connection (efficiency)
- `key TacacsSharedKey` — shared secret; must match the server
- `aaa group server tacacs+ TAC-GROUP` — group servers so several can be listed for redundancy
- `server name TAC1` — add the server to the group

#### RADIUS server (common for network access / 802.1X)

```
radius server RAD1
 address ipv4 10.1.1.60 auth-port 1812 acct-port 1813
 key RadiusSharedKey
aaa group server radius RAD-GROUP
 server name RAD1
```

- `radius server RAD1` — define a RADIUS server object
- `address ipv4 10.1.1.60 auth-port 1812 acct-port 1813` — server IP and the standard RADIUS ports
- `key RadiusSharedKey` — shared secret
- `aaa group server radius RAD-GROUP` / `server name RAD1` — create a redundancy group and add the server

#### Method lists — tie it together (order = fallback order)

| Command | Explanation |
|---|---|
| `aaa authentication login default group TAC-GROUP local` | Try TACACS+ first; fall back to local users if the servers are unreachable |
| `aaa authentication enable default group TAC-GROUP enable` | Authenticate `enable` via TACACS+, fall back to the enable secret |
| `aaa authorization exec default group TAC-GROUP local` | Server decides privilege level / shell access |
| `aaa authorization commands 15 default group TAC-GROUP local` | Per-command authorization for priv-15 commands (TACACS+ only) |
| `aaa accounting exec default start-stop group TAC-GROUP` | Log session start/stop to the server |
| `aaa accounting commands 15 default start-stop group TAC-GROUP` | Log every priv-15 command executed (audit trail) |

**Always end method lists with `local`** — if every server is down and there's no fallback, you cannot log in.

**TACACS+ vs RADIUS at a glance:** TACACS+ uses TCP/49, encrypts the whole payload, and separates authentication/authorization (enables per-command control) — best for managing the devices themselves. RADIUS uses UDP/1812-1813, encrypts only the password, combines authen+author — best for authenticating end users (Wi-Fi, VPN, 802.1X).

#### AAA verification & troubleshooting

| Command | Explanation |
|---|---|
| `show aaa servers` | Server states and request counters |
| `show tacacs` | TACACS+ server statistics |
| `show aaa sessions` | Active AAA user sessions |
| `test aaa group TAC-GROUP admin StrongPass123 legacy` | Test authentication against the server without logging out |
| `debug aaa authentication` | Watch the method lists being evaluated live |
| `debug tacacs` | Watch TACACS+ exchanges (lab use; `undebug all` to stop) |

### SSH with AAA (complete recipe)

Combines the SSH setup above with AAA-based login — the production-grade remote access config:

```
hostname R1
ip domain-name example.com
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret StrongPass123
aaa new-model
tacacs server TAC1
 address ipv4 10.1.1.50
 key TacacsSharedKey
aaa group server tacacs+ TAC-GROUP
 server name TAC1
aaa authentication login SSH-AUTH group TAC-GROUP local
aaa authorization exec default group TAC-GROUP local
line vty 0 4
 transport input ssh
 login authentication SSH-AUTH
 exec-timeout 10 0
 access-class 10 in
```

- `crypto key generate rsa modulus 2048` — generate the keypair inline (no prompt)
- `username admin privilege 15 secret StrongPass123` — local fallback account; **create it first**
- `aaa authentication login SSH-AUTH group TAC-GROUP local` — named method list: TACACS+ first, local fallback
- `aaa authorization exec default group TAC-GROUP local` — the server assigns the shell/privilege level
- `transport input ssh` — SSH only, no Telnet
- `login authentication SSH-AUTH` — use the AAA method list instead of `login local`
- `exec-timeout 10 0` — kill idle sessions after 10 minutes
- `access-class 10 in` — optional: ACL 10 restricts which source IPs may connect

**Note:** once `aaa new-model` is active, `login local` on a line is ignored — AAA method lists take over. Keep a console session open while testing so a mistake can't lock you out.

### Login protection

| Command | Explanation |
|---|---|
| `login block-for 120 attempts 3 within 60` | Block logins for 120 s after 3 failed attempts in 60 s (brute-force defense) |
| `login on-failure log` | Log failed login attempts |
| `login on-success log` | Log successful logins |

### Disable unused/risky services

| Command | Explanation |
|---|---|
| `no ip http server` | Disable the HTTP management interface |
| `no ip http secure-server` | Disable HTTPS management (if unused) |
| `no cdp run` | Disable CDP globally (it leaks device info) |
| `no cdp enable` | (Interface mode) Disable CDP on one untrusted interface only |
| `shutdown` | (Interface mode) Administratively disable any unused interface |

### Switch port security

Limits which/how many MAC addresses may use an access port:

```
interface f0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
```

- `switchport mode access` — port security requires a static access (or trunk) mode
- `switchport port-security` — enable port security
- `switchport port-security maximum 2` — allow at most 2 MAC addresses (default 1)
- `switchport port-security mac-address sticky` — learn MACs dynamically and save them to the config (alternatively `switchport port-security mac-address aaaa.bbbb.cccc` statically defines an allowed MAC)
- `switchport port-security violation shutdown` — on violation: err-disable the port (default). Other options: `restrict` (drop + log + counter), `protect` (drop silently)

Verification:

| Command | Explanation |
|---|---|
| `show port-security` | Summary of all secured ports |
| `show port-security interface f0/1` | Detailed status of one port (see §18.10 for annotated output) |
| `show port-security address` | The secure MAC address table |

Recovering an err-disabled port: `shutdown` then `no shutdown` on the interface, or automate:

| Command | Explanation |
|---|---|
| `errdisable recovery cause psecure-violation` | Auto-recover from port-security violations |
| `errdisable recovery interval 300` | ...after 300 seconds |

### DHCP snooping & Dynamic ARP Inspection (switch)

```
ip dhcp snooping
ip dhcp snooping vlan 10,20
interface g0/1
 ip dhcp snooping trust
interface range f0/1 - 24
 ip dhcp snooping limit rate 10
ip arp inspection vlan 10,20
interface g0/1
 ip arp inspection trust
```

- `ip dhcp snooping` — enable DHCP snooping globally (blocks rogue DHCP servers)
- `ip dhcp snooping vlan 10,20` — apply it to specific VLANs
- `ip dhcp snooping trust` — mark the uplink toward the real DHCP server as trusted
- `ip dhcp snooping limit rate 10` — rate-limit DHCP messages on untrusted ports (packets/second)
- `ip arp inspection vlan 10,20` — enable DAI, which validates ARP packets against the snooping binding table
- `ip arp inspection trust` — trust the uplink for ARP as well

---

## 5. VLANs & Trunking (Switch)

### Creating VLANs

```
vlan 10
 name SALES
vlan 20
 name ENGINEERING
vlan 99
 name MANAGEMENT
```

- `vlan 10` — create VLAN 10 and enter VLAN configuration mode
- `name SALES` — assign a human-readable name

### Access ports (end devices)

```
interface f0/5
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 150
interface range f0/6 - 10
 switchport mode access
 switchport access vlan 20
```

- `switchport mode access` — force the port to be an access port (single VLAN)
- `switchport access vlan 10` — assign it to VLAN 10
- `switchport voice vlan 150` — optional: separate voice VLAN for an attached IP phone
- `interface range f0/6 - 10` — configure multiple ports at once

### Trunk ports (switch-to-switch / switch-to-router)

```
interface g0/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,99
 switchport nonegotiate
```

- `switchport mode trunk` — force trunking (carries multiple VLANs, 802.1Q tagged)
- `switchport trunk native vlan 99` — untagged traffic belongs to VLAN 99 (change from the default 1 for security)
- `switchport trunk allowed vlan 10,20,99` — only permit these VLANs on the trunk (prune everything else). Use `switchport trunk allowed vlan add 30` to add a VLAN without wiping the list — **omitting `add` replaces the entire list!**
- `switchport nonegotiate` — disable DTP negotiation frames (hardening)
- On some older switches you must first set `switchport trunk encapsulation dot1q`

### DTP (Dynamic Trunking Protocol) modes

| Command | Explanation |
|---|---|
| `switchport mode dynamic desirable` | Actively tries to form a trunk |
| `switchport mode dynamic auto` | Forms a trunk only if the other side asks |

**Best practice:** hard-code `switchport mode access` or `switchport mode trunk` + `switchport nonegotiate`; never rely on DTP (VLAN-hopping risk).

### Verification & cleanup

| Command | Explanation |
|---|---|
| `show vlan brief` | VLANs and which ports belong to each (see §18.4) |
| `show interfaces trunk` | Trunk ports, native VLANs, allowed VLANs (see §18.5) |
| `show interfaces f0/5 switchport` | Full L2 details for one port |
| `no vlan 10` | Delete VLAN 10 |
| `delete vlan.dat` | Erase the VLAN database (factory reset step 2 on switches) |

---

## 6. Inter-VLAN Routing

### Router-on-a-stick (subinterfaces)

```
interface g0/0
 no shutdown
interface g0/0.10
 encapsulation dot1q 10
 ip address 192.168.10.1 255.255.255.0
interface g0/0.20
 encapsulation dot1q 20
 ip address 192.168.20.1 255.255.255.0
interface g0/0.99
 encapsulation dot1q 99 native
 ip address 192.168.99.1 255.255.255.0
```

- `interface g0/0` + `no shutdown` — the physical interface must be up; it carries the trunk
- `interface g0/0.10` — subinterface for VLAN 10 (the number after the dot is arbitrary, but matching the VLAN keeps you sane)
- `encapsulation dot1q 10` — tag/untag frames for VLAN 10 (must come before the IP address)
- `ip address 192.168.10.1 255.255.255.0` — the default gateway for VLAN 10 hosts
- `encapsulation dot1q 99 native` — match the trunk's native VLAN here

### Layer 3 switch (SVIs)

```
ip routing
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
interface g0/1
 no switchport
 ip address 10.0.0.1 255.255.255.252
```

- `ip routing` — enable routing on the multilayer switch (off by default)
- `interface vlan 10` + `ip address ...` — an SVI acts as the gateway for that VLAN
- `no switchport` — convert a switch port into a routed (L3) port
- `ip address 10.0.0.1 255.255.255.252` — a routed uplink, typically a /30 point-to-point

---

## 7. Spanning Tree Protocol (STP)

STP prevents Layer-2 loops by electing a root bridge and blocking redundant paths.

### Global configuration

| Command | Explanation |
|---|---|
| `spanning-tree mode rapid-pvst` | Use Rapid PVST+ (fast convergence; recommended). Others: `pvst`, `mst` |
| `spanning-tree vlan 10 root primary` | Make this switch the root bridge for VLAN 10 (sets priority to 24576 or lower) |
| `spanning-tree vlan 10 root secondary` | Backup root (priority 28672) |
| `spanning-tree vlan 10 priority 4096` | Or set the priority manually (multiple of 4096; lower wins) |
| `spanning-tree portfast default` | PortFast on all access ports at once |
| `spanning-tree portfast bpduguard default` | BPDU Guard on all PortFast ports |

### Per-interface features

| Command | Explanation |
|---|---|
| `spanning-tree portfast` | Skip listening/learning on an access port — host ports come up instantly |
| `spanning-tree bpduguard enable` | Err-disable the port if a BPDU arrives (a switch was plugged into a host port) |
| `spanning-tree guard root` | Root Guard: block a downstream switch from becoming root via this port |
| `spanning-tree cost 10` | Manipulate path cost (lower preferred) |
| `spanning-tree port-priority 64` | Tie-breaker between equal-cost ports (lower wins, multiple of 32) |

### Verification

| Command | Explanation |
|---|---|
| `show spanning-tree` | Root bridge, port roles/states for every VLAN (see §18.6) |
| `show spanning-tree vlan 10` | Same, for one VLAN |
| `show spanning-tree summary` | Modes and feature status (PortFast, BPDU Guard, etc.) |

**Reading the output:** the root bridge shows `This bridge is the root`. Port roles: Root (best path to root), Designated (forwarding), Alternate/Blocked.

---

## 8. EtherChannel (Link Aggregation)

Bundles multiple physical links into one logical link (more bandwidth, no STP blocking between them).

```
interface range g0/1 - 2
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
```

- `channel-group 1 mode active` — LACP active, actively negotiates (best practice). Other modes: `passive` = LACP, waits for the other side; `desirable` / `auto` = PAgP (Cisco proprietary); `on` = static bundle, no protocol (both sides must be `on`)
- `interface port-channel 1` — configure the logical interface; settings apply to all members
- `switchport mode trunk` / `switchport trunk allowed vlan 10,20,99` — trunk settings applied to the bundle as a whole

Verify with `show etherchannel summary` (flags: `SU` = Layer 2, in use; `P` = port bundled — see §18.11).

**Rule:** all member ports must match exactly (speed, duplex, mode, allowed VLANs), or the bundle will fail.

---

## 9. IPv4 Static Routing

### Route variants

| Command | Explanation |
|---|---|
| `ip route 10.2.0.0 255.255.255.0 192.168.1.2` | Route to 10.2.0.0/24 via next-hop 192.168.1.2 |
| `ip route 10.2.0.0 255.255.255.0 g0/1` | Exit-interface form (fine on point-to-point links) |
| `ip route 10.2.0.0 255.255.255.0 g0/1 192.168.1.2` | Fully specified (both) — most robust |
| `ip route 0.0.0.0 0.0.0.0 203.0.113.1` | Default route ("gateway of last resort") — matches everything |
| `ip route 10.2.0.0 255.255.255.0 172.16.0.2 90` | Floating static: AD 90 makes it a backup to a routing protocol |
| `ip route 10.2.0.0 255.255.255.128 192.168.1.2` | Static route to a specific subnet (e.g., a /25) |

### Administrative Distance (AD) — trustworthiness of a route source (lower wins)

| Source | AD |
|---|---|
| Connected | 0 |
| Static | 1 |
| eBGP | 20 |
| EIGRP | 90 |
| OSPF | 110 |
| RIP | 120 |

### Verification

| Command | Explanation |
|---|---|
| `show ip route` | The routing table: `C`=connected, `S`=static, `O`=OSPF, `S*`=default static (see §18.1) |
| `show ip route 10.2.0.5` | Which route would be used to reach this exact address |
| `show ip route static` | Only static routes |
| `ping 10.2.0.1` | Basic reachability test |
| `ping 10.2.0.1 source g0/1` | Ping sourced from a specific interface |
| `traceroute 10.2.0.1` | Hop-by-hop path |

---

## 10. IPv6

### Enabling and addressing

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

### IPv6 static routing

| Command | Explanation |
|---|---|
| `ipv6 route 2001:db8:acad:2::/64 2001:db8:acad:12::2` | Via a global next-hop |
| `ipv6 route 2001:db8:acad:2::/64 g0/1 fe80::2` | Via a link-local next-hop — the interface is **mandatory** here |
| `ipv6 route ::/0 2001:db8:acad:12::2` | IPv6 default route |

### SLAAC / RA control (interface mode)

| Command | Explanation |
|---|---|
| `ipv6 nd other-config-flag` | RA: use SLAAC for the address, DHCPv6 for other info (DNS etc.) |
| `ipv6 nd managed-config-flag` | RA: hosts should use stateful DHCPv6 for addressing |
| `ipv6 nd router-preference high` | Make hosts prefer this router |
| `ipv6 nd ra suppress` | Stop sending Router Advertisements on this interface |

### Verification

| Command | Explanation |
|---|---|
| `show ipv6 interface brief` | IPv6 addresses per interface |
| `show ipv6 route` | The IPv6 routing table |
| `show ipv6 neighbors` | IPv6 equivalent of the ARP table (NDP cache) |
| `ping 2001:db8:acad:2::1` | IPv6 ping |

---

## 11. DHCP

### DHCPv4 server (on a router)

```
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

### DHCPv6 — stateless (SLAAC address + DHCP options)

```
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

```
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

## 12. OSPF (Open Shortest Path First)

### OSPFv2 (IPv4) basic setup

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

### OSPF cost & reference bandwidth (metric tuning)

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

### Interface-level OSPF (modern alternative to `network`)

| Command | Explanation |
|---|---|
| `ip ospf 1 area 0` | Enable OSPF directly on the interface |
| `ip ospf cost 15` | Manually set cost (lower preferred); cost = reference-bw ÷ interface-bw |
| `ip ospf priority 100` | DR election priority (`0` = never DR; higher wins; default 1) |
| `ip ospf hello-interval 5` | Hello timer (must match the neighbor; default 10 s on Ethernet) |
| `ip ospf dead-interval 20` | Dead timer (must match; default 4× hello) |
| `ip ospf network point-to-point` | Skip DR/BDR election on point-to-point Ethernet links |

### OSPF authentication (hardening)

```
interface g0/1
 ip ospf message-digest-key 1 md5 SecretKey
router ospf 1
 area 0 authentication message-digest
```

- `ip ospf message-digest-key 1 md5 SecretKey` — configure the MD5 key on the link (key ID must match the neighbor)
- `area 0 authentication message-digest` — require MD5 authentication for the whole area

### OSPFv3 (IPv6)

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

### Verification

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

## 13. Access Control Lists (ACLs)

### Numbered standard ACL (filters by SOURCE only; 1–99, 1300–1999)

```
access-list 10 permit 192.168.10.0 0.0.0.255
access-list 10 deny host 192.168.20.5
access-list 10 permit any
```

- `access-list 10 permit 192.168.10.0 0.0.0.255` — permit the whole subnet (wildcard mask again)
- `access-list 10 deny host 192.168.20.5` — `host` = wildcard `0.0.0.0` (one address)
- `access-list 10 permit any` — allow everything else (**every ACL ends with an implicit deny all**)

### Numbered extended ACL (source, destination, protocol, ports; 100–199, 2000–2699)

```
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

### Named ACLs (editable, self-documenting — preferred)

```
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

### Applying ACLs

| Command | Mode | Explanation |
|---|---|---|
| `ip access-group 100 in` | Interface | Filter traffic entering this interface |
| `ip access-group BLOCK-WEB out` | Interface | Filter traffic leaving this interface |
| `access-class 10 in` | Line (vty) | Restrict who may SSH/Telnet to the device itself |

**Placement rules of thumb:** standard ACLs close to the *destination* (they can't see destinations); extended ACLs close to the *source* (kill unwanted traffic early). One ACL per interface, per direction, per protocol.

### IPv6 ACLs

```
ipv6 access-list BLOCK-V6
 deny tcp 2001:db8:acad:1::/64 any eq 23
 permit ipv6 any any
interface g0/0
 ipv6 traffic-filter BLOCK-V6 in
```

- `ipv6 access-list BLOCK-V6` — IPv6 ACLs are always named
- `ipv6 traffic-filter BLOCK-V6 in` — note: `traffic-filter`, not `access-group`

### Verification

| Command | Explanation |
|---|---|
| `show access-lists` | All ACLs with per-line match counters (see §18.9) |
| `show ip interface g0/0` | Shows which ACLs are applied where |
| `clear access-list counters` | Reset the hit counters |

---

## 14. NAT (Network Address Translation)

Terminology: **inside local** = private IP of a host; **inside global** = its public representation. The `ip nat inside`/`ip nat outside` interface marks are mandatory for any NAT to work.

### Mark the interfaces (required for every NAT type)

```
interface g0/0
 ip nat inside
interface g0/1
 ip nat outside
```

- `ip nat inside` — this interface faces the private LAN
- `ip nat outside` — this interface faces the Internet/ISP

### Static NAT (one-to-one; for servers that must be reachable from outside)

| Command | Explanation |
|---|---|
| `ip nat inside source static 192.168.10.5 203.0.113.5` | Permanently map private ↔ public |

### Dynamic NAT (pool of public addresses)

```
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat pool PUBLIC 203.0.113.10 203.0.113.20 netmask 255.255.255.0
ip nat inside source list 1 pool PUBLIC
```

- `access-list 1 permit 192.168.10.0 0.0.0.255` — define **who** gets translated
- `ip nat pool PUBLIC 203.0.113.10 203.0.113.20 netmask 255.255.255.0` — define the public address pool
- `ip nat inside source list 1 pool PUBLIC` — tie the ACL and pool together

### PAT / NAT overload (many-to-one; the typical home/office setup)

```
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface g0/1 overload
```

- `access-list 1 permit 192.168.10.0 0.0.0.255` — who gets translated
- `ip nat inside source list 1 interface g0/1 overload` — everyone shares the outside interface's IP, distinguished by source port. Alternative: `ip nat inside source list 1 pool PUBLIC overload` overloads onto a pool instead

### Port forwarding (static PAT)

| Command | Explanation |
|---|---|
| `ip nat inside source static tcp 192.168.10.5 80 203.0.113.5 8080` | Outside `:8080` → inside server `:80` |

### Verification

| Command | Explanation |
|---|---|
| `show ip nat translations` | The active translation table (see §18.8) |
| `show ip nat statistics` | Hit/miss counters, interface roles, pool usage |
| `clear ip nat translation *` | Flush dynamic translations (needed before changing NAT config) |
| `debug ip nat` | Watch translations happen live (lab use; `undebug all` to stop) |

---

## 15. First-Hop Redundancy — HSRP

Two routers share a virtual gateway IP so hosts survive a router failure.

```
interface g0/0
 ip address 192.168.10.2 255.255.255.0
 standby version 2
 standby 1 ip 192.168.10.1
 standby 1 priority 150
 standby 1 preempt
```

- `standby version 2` — HSRPv2 (supports more groups and IPv6)
- `standby 1 ip 192.168.10.1` — the virtual gateway IP that hosts point at
- `standby 1 priority 150` — higher priority becomes Active (default 100)
- `standby 1 preempt` — reclaim the Active role when this router comes back up

Verify with `show standby [brief]` — states: Active / Standby (see §18.12).

---

## 16. Logging, Monitoring & Housekeeping

| Command | Explanation |
|---|---|
| `logging host 10.1.1.100` | Send syslog messages to a server |
| `logging trap warnings` | Only send severity 4 (warnings) and worse |
| `service timestamps log datetime msec` | Timestamp log entries precisely |
| `logging buffered 16384` | Keep logs in a local RAM buffer |
| `show logging` | View the buffered logs |
| `terminal monitor` | Show log messages on your SSH session (console gets them by default) |
| `show processes cpu sorted` | CPU usage per process |
| `show memory statistics` | Memory usage |
| `show flash:` | Files on flash storage (IOS images) |
| `show tech-support` | Everything at once (for TAC cases) |

---

## 17. Low-Level Device Operations

### 17.1 File system navigation

IOS treats storage locations as file systems: `flash:` (IOS images, local files), `nvram:` (startup-config), `usbflash0:` (USB stick), plus network "file systems" like `tftp:`, `ftp:`, `scp:`, `http:`.

| Command | Explanation |
|---|---|
| `show file systems` | List all available file systems and free space |
| `dir flash:` | List files on flash (also just `dir`) |
| `dir usbflash0:` | List files on a USB stick |
| `dir nvram:` | See startup-config and private-config |
| `more flash:somefile.txt` | View a text file's contents |
| `copy flash:a.bin usbflash0:a.bin` | Copy between file systems |
| `delete flash:old-image.bin` | Delete a file |
| `mkdir flash:backups` | Create a directory |
| `verify flash:image.bin` | Check a file's embedded integrity hash |
| `verify /md5 flash:image.bin <hash>` | Compare against the MD5 from Cisco's download page |
| `fsck flash:` | Check/repair the file system |
| `format flash:` | Wipe a file system entirely (careful!) |

### 17.2 Firmware (IOS) upgrade

**Procedure:** check space → copy the new image → verify its hash → point the boot variable at it → save → reload → confirm.

```
show version
show flash:
copy tftp: flash:
verify /md5 flash:new-image.bin
configure terminal
 no boot system
 boot system flash:new-image.bin
 boot system flash:old-image.bin
 end
copy running-config startup-config
reload
show version
```

- `show version` — note the current IOS version and which file booted (`System image file is...`)
- `show flash:` — confirm there's enough free space for the new image
- `copy tftp: flash:` — pull the image from a TFTP server (prompts for server IP + filename). Alternatives: `copy usbflash0:new-image.bin flash:` from USB, or `copy ftp://user:pass@10.1.1.50/new-image.bin flash:` via FTP (handles big files better than TFTP)
- `verify /md5 flash:new-image.bin` — **always** verify against Cisco's published MD5 before booting it
- `no boot system` — clear any old boot statements first
- `boot system flash:new-image.bin` — boot the new image first...
- `boot system flash:old-image.bin` — ...and keep the old one as an automatic fallback
- `copy running-config startup-config` — boot statements live in the config; save or they're lost
- `reload` — reboot into the new image
- `show version` — confirm the new version is running, then optionally delete the old image

On newer platforms in **install mode** (Catalyst 9k, ISR/Cat8k):

| Command | Explanation |
|---|---|
| `install add file flash:image.bin activate commit` | One-shot: unpack, activate, and commit the new image |
| `install remove inactive` | Clean up superseded packages |
| `show install summary` | Which image is committed/active |

**ROMMON rescue** — if the device won't boot (bad/deleted image), from the `rommon >` prompt:

```
dir flash:
boot flash:known-good.bin
```

- `dir flash:` — see what images exist
- `boot flash:known-good.bin` — boot a specific image manually

Or stage a TFTP recovery from ROMMON:

```
IP_ADDRESS=10.1.1.2
IP_SUBNET_MASK=255.255.255.0
DEFAULT_GATEWAY=10.1.1.1
TFTP_SERVER=10.1.1.50
TFTP_FILE=new-image.bin
tftpdnld
```

- The five `VARIABLE=value` lines give ROMMON just enough IP configuration to reach the TFTP server
- `tftpdnld` — download the image straight into flash

### 17.3 Configuration management — backup & restore

| Command | Explanation |
|---|---|
| `copy running-config startup-config` | Save active config to NVRAM (the everyday "save") |
| `copy running-config tftp:` | Back up to a TFTP server (prompts for IP + filename) |
| `copy running-config usbflash0:R1-backup.cfg` | Back up to USB |
| `copy running-config scp://admin@10.1.1.50/R1.cfg` | Back up over SSH (encrypted — best practice) |
| `copy startup-config tftp:` | Back up the saved config instead of the running one |
| `copy tftp: running-config` | Restore: **merges** the file into the running config (doesn't erase extras!) |
| `copy tftp: startup-config` | Restore into NVRAM, then `reload` for a clean, exact restore |
| `configure replace flash:R1-backup.cfg` | **True restore**: makes running-config exactly match the file (adds AND removes) |

**Merge vs replace:** copying into running-config only *adds/overwrites* lines — commands present on the device but absent from the backup survive. Use `configure replace` (or restore to startup-config + reload) when you need the device to match the backup exactly.

#### Archive & rollback (automatic config versioning)

```
configure terminal
 archive
  path flash:backups/R1-config
  maximum 10
  write-memory
  time-period 1440
 exit
 archive log config
  logging enable
  notify syslog
```

- `archive` — enter the archive configuration
- `path flash:backups/R1-config` — where to store versions (appends `-1`, `-2`, ...)
- `maximum 10` — keep the last 10 versions
- `write-memory` — auto-archive every time someone saves
- `time-period 1440` — also archive every 24 h (value in minutes)
- `archive log config` + `logging enable` — log every configuration command entered, with username
- `notify syslog` — send those entries to syslog too

| Command | Explanation |
|---|---|
| `show archive` | List stored versions |
| `show archive log config all` | Who changed what, and when |
| `configure replace flash:backups/R1-config-3` | Roll back to version 3 |

#### Safe remote changes (don't lock yourself out)

| Command | Explanation |
|---|---|
| `configure terminal revert timer 5` | Auto-rollback to the saved config in 5 min unless confirmed |
| `configure confirm` | "I'm still in — keep my changes" (cancels the revert timer) |

### 17.4 Reboot & scheduled reloads

| Command | Explanation |
|---|---|
| `reload` | Reboot now (warns if the config is unsaved) |
| `reload in 10` | Reboot in 10 minutes — schedule this **before** risky remote changes: if you cut yourself off, the reboot restores the saved config |
| `reload at 02:00 14 July` | Reboot at a specific time (requires a correct clock) |
| `reload cancel` | Made it through? Cancel the pending reload |
| `show reload` | See any scheduled reload |

### 17.5 Password reset / recovery (router)

Requires physical console access — this is why console ports must be physically secured.

1. Power-cycle the router and send the **Break** sequence during the first ~60 s of boot to land in ROMMON.
2. `confreg 0x2142` — sets the config register to *ignore startup-config* at boot.
3. `reset` — reboot; the router comes up "blank". Answer **no** to the setup dialog.
4. `enable` → `copy startup-config running-config` — load the real config *without* having authenticated through it. (Note: interfaces come back shut down — `no shutdown` them.)
5. Change the credentials: `enable secret NewPass`, fix `username` lines, etc.
6. `config-register 0x2102` in config mode — restore normal boot behavior. **Forgetting this = the router boots blank forever.**
7. `copy running-config startup-config`, then `reload`. Verify with `show version` (config register shown on the last line).

**Catalyst switch variant** — hold the **Mode** button while powering on to reach the `switch:` prompt, then:

```
flash_init
rename flash:config.text flash:config.old
boot
```

- `flash_init` — initialize the flash file system
- `rename flash:config.text flash:config.old` — hide the config from the boot process
- `boot` — boot without a config; afterwards, rename the file back, `copy` it to running-config, change the passwords, and save

To defeat this procedure on stolen devices: `no service password-recovery` (drastic — recovery then requires wiping the config).

### 17.6 Factory reset / configuration reset

```
erase startup-config
delete vlan.dat
reload
```

- `erase startup-config` — wipe the saved configuration from NVRAM (`write erase` also works)
- `delete vlan.dat` — **switches only**: the VLAN database lives outside the config and must be deleted separately
- `reload` — reboot; the device comes up unconfigured (say **no** to the setup wizard)

Newer IOS-XE platforms have a genuine one-command wipe:

| Command | Explanation |
|---|---|
| `factory-reset all` | Erase configs, files, logs, and crash data (keeps the boot image) |
| `factory-reset all secure` | Same, with secure (unrecoverable) erase — for device disposal |

**Partial resets:**

| Command | Explanation |
|---|---|
| `default interface g0/1` | Reset **one** interface to factory defaults |
| `configure replace flash:golden.cfg` | Reset to a known-good baseline instead of to nothing |
| `clear ip nat translation *` | Clear the NAT table without touching config |
| `clear mac address-table dynamic` | Flush learned MACs |
| `clear counters` | Zero interface statistics (great before troubleshooting) |

### 17.7 Licensing (quick reference)

| Command | Explanation |
|---|---|
| `show license status` | Smart Licensing state (modern IOS-XE) |
| `show license summary` | Which licenses are in use |
| `license boot level network-advantage` | Set the license boot level (reload required) |

---

## 18. Reading Command Output — Annotated Examples

Realistic output from the most important verification commands, with explanations of what to look for.

### 18.1 `show ip route`

```
R1# show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       * - candidate default, U - per-user static route

Gateway of last resort is 203.0.113.1 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 203.0.113.1
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.1.0/30 is directly connected, GigabitEthernet0/1
L        10.1.1.1/32 is directly connected, GigabitEthernet0/1
O     192.168.20.0/24 [110/2] via 10.1.1.2, 00:12:45, GigabitEthernet0/1
S     192.168.30.0/24 [1/0] via 10.1.1.2
      192.168.10.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.10.0/24 is directly connected, GigabitEthernet0/0
L        192.168.10.1/32 is directly connected, GigabitEthernet0/0
```

**How to read it:**
- **The letter code** is the route's source: `C` = directly connected network, `L` = the router's own address on that link (always /32), `S` = static, `O` = OSPF-learned, `S*` = a static route that is also the **default** route.
- `Gateway of last resort is 203.0.113.1` — a default route exists; anything not matching a more specific route goes there. If this says *"is not set"*, unknown destinations are dropped.
- `[110/2]` = **[administrative distance / metric]**. AD 110 identifies OSPF; metric 2 is the OSPF cost of the path. When two sources offer the same prefix, lower AD wins; between routes from the same protocol, lower metric wins.
- `via 10.1.1.2` — the next-hop address; `00:12:45` — how long the route has been stable (a young timer on an "old" route hints at flapping); `GigabitEthernet0/1` — the exit interface.
- **Longest prefix wins:** a packet to 192.168.20.5 matches `192.168.20.0/24` (specific) over `0.0.0.0/0` (default), regardless of AD or metric.

### 18.2 `show ip interface brief`

```
R1# show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     192.168.10.1    YES manual up                    up
GigabitEthernet0/1     10.1.1.1        YES manual up                    up
GigabitEthernet0/2     unassigned      YES unset  administratively down down
Serial0/0/0            203.0.113.2     YES DHCP   up                    down
Vlan1                  unassigned      YES unset  down                  down
```

**How to read it — the last two columns tell the story:**

| Status | Protocol | Meaning |
|---|---|---|
| up | up | Working normally |
| administratively down | down | Someone typed `shutdown` — fix with `no shutdown` |
| up | down | Physical layer OK, data-link broken: clock/encapsulation mismatch, keepalives failing, far end misconfigured |
| down | down | No signal at all: cable unplugged/bad, far-end device off, wrong port |

- **Method** shows where the IP came from: `manual` (typed), `DHCP`, `unset` (none).

### 18.3 `show interfaces g0/0` (key excerpts)

```
GigabitEthernet0/0 is up, line protocol is up
  Hardware is iGbE, address is 0c1a.2b3c.4d00
  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec,
  Full-duplex, 1000Mb/s, media type is RJ45
  Last input 00:00:02, output 00:00:01
  5 minute input rate 2000 bits/sec, 3 packets/sec
     10233 packets input, 1523340 bytes
     0 input errors, 0 CRC, 0 frame, 0 overrun
     8811 packets output, 923412 bytes
     0 output errors, 0 collisions, 2 interface resets
     0 late collision, 0 deferred
```

**What to look for:**
- `Full-duplex, 1000Mb/s` — verify both ends agree. A **duplex mismatch** classically shows as *CRC/late collisions* on the half-duplex side and runts on the other.
- `input errors` / `CRC` climbing → bad cable, bad SFP, or duplex mismatch. Re-run the command twice; what matters is whether counters are *increasing*, not their absolute value (use `clear counters` first).
- `output drops` (in the queue line, not shown here) → congestion: traffic exceeds the interface's capacity.
- `BW 1000000 Kbit/sec` — the bandwidth value OSPF/EIGRP use for metrics; it can be set with the `bandwidth` command without changing real speed.

### 18.4 `show vlan brief`

```
S1# show vlan brief
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi0/3, Gi1/0, Gi1/1, Gi1/2
10   SALES                            active    Fa0/5, Fa0/6
20   ENGINEERING                      active    Fa0/7, Fa0/8
99   MANAGEMENT                       active    
1002 fddi-default                     act/unsup
```

**How to read it:**
- Only **access** ports are listed — trunks don't appear here (they carry all VLANs; see `show interfaces trunk`).
- A host in the wrong VLAN? This is the first place to look: find its port in the wrong row.
- A VLAN that exists but shows no ports (like 99 here) is fine for SVI-only/management VLANs.
- VLANs 1002–1005 are legacy FDDI/Token-Ring placeholders — ignore them; they can't be deleted.

### 18.5 `show interfaces trunk`

```
S1# show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Gi0/1       on           802.1q         trunking      99

Port        Vlans allowed on trunk
Gi0/1       10,20,99

Port        Vlans allowed and active in management domain
Gi0/1       10,20,99

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/1       10,20
```

**How to read it:**
- Mode `on` = statically configured trunk; `desirable`/`auto` mean DTP negotiated it (harden this).
- `Native vlan 99` — must match on both ends or you'll see CDP "native VLAN mismatch" log messages and inter-VLAN leakage.
- The three VLAN lists narrow down: *allowed* (your config) → *active* (VLAN actually exists) → *forwarding* (STP isn't blocking it). **A VLAN missing from a lower list than the one above it is your fault line:** allowed but not active = VLAN not created; active but not forwarding = STP blocking or pruned (here VLAN 99 is not forwarding — it has no member ports, which is normal).

### 18.6 `show spanning-tree vlan 10`

```
S2# show spanning-tree vlan 10
VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    24586
             Address     0c1a.aaaa.0001
             Cost        4
             Port        25 (GigabitEthernet0/1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32778  (priority 32768 sys-id-ext 10)
             Address     0c1a.bbbb.0002

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- ------------------------
Gi0/1               Root FWD 4         128.25   P2p
Gi0/2               Altn BLK 4         128.26   P2p
Fa0/5               Desg FWD 19        128.7    P2p Edge
```

**How to read it:**
- **Root ID vs Bridge ID:** Root ID is the elected root bridge; Bridge ID is *this* switch. If they showed the same MAC, you'd see `This bridge is the root` instead of a Cost/Port line. Here, this switch is NOT the root.
- Root priority `24586` = 24576 + VLAN 10 → someone configured `spanning-tree vlan 10 root primary`. This switch's `32778` = default 32768 + 10.
- **Port roles:** `Root`/`FWD` = best path toward the root; `Desg`/`FWD` = designated (forwarding) for its segment; `Altn`/`BLK` = the **blocked redundant link** — this is the loop prevention working. If a "missing bandwidth" mystery arises, this blocked port is usually why.
- `P2p Edge` on Fa0/5 = PortFast is active (host port, skips listening/learning).

### 18.7 `show ip ospf neighbor`

```
R1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           1   FULL/DR         00:00:35    10.1.1.2        GigabitEthernet0/1
3.3.3.3           1   FULL/BDR        00:00:38    10.1.1.3        GigabitEthernet0/1
4.4.4.4           0   2WAY/DROTHER    00:00:31    10.1.1.4        GigabitEthernet0/1
5.5.5.5           1   FULL/  -        00:00:33    10.2.2.2        Serial0/0/0
```

**How to read it:**
- `FULL` = healthy adjacency, databases synchronized. `FULL/ -` (dash) = point-to-point link, where no DR/BDR is elected — also healthy.
- `2WAY/DROTHER` is **normal, not broken**, on a multi-access segment: two non-DR routers stay at 2WAY with each other and only go FULL with the DR and BDR. (Pri `0` = that router refuses to be DR.)
- **Genuinely broken states:** stuck in `INIT` = your hellos reach them, theirs don't return (ACL? unidirectional link?); stuck in `EXSTART`/`EXCHANGE` = classic **MTU mismatch**; neighbor missing entirely = subnet/area/timer/authentication mismatch or `passive-interface`.
- **Dead Time** counts down from the dead interval and resets with each hello — it should hover, never reach zero.

### 18.8 `show ip nat translations`

```
R1# show ip nat translations
Pro  Inside global       Inside local        Outside local       Outside global
tcp  203.0.113.5:34122   192.168.10.11:34122 198.51.100.7:443    198.51.100.7:443
tcp  203.0.113.5:51920   192.168.10.42:51920 93.184.216.34:80    93.184.216.34:80
icmp 203.0.113.5:1       192.168.10.11:1     8.8.8.8:1           8.8.8.8:1
---  203.0.113.9         192.168.10.5        ---                 ---
```

**How to read it (each row left→right is "outside view → inside truth"):**
- **Inside local** = the host's real private IP; **inside global** = what the outside world sees. Rows sharing one inside-global IP with different ports (first three rows) = **PAT/overload** at work.
- The last row with no protocol/ports and no outside addresses = a **static NAT** entry — it exists permanently, even with no traffic.
- Empty table when you expect translations? Check: interfaces marked `ip nat inside`/`ip nat outside`, the ACL actually matches the source, and remember translations only appear when traffic flows.

### 18.9 `show access-lists`

```
R1# show access-lists
Extended IP access list BLOCK-WEB
    10 deny tcp 192.168.10.0 0.0.0.255 any eq www (152 matches)
    15 deny tcp 192.168.10.0 0.0.0.255 any eq 443 (89 matches)
    20 permit ip any any (10334 matches)
```

**How to read it:**
- `(152 matches)` — the counters are the whole point: they prove the line is actually catching traffic. A rule you expect to fire showing **no counter at all** = never matched — likely ordered below a broader line that grabs the traffic first, applied to the wrong interface/direction, or the traffic isn't what you think.
- Matches only count on the interface(s) where the ACL is applied, and the implicit deny at the end has **no counter** — add an explicit `deny ip any any log` line to see what's being silently dropped.

### 18.10 `show port-security interface f0/5`

```
S1# show port-security interface f0/5
Port Security              : Enabled
Port Status                : Secure-shutdown
Violation Mode             : Shutdown
Maximum MAC Addresses      : 2
Total MAC Addresses        : 2
Sticky MAC Addresses       : 2
Last Source Address:Vlan   : 000c.dead.beef:10
Security Violation Count   : 1
```

**How to read it:**
- `Secure-shutdown` = the port is **err-disabled** right now — a violation occurred. (Healthy states: `Secure-up`; `Secure-down` just means the port is down/idle.)
- `Last Source Address` = the MAC that triggered the violation — your evidence of *what* was plugged in.
- Recover: fix the physical issue, then `shutdown` / `no shutdown` on the interface (the violation count survives; clear with `clear port-security sticky`).

### 18.11 `show etherchannel summary`

```
S1# show etherchannel summary
Flags:  D - down        P - bundled in port-channel
        S - Layer2      U - in use      s - suspended
        I - stand-alone

Group  Port-channel  Protocol    Ports
------+-------------+-----------+----------------------------
1      Po1(SU)         LACP      Gi0/1(P)  Gi0/2(P)
2      Po2(SD)         LACP      Gi0/3(I)  Gi0/4(s)
```

**How to read it:**
- `Po1(SU)` = Layer-2 channel, **U**p and in use, with both members `(P)` bundled — healthy.
- `Po2(SD)` is down: `Gi0/3(I)` = operating stand-alone (LACP got no response — is the far side configured?), `Gi0/4(s)` = **suspended**, almost always a **member mismatch** (VLAN list, speed, duplex, or mode differs from the channel). Fix the member config or the far end.

### 18.12 `show standby brief` (HSRP)

```
R1# show standby brief
                     P indicates configured to preempt.
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Gi0/0       1    150 P Active  local           192.168.10.3    192.168.10.1
```

**How to read it:**
- State `Active` + Active = `local` → this router currently owns the virtual IP 192.168.10.1. The peer at `.3` is `Standby`, ready to take over.
- `P` = preempt is on — after recovering from a failure, this higher-priority (150) router will reclaim the Active role. Without preempt it would wait as Standby forever.
- Both routers showing `Active` = **split brain**: they can't hear each other's hellos — check the link/VLAN between them.

---

## Quick "is it working?" checklist

| Question | Command |
|---|---|
| Are my interfaces up? | `show ip interface brief` |
| Can I reach it? | `ping` / `traceroute` |
| What path is used? | `show ip route <dest>` |
| L2: right VLAN? | `show vlan brief`, `show interfaces trunk` |
| STP blocking something? | `show spanning-tree` |
| OSPF neighbors up? | `show ip ospf neighbor` |
| DHCP leasing? | `show ip dhcp binding` |
| NAT translating? | `show ip nat translations` |
| ACL dropping traffic? | `show access-lists` (watch counters) |
| Who's plugged in where? | `show mac address-table`, `show cdp neighbors` |
