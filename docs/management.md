# Management and Monitoring

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

## 3. Logging
| Command | Explanation |
|---|---|
| `[no] logging console` | Global command that enables (or disables with the no option) logging to the console device.|
| `[no] logging monitor` | Global command that enables (or disables with the no option) logging to users connected to the device with SSH or Telnet. |
| `[no] logging buffered` | Global command that enables (or disables with the no option) logging to an internal buffer. |
| `logging [host] <ip-address> / hostname` | Global command that enables logging to a syslog server. |
| `logging console level-name / level-number` | Global command that sets the log message level for console log messages.|
| `logging monitor level-name / level-number` | Global command that sets the log message level for log messages sent to SSH and Telnet users.|
| `logging buffered level-name / level-number` | Global command that sets the log message level for buffered log messages displayed later by the `show logging` command. |
| `logging trap level-name / level-number` | Global command that sets the log message level for messages sent to syslog servers.|
| `[no] service sequence-numbers` | Global command to enable or disable (with the no option) the use of sequence numbers in log messages. |

## 4. Time
| Command | Explanation |
|---|---|
| `clock timezone name +/– hours-offset [minutes-offset]` | Global command that names a time zone and defines the +/– offset versus UTC.|
| `clock summertime name recurring` | Global command that names a daylight savings time for a time zone and tells IOS to adjust the clock automatically. |
| `ntp server address / hostname` | Global command that configures the device as an NTP client by referring to the address or name of an NTP server.|
| `ntp master stratum-level` | Global command that configures the device as an NTP server and assigns its local clock stratum level.|
| `ntp source name/number` | Global command that tells NTP to use the listed interface (by name/number) for the source IP address for NTP messages.|

## Verification

| Command | Explanation |
|---|---|
| `show cdp neighbors [type number]` | Lists one summary line of information about each neighbor; optionally, lists neighbors off the listed interface|
| `show lldp neighbors [type number]` | Lists one summary line of information about each neighbor; optionally, lists neighbors off the listed interface|
| `show cdp neighbors detail` | Lists one large set of information (approximately 15 lines) for every neighbor |
| `show lldp neighbors detail` | Lists one large set of information (approximately 15 lines) for every neighbor |
| `show cdp interface [type number]` | States whether CDP is enabled on each interface or a single interface if the interface is listed |
| `show lldp interface [type number]` | States whether LLDP is enabled on each interface or a single interface if the interface is listed |
| `show cdp traffic` | Displays global statistics for the number of CDP advertisements sent and received|
| `show lldp traffic` | Displays global statistics for the number of LLDP advertisements sent and received|
| `show logging` | Lists the current logging configuration and lists buffered log messages at the end |
| `terminal [no] monitor` | For a user (SSH or Telnet) session, toggles on or off the receipt of log messages, for that one session, if `logging monitor` is also configured|
| `[no] debug {various}` | EXEC command to enable or disable one of a multitude of debug options |
| `show clock` | Lists the time of day and the date per the local device |
| `show ntp associations` | Shows all NTP clients and servers with which the local device is attempting to synchronize with NTP |
| `show ntp status` | Shows current NTP client status in detail|
