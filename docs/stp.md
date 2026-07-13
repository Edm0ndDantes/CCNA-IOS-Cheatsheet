# Spanning Tree Protocol (STP)

STP prevents Layer-2 loops by electing a root bridge and blocking redundant paths.

## Global configuration

| Command | Explanation |
|---|---|
| `spanning-tree mode rapid-pvst` | Use Rapid PVST+ (fast convergence; recommended). Others: `pvst`, `mst` |
| `spanning-tree vlan 10 root primary` | Make this switch the root bridge for VLAN 10 (sets priority to 24576 or lower) |
| `spanning-tree vlan 10 root secondary` | Backup root (priority 28672) |
| `spanning-tree vlan 10 priority 4096` | Or set the priority manually (multiple of 4096; lower wins) |
| `spanning-tree portfast default` | PortFast on all access ports at once |
| `spanning-tree portfast bpduguard default` | BPDU Guard on all PortFast ports |

## Per-interface features

| Command | Explanation |
|---|---|
| `spanning-tree portfast` | Skip listening/learning on an access port — host ports come up instantly |
| `spanning-tree bpduguard enable` | Err-disable the port if a BPDU arrives (a switch was plugged into a host port) |
| `spanning-tree guard root` | Root Guard: block a downstream switch from becoming root via this port |
| `spanning-tree cost 10` | Manipulate path cost (lower preferred) |
| `spanning-tree port-priority 64` | Tie-breaker between equal-cost ports (lower wins, multiple of 32) |

## Verification

| Command | Explanation |
|---|---|
| `show spanning-tree` | Root bridge, port roles/states for every VLAN (see §18.6) |
| `show spanning-tree vlan 10` | Same, for one VLAN |
| `show spanning-tree summary` | Modes and feature status (PortFast, BPDU Guard, etc.) |

**Reading the output:** the root bridge shows `This bridge is the root`. Port roles: Root (best path to root), Designated (forwarding), Alternate/Blocked.

---