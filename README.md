# VCF91 — The VCF 9.1 Handbook

**Understand VCF 9.1. One architecture at a time.**

An independent visual handbook for learning, designing, deploying and operating VMware Cloud Foundation 9.1.

> **Disclaimer:** VCF91.com is an independent educational project and is not affiliated with or endorsed by Broadcom or VMware.
>
> **Accuracy note:** This handbook is written from general platform knowledge. Product names, UI labels, supported versions, sizing and limits change between builds. **Always validate against the official VCF 9.1 documentation, release notes, Interoperability Matrix and Broadcom Compatibility Guide before acting on anything here**, especially the deployment, sizing and lifecycle sections.

---

## How to use this handbook

| If you are… | Read | Goal |
|---|---|---|
| **Beginner** | Chapters 1–2, Glossary | Understand fundamentals and terminology |
| **Engineer** | Chapters 3–8, 11, 12, Troubleshooting | Build, deploy and operate |
| **Architect** | Chapters 2, 5, 9, 13, 14, Decisions | Design environments and justify decisions |

Callout legend used throughout:

> 🧠 **Architect's Note** — judgement that comes from experience.
> ⚠️ **Common Mistake** — what goes wrong in real projects.
> 🔎 **Deep Dive** — optional detail.

---

## Table of Contents

1. [VCF Fundamentals](#01--vcf-fundamentals)
2. [VCF Architecture](#02--vcf-architecture)
3. [Compute](#03--compute)
4. [Storage / vSAN](#04--storage--vsan)
5. [NSX Networking](#05--nsx-networking)
6. [Management](#06--management)
7. [VCF Operations](#07--vcf-operations)
8. [VCF Automation](#08--vcf-automation)
9. [Security](#09--security)
10. [Kubernetes / VKS](#10--kubernetes--vks)
11. [Deployment (step by step)](#11--deployment-step-by-step)
12. [Lifecycle](#12--lifecycle)
13. [Design Patterns](#13--design-patterns)
14. [Backup & Disaster Recovery](#14--backup--disaster-recovery)
15. [Architecture Decisions](#15--architecture-decisions)
16. [Troubleshooting Flows](#16--troubleshooting-flows)
17. [Glossary](#17--glossary)
18. [Interview Questions](#18--interview-questions)
19. [Appendix: Worksheets and Further Reading](#19--appendix)

---

# 01 — VCF Fundamentals

## Short introduction

What VCF is, why it exists, and the vocabulary every later chapter relies on.

## Architecture diagram

```text
                     VCF PRIVATE CLOUD
                            │
                        VCF FLEET
                            │
                      VCF INSTANCE
                            │
              ┌─────────────┴─────────────┐
              │                           │
      MANAGEMENT DOMAIN            WORKLOAD DOMAIN
              │                           │
       ┌──────┼──────┐              ┌─────┼─────┐
       │      │      │              │     │     │
    vCenter  NSX  Operations       VM    App   K8s
```

## Core concept

VMware Cloud Foundation (VCF) is an integrated private-cloud platform. It bundles compute (ESXi, vCenter), storage (vSAN), networking and security (NSX), operations and automation into one lifecycle-managed stack, so that a private cloud behaves like a cloud service.

## Detailed explanation

### What is VCF?

Instead of buying, integrating, validating and upgrading separate products, VCF delivers a tested combination of components with one operating model.

### Why VCF?

- **Consistency:** the same architecture everywhere, from core data centre to edge.
- **Lifecycle:** updates are sequenced so components stay in supported combinations.
- **Self-service:** consumers request infrastructure through a catalogue rather than tickets.
- **Security by design:** micro-segmentation, identity and compliance are platform features.
- **Unified VM and container platform.**

### Core vocabulary

| Term | Meaning |
|---|---|
| **Private Cloud** | The overall VCF-based environment delivering cloud-style services on your infrastructure |
| **VCF Fleet** | A group of one or more VCF Instances managed together |
| **VCF Instance** | One deployed VCF environment: a management domain plus workload domains |
| **Management Domain** | Runs the platform's own control-plane components |
| **Workload Domain** | Isolated capacity (clusters) for applications, with its own lifecycle |
| **ESXi** | The bare-metal hypervisor |
| **vCenter** | Central management of ESXi hosts, clusters and VMs |
| **NSX** | Software-defined networking, routing and security |
| **vSAN** | Distributed storage pooled from host-local devices |
| **VCF Operations** | Monitoring, capacity, performance, logs and compliance |
| **VCF Automation** | Self-service catalogue, projects, policies and governance |

### The hierarchy to memorise

```text
Fleet → Instance → Domain → Cluster → Host → VM / Pod
```

> 🧠 **Architect's Note:** Learn the hierarchy first. Every later topic (lifecycle, blast radius, licensing, security boundaries) hangs on it.

> ⚠️ **Common Mistake:** Treating the Management Domain as "just another cluster". It runs the platform itself and deserves stricter change control.

> 🔎 **Deep Dive:** Terminology evolves between releases (older material is heavily SDDC Manager–centric). Check release notes for your exact build to confirm which component performs which function.

## Example

A company runs one Instance: a Management Domain, a production Workload Domain and a VDI Workload Domain. A second Instance in another region is added to the same Fleet.

## Key Takeaways

- VCF is an integrated, lifecycle-managed private cloud.
- Fleet > Instance > Domain is the core hierarchy.
- Management and workload planes are separated by design.

---

# 02 — VCF Architecture

## Short introduction

The management plane, the workload plane and the infrastructure services underneath both.

## Architecture diagram

```text
┌──────────────────────── VCF INSTANCE ────────────────────────┐
│                                                              │
│  MANAGEMENT PLANE                 WORKLOAD PLANE             │
│  ┌────────────────────┐           ┌───────────────────────┐  │
│  │ Management Domain  │           │ Workload Domain A      │  │
│  │ vCenter  NSX Mgr   │           │ vCenter  NSX  Cluster  │  │
│  │ Operations Auto    │           ├───────────────────────┤  │
│  │ ESXi + vSAN        │           │ Workload Domain B      │  │
│  └────────────────────┘           │ (Kubernetes / VKS)     │  │
│                                   └───────────────────────┘  │
│  ──────────── INFRASTRUCTURE SERVICES ────────────────────   │
│   DNS • NTP • Identity • Certificates • Routing • Physical   │
└──────────────────────────────────────────────────────────────┘
```

## Core concept

Separate the control plane from the workloads it controls. Each domain is the unit of isolation, capacity and lifecycle.

## Detailed explanation

- **Management plane:** vCenter, NSX Manager, operations and automation components, lifecycle tooling. Small, protected, stable.
- **Workload plane:** tenant VMs, applications and Kubernetes clusters.
- **Infrastructure services:** DNS, NTP, certificates, identity and routing. Most failed deployments trace back to these rather than to the VMware software.
- **Compute, storage and networking:** every domain consists of clusters of ESXi hosts, a storage layer (vSAN or supported external storage) and NSX overlay networking.
- **Shared vs dedicated NSX:** a workload domain may use its own NSX Manager cluster or share one. This is a key isolation and blast-radius decision (see [Decisions](#15--architecture-decisions)).

> 🧠 **Architect's Note:** Draw the two planes on a whiteboard before touching an installer. If you can't, you aren't ready to deploy.

> ⚠️ **Common Mistake:** Sizing the Management Domain only for today's workload domains. It has to scale with your fleet.

## Example

Management Domain: 4 hosts on vSAN. Workload Domain A: 8 hosts for production. Workload Domain B: 6 hosts dedicated to Kubernetes.

## Key Takeaways

- Separate management from workload.
- Infrastructure services are prerequisites, not afterthoughts.
- Domains are the unit of isolation and lifecycle.

---

# 03 — Compute

## Short introduction

ESXi, vCenter and clusters: how capacity is pooled, protected and balanced.

## Architecture diagram

```text
vCenter
   │
Cluster
   │
┌──┴───┬──────┬──────┐
│      │      │      │
ESXi  ESXi  ESXi  ESXi
```

## Core concept

vCenter manages clusters of ESXi hosts. Cluster services provide resilience (HA) and balance (DRS).

## Detailed explanation

| Feature | What it does | Design note |
|---|---|---|
| **vSphere HA** | Restarts VMs after host failure | Set admission control to N+1 or N+2 deliberately |
| **DRS** | Balances load, places VMs, uses vMotion | Keep it enabled; tune automation level |
| **EVC** | Masks CPU features for mixed generations | Required when mixing hardware generations in a cluster |
| **Resource pools** | Group and prioritise VMs | Avoid deep, complex hierarchies |
| **Shares / reservations / limits** | Control contention | Reservations consume guaranteed capacity |

**VM lifecycle:** create → configure → power on → monitor → migrate (vMotion) → snapshot/clone → retire.

> 🧠 **Architect's Note:** Admission control is capacity you pay for in advance. Decide N+1 or N+2 on purpose.

> ⚠️ **Common Mistake:** Disabling DRS "because it moves my VMs". You lose the balancing that keeps latency predictable.

> 🔎 **Deep Dive:** Rolling lifecycle updates evacuate one host at a time, so they depend on spare cluster capacity.

## Example

An 8-host cluster with N+1 admission control tolerates one host failure and still starts all protected VMs.

## Key Takeaways

- Clusters provide HA and balance.
- EVC enables mixed CPU generations.
- Spare capacity makes upgrades possible.

---

# 04 — Storage / vSAN

## Short introduction

How physical devices become policy-driven virtual disks.

## Architecture diagram

```text
Physical Storage (NVMe in each host)
        ↓
      vSAN (distributed datastore)
        ↓
Storage Policy (FTT, RAID, performance)
        ↓
Virtual Disk (components placed across hosts)
        ↓
       VM
```

## Core concept

vSAN aggregates host-local devices into a shared datastore. **Storage policies** declare intent; vSAN decides placement and reconfigures data when the policy changes.

## Detailed explanation

- **ESA (Express Storage Architecture):** NVMe-based, single-tier design with improved performance and space efficiency.
- **Storage policies:** failures to tolerate (FTT), RAID-1 mirroring or RAID-5/6 erasure coding, plus performance and encryption options.
- **Resilience and capacity:** protection consumes raw capacity. Keep slack space so rebuilds after a disk or host failure can complete.
- **Disk failures:** vSAN rebuilds affected components elsewhere in the cluster; health checks surface the state.
- **RAID concepts:** mirroring is simple with higher overhead; erasure coding is space-efficient but needs enough hosts for the stripe width.

| Policy intent | Typical approach | Trade-off |
|---|---|---|
| Max performance | RAID-1 | Higher capacity use |
| Capacity efficiency | RAID-5/6 | Needs enough hosts |
| Higher resilience | Higher FTT | More capacity and hosts |

> 🧠 **Architect's Note:** Policies are intent, not placement. Change the policy and vSAN reconfigures data in the background, so plan the resync traffic.

> ⚠️ **Common Mistake:** Running the datastore near full. Rebuilds can't complete and resilience silently degrades.

## Example

FTT=1 with RAID-1 stores two copies plus a witness; RAID-5 achieves the same tolerance with less raw capacity.

## Key Takeaways

- Policies drive resilience.
- ESA is NVMe-optimised.
- Keep slack capacity for rebuilds.

---

# 05 — NSX Networking

## Short introduction

Overlay networking, routing and security. This is one of the most important chapters for both design and troubleshooting.

## Architecture diagram

```text
 Host                               Edge Cluster
┌─────────────────────┐            ┌──────────────────────┐
│ VM ─ Segment ─ DFW  │            │  Tier-0 (SR)  ── BGP ───► Physical routers
│       │             │            │     │                │
│   Tier-1 (DR)───────┼─ Geneve ──►│  Tier-1 (SR)         │
└─────────────────────┘   TEP      └──────────────────────┘
```

## Core concept

NSX builds virtual networks in software. Traffic between VMs stays in the hypervisor; traffic leaving the environment crosses an Edge node.

## Detailed explanation

### Building blocks

| Object | Role |
|---|---|
| **Transport Node** | Host or Edge that participates in NSX |
| **Transport Zone** | Defines the scope of an overlay (or VLAN) network |
| **Segment** | A Layer 2 network (overlay or VLAN-backed) |
| **Edge Node / Edge Cluster** | Appliances providing north-south connectivity and stateful services |
| **Tier-0 Gateway** | Connects to the physical network, usually via BGP |
| **Tier-1 Gateway** | Tenant or application gateway; connects up to a Tier-0 |
| **VPC** | Self-service isolated network construct for consumers |

### Routing

- Gateways have a **distributed router (DR)** on every host and, where services are needed, a **service router (SR)** on an Edge.
- Tier-0 commonly runs **BGP** with the physical network, advertising routes with ECMP for scale.
- **Active-active** Tier-0 scales throughput; **active-standby** supports stateful services such as NAT.

### NAT, firewalling and micro-segmentation

- **NAT** runs on gateways (SNAT, DNAT).
- **Gateway Firewall** secures north-south traffic.
- **Distributed Firewall (DFW)** enforces policy at each VM's vNIC for east-west traffic. This enables **micro-segmentation**: per-workload policy regardless of subnet.

### Edge design and HA

Pick Edge size by throughput and services (NAT, load balancing, gateway firewall). Place Edge nodes on separate hosts and failure domains inside an Edge Cluster.

### MTU

Overlay encapsulation adds overhead. Use a minimum of 1600 bytes (1700 is a common recommendation) on every device in the TEP path.

## Interactive packet flow: "Follow the Packet"

Walk through each step in order:

| Step | Stage | What happens |
|---|---|---|
| 1 | **VM** | Workload sends traffic from its vNIC onto an NSX segment |
| 2 | **Segment** | Overlay segment; the host's tunnel endpoint (TEP) encapsulates if the destination is on another host |
| 3 | **DFW** | Distributed Firewall rules are evaluated at the vNIC |
| 4 | **Tier-1** | Distributed router forwards toward the Tier-1 service router if services are needed |
| 5 | **Tier-0** | Routing (BGP) and optional NAT or gateway firewall |
| 6 | **Edge** | The Edge node forwards out its physical uplink |
| 7 | **External Network** | Packet reaches the physical network and beyond |

```text
VM → Segment → DFW → Tier-1 → Tier-0 → Edge → External Network
```

> 🧠 **Architect's Note:** East-west traffic stays in the hypervisor; only north-south crosses an Edge. Size Edges for north-south throughput.

> ⚠️ **Common Mistake:** Ignoring MTU. Most "random" overlay problems are an MTU mismatch somewhere in the underlay.

> 🔎 **Deep Dive:** DFW rules apply at the vNIC *before* routing, so a rule applies even between VMs on the same host.

## Example

A web VM calls an external API: segment → DFW rule check → Tier-1 → Tier-0 → Edge uplink → physical router.

## Key Takeaways

- Segments, Tier-1, Tier-0 and Edges form the path.
- DFW secures at the vNIC.
- Plan MTU and BGP up front.

---

# 06 — Management

## Short introduction

The components that run and govern the platform itself.

## Core concept

The Management Domain hosts the control plane. Its health determines your ability to operate everything else.

## Detailed explanation

| Component | Purpose |
|---|---|
| **vCenter** | Compute management for clusters |
| **NSX Manager** | Networking and security control plane |
| **VCF Operations** | Monitoring and compliance |
| **VCF Automation** | Self-service and governance |
| **Identity** | SSO and directory integration; MFA |
| **Certificates** | Trust between components; needs a lifecycle plan |
| **Lifecycle tooling** | Sequenced, validated updates |

> 🧠 **Architect's Note:** Certificates expire at the worst possible time. Put expiry on a calendar and automate renewal where you can.

> ⚠️ **Common Mistake:** Patching components individually outside the lifecycle workflow.

## Key Takeaways

- Management components need protection.
- Identity and certificates are platform concerns.
- Always use lifecycle tooling.

---

# 07 — VCF Operations

## Short introduction

Know the health, capacity and performance of the private cloud.

## Architecture diagram

```text
Collect → Analyse → Alert → Act
(metrics,  (baselines,  (symptoms,   (remediate,
 logs,      capacity     recommen-    resize,
 events)    models)      dations)     plan growth)
```

## Detailed explanation

- **Monitoring and health:** metrics across compute, storage and network.
- **Alerts:** symptom-based alerts with recommended actions.
- **Capacity:** time-remaining forecasts and right-sizing.
- **Performance:** contention, latency and anomaly detection.
- **Logs:** centralised search; correlate logs with metrics.
- **Compliance:** configuration checks against hardening guides and internal standards.

> 🧠 **Architect's Note:** Alert on symptoms users feel, not on every metric you can graph.

> ⚠️ **Common Mistake:** Leaving default alert thresholds untouched, then ignoring the noise.

## Key Takeaways

- Observe health, capacity and performance.
- Tune alerts.
- Run compliance checks continuously.

---

# 08 — VCF Automation

## Short introduction

Self-service delivery with governance.

## Architecture diagram

```text
User
 ↓
VCF Automation (projects, policies)
 ↓
Catalog
 ↓
Template
 ↓
Deployment
```

## Detailed explanation

- **Self-service:** users request approved items without tickets.
- **Projects:** scope resources, quotas and membership.
- **Policies:** placement, naming, approvals, lease and day-2 actions.
- **Catalogs:** curated items users are allowed to request.
- **Templates:** blueprints (ideally infrastructure-as-code in Git).
- **Deployments:** running results tracked through their lifecycle.
- **Governance:** the guardrails that make speed safe.

> 🧠 **Architect's Note:** Governance is the product. Speed without guardrails is just faster sprawl.

> ⚠️ **Common Mistake:** Publishing every template. Curate a small, well-supported catalogue.

## Key Takeaways

- Catalogs enable self-service.
- Projects set boundaries.
- Templates are code.

---

# 09 — Security

## Short introduction

Layered defence from identity to recovery.

## Architecture diagram

```text
┌───────────────────────────────────────────────┐
│ Visibility: logging • monitoring • compliance │
├───────────────────────────────────────────────┤
│ Data: encryption • backup • DR                │
├───────────────────────────────────────────────┤
│ Platform: hardening • certificates • patching │
├───────────────────────────────────────────────┤
│ Network: gateway FW • DFW micro-segmentation  │
├───────────────────────────────────────────────┤
│ Identity: SSO • RBAC • MFA                    │
└───────────────────────────────────────────────┘
```

## Detailed explanation

- **IAM, RBAC, MFA:** federate identity, grant least privilege, require MFA for admin access.
- **NSX DFW and micro-segmentation:** apply a Zero Trust east-west model.
- **Management security:** restrict management networks, harden and patch management components.
- **Certificates:** a documented replacement process.
- **Logging and monitoring:** forward logs off-platform; alert on privileged actions.
- **Compliance:** continuous configuration checks.
- **Backup and DR:** security controls too, since ransomware recovery depends on them.

### Rolling out micro-segmentation

1. Inventory applications and flows (monitoring only).
2. Group workloads with tags.
3. Write allow rules for known flows.
4. Move to deny-by-default for one application at a time.
5. Review logs; iterate.

> 🧠 **Architect's Note:** Default-deny is the end state. Get there with monitoring-first rollouts.

> ⚠️ **Common Mistake:** Shared admin accounts. You lose auditing and accountability.

## Key Takeaways

- Layer your controls.
- Zero Trust via micro-segmentation.
- Test restores.

---

# 10 — Kubernetes / VKS

## Short introduction

How Kubernetes sits on top of VCF.

## Architecture diagram

```text
VCF
 ↓
vSphere
 ↓
Supervisor
 ↓
VKS (vSphere Kubernetes Service)
 ↓
Kubernetes clusters
 ↓
Applications
```

## Detailed explanation

- **Supervisor:** enabling it on a vSphere cluster makes Kubernetes a native control plane.
- **VKS:** provisions and lifecycles conformant Kubernetes clusters on the Supervisor.
- **Namespaces:** isolation units with quotas and RBAC.
- **Storage:** storage policies surface as Kubernetes StorageClasses (links Chapter 4 to this chapter).
- **Networking:** NSX provides pod and service connectivity; plan IP ranges early.
- **Ownership:** platform teams own layers up to VKS; application teams own what runs above.

> 🧠 **Architect's Note:** Treat Kubernetes IP ranges (pods, services, ingress) as a first-class design input.

> ⚠️ **Common Mistake:** Planning Kubernetes networking after the infrastructure is built.

## Key Takeaways

- Supervisor is the control plane.
- VKS builds clusters on it.
- VMs and containers share one platform.

---

# 11 — Deployment (step by step)

## Short introduction

A deployment journey from requirements to applications. Preparation takes most of the time; the installer is the short part.

## Architecture diagram

```text
Requirements
 ↓
DNS / NTP
 ↓
Network Preparation
 ↓
Physical Infrastructure
 ↓
ESXi
 ↓
VCF Installer
 ↓
Management Domain
 ↓
VCF Services
 ↓
Workload Domain
 ↓
Applications
```

> ⚠️ **Read first:** Exact installer screens, field names, file formats and supported versions vary by build. This guide gives the *order of work and what to verify at each step*. Follow the official VCF 9.1 deployment guide and Planning and Preparation Workbook for exact values.

## Phase 0 — Requirements and design

- [ ] Business requirements captured: scale, availability, compliance, growth
- [ ] Architecture pattern chosen ([Chapter 13](#13--design-patterns))
- [ ] Hardware validated against the Broadcom Compatibility Guide (servers, NICs, storage devices, firmware, drivers)
- [ ] Target VCF 9.1 build identified; release notes and Interoperability Matrix read
- [ ] Licences and entitlements available
- [ ] Sizing completed for Management Domain and first Workload Domain
- [ ] Naming convention agreed (hosts, clusters, domains, FQDNs)
- [ ] Owners assigned: network, DNS, security, storage, virtualisation
- [ ] Change windows and approvals booked

## Phase 1 — DNS and NTP

**Step 1.1 — Create DNS records (forward and reverse)** for every component. Example plan (replace `example.local` and addresses):

| Component | FQDN (example) | IP (example) |
|---|---|---|
| ESXi host 1 | `esx01.example.local` | 10.10.10.11 |
| ESXi host 2 | `esx02.example.local` | 10.10.10.12 |
| ESXi host 3 | `esx03.example.local` | 10.10.10.13 |
| ESXi host 4 | `esx04.example.local` | 10.10.10.14 |
| VCF Installer | `installer.example.local` | 10.10.10.5 |
| Management vCenter | `vc-mgmt.example.local` | 10.10.10.20 |
| NSX Manager VIP | `nsx-mgmt.example.local` | 10.10.10.30 |
| NSX Manager nodes | `nsx-mgmt-01…03` | 10.10.10.31–33 |
| Operations | `ops.example.local` | 10.10.10.40 |
| Automation | `auto.example.local` | 10.10.10.50 |

**Step 1.2 — Verify resolution** from a machine in the management network:

```bash
nslookup esx01.example.local
nslookup 10.10.10.11
dig +short vc-mgmt.example.local
dig +short -x 10.10.10.20
```

Every name must resolve forward *and* reverse to the same address.

**Step 1.3 — NTP:** provide at least two reachable time sources. All hosts and appliances must agree on time (certificate and authentication failures follow time drift).

- [ ] Forward and reverse DNS verified for all names
- [ ] NTP servers reachable from the management network
- [ ] DNS suffix and search domain defined

## Phase 2 — Network preparation

**Step 2.1 — Define VLANs and subnets.** Example plan:

| Purpose | VLAN (example) | Subnet (example) | MTU |
|---|---|---|---|
| Management | 10 | 10.10.10.0/24 | 1500 |
| vMotion | 11 | 10.10.11.0/24 | 9000 or 1500 |
| vSAN | 12 | 10.10.12.0/24 | 9000 or 1500 |
| NSX Host TEP | 13 | 10.10.13.0/24 | 1700+ |
| NSX Edge TEP | 14 | 10.10.14.0/24 | 1700+ |
| Edge uplink A | 15 | 10.10.15.0/24 | 1500 |
| Edge uplink B | 16 | 10.10.16.0/24 | 1500 |

**Step 2.2 — Configure the physical switches:**

- VLANs created and trunked to every host port
- Consistent MTU end to end (overlay path at least 1600, ideally 1700+)
- Redundant uplinks and (if used) LACP or equivalent settings
- Spanning-tree edge/portfast style settings on host ports
- Gateways/SVIs for each routed subnet

**Step 2.3 — Plan routing:** BGP ASNs, neighbour IPs, route filters, BFD if used, and upstream firewall rules for BGP (TCP 179).

**Step 2.4 — Allocate IP pools** for TEPs, management, and Kubernetes (pod, service and ingress ranges) before deployment.

- [ ] VLAN and subnet plan approved
- [ ] MTU verified end to end
- [ ] BGP peering details agreed with the network team
- [ ] Firewall rules for installer, management and NSX flows opened

## Phase 3 — Physical infrastructure and ESXi

**Step 3.1 — Rack, cable and power** hosts. Verify redundant paths to two switches.

**Step 3.2 — Align firmware and drivers** to the compatibility guide and the VCF bill of materials.

**Step 3.3 — Install the required ESXi build** on each management host.

**Step 3.4 — Configure each host:**

- Management IP, subnet, gateway, VLAN
- DNS servers, hostname and FQDN
- NTP
- Certificate regenerated *after* the final hostname is set

**Step 3.5 — Verify the host from the ESXi shell:**

```bash
esxcli network ip interface list
esxcli system time get
esxcli storage core device list | less
vdq -q
```

For vSAN, `vdq -q` should report devices as eligible; clean any old partitions first.

**Step 3.6 — Verify MTU between hosts** (example for a 1500 MTU path; adjust size for your MTU: payload = MTU − 28):

```bash
vmkping -I vmk0 -d -s 1472 10.10.10.12
```

- [ ] All hosts reachable, correct hostname, correct time
- [ ] Storage devices clean and visible
- [ ] Network path and MTU verified between hosts

## Phase 4 — VCF Installer and management domain

> Terminology and exact tooling differ by release. The steps below describe the standard flow: deploy an installer appliance, supply a validated configuration, run validation, then deploy.

**Step 4.1 — Download** the installer and required component binaries for your target VCF 9.1 build (or configure an online/offline depot as documented).

**Step 4.2 — Deploy the installer appliance** to a temporary location reachable from the management network. Give it its FQDN, IP, DNS and NTP.

**Step 4.3 — Connect the depot** and make the required bundles available to the installer.

**Step 4.4 — Complete the deployment specification** (workbook or wizard), including:

- Management domain name, SSO domain and passwords (follow the password policy; store in a vault)
- Host FQDNs and credentials
- vCenter, NSX Manager and other component FQDNs and IPs
- Network pools (vMotion, vSAN, TEP) and VLAN IDs
- Edge cluster and Tier-0 settings (if deployed at bring-up)
- Certificate approach

**Step 4.5 — Run validation.** Fix every failure. Typical failures are DNS, NTP, MTU, VLAN or credentials.

**Step 4.6 — Start the deployment.** Monitor progress and logs. Do not interrupt it. If a step fails, read the error, fix the cause, then retry the failed step following official guidance.

**Step 4.7 — Verify the Management Domain:**

- [ ] vCenter reachable; hosts in a healthy cluster
- [ ] vSAN healthy, no unresolved health alerts
- [ ] NSX Manager cluster healthy (all nodes up, VIP responds)
- [ ] Transport nodes prepared; TEP connectivity confirmed
- [ ] Time, DNS and certificates healthy
- [ ] Backups configured for management components
- [ ] Admin access, RBAC and MFA configured

## Phase 5 — VCF services

**Step 5.1 — Deploy VCF Operations** (monitoring, capacity, logs, compliance). Connect to vCenter and NSX; verify data collection.

**Step 5.2 — Deploy VCF Automation.** Integrate identity; create the first project and a basic catalogue item.

**Step 5.3 — Identity integration:** connect your directory/identity provider, map groups to roles, require MFA for administrators.

**Step 5.4 — Baseline operations:** tune alerts, create dashboards, enable compliance checks.

- [ ] Operations collecting from all domains
- [ ] Automation reachable; test deployment succeeds
- [ ] Identity federated; admin accounts individual, not shared

## Phase 6 — Workload domain

**Step 6.1 — Prepare hosts:** ESXi installed and configured as in Phase 3; clean disks; network ready.

**Step 6.2 — Commission hosts** into the platform inventory (the tooling validates them).

**Step 6.3 — Create the workload domain:** choose storage type, vCenter and NSX model (shared or dedicated; see [Decisions](#15--architecture-decisions)), network pools, host selection and licensing.

**Step 6.4 — Edge and routing (if not already deployed):** create the Edge cluster, Tier-0 (BGP) and Tier-1 gateways. Verify BGP neighbours are established and routes are exchanged.

**Step 6.5 — Apply baseline policy:** storage policies, DFW default policy (monitoring first), RBAC, tags.

**Step 6.6 — Validate with a test VM:** place on a segment, check DHCP/IP, reach the gateway, reach an external address, confirm DFW logging.

- [ ] Domain healthy; hosts in expected cluster(s)
- [ ] BGP up; external reachability confirmed
- [ ] Storage policies applied; test VM resilient
- [ ] Operations shows the new domain

## Phase 7 — Kubernetes (optional)

1. Confirm networking plan (pod, service, ingress ranges) and an appropriate storage policy.
2. Enable the Supervisor on a prepared cluster.
3. Create a namespace with quotas and RBAC.
4. Provision a VKS cluster; verify node readiness and pod networking.
5. Deploy a test application with a persistent volume and a service.

- [ ] Supervisor healthy
- [ ] VKS cluster created; test workload running

## Phase 8 — Applications and handover

- [ ] Catalogue items published and tested
- [ ] Security policy in place (DFW, RBAC, logging)
- [ ] Backups and a restore test completed
- [ ] DR runbook written
- [ ] Monitoring and alert routing confirmed
- [ ] As-built documentation complete (IPs, credentials ownership, diagrams)
- [ ] Operational handover and training done

> 🧠 **Architect's Note:** A two-week preparation for a one-day install is normal. Validate DNS first; it prevents the most failures.

> ⚠️ **Common Mistake:** Skipping the planning workbook and typing values live into the installer.

> 🔎 **Deep Dive:** Document every IP, VLAN, certificate owner and credential owner *before* day one. Rebuilding that knowledge afterwards is far harder.

## Deployment troubleshooting quick table

| Symptom | Likely cause | First check |
|---|---|---|
| Validation fails on hosts | DNS or time | Forward/reverse DNS, `esxcli system time get` |
| TEP ping fails | MTU or VLAN | `vmkping -d -s <size>` on the TEP VLAN |
| vSAN claims no disks | Existing partitions | `vdq -q`, clean devices |
| NSX Manager node won't join | DNS, time or firewall | Node logs, name resolution, ports |
| BGP stays idle | ASN, IP or ACL | Neighbour config, TCP 179, uplink VLAN |

## Key Takeaways

- Prepare, then install.
- DNS and NTP first.
- Checklist every stage.

---

# 12 — Lifecycle

## Short introduction

Keeping the platform current, supported and recoverable through upgrades and capacity changes.

## Architecture diagram

```text
Plan → Pre-check → Back up → Update → Validate
```

## Detailed explanation

- **Why it matters:** components must stay in tested version combinations; lifecycle tooling sequences updates so you aren't solving compatibility by hand.
- **Order of operations:** review release notes and the interoperability matrix → run prechecks → back up management components → update the management domain first → update workload domains → validate.
- **Host patching:** maintenance mode → evacuate with vMotion → patch → reboot → exit. Gating factors are spare capacity and a healthy vSAN resync.
- **Capacity changes:** commission hosts, expand clusters or domains; decommission by evacuating workloads and data first.
- **Staging:** download and stage bundles before the window.

### Upgrade runbook

1. Read release notes and known issues.
2. Confirm the supported upgrade path.
3. Resolve all health warnings.
4. Verify backups and take fresh ones.
5. Stage bundles.
6. Run prechecks; fix failures.
7. Update management domain components in order.
8. Update workload domains one at a time.
9. Validate health, alerts and a sample workload.
10. Record results and update documentation.

> 🧠 **Architect's Note:** The best upgrade is a boring one. Fix warnings before the window, not during it.

> ⚠️ **Common Mistake:** Upgrading while a vSAN resync is running, or on a cluster with no spare capacity.

## Key Takeaways

- Lifecycle tooling keeps versions compatible.
- Management first, workloads after.
- Spare capacity makes rolling updates possible.

---

# 13 — Design Patterns

## Short introduction

Reference designs you can adapt, and the trade-offs behind each.

## Detailed explanation

| Pattern | Use when | Trade-off |
|---|---|---|
| **Consolidated** | Labs, edge, small sites | Less isolation; shared capacity |
| **Standard** | Most production environments | More hosts; clear separation |
| **Stretched cluster** | Site-level availability, near-zero RPO | Needs low-latency links and a witness; complexity |
| **Multi-instance with replication DR** | Independence per site, tolerant of latency | Longer RTO; more to operate |
| **Multi-tenant** | Shared platform, separate teams | Needs strong policy: projects, VPCs, DFW |
| **Edge / remote** | Many small sites | Minimal footprint; central management |

```text
Requirements → Constraints → Pattern → Validate failure scenarios
```

> 🧠 **Architect's Note:** Start from the failure you must survive, then pick the pattern.

> ⚠️ **Common Mistake:** Choosing a stretched design for "availability" without testing the inter-site network.

## Key Takeaways

- Patterns encode trade-offs.
- Design from failure scenarios.
- Isolation costs capacity.

---

# 14 — Backup & Disaster Recovery

## Short introduction

Protecting the platform and the workloads on it.

## Architecture diagram

```text
Classify → Protect → Isolate → Test
```

## Detailed explanation

- **What to protect:** management components (vCenter, NSX Manager, operations, automation), their configuration, and tenant workloads.
- **Methods:** file-based backups of management components to an external target; image-level backups for workloads; keep one copy outside the platform.
- **DR:** define **RPO** and **RTO** per application tier, then choose replication, stretched clusters or restore-from-backup.
- **Ransomware recovery:** immutable copies, an isolated recovery environment and known-clean restore points.
- **Restore order:** identity, DNS and time → management components → workloads.

| Tier | Example RPO | Example RTO | Typical approach |
|---|---|---|---|
| 1 | Minutes | < 1 hour | Replication or stretched cluster |
| 2 | Hours | Hours | Replication with scheduled failover |
| 3 | 24 h | Day | Nightly backup restore |

> 🧠 **Architect's Note:** A backup you haven't restored is a hope, not a control.

> ⚠️ **Common Mistake:** Storing backups on the same platform they protect.

## Key Takeaways

- Classify before you protect.
- Keep copies off-platform.
- Test restores.

---

# 15 — Architecture Decisions

| # | Question | Option A | Option B | Guidance |
|---|---|---|---|---|
| 1 | NSX per workload domain | Shared NSX: lower footprint, simpler cross-domain networking | Dedicated NSX: stronger isolation, independent upgrades | Dedicated for regulated or multi-tenant boundaries |
| 2 | vSAN ESA vs external storage | ESA on NVMe: performance and efficiency | External arrays: reuse investments, capacity-only growth | ESA for new builds unless existing arrays justify otherwise |
| 3 | One large cluster vs several | Efficient pooling | Smaller failure and licence domains | Balance against N+1 overhead |
| 4 | Stretched cluster vs two instances | Near-zero RPO, automated failover; needs low latency and a witness | Tolerates latency, isolates failures; longer RTO | Decide from RPO/RTO and inter-site network quality |
| 5 | Tier-0 active-active vs active-standby | Scales throughput | Supports stateful services (NAT) | Match to services needed |
| 6 | Management domain scope | One per instance (standard) | Separate instances per region | Separate instances for independent lifecycles |
| 7 | Kubernetes in or beside VCF | Unified networking, policy and operations | Team autonomy, duplicated tooling | In VCF unless there is a strong reason |
| 8 | Backup target | External, immutable, off-platform | On-platform | Always off-platform; size for restore speed |

### Decision record template

```text
Title:
Date:
Status: Proposed | Accepted | Superseded
Context:
Options considered:
Decision:
Consequences (positive / negative):
Review date:
```

---

# 16 — Troubleshooting Flows

### 1. Host won't join the cluster

1. Verify forward and reverse DNS.
2. Check time sync.
3. Confirm management VLAN and MTU.
4. Review vCenter and host logs.

### 2. VMs can't reach the external network

1. Check DFW rules and logs.
2. Verify Tier-1 route advertisement.
3. Verify Tier-0 BGP neighbours.
4. Check Edge node health and MTU.

```text
VM → Segment → DFW → T1 → T0 → Edge → External   (check each hop in order)
```

### 3. vSAN resync stuck

1. Check capacity slack.
2. Look for failed disks or hosts.
3. Check inter-host network.
4. Review resync throttling.

### 4. Certificate warnings or login failures

1. Check expiry dates on all components.
2. Verify time sync across appliances.
3. Check identity provider and SSO health.
4. Replace through lifecycle tooling, not by hand.

### 5. Lifecycle precheck fails

1. Read the failing check, not just the summary.
2. Check vSAN health and resync state.
3. Confirm spare capacity for host evacuation.
4. Verify backups and DNS resolution.

### 6. BGP neighbour down on Tier-0

1. Check physical router config and ASNs.
2. Verify uplink VLANs and Edge interfaces.
3. Check MTU and firewall rules on TCP 179.
4. Review Edge node health and logs.

### 7. VM slow or unresponsive

1. Check CPU ready and memory contention.
2. Check storage latency and policy compliance.
3. Check host and DRS imbalance.
4. Look at network drops and DFW rules.

---

# 17 — Glossary

| Term | Definition |
|---|---|
| **BGP** | Routing protocol used between Tier-0 and the physical network |
| **DFW** | Distributed Firewall enforced at the vNIC |
| **Domain** | Isolated unit of capacity and lifecycle |
| **DRS** | Load balancing and placement using vMotion |
| **Edge Cluster** | Group of Edge nodes for redundancy |
| **Edge Node** | NSX appliance providing north-south services |
| **ESA** | vSAN Express Storage Architecture |
| **ESXi** | Bare-metal hypervisor |
| **EVC** | CPU feature masking for mixed-generation hosts |
| **Fleet** | Group of VCF Instances |
| **FTT** | Failures to tolerate in a vSAN policy |
| **Geneve** | Encapsulation protocol used by the NSX overlay |
| **HA** | Restarts VMs after host failure |
| **Instance** | One deployed VCF private cloud |
| **Micro-segmentation** | Per-workload firewall policy rather than per-subnet |
| **Namespace** | Kubernetes isolation unit with quotas |
| **Project** | Scope for resources, quotas and policy in Automation |
| **RBAC** | Role-based access control |
| **RPO** | Maximum tolerable data loss, measured in time |
| **RTO** | Maximum tolerable time to restore service |
| **Segment** | NSX Layer 2 network |
| **Storage policy** | Declared resilience and performance intent |
| **Supervisor** | Kubernetes control plane on vSphere |
| **TEP** | Tunnel endpoint used for overlay encapsulation |
| **Tier-0** | Gateway to the physical network |
| **Tier-1** | Tenant or application gateway |
| **Transport Node** | Host or Edge participating in NSX |
| **Transport Zone** | Scope of an NSX network |
| **VCF Automation** | Self-service catalogue and governance |
| **VCF Operations** | Monitoring, capacity and compliance |
| **vCenter** | Management server for ESXi clusters |
| **VKS** | vSphere Kubernetes Service |
| **vMotion** | Live migration of a running VM between hosts |
| **VPC** | Self-service isolated NSX network |
| **vSAN** | Hyperconverged distributed storage |
| **Witness** | Tie-breaker component in stretched designs |
| **Zero Trust** | Deny by default; allow only verified, needed access |

---

# 18 — Interview Questions

<details><summary><b>1. What is the difference between a VCF Instance and a Fleet?</b></summary>

A Fleet groups multiple Instances. An Instance is one deployed private cloud with its own management domain.
</details>

<details><summary><b>2. Why separate management and workload domains?</b></summary>

Isolation, independent lifecycle, and protecting the control plane from tenant impact.
</details>

<details><summary><b>3. Explain a north-south packet in NSX.</b></summary>

VM → segment → DFW → Tier-1 → Tier-0 → Edge → physical network.
</details>

<details><summary><b>4. How does vSAN tolerate failures?</b></summary>

The storage policy sets FTT and RAID level; components are distributed across hosts or fault domains, and rebuilt elsewhere on failure.
</details>

<details><summary><b>5. How does Kubernetes fit into VCF?</b></summary>

The Supervisor in vSphere hosts the control plane; VKS creates conformant clusters on it.
</details>

<details><summary><b>6. What does VCF Operations add over vCenter alone?</b></summary>

Cross-domain monitoring, capacity forecasting, alert tuning and compliance checks.
</details>

<details><summary><b>7. What is the difference between Tier-0 and Tier-1?</b></summary>

Tier-0 connects to the physical network (BGP); Tier-1 serves tenants or apps and connects up to Tier-0.
</details>

<details><summary><b>8. How do you size the Management Domain?</b></summary>

For all management components now, plus growth in domains and operations data, with headroom for lifecycle operations.
</details>

<details><summary><b>9. Walk through an upgrade.</b></summary>

Plan, precheck, back up, update management first, then workload domains, then validate.
</details>

<details><summary><b>10. When would you choose a stretched cluster?</b></summary>

When you need site-level availability with near-zero RPO and have low-latency, reliable inter-site links.
</details>

<details><summary><b>11. How do you design a DR strategy?</b></summary>

Classify by RPO and RTO, choose replication or backup per tier, keep immutable copies, and test restores.
</details>

<details><summary><b>12. A VM cannot reach the internet. How do you troubleshoot?</b></summary>

Follow the packet: segment, DFW logs, Tier-1 routes, Tier-0 BGP, Edge health, MTU, then the physical network.
</details>

<details><summary><b>13. Why does MTU matter in NSX?</b></summary>

Geneve encapsulation adds overhead; an undersized MTU in the TEP path causes drops or fragmentation.
</details>

<details><summary><b>14. How would you introduce micro-segmentation to a live environment?</b></summary>

Monitor flows first, group with tags, write allow rules, then enforce deny-by-default one application at a time.
</details>

---

# 19 — Appendix

## A. Deployment planning worksheet

| Item | Value | Owner | Verified |
|---|---|---|---|
| DNS domain | | | ☐ |
| DNS servers | | | ☐ |
| NTP servers | | | ☐ |
| Management VLAN / subnet / gateway | | | ☐ |
| vMotion VLAN / subnet | | | ☐ |
| vSAN VLAN / subnet | | | ☐ |
| Host TEP VLAN / pool | | | ☐ |
| Edge TEP VLAN / pool | | | ☐ |
| Edge uplink VLANs / subnets | | | ☐ |
| BGP local ASN / peer ASNs / peer IPs | | | ☐ |
| Host FQDNs and IPs | | | ☐ |
| vCenter / NSX / Ops / Automation FQDNs and IPs | | | ☐ |
| Certificate authority and process | | | ☐ |
| Identity provider and groups | | | ☐ |
| Backup target | | | ☐ |
| Kubernetes pod / service / ingress ranges | | | ☐ |

## B. Contributing a new chapter

Use this skeleton to keep chapters consistent:

```markdown
# NN — Title

## Short introduction
## Architecture diagram
## Core concept
## Detailed explanation
> 🧠 **Architect's Note:**
> ⚠️ **Common Mistake:**
> 🔎 **Deep Dive:**
## Example
## Key Takeaways
```

## C. Further reading

- Official VCF documentation and release notes (Broadcom TechDocs)
- Broadcom Compatibility Guide
- VCF Planning and Preparation Workbook and Interoperability Matrix for your release

---

*VCF91.com is an independent educational project and is not affiliated with or endorsed by Broadcom or VMware.*
