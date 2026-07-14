## Creating VLANs

``` linenums="1"
vlan 10
 name SALES
vlan 20
 name ENGINEERING
vlan 99
 name MANAGEMENT
```

- `vlan 10` — create VLAN 10 and enter VLAN configuration mode
- `name SALES` — assign a human-readable name

## Access ports (end devices)

``` linenums="1"
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

## Trunk ports (switch-to-switch / switch-to-router)

``` linenums="1"
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

## DTP (Dynamic Trunking Protocol) modes

| Command | Explanation |
|---|---|
| `switchport mode dynamic desirable` | Actively tries to form a trunk |
| `switchport mode dynamic auto` | Forms a trunk only if the other side asks |

**Best practice:** hard-code `switchport mode access` or `switchport mode trunk` + `switchport nonegotiate`; never rely on DTP (VLAN-hopping risk).

## Verification & cleanup

| Command | Explanation |
|---|---|
| `show vlan brief` | VLANs and which ports belong to each (see §18.4) |
| `show interfaces trunk` | Trunk ports, native VLANs, allowed VLANs (see §18.5) |
| `show interfaces f0/5 switchport` | Full L2 details for one port |
| `no vlan 10` | Delete VLAN 10 |
| `delete vlan.dat` | Erase the VLAN database (factory reset step 2 on switches) |

---

## Inter-VLAN Routing

### Router-on-a-stick (subinterfaces)

``` linenums="1"
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

``` linenums="1"
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