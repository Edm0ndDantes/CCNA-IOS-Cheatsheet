# Network Security & Hardening

## Network Security Fundamentals

### S.1 Malware types

| Type | Defining trait |
|---|---|
| **Virus** | Malicious code that **requires end-user activation** (open a file/app, power on); can lie **dormant** then trigger on a date; spreads by infecting other files |
| **Worm** | Self-replicates and spreads **independently**, exploiting vulnerabilities with no user action |
| **Trojan horse** | Malware hidden inside a **seemingly legitimate** program |
| **Botnet / zombies** | A network of infected hosts ("zombies") remotely **controlled by an attacker** to attack others or steal data |
| **Spyware / rootkit** | Covertly gathers data / hides deep in the OS to retain privileged access |

*(Note: the courseware also lists "enabling vulnerability + propagation mechanism + payload" as the anatomy of a **worm**, not a virus.)*

### S.2 Attack categories

- **Reconnaissance** — information gathering: **scan for accessibility**, map devices/services (precursor to an attack).
- **Access** — gain/escalate unauthorized access, **retrieve or modify data**, privilege escalation.
- **DoS / DDoS** — overwhelm a host/interface with **extreme volumes of data** to deny service (DDoS uses a botnet).

### S.3 Specific L2/infrastructure attacks

| Attack | Mechanism | Mitigation |
|---|---|---|
| **DHCP spoofing** | Rogue DHCP server hands clients a **false default gateway** to intercept traffic | **DHCP snooping** |
| **ARP cache poisoning** | Forged ARP maps attacker's MAC to a legitimate IP | Dynamic ARP Inspection |
| **DNS attacks on open resolvers** | **Amplification & reflection** (mask/inflate the attack), and **resource-utilization** DoS; also **cache poisoning** | Restrict/secure resolvers |
| **TCP SYN flood** | Half-open connections exhaust the server | — |

### S.4 Penetration-testing tool categories

| Tool type | Purpose | Examples |
|---|---|---|
| **Password crackers** | Repeatedly **guess/crack passwords** | John the Ripper, THC Hydra, RainbowCrack, Medusa |
| **Packet sniffers** | **Capture & analyze** LAN/WLAN traffic | Wireshark, tcpdump |
| **Debuggers** | **Reverse-engineer binaries** to write exploits / analyze malware | GDB, WinDbg, IDA Pro |
| Fuzzers / packet crafters | Probe robustness with forged packets | — |

### S.5 The four goals of secure communication

| Goal | Guarantees |
|---|---|
| **Confidentiality** | Only authorized parties can read it — via **encryption** |
| **Integrity** | The message wasn't altered — via a **hash** (MD5, SHA) |
| **Origin authentication** | It genuinely came from the claimed sender — hash recalculated with a **predetermined/secret key** (HMAC) |
| **Non-repudiation** | The sender can't later deny sending it |

### S.6 Symmetric vs asymmetric keys

- **Symmetric** — one shared secret key encrypts and decrypts (fast; AES, 3DES). Used for **bulk data confidentiality**.
- **Asymmetric** — a **public/private key pair**; the *complementary* key decrypts:
  - **Public key encrypts → private key decrypts** (confidentiality to the key owner)
  - **Private key encrypts → public key decrypts** (origin authentication / signatures)
  - Keys are **not** interchangeable — you can't decrypt with the same key that encrypted.
- **Common algorithm roles** (as IPsec uses them): **AES** = encryption (confidentiality); **MD5 / SHA** = hashing (integrity); **DH** = key exchange; **RSA** = authentication.

## Security Hardening

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

``` linenums="1"
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

``` linenums="1"
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

``` linenums="1"
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

``` linenums="1"
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

``` linenums="1"
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

``` linenums="1"
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

``` linenums="1"
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