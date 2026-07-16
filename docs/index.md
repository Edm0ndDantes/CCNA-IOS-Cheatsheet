# Cisco Handbook IOS Command Cheatsheet for CCNA

A practical reference for configuring Cisco routers and switches. 
Single commands are presented in tables; multi-step setup processes are shown as code blocks with each command explained below.

Commands are entered from these prompt levels:

| Prompt | Mode | How to get there |
|---|---|---|
| `Router>` | User EXEC | Default on login |
| `Router#` | Privileged EXEC | `enable` |
| `Router(config)#` | Global configuration | `configure terminal` |
| `Router(config-if)#` | Interface configuration | `interface g0/0` |
| `Router(config-line)#` | Line configuration | `line console 0` / `line vty 0 4` |
| `Router(config-router)#` | Router (routing protocol) config | `router ospf 1` |