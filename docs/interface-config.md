## Router interface (routed port)

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

## Switch management IP (SVI)

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

# Loopback interface

```
interface loopback 0
 ip address 10.0.0.1 255.255.255.255
```

- `interface loopback 0` — virtual interface, always up; used for router IDs, testing, management
- `ip address 10.0.0.1 255.255.255.255` — typically a /32 host route

---