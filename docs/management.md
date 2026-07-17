# Network Management & Monitoring

## M.1 Device discovery — CDP & LLDP

## 1. Cisco Discovery Protocol (CDP)
A media- and protocol-independent device-discovery protocol that runs on most Cisco-manufactured equipment, including routers, access servers, and switches. 
Using CDP, a device can advertise its existence to other devices and receive information about other devices on the same LAN or on the remote side of a WAN.

| Command | Explanation |
|---|---|
| `[no] cdp run` | Global command that enables and disables (with the no option) CDP for the entire switch or router. |
| `[no] cdp enable` | Interface subcommand to enable and disable (with the no option) CDP for a particular interface. |

## 2. Link Layer Discovery Protocol (LLDP)
An IEEE standard protocol (IEEE 802.1AB) that defines messages, encapsulated directly in Ethernet frames so they do not rely on a working IPv4 or IPv6 network, for the purpose of giving devices a means of announcing basic device information to other devices on the LAN. 
It is a standardized protocol similar to Cisco Discovery Protocol (CDP).

| Command | Explanation |
|---|---|
| `[no] lldp run` | Global command to enable and disable (with the no option) LLDP for the entire switch or router. |


| Protocol | Vendor | Purpose |
|---|---|---|
| **CDP** (Cisco Discovery Protocol) | Cisco proprietary | Periodic **Layer 2** advertisements between directly connected Cisco devices: **device name, IOS version, platform, number/type of interfaces**, management address |
| **LLDP** (Link Layer Discovery Protocol) | **Vendor-neutral** (IEEE 802.1AB) | Same idea for **multi-vendor** networks — advertises identity/capabilities to connected L2 neighbors |

```text linenums="1"
show cdp neighbors [detail]     ! discovered Cisco neighbors (detail adds IP/IOS)
cdp run / no cdp run            ! enable/disable CDP globally
show lldp neighbors [detail]    ! vendor-neutral equivalent
lldp run                        ! LLDP is OFF by default; enable it globally
```

## M.2 NTP — time synchronization

Consistent clocks are essential for correlating **log timestamps** during troubleshooting. NTP syncs a device's software clock to a **private master clock** or a **public Internet NTP server**; sources are ranked by **stratum** (lower = closer to the authoritative time source).

```text linenums="1"
ntp server 10.1.1.100           ! sync this device to an NTP server
ntp master 3                    ! act as an NTP master at stratum 3 (lab/private clock)
show ntp associations [detail]  ! peers; the master's IP is shown here
show ntp status                 ! sync state and stratum
```

| Command | Explanation |
|---|---|
| `clock timezone <name> +/– <hours-offset> <[minutes-offset]>` | Global command that names a time zone and defines the +/– offset versus UTC.|
| `clock summertime <name> recurring` | Global command that names a daylight savings time for a time zone and tells IOS to adjust the clock automatically. |
| `ntp server <address> / <hostname>` | Global command that configures the device as an NTP client by referring to the address or name of an NTP server.|
| `ntp master <stratum-level>` | Global command that configures the device as an NTP server and assigns its local clock stratum level.|
| `ntp source <name/number>` | Global command that tells NTP to use the listed interface (by name/number) for the source IP address for NTP messages.|

- Reading `show ntp associations`: the entry marked as the **synced peer** identifies the **master's IP**; the client is the router running the command.

## M.3 Syslog — centralized logging

A **syslog server** keeps a **historical record** of messages from monitored devices. Severity runs **0 (emergency) → 7 (debugging)**; `logging trap <level>` sends that level and worse.

```text linenums="1"
logging host 10.1.1.100
logging trap warnings           ! severity 4 and more severe
service timestamps log datetime msec
show logging
```

| Command | Explanation |
|---|---|
| `[no] logging console` | Global command that enables (or disables with the no option) logging to the console device.|
| `[no] logging monitor` | Global command that enables (or disables with the no option) logging to users connected to the device with SSH or Telnet. |
| `[no] logging buffered` | Global command that enables (or disables with the no option) logging to an internal buffer. |
| `logging [host] <ip-address> / <hostname>` | Global command that enables logging to a syslog server. |
| `logging console <level-name> / <level-number>` | Global command that sets the log message level for console log messages.|
| `logging monitor <level-name> / <level-number>` | Global command that sets the log message level for log messages sent to SSH and Telnet users.|
| `logging buffered <level-name> / <level-number>` | Global command that sets the log message level for buffered log messages displayed later by the `show logging` command. |
| `logging trap <level-name> / <level-number>` | Global command that sets the log message level for messages sent to syslog servers.|
| `[no] service sequence-numbers` | Global command to enable or disable (with the no option) the use of sequence numbers in log messages. |

## M.4 SNMP — Simple Network Management Protocol

An SNMP system has **three elements**: the **SNMP manager** (NMS software), **agents** (software on each managed device that collects/stores local data), and the **MIB** (the information database of objects on the device).

- **Polling**: the manager queries agents periodically.
- **Traps**: **unsolicited** agent→manager messages fired when an event occurs — they **eliminate some periodic polling** and **reduce load** on network and agent resources (the two Q66 benefits).

**SNMPv3 security levels** (each stronger than the last):

| Level | Authentication | Encryption |
|---|---|---|
| **noAuth** | Username/community-string match only | None |
| **auth** | **HMAC with MD5 or SHA** | None |
| **priv** | HMAC MD5/SHA | **+ DES/3DES/AES encryption** |

## M.5 Network documentation & baselines

- A **network baseline** establishes **normal** performance to serve as a **reference point for future evaluations** (detecting anomalies) — not merely a statistical average.
- **Logical topology diagram** includes: **interface identifiers** and **connection types** (LAN/WAN/point-to-point).
- **Physical topology diagram** includes: device type, cable type/identifier & specification, OS/IOS version.




## Verification

| Command | Explanation |
|---|---|
| `show cdp neighbors <[type number]>` | Lists one summary line of information about each neighbor; optionally, lists neighbors off the listed interface|
| `show lldp neighbors <[type number]>` | Lists one summary line of information about each neighbor; optionally, lists neighbors off the listed interface|
| `show cdp neighbors detail` | Lists one large set of information (approximately 15 lines) for every neighbor |
| `show lldp neighbors detail` | Lists one large set of information (approximately 15 lines) for every neighbor |
| `show cdp interface <[type number]>` | States whether CDP is enabled on each interface or a single interface if the interface is listed |
| `show lldp interface <[type number]>` | States whether LLDP is enabled on each interface or a single interface if the interface is listed |
| `show cdp traffic` | Displays global statistics for the number of CDP advertisements sent and received|
| `show lldp traffic` | Displays global statistics for the number of LLDP advertisements sent and received|
| `show logging` | Lists the current logging configuration and lists buffered log messages at the end |
| `terminal [no] monitor` | For a user (SSH or Telnet) session, toggles on or off the receipt of log messages, for that one session, if `logging monitor` is also configured|
| `[no] debug <{various}>` | EXEC command to enable or disable one of a multitude of debug options |
| `show clock` | Lists the time of day and the date per the local device |
| `show ntp associations` | Shows all NTP clients and servers with which the local device is attempting to synchronize with NTP |
| `show ntp status` | Shows current NTP client status in detail|


## Network Management Resources
[LibreNMS](https://www.librenms.org/#downloads)