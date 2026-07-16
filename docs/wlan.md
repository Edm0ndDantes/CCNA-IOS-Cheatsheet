# WLANs (Wireless LANs)

## W.1 RF fundamentals

Wi-Fi (IEEE 802.11) carries frames over radio instead of cable — and radio changes the rules:

- **Half-duplex, shared medium:** all devices on a channel share airtime; only one transmits at a time. Wireless uses **CSMA/CA** (Collision *Avoidance*): listen first, wait a random backoff, transmit, and require an **ACK** for every frame — collisions can't be *detected* over radio (you can't hear while transmitting), so they're avoided and repaired instead.
- **Bands and channels:**

| Band | Channels | Character |
|---|---|---|
| **2.4 GHz** | 1–13 (14 in Japan) — but only **1, 6, 11 are non-overlapping** | Longer range, penetrates walls, crowded (Bluetooth, microwaves); channels are 22 MHz wide but spaced 5 MHz apart, hence the 1/6/11 rule |
| **5 GHz** | ~24 non-overlapping 20 MHz channels | Shorter range, much more spectrum, supports 40/80/160 MHz channel bonding for speed; some channels require **DFS** (radar detection) |
| **6 GHz** (Wi-Fi 6E) | 59 additional 20 MHz channels | New, clean spectrum; WPA3 mandatory |

- **Adjacent APs must use non-overlapping channels** — same-channel neighbors share airtime (co-channel interference), overlapping-channel neighbors corrupt each other. The classic design is a honeycomb of 1/6/11 in 2.4 GHz.
- **802.11 standards:**

| Standard | Marketing name | Band | Max (theoretical) |
|---|---|---|---|
| 802.11b / a / g | — | 2.4 / 5 / 2.4 GHz | 11 / 54 / 54 Mb/s |
| 802.11n | Wi-Fi 4 | 2.4 + 5 | ~600 Mb/s (MIMO introduced) |
| 802.11ac | Wi-Fi 5 | 5 only | ~6.9 Gb/s (MU-MIMO downlink) |
| 802.11ax | Wi-Fi 6 / 6E | 2.4 + 5 (+6) | ~9.6 Gb/s (OFDMA, uplink MU-MIMO, better density) |

## W.2 Service sets & topology terms

| Term | Meaning |
|---|---|
| **SSID** | The network *name* — what users see |
| **BSS** | Basic Service Set: one AP and its clients |
| **BSSID** | The BSS's unique ID = the AP radio's MAC address (one SSID on many APs = same SSID, different BSSIDs) |
| **ESS** | Extended Service Set: multiple APs advertising the same SSID over a wired **distribution system**, enabling roaming |
| **IBSS / ad hoc** | Client-to-client, no AP |
| **Mesh** | APs backhaul wirelessly to each other; only some are cabled |

A client **associates** with one BSS at a time; **roaming** = re-associating to a different BSSID of the same ESS. One physical AP typically broadcasts several SSIDs, each mapped to a different **VLAN** — this SSID↔VLAN mapping is the core of enterprise WLAN design.

## W.3 AP architectures — autonomous vs lightweight (CAPWAP)

| Architecture | How it works | Scale |
|---|---|---|
| **Autonomous** | Each AP is fully self-contained, configured individually (CLI/GUI); its switch port is a **trunk** (the AP bridges SSIDs to VLANs locally) | A handful of APs |
| **Lightweight + WLC** | APs keep only the radio functions; all config, authentication, and roaming logic centralizes on a **Wireless LAN Controller**. This division of labor is **split-MAC**. AP switch ports are **access** ports | Enterprise |
| **Cloud-managed** (e.g., Meraki) | Autonomous data plane, cloud control plane | Enterprise, ops-light |

**CAPWAP** — the lightweight AP ↔ WLC protocol: two UDP tunnels, **5246 (control, DTLS-encrypted)** and **5247 (data)**. Client traffic is tunneled from the AP to the WLC and dropped onto the wired VLAN *there* — which is why AP ports are simple access ports and the **WLC's port is the trunk**.

**How a lightweight AP finds its WLC (discovery order worth knowing):** subnet broadcast → previously stored WLC IPs → **DHCP option 43** (WLC address handed out with the lease) → DNS lookup of `CISCO-CAPWAP-CONTROLLER.<domain>`.

**Lightweight AP modes:**

| Mode | Purpose |
|---|---|
| **Local** | Default: serves clients, tunnels all traffic to the WLC |
| **FlexConnect** | Branch office: can switch traffic *locally* and survive a WAN/WLC outage |
| **Monitor / Sniffer** | Dedicated sensor: rogue detection / packet capture (serves no clients) |
| **Rogue detector** | Wired-side rogue correlation |
| **Bridge / Mesh** | Point-to-point links, mesh backhaul |

## W.4 Wireless security

| Method | Encryption / auth | Verdict |
|---|---|---|
| Open / WEP | none / RC4, broken since 2001 | Never (open + captive portal only for guest) |
| WPA | TKIP (RC4 wrapper) | Legacy stopgap — avoid |
| **WPA2** | **AES-CCMP** | Today's baseline |
| **WPA3** | AES-GCMP, **SAE** replaces the PSK handshake (immune to offline dictionary attacks), PMF mandatory | Deploy where supported |

Orthogonal to the version, the **authentication model**:

- **Personal (PSK / SAE):** one shared passphrase for everyone. Simple; no per-user revocation.
- **Enterprise (802.1X/EAP):** each user authenticates individually against a **RADIUS** server (common methods: PEAP = username/password inside TLS, EAP-TLS = client certificates). The RADIUS config from the AAA section is exactly what backs this.

## W.5 Switch configuration for APs and WLC

Port for a **lightweight AP** (CAPWAP tunnels everything, so one VLAN suffices):

```text linenums="1"
interface g1/0/10
 description LIGHTWEIGHT-AP
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
 power inline auto
```

- `switchport access vlan 100` — the AP management VLAN; client SSIDs don't need to exist here — they emerge from the CAPWAP tunnel at the WLC
- `spanning-tree portfast` — the AP is an edge device; let it come up immediately
- `power inline auto` — PoE for the AP (default on PoE switches; shown for completeness — `show power inline` to verify draw)

Port for an **autonomous AP** (bridges SSIDs to VLANs itself):

```text linenums="1"
interface g1/0/11
 description AUTONOMOUS-AP
 switchport mode trunk
 switchport trunk native vlan 100
 switchport trunk allowed vlan 100,110,120
```

- Trunk because each SSID maps to a VLAN tag on the wire; native VLAN = the AP's management VLAN

Port(s) for the **WLC** — a trunk, usually bundled (WLC LAG requires `mode on`, not LACP):

```text linenums="1"
interface range g1/0/23 - 24
 channel-group 5 mode on
interface port-channel 5
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120
```

- `channel-group 5 mode on` — static EtherChannel; **Cisco WLCs don't speak LACP/PAgP**, so `mode on` is mandatory
- Allowed VLANs = the AP management VLAN plus every VLAN any SSID maps to

**DHCP option 43** — point APs at the WLC from the router/IOS DHCP pool:

| Command | Explanation |
|---|---|
| `option 43 hex f104.0a01.0105` | Inside the AP pool: TLV `f1` + length `04` + WLC IP in hex (`0a01.0105` = 10.1.1.5). Multiple WLCs: length `08` + two IPs |

## W.6 WLC quick reference (AireOS CLI)

Day-to-day WLAN creation is GUI work (WLAN → SSID → security → interface/VLAN → enable), but the CLI verifies fast:

| Command | Explanation |
|---|---|
| `show wlan summary` | All WLANs (SSIDs), their IDs, status, and mapped interface |
| `show ap summary` | Every joined AP: model, IP, and how many clients |
| `show ap join stats summary all` | Why an AP did/didn't join — the CAPWAP troubleshooting goldmine |
| `show client summary` | Associated clients, their AP, WLAN, and state |
| `show client detail <mac>` | One client's full story: RSSI, SNR, security, throughput |
| `show interface summary` | WLC's dynamic interfaces (the VLAN mappings) |
| `show sysinfo` | WLC version, AP capacity |

## W.7 Troubleshooting quick hits

| Symptom | First checks |
|---|---|
| AP won't join the WLC | `show ap join stats` on the WLC; DHCP option 43 / DNS record; UDP 5246-5247 open in the path; AP and WLC code versions |
| AP port err-disabled / no power | `show power inline` (PoE budget), BPDU Guard vs an AP in bridge mode |
| Clients associate but get no IP | SSID→VLAN mapping on the WLC, DHCP scope for *that* VLAN, trunk allowed-VLAN list to the WLC |
| Works near the AP, dies far away | Client transmits weaker than the AP — coverage designed on AP power alone; check RSSI/SNR in `show client detail` |
| 2.4 GHz slow, 5 GHz fine | Co-channel interference — neighbors off the 1/6/11 plan |
