# Navigation & Basics

| Command | Explanation |
|---|---|
| `enable` | Enter privileged EXEC mode |
| `disable` | Return to user EXEC mode |
| `configure terminal` | Enter global configuration mode (`conf t`) |
| `exit` | Go back one mode level |
| `end` | Return directly to privileged EXEC (also **Ctrl+Z**) |
| `do <command>` | Run an EXEC command from config mode, e.g. `do show ip int brief` |
| `?` | Context-sensitive help (list available commands) |
| `show history` | Show recently entered commands |
| `terminal length 0` | Disable `--More--` paging for the session |
| `no <command>` | Negate/remove a command (e.g. `no shutdown`) |

**Tip:** Commands can be abbreviated as long as they're unambiguous (`conf t` = `configure terminal`, `sh ip int br` = `show ip interface brief`). Use **Tab** to auto-complete.