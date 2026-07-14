## EtherChannel (Link Aggregation)

Bundles multiple physical links into one logical link (more bandwidth, no STP blocking between them).

``` linenums="1"
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