# Dynamic Routing over Tunnels (OSPF, EIGRP, BGP)

## Which protocol works over which tunnel?

Routing protocols that discover neighbors via **multicast** (OSPF `224.0.0.5/6`, EIGRP `224.0.0.10`) cannot run over a pure crypto-map IPsec VPN — ESP carries unicast IP only. They need an actual tunnel *interface* to ride on:

| Tunnel type | OSPF / EIGRP | BGP | Why |
|---|---|---|---|
| GRE | **Yes** | Yes | GRE encapsulates multicast like any other payload |
| IPsec crypto map (no interface) | **No** | Yes (unicast TCP/179 works if the crypto ACL permits it) | No interface, no multicast |
| IPsec SVTI (`tunnel mode ipsec ipv4`) | **Yes** | Yes | It's a real interface; IOS handles the multicast internally |
| GRE over IPsec | **Yes** | Yes | GRE does the carrying, IPsec does the protecting |

**Universal rules for any protocol over any tunnel:**

- **Recursive routing kills tunnels.** The route to the `tunnel destination` (the peer's public IP) must NEVER be learned through the tunnel itself. Keep the underlay route static (or in a separate process) and never advertise the public/underlay networks into the overlay protocol. Symptom: `%TUN-5-RECURDOWN`, tunnel flapping every ~30 s.
- **Fix the tunnel bandwidth.** Tunnel interfaces default to `BW 100 Kbit/sec`, which poisons OSPF cost and EIGRP metrics. Set `bandwidth` to a realistic value (in Kb/s) on both ends.
- **Advertise the tunnel subnet + the LANs; nothing else.** The overlay protocol should know the `172.16.0.0/30` link and the site LANs — not the physical interfaces.

## OSPF over a tunnel

Works identically over GRE, SVTI, and GRE-over-IPsec — the tunnel is just a point-to-point link to OSPF:

```text linenums="1"
interface tunnel0
 bandwidth 100000
 ip ospf 1 area 0
 ip ospf network point-to-point
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface tunnel0
 network 192.168.1.0 0.0.0.255 area 0
```

- `bandwidth 100000` — pretend the tunnel is 100 Mb/s so OSPF cost is sane (value in Kb/s)
- `ip ospf 1 area 0` — enable OSPF directly on the tunnel (equivalent: a `network 172.16.0.0 0.0.0.3 area 0` statement)
- `ip ospf network point-to-point` — GRE/SVTI tunnels are point-to-point by default, so this is usually already correct; stating it explicitly avoids DR/BDR surprises and speeds adjacency
- `no passive-interface tunnel0` — hellos must flow on the tunnel; keep everything else passive
- `network 192.168.1.0 0.0.0.255 area 0` — advertise the local LAN into the overlay
- **Do not** add a `network` statement covering `g0/1` (the public interface) — that's how recursive routing starts

**OSPF-specific gotchas:**

- Adjacency stuck in `EXSTART` over a tunnel = **MTU mismatch** — both tunnel ends need the same `ip mtu` (e.g., 1400). Lab escape hatch: `ip ospf mtu-ignore`, but fix the MTU properly instead.
- The tunnel's OSPF cost comes from the `bandwidth` value — if a site has two tunnels (primary/backup), steer traffic with `ip ospf cost` on the tunnel interfaces.
- Hellos every 10 s / dead 40 s is slow failover; either tune `ip ospf hello-interval 3` + `ip ospf dead-interval 12` on both ends, or rely on GRE `keepalive` to down the interface faster.

## EIGRP over a tunnel

```text linenums="1"
interface tunnel0
 bandwidth 100000
 ip address 172.16.0.2 255.255.255.252
 tunnel source g0/1 !or tunnel source 198.51.100.2
 tunnel destination 203.0.113.1
 keepalive 5 3
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip hello-interval eigrp 100 5
 ip hold-time eigrp 100 15
router eigrp 100
 network 172.16.0.0 0.0.0.3
 network 192.168.1.0 0.0.0.255
 passive-interface default
 no passive-interface tunnel0
```

- `bandwidth 100000` — EIGRP's metric is built from bandwidth + delay, so the default 100 Kb/s makes every tunnel look terrible; **also**, EIGRP throttles itself to 50% of interface bandwidth for its own packets — at the default that's 50 Kb/s and large topologies fail to converge
- `ip hello-interval eigrp 100 5` / `ip hold-time eigrp 100 15` — faster failure detection (defaults 5/15 on p2p are fine; shown for completeness — tune per need)
- `network 172.16.0.0 0.0.0.3` — form the adjacency across the tunnel
- `network 192.168.1.0 0.0.0.255` — advertise the LAN

**EIGRP-specific gotchas:**

- **Split horizon** bites hub-and-spoke designs: on a multipoint hub interface, the hub won't re-advertise one spoke's routes to another spoke. Fix on the hub interface: `no ip split-horizon eigrp 100`. (Irrelevant for a single point-to-point tunnel.)
- On DMVPN/mGRE hubs you also need `no ip next-hop-self eigrp 100` for spoke-to-spoke shortcuts.
- EIGRP forms adjacencies fast and flaps fast — pair with GRE `keepalive` so the interface state tracks reality.

## BGP over a tunnel

BGP is unicast TCP (port 179), so it runs over *anything* — including a bare crypto-map IPsec VPN (just permit `tcp` between the peering addresses in the crypto ACL). Over a tunnel interface, peer on the tunnel IPs:

```text linenums="1"
router bgp 65001
 neighbor 172.16.0.2 remote-as 65002
 neighbor 172.16.0.2 timers 10 30
 address-family ipv4
  network 192.168.1.0 mask 255.255.255.0
  neighbor 172.16.0.2 activate
```

- `neighbor 172.16.0.2 remote-as 65002` — peer with the far end's **tunnel** IP; since it's directly connected across the tunnel, no `ebgp-multihop` or `update-source` tricks are needed
- `neighbor 172.16.0.2 timers 10 30` — default BGP timers (60/180) mean up to 3 minutes of blackholing after a failure; tighten them, or better, rely on the tunnel interface going down (GRE `keepalive`) to reset the session instantly
- `network 192.168.1.0 mask 255.255.255.0` — advertise the LAN (must exist in the routing table)
- iBGP over tunnels works the same; peer on loopbacks with `update-source loopback0` only if an IGP already provides loopback reachability through the tunnel

**BGP-specific gotchas:**

- BGP's TCP session tolerates anything that passes unicast — this makes it the **only** option over crypto-map IPsec, and the most robust choice over unstable tunnels.
- The recursive-routing rule still applies with a twist: never advertise the tunnel **endpoints'** public networks in BGP over the tunnel, or the underlay route can be overridden after a flap.
- MTU: BGP exchanges large UPDATE packets at session start; a broken path-MTU inside the tunnel shows up as a session that establishes and then hangs in `OpenConfirm`/drops on bulk transfer — another reason `ip tcp adjust-mss` belongs on every tunnel.

## Verification (any protocol over any tunnel)

| Command | Explanation |
|---|---|
| `show ip ospf neighbor` / `show ip eigrp neighbors` / `show ip bgp summary` | Is the adjacency/session up across the tunnel? |
| `show ip route <remote LAN>` | The proof: the far LAN must resolve via the tunnel IP / `Tunnel0` |
| `show ip route <tunnel destination public IP>` | The anti-proof: this must **not** point at `Tunnel0` (recursive routing check) |
| `show interfaces tunnel0 \| include bandwidth` | Confirm the metric-relevant `bandwidth` is set |
| `ping <remote LAN IP> source <local LAN interface>` | End-to-end test through overlay routing, not just tunnel-IP to tunnel-IP |
```

Notes on the addition: it's written as **Section 5** assuming it slots in after MPLS (before the final checklist) in the VPN & Tunnels file — renumber the heading if you place it elsewhere. It follows the established conventions: comparison table first (which protocol survives which tunnel type, with the multicast reasoning), one recipe per protocol with bulleted explanations, protocol-specific gotcha lists (recursive routing, `EXSTART`/MTU, EIGRP split horizon and the 50%-bandwidth throttle, BGP timers), and a verification table that includes the recursive-routing "anti-proof" check.