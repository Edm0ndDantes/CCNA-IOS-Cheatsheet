## Logging, Monitoring & Housekeeping

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

## Low-Level Device Operations

### File system navigation

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

### Firmware (IOS) upgrade

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

### Configuration management — backup & restore

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

### Reboot & scheduled reloads

| Command | Explanation |
|---|---|
| `reload` | Reboot now (warns if the config is unsaved) |
| `reload in 10` | Reboot in 10 minutes — schedule this **before** risky remote changes: if you cut yourself off, the reboot restores the saved config |
| `reload at 02:00 14 July` | Reboot at a specific time (requires a correct clock) |
| `reload cancel` | Made it through? Cancel the pending reload |
| `show reload` | See any scheduled reload |

### Password reset / recovery (router)

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

### Factory reset / configuration reset

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

### Licensing (quick reference)

| Command | Explanation |
|---|---|
| `show license status` | Smart Licensing state (modern IOS-XE) |
| `show license summary` | Which licenses are in use |
| `license boot level network-advantage` | Set the license boot level (reload required) |

---