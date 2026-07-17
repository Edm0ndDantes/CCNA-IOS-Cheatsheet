# Enterprise Campus Design (Hierarchical Model)

## E.1 The three layers

| Layer | Role | Typical devices |
|---|---|---|
| **Access** | Connects **end users/devices** to the network; provides **PoE**, port security, VLANs | Fixed-config access switches (e.g. Catalyst 2960) |
| **Distribution** | Aggregates access switches; **inter-VLAN routing, policy, ACLs/security, connects to other networks** | Multilayer switches / routers |
| **Core** | **High-speed backbone** — fast transport between distribution blocks, minimal policy | High-throughput L3 switches |

- **Access layer** functions: connect users, PoE, first-hop security.
- **Distribution layer** functions: **connect remote networks** and **provide data-traffic security** (the two answers Q68 wants) — routing and policy live here.
- **Core layer** function: the high-speed backbone only.
- **Collapsed core**: core and distribution combined into one device — cheaper, used in smaller networks, but **increases the blast radius** of a single failure.

## E.2 Failure domains & redundancy

A **failure domain** is the portion of the network knocked out when one device fails. Design goal: **keep failure domains small**.

- The **switch block / building-block approach** (deploying distribution devices in **redundant pairs**) limits how far a single failure propagates — the correct way to bound a failure domain (Q1).
- Redundant **power supplies** protect against one failure mode only (power) — not a general reliability fix.
- To reason about a failure domain: when a switch loses power, everything **downstream that depends solely on it** (its directly attached hosts/APs/switches with no alternate path) is in the domain.
- **Extending the access layer over wireless** adds flexibility and reduces cost, but **increases the number of single points of failure** and does **not** add bandwidth (Q71).

## E.3 Catalyst 2960 characteristics (courseware specifics)

- **Fixed-configuration** (non-modular) **access-layer** switches.
- Support **one active SVI** (for management) — on IOS **prior to 15.x** only one SVI is active.
- 2960-C models support **PoE pass-through** (powered by an upstream PoE source, then power their own downstream devices).