# Quality of Service (QoS)

## Q.1 Why QoS — the problem statement

Without QoS every packet is equal: **FIFO** (first-in, first-out) queuing and, when a queue fills, **tail drop** — whatever arrives next is discarded regardless of importance. That's fine until a link congests; then a backup job can starve a phone call. QoS is the toolset of **per-hop behaviors (PHB)**: each device, independently at each hop, classifies traffic and gives some of it better treatment — there is no end-to-end reservation, just consistent per-hop policy.

The four impairments QoS manages, and what interactive voice tolerates (the exam numbers):

| Characteristic | Definition | One-way voice target |
|---|---|---|
| **Bandwidth** | Link capacity available to a class | ~30–100 kb/s per call (codec-dependent) |
| **Delay (latency)** | Time for a packet to cross the network | ≤ **150 ms** |
| **Jitter** | *Variation* in delay between packets | ≤ **30 ms** |
| **Loss** | Percentage of packets dropped | ≤ **1%** |

Voice/video are the drivers: TCP data retransmits and survives; real-time UDP streams cannot wait or recover. (Endpoints de-jitter with a small playout buffer — but only within limits.)

## Q.2 Classification & marking

**Order of operations:** traffic must be **classified** *before* it can be **marked**. Mark **as close to the source as possible**; the **trust boundary** is where markings start being believed — a Cisco **IP phone is a trusted marking point** (Trust Boundary 1 in the courseware).

**Layer-2 vs Layer-3 marking:**
| Marking | Layer | Field | Notes |
|---|---|---|---|
| **CoS** | L2 | 802.1Q tag, 3 bits (0–7) | Applies to **Ethernet frames**; lost across routed hops |
| **IP Precedence** | L3 | ToS byte, 3 bits | Legacy |
| **DSCP** | L3 | ToS byte, 6 bits (0–63) | Survives end-to-end; modern standard |

**Traffic-type characteristics:**
| | Voice | Video | Data |
|---|---|---|---|
| Packets | Small (~200 B), **predictable, smooth** | **Large, bursty, unpredictable, inconsistent** | Varies |
| Latency budget | ≤ **150 ms** one-way (Cisco design std) | courseware cites ≤ **400 ms** | Not real-time |
| Loss tolerance | Low; **more** resilient than video | **Less** resilient to loss than voice | Retransmits (TCP) |
| Bandwidth | ~30–128 kb/s per call | High (≥ 384 kb/s+) | Varies |

*(The 400 ms video figure is the ENSA courseware value; real-world/other Cisco material treats interactive video like voice at ~150 ms. Voice's ≤150 ms is the reliable number.)*

**Congestion tools recap:**
- **Policing** — **drops** (or re-marks) excess; sawtooth output; in or out.
- **Shaping** — **buffers** excess and sends later; smooth output; **egress only**.
- **WRED** — congestion **avoidance**: drops selectively before the queue is full.

**Classification** = identifying which class a packet belongs to (by ACL, protocol/**NBAR** deep inspection, ingress interface, or an existing marking). **Marking** = writing that decision into the header so every later hop can classify by a cheap field lookup instead of re-inspecting:

| Marking | Where | Bits | Values |
|---|---|---|---|
| **CoS** (Class of Service, 802.1p) | The **802.1Q trunk tag** (L2) | 3 | 0–7 — **lost the moment the frame crosses a routed hop or an access port** (the tag is stripped) |
| **IP Precedence** | IP ToS byte (L3), legacy | 3 | 0–7 |
| **DSCP** | The same ToS byte, redefined (L3) | 6 | 0–63 — survives end-to-end; the modern standard |

**DSCP values to memorize:**

| Name | Decimal | Meaning |
|---|---|---|
| **DF / CS0** (default) | 0 | Best effort |
| **EF** (Expedited Forwarding) | **46** | Voice payload — low loss/latency/jitter, the priority-queue class |
| **AFxy** (Assured Forwarding) | e.g. AF41 = 34 | 4 classes (x = 1–4, higher = better queue) × 3 drop precedences (y = 1–3, **higher = dropped first**); decimal = 8x + 2y |
| **CS1–CS7** (Class Selector) | 8, 16, …, 56 | Backward compatible with IP Precedence (top 3 bits); CS1 = scavenger (worse than best effort) |

Convention: voice payload **EF**, interactive video **AF4x**, call signaling **CS3**, bulk/scavenger **CS1**.

**Trust boundary:** markings are just header bits — any host can set DSCP 46 on its torrents. The trust boundary is the point where markings start being believed; everything outside it gets re-marked. Best practice: push it as close to the edge as possible — **trust a Cisco IP phone** (the phone re-marks its attached PC's traffic to 0), **don't trust user PCs**. On switches: `mls qos trust dscp` / `mls qos trust cos` / `mls qos trust device cisco-phone` (platform-dependent; newer platforms trust by default and use ingress policies instead).

## Q.3 Queuing — congestion management

When the interface is congested, queuing decides **what transmits next**:

| Mechanism | Behavior | Verdict |
|---|---|---|
| **FIFO** | One queue, arrival order | The default on fast interfaces; no differentiation |
| **CBWFQ** (Class-Based Weighted Fair Queuing) | Each class gets its own queue with a guaranteed **minimum bandwidth** share during congestion | The workhorse for data classes |
| **LLQ** (Low Latency Queuing) | CBWFQ **plus one strict-priority queue**: packets there always transmit first — but the priority class is **policed to its configured rate**, so it cannot starve the other queues | The standard design; voice (EF) goes in the priority queue |

Why plain priority queuing needed fixing: an unpoliced strict-priority queue under attack/overload starves everything else — LLQ's built-in policer is the safeguard. Bandwidth guarantees are *minimums that matter only during congestion*; idle bandwidth is shared freely.

## Q.4 Congestion avoidance — WRED

Tail drop has two diseases: it drops indiscriminately, and when a full queue drops many TCP flows at once they all back off and later speed up together — **TCP global synchronization**, a sawtooth of congestion and underuse.

**RED/WRED** (Weighted Random Early Detection) drops *early and selectively*: as the queue's average depth grows past a minimum threshold, it discards a **random, increasing fraction** of packets *before* the queue is full — nudging individual TCP flows to slow down at different times. The **W** = drop probability weighted by marking: **higher AF drop precedence (AFx3) is dropped before AFx1**, and EF is typically never WRED-dropped. Useless against UDP (voice/video don't slow down) — which is why voice belongs in the LLQ priority queue, not a WRED-managed data queue.

## Q.5 Policing & shaping — rate limiting

Both enforce a rate; they differ in what happens to the excess:

| | **Policing** | **Shaping** |
|---|---|---|
| Excess traffic | **Dropped** (or re-marked to a worse class) | **Buffered** and sent later — smooths bursts |
| Effect on TCP | Drops → retransmits | Delay → added latency/jitter |
| Direction | Inbound or outbound | **Outbound only** (you can't buffer what you already received) |
| Typical place | **Provider ingress**: enforce the contract; also the LLQ priority-queue limiter | **Customer egress**: slow to the contracted rate *before* the provider's policer drops the excess |

Classic pairing: ISP sells 200 Mb/s on a 1 Gb/s handoff — the ISP **polices** inbound at 200 Mb/s, so you **shape** outbound to 200 Mb/s and keep the drop decision (and your QoS priorities) on your side.

## Q.6 Configuration — MQC (Modular QoS CLI)

MQC is a three-step pattern: **class-map** (match traffic) → **policy-map** (act on the classes) → **service-policy** (apply to an interface, per direction).

```text linenums="1"
class-map match-any VOICE
 match dscp ef
class-map match-any SIGNALING
 match dscp cs3
class-map match-any VIDEO
 match dscp af41
policy-map WAN-EDGE
 class VOICE
  priority percent 10
 class SIGNALING
  bandwidth percent 5
 class VIDEO
  bandwidth percent 30
 class class-default
  fair-queue
  random-detect dscp-based
interface g0/1
 service-policy output WAN-EDGE
```

- `class-map match-any VOICE` — a class matches on one or more criteria; `match-any` = OR, `match-all` = AND (alternatives to `match dscp`: `match protocol` for NBAR, `match access-group` for an ACL)
- `priority percent 10` — **LLQ**: strict priority for voice, policed at 10% of interface bandwidth
- `bandwidth percent 30` — **CBWFQ**: guaranteed minimum during congestion for the class
- `class-default` — everything unmatched lands here; `fair-queue` balances the flows within it, `random-detect dscp-based` enables WRED honoring drop precedences
- `service-policy output WAN-EDGE` — activate on the congested (usually WAN) interface, egress

Marking at the trust boundary (ingress) uses the same pattern with `set`:

```text linenums="1"
class-map match-any FROM-PHONES
 match protocol rtp
policy-map MARK-IN
 class FROM-PHONES
  set dscp ef
 class class-default
  set dscp default
interface g0/0
 service-policy input MARK-IN
```

- `set dscp ef` — mark voice at the edge so every later hop just matches DSCP
- `set dscp default` — re-mark everything else to 0: the distrust half of the trust boundary

## Q.7 Verification

| Command | Explanation |
|---|---|
| `show policy-map interface g0/1` | **The** QoS command: per-class packet/byte counters, drops, queue depth, offered vs sent rate — proves classification matches and the policy acts |
| `show class-map` | Configured classes and match criteria |
| `show policy-map WAN-EDGE` | The policy definition |
| `show mls qos interface g0/1` | (Legacy switches) Trust state of a port |
| `show interfaces g0/1` | `Total output drops` climbing = congestion exists — the reason to deploy QoS on this interface |

**Reading `show policy-map interface`:** a class showing 0 packets = classification failure (marking not set upstream, wrong match criteria, policy on the wrong direction). Drops inside the `priority` class = voice exceeding its policer — raise the percent or you're clipping calls.
