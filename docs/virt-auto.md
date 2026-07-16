# Virtualization & Network Automation

## V.1 Server virtualization — hypervisors, VMs, containers

A **hypervisor** lets one physical server run many isolated **virtual machines**, each with its own OS, vCPUs, vRAM, vNICs:

| Type | Runs | Examples | Use |
|---|---|---|---|
| **Type 1 (bare-metal)** | Directly on hardware | ESXi, Hyper-V, KVM, Proxmox | Data centers — efficiency, production |
| **Type 2 (hosted)** | As an app on a normal OS | VirtualBox, VMware Workstation | Labs, desktops (your CML/EVE-NG labs) |

**Containers** virtualize at the OS level instead: all containers share the host's kernel, packaging only the app + dependencies (Docker; orchestrated at scale by Kubernetes):

| | VM | Container |
|---|---|---|
| Contains | Full guest OS | App + libraries only |
| Size / boot | GBs, minutes | MBs, seconds |
| Isolation | Strong (hardware-level) | Process-level (shared kernel) |
| Density | Tens per host | Hundreds per host |

**The networking side:** VMs attach to a **virtual switch (vSwitch)** inside the hypervisor — a software L2 switch mapping VM vNICs to VLANs and to the physical NICs (uplinks). Consequences for the network engineer: the switchport facing a virtualization host is always a **trunk** carrying the VM VLANs; and VM-to-VM traffic on the same host **never touches your physical switch** — invisible to SPAN/ACLs there.

**Network functions virtualize too (NFV):** routers, firewalls, and load balancers as VMs (Catalyst 8000V, ASAv, FTDv) — same IOS commands, virtual hardware.

*(VRFs — virtualization of the routing table — are covered in the MPLS/VPN file, §4.3: same idea applied to a router.)*

## V.2 Traditional vs controller-based networking

Every network device has three **planes**:

| Plane | Job | Examples |
|---|---|---|
| **Data plane** | Forward frames/packets | The ASIC doing MAC/FIB lookups, NAT rewrites, ACL drops |
| **Control plane** | Decide *how* to forward | OSPF, STP, ARP — builds the tables the data plane uses |
| **Management plane** | Configure & monitor the device | SSH, SNMP, NETCONF, HTTPS |

- **Traditional networking:** every device runs its own control plane and is configured **one CLI at a time** (distributed intelligence). Consistency depends on humans; per-device config drift is the norm.
- **Controller-based (SDN):** the control plane (partly or fully) moves to a central **controller** with a network-wide view; devices are programmed through APIs.

**The API directions (memorize these):**

| API | Between | Examples |
|---|---|---|
| **Southbound (SBI)** | Controller → network devices | NETCONF/RESTCONF, OpenFlow, SSH/CLI, SNMP, OpenConfig/gNMI |
| **Northbound (NBI)** | Apps/scripts → controller | **REST APIs** — your Python script asks the controller, never the 500 switches individually |

**Cisco's controllers:** **Catalyst Center** (formerly DNA Center) for the enterprise campus; **ACI/APIC** in the data center; **Meraki dashboard** and **vManage (SD-WAN)** in the cloud.

## V.3 Intent-Based Networking (IBN)

IBN is the layer *above* SDN: you declare the **business intent**, the system translates, deploys, and — the part that makes it IBN rather than just automation — **continuously verifies** the network still fulfills it. The canonical loop:

| Phase | What happens | Catalyst Center feature |
|---|---|---|
| **1. Translation** | Intent ("guests never reach corporate", "voice gets priority") is captured and converted into policies, validated against the network model | Policy / Design workflows |
| **2. Activation** | Policies are rendered into device configurations and pushed via the SBI to every relevant device | Provision (templates, fabric, SGT/TrustSec policy) |
| **3. Assurance** | Telemetry streams back; the controller compares **actual state vs declared intent**, surfaces drift and failures, suggests or applies remediation | Assurance (health scores, issues, sensor tests) |

- The loop is **closed**: assurance findings feed back into translation/activation — configure → verify → correct, continuously. Traditional automation is fire-and-forget; IBN checks its own homework.
- Intent is **abstracted from implementation**: "marketing can't reach finance" doesn't name VLANs, ACLs, or IPs — the controller picks the mechanism (SGT policy, ACL, VRF) per device. You manage the *what*; it owns the *how*.
- Exam phrasing to recognize: *fulfillment* (translation + activation) and *assurance* are the two halves of IBN; Catalyst Center is Cisco's IBN controller for the campus.

## V.4 Overlay, underlay, fabric (SD-Access / SD-WAN vocabulary)

| Term | Meaning |
|---|---|
| **Underlay** | The physical network — links and a simple routed IGP whose only job is reachability between devices |
| **Overlay** | Virtual tunnels built on top (**VXLAN** in SD-Access, **IPsec** in SD-WAN) that carry the actual user traffic and policy |
| **Fabric** | Underlay + overlay + the controller managing both, viewed as one system |

Same underlay/overlay logic as the GRE/IPsec chapter, applied network-wide and automated. In SD-Access the VXLAN header carries an **SGT** (security group tag), enabling policy independent of IP addressing; edge switches encapsulate, the controller orchestrates.

## V.5 REST APIs

A **REST API** exchanges structured data over HTTP(S). The verbs map to **CRUD**:

| HTTP verb | CRUD | Meaning |
|---|---|---|
| `POST` | Create | Add a new resource |
| `GET` | Read | Retrieve (never modifies) |
| `PUT` / `PATCH` | Update | Replace / partially modify |
| `DELETE` | Delete | Remove |

Anatomy of `GET https://catalystcenter/dna/intent/api/v1/network-device?managementIpAddress=10.1.1.1`:

- **URI path** identifies the resource; **query parameters** after `?` filter it
- **Headers** carry `Content-Type: application/json`, `Accept:`, and credentials
- **Response** = status code + usually a JSON body

| Status code | Meaning |
|---|---|
| `200` / `201` / `204` | OK / Created / OK-no-body |
| `400` | Bad request (your syntax/body) |
| `401` / `403` | Bad or missing auth / authenticated but not permitted |
| `404` | No such resource (check the URI) |
| `429` | Rate-limited — slow down |
| `5xx` | Server-side failure |

REST is **stateless**: every request carries everything needed; the server remembers nothing between calls — which is exactly why *every* request must carry authentication:

### REST authentication methods

| Method | How it travels | Character |
|---|---|---|
| **Basic auth** | `Authorization: Basic <base64(user:pass)>` header | Simplest; base64 is **encoding, not encryption** — credentials are effectively cleartext, so HTTPS is mandatory. Sent on every call |
| **API key** | Static secret in a header (`X-API-Key: ...`) or query param | Simple machine identity; no expiry unless rotated — treat like a password, never in URLs (they end up in logs) |
| **Bearer token** | `Authorization: Bearer <token>` header | Obtained by an initial login call, **expires** — limits the damage of a leak. The dominant pattern |
| **OAuth 2.0** | Token *framework*: an authorization server issues scoped, short-lived access tokens (+ refresh tokens) | The standard behind "login with X" and most cloud APIs; what bearer tokens usually come from |

The token workflow, using Catalyst Center as the example:

```text
1) POST https://cc/dna/system/api/v1/auth/token        (Basic auth just this once)
   <= 200  {"Token": "eyJhbGciOi..."}
2) GET  https://cc/dna/intent/api/v1/network-device    (every subsequent call)
   Header: X-Auth-Token: eyJhbGciOi...
   <= 200  [ ...device list... ]
3) Token expires => a call returns 401 => repeat step 1
```

- Statelessness in practice: the token **is** the session; there is no "logged in" state on the server side.
- `401` mid-script almost always = expired token, not wrong credentials — re-authenticate and retry.

## V.6 Data formats — JSON, XML, YAML

### JSON (what APIs speak — interpreting it is an exam certainty)

| Element | Syntax | Example |
|---|---|---|
| Object | `{ }` — unordered key:value pairs | `{"hostname": "R1"}` |
| Array | `[ ]` — ordered list | `["Gi0/0", "Gi0/1"]` |
| Key | Always a `"quoted string"` before `:` | `"uptime":` |
| Value | String `"..."`, number, `true`/`false`, `null`, nested object/array | `"14.2"`, `92`, `false` |

```json
{
  "device": {
    "hostname": "R1",
    "isManaged": true,
    "interfaces": [
      { "name": "GigabitEthernet0/0", "vlan": 10 },
      { "name": "GigabitEthernet0/1", "vlan": 20 }
    ]
  }
}
```

- Navigation skill: the VLAN of the second interface = `device` → `interfaces` → `[1]` (arrays count from 0) → `vlan` → `20`
- Validity traps: keys must be quoted; `true`/`false` lowercase, unquoted; no trailing comma; `10` (number) ≠ `"10"` (string)

### YAML (what Ansible speaks)

YAML is a superset of JSON built for humans: **indentation** (spaces, never tabs) replaces braces, `-` introduces list items, `#` comments. The same data as above:

```yaml
# YAML equivalent of the JSON example
device:
  hostname: R1
  isManaged: true
  interfaces:
    - name: GigabitEthernet0/0
      vlan: 10
    - name: GigabitEthernet0/1
      vlan: 20
```

| Concept | JSON | YAML |
|---|---|---|
| Object / mapping | `{"key": "value"}` | `key: value` (nesting by indentation) |
| List | `["a", "b"]` | `- a` / `- b` on separate lines (inline `[a, b]` also legal) |
| String quoting | Mandatory | Optional (quote when ambiguous — `"yes"`, `"01:00"`) |
| Comments | None | `# like this` |

**XML** completes the trio: tag-based (`<hostname>R1</hostname>`), verbose, schema-friendly — what **NETCONF** uses; RESTCONF speaks both XML and JSON.

## V.7 Configuration management & IaC — Ansible and Terraform

**The problem:** hundreds of devices, manual CLI = drift, typos, no audit trail. **The fix:** define configuration as versioned files (**Infrastructure as Code**), keep them in Git, let a tool enforce them. Both tools below are **idempotent** — applying the same definition twice changes nothing the second time (contrast with pasting a CLI script twice).

### Ansible

| Term | Meaning |
|---|---|
| **Control node** | The machine running Ansible — nothing is installed on the network devices (**agentless**); transport is **SSH** (or HTTPS APIs) |
| **Inventory** | INI/YAML file listing managed hosts, grouped, with connection variables |
| **Playbook** | YAML file: ordered **plays**, each a list of **tasks** run against a host group |
| **Module** | The unit of work a task calls (`cisco.ios.ios_config`, `ios_facts`, `ios_command`) |
| **Model** | **Push, procedural** — you (or a schedule/CI job) run the playbook; tasks execute in order |

```yaml
# inventory.yml
routers:
  hosts:
    r1:
      ansible_host: 10.1.1.1
  vars:
    ansible_network_os: cisco.ios.ios
    ansible_connection: ansible.netcommon.network_cli
    ansible_user: admin
```

```yaml
# playbook.yml — ensure NTP + a VLAN exist on all routers
- name: Baseline config
  hosts: routers
  gather_facts: no
  tasks:
    - name: Ensure NTP server is configured
      cisco.ios.ios_config:
        lines:
          - ntp server 192.168.1.100

    - name: Ensure VLAN 30 exists
      cisco.ios.ios_vlans:
        config:
          - vlan_id: 30
            name: IOT
        state: merged
```

- Run with `ansible-playbook -i inventory.yml playbook.yml`; a second run reports `ok` instead of `changed` — idempotence in action
- Note what Ansible rides on: the **SSH + local user** groundwork from the hardening section *is* its transport — automation lives on the management plane you already built

### Terraform

| Term | Meaning |
|---|---|
| **HCL** | The declarative language: you write the **end state**, Terraform computes what must change |
| **Provider** | Plugin that knows an API (AWS, Azure, IOS-XE, Meraki, Catalyst Center) |
| **State file** | Terraform's record of what it built — maps code to real resource IDs; drift is detected by comparing code ↔ state ↔ reality. Protect it (it holds secrets) |
| **Workflow** | `terraform init` (fetch providers) → `terraform plan` (**preview the diff** — the killer feature) → `terraform apply` (execute) → `terraform destroy` (tear down) |

```hcl
# main.tf — declare that VLAN 30 exists on an IOS-XE switch (RESTCONF provider)
resource "iosxe_vlan" "iot" {
  vlan_id = 30
  name    = "IOT"
}
```

- Declarative vs procedural in one line: Ansible = "run these steps"; Terraform = "make reality equal this file". Deleting the `resource` block and applying **removes** the VLAN — the file is the single source of truth
- Sweet spots: Terraform = *provisioning* infrastructure (cloud VPCs/VMs, controller objects, increasingly network device state); Ansible = *configuring* what exists (fleet-wide IOS changes, upgrades). Real shops use both: Terraform builds, Ansible configures

*(Older exam versions named Puppet and Chef here — agent-based, pull-model tools; know the names as the contrast to Ansible's agentless push.)*

## V.8 AI & machine learning in network operations (new in 200-301 v1.1)

| Concept | Meaning in the network context |
|---|---|
| **Predictive AI/ML** | Learn the network's normal from telemetry, then forecast and flag: anomaly detection, failure prediction, capacity trending — Catalyst Center **Assurance** (the IBN assurance phase, §V.3, is where this ML lives) and Meraki health scores |
| **Generative AI** | Produces content: draft configs, explain a routing table, natural-language interfaces to the controller |
| **AIOps** | The umbrella: telemetry in → ML correlation → suggested/automated remediation |
| Caveats the exam expects | Model quality depends on training data; recommendations need human validation; telemetry export = privacy/security consideration |

## V.9 Where the device CLI still shows up

| Command | Explanation |
|---|---|
| `ip http secure-server` | Enables the HTTPS server that RESTCONF rides on |
| `restconf` | Enable the RESTCONF API (IOS-XE) — JSON/XML over HTTPS, YANG-modeled |
| `netconf-yang` | Enable NETCONF (SSH port 830) — the XML-based, standards southbound |
| `show netconf-yang sessions` | Who's currently talking NETCONF to this box |
| `show platform software yang-management process` | Are the model-driven-API daemons actually running |
| `crypto key generate rsa` + `username ... privilege 15` | The SSH/local-auth groundwork Ansible's connection uses |
| `show sdm prefer` | (Switch) TCAM template — relevant when fabric/SGT features need a different one |


## V.10 Network Automation Resources

- [Netmiko](https://python-automation-book.readthedocs.io/en/1.0/12_netmiko/01_netmiko.html)

- [NAPALM](https://napalm.readthedocs.io/en/latest/)