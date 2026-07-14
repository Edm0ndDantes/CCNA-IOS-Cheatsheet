# Basic Device Configuration

## Global settings

| Command | Explanation |
|---|---|
| `hostname R1` | Set the device name (shows in prompt) |
| `banner motd #Unauthorized access prohibited!#` | Message-of-the-day banner shown at login; `#` is the delimiter |
| `no ip domain-lookup` | Stop the device trying to DNS-resolve mistyped commands (prevents annoying hangs) |
| `clock set 14:30:00 13 July 2026` | Set the clock manually (privileged EXEC, not config mode) |
| `ntp server 192.168.1.100` | Sync the clock from an NTP server instead |

## Console line setup

``` linenums="1"
line console 0
 password cisco
 login
 logging synchronous
 exec-timeout 5 0
```

- `line console 0` — enter console line configuration mode
- `password cisco` — set the console password
- `login` — require the password at login (without this, the password is ignored)
- `logging synchronous` — stop log messages from interrupting what you're typing
- `exec-timeout 5 0` — log out idle sessions after 5 minutes 0 seconds (`0 0` = never, lab only)

## VTY lines setup (Telnet/SSH)

``` linenums="1"
line vty 0 4
 password cisco
 login
 transport input ssh
```

- `line vty 0 4` — configure virtual terminal lines 0–4 (remote access sessions)
- `password cisco` — password for remote login
- `login` — require the password (use `login local` for username-based authentication)
- `transport input ssh` — allow only SSH (options: `telnet`, `ssh`, `all`, `none`)

## Saving and managing configuration

| Command | Explanation |
|---|---|
| `copy running-config startup-config` | Save the active config to NVRAM (survives reboot); `wr` also works |
| `show running-config` | View the active configuration in RAM |
| `show startup-config` | View the saved configuration in NVRAM |
| `erase startup-config` | Delete the saved config (factory reset step 1) |
| `reload` | Reboot the device |

## Key verification commands

| Command | Explanation |
|---|---|
| `show ip interface brief` | One-line summary of every interface: IP, status, protocol |
| `show interfaces g0/0` | Detailed stats: errors, duplex, speed, drops |
| `show version` | IOS version, uptime, hardware, config register |
| `show protocols` | Interfaces with their IP addresses and status |
| `show cdp neighbors [detail]` | Discover directly connected Cisco devices (`detail` adds IPs/IOS) |
| `show lldp neighbors` | Vendor-neutral equivalent of CDP |
| `show mac address-table` | (Switch) Learned MAC addresses and their ports |
| `show arp` | IP-to-MAC bindings the device has learned |

---