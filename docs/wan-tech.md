# WAN Technologies & Concepts

## W.1 What a WAN is, and where it fits

A **WAN** provides connectivity over a **large geographic area** (between cities, countries, continents) — contrast a LAN (single building/campus) and a MAN (a city). WANs interconnect remote sites, users, and data centers; they are typically **leased from a service provider** rather than owned, because laying long-haul infrastructure is prohibitively expensive.

- A **converged network** carries voice, video, and data over **one common infrastructure** instead of separate parallel networks — the cost saving and the reason QoS matters.
- A **data center** is the dedicated facility for storage/compute; for disaster recovery it serves as the **backup site** (often leased off-site to avoid ownership cost).

## W.2 WAN terminology — the physical demarcation

| Term | Meaning |
|---|---|
| **CPE** (Customer Premises Equipment) | Devices + inside wiring at the enterprise edge that connect to the carrier link |
| **DTE** (Data Terminal Equipment) | The customer device passing data from the LAN for WAN transmission (typically the router) |
| **DCE** (Data Communications Equipment) | The provider's device that gives the customer an interface into the WAN cloud (sets the clock rate) |
| **Local loop / "last mile"** | The physical cable from the customer to the provider's nearest **POP** (Point of Presence) |
| **CO** (Central Office) | The provider facility where the local loop terminates |
| **DSLAM** | Concentrator at the CO that multiplexes many DSL subscribers onto a high-capacity backhaul (e.g. a T3) |

## W.3 Private vs public WAN infrastructure

| Category | Technologies |
|---|---|
| **Private** (dedicated, provider-isolated) | **Leased line** (dedicated point-to-point), **Ethernet WAN** (Metro Ethernet), ISDN/PSTN (dial), ATM, Frame Relay, **MPLS** (provider-managed) |
| **Public** (shared, Internet-based) | **DSL**, **cable**, satellite/**VSAT**, municipal Wi-Fi, WiMAX, cellular 3G/4G/5G |

**Access technology quick facts (exam favorites):**

- **DSL** — high-speed over existing copper telephone lines; DSLAM multiplexes subscribers at the CO onto a T3/fiber backhaul.
- **Ethernet WAN (Metro Ethernet)** — switched, high-bandwidth **Layer 2**; integrates trivially with existing Ethernet LANs; carries converged voice/video/data. The right answer whenever a question wants **high-speed metro connectivity that plugs into the existing LAN**.
- **Leased line** — dedicated, always-on, point-to-point; simple but expensive per-Mbps.
- **MPLS** — provider-managed; flexible (rides T/E-carrier, Ethernet, DSL, etc.) but generally lower raw speed than a dedicated Ethernet WAN.
- **Bandwidth-to-cost intuition:** dedicated Ethernet WAN > MPLS > DSL/cable > ISDN/VSAT for raw speed.