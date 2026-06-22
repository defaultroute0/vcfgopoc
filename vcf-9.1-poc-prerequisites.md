# VMware Cloud Foundation 9.1 — PoC Prerequisites

Everything that must be in place **before** deploying a VMware Cloud Foundation (**VCF**) 9.1 Proof of Concept (PoC).

> Tables marked **`TBA`** are worksheets for **you to complete** with your environment's values before the PoC begins.

> **What's new in 9.1:** several management appliances that used to be separate are now consolidated into one new containerised **VCF Management Services** cluster, and a **centralised VCF License Server** is added. The Management Services cluster (Section 1) needs IP addresses, DNS names and capacity reserved up front — plan it from the start.

## Contents

1. [VCF Management Services](#1-vcf-management-services)
2. [Internet and Proxy](#2-internet-and-proxy)
3. [VLANs](#3-vlans)
4. [MTU / Jumbo Frames](#4-mtu--jumbo-frames)
5. [Dynamic Routing (BGP)](#5-dynamic-routing-bgp)
6. [DNS Records](#6-dns-records)
7. [DNS and NTP Servers](#7-dns-and-ntp-servers)
8. [Management Hosts (ESX)](#8-management-hosts-esx)
9. [Storage](#9-storage)
10. [Licensing](#10-licensing)
11. [Firewall — Ports and URLs](#11-firewall--ports-and-urls)
12. [Software and Depot](#12-software-and-depot)
13. [Certificates](#13-certificates)
14. [Jump Host](#14-jump-host)
15. [Naming Standard](#15-naming-standard)
16. [Passwords](#16-passwords)

---

## 1. VCF Management Services

A cluster of small virtual appliances on the management domain that runs VCF's lifecycle and platform services (Fleet Lifecycle, SDDC Lifecycle, the Identity Broker that provides single sign-on, and so on). It is **mandatory** in 9.1 and is built during deployment, so reserve its addresses, names and capacity now.

**Capacity to plan for:** for a PoC the cluster deploys at its **`Simple`** model — about **4 nodes** (1 control-plane + 3 workers) at roughly **12 vCPU / 24 GB RAM** each (planning guideline — confirm current figures in Broadcom's *Management Services Models* documentation), scaling up automatically as services are added. Size your management hosts to accommodate this alongside the other management appliances.

**Addresses and names to reserve:**

| Item | Requirement |
|---|---|
| IP addresses (worker nodes) | **12 minimum** (a `/28` block); **30 recommended** (a `/27`) |
| Internal cluster network | Defaults to `198.18.0.0/15` — **must not overlap** your network; changeable at deploy time |
| Service names (FQDNs) | **4 required**, all lowercase, on IPs **outside** the worker-node block: *services runtime, fleet, instance, identity broker*. The centralised **License Server** is a separate appliance with its own FQDN (see Section 10). |

*FQDN = Fully Qualified Domain Name, e.g. `vcf-runtime.example.com`.*

---

## 2. Internet and Proxy

VCF needs internet access — directly or through a proxy — to download software during installation and for ongoing updates (used by the VCF Installer, SDDC Manager and VCF Operations). **If there is no internet**, use an offline depot instead (Section 12) and plan for it early.

A **proxy** is supported. Configure the same proxy in two places: the **VCF Installer** (initial build) and **SDDC Manager / VCF Operations** (ongoing downloads). Two limits to note: **NTLMv2 proxy authentication is not supported**, and if the proxy inspects SSL traffic you must load its root certificate into the VCF Installer's trust store.

| Proxy setting | Value |
|---|---|
| Host : Port | TBA |
| Authentication (none, or username + password) | TBA |
| Addresses to bypass (internal subnets, depot) | TBA |

---

## 3. VLANs

Provision **new, unused VLANs** to avoid IP clashes and leave room to grow. (*TEP = Tunnel End Point — the interfaces NSX, VMware's network-virtualisation layer, uses to carry virtual-network traffic.*)

| VLAN | ID | Purpose |
|---|---|---|
| ESXi Management | TBA | Physical host management |
| VM Management | TBA | Management VMs and all VCF Management Services addresses |
| vMotion | TBA | Live VM migration (needs jumbo frames) |
| vSAN | TBA | Only if vSAN is your storage (needs jumbo frames) |
| FC / VMFS | TBA | Only if standing up on Fibre Channel storage |
| NSX Host TEP | TBA | Host overlay networking (needs jumbo frames) |
| NSX Edge TEP | TBA | Edge overlay networking (needs jumbo frames) |
| Edge BGP Peer Link 1 | TBA | Point-to-point link, Edge uplink 1 to your router (BGP runs here) |
| Edge BGP Peer Link 2 | TBA | Point-to-point link, Edge uplink 2 to your router (BGP runs here) |
| VPC External Block | N/A | External IP range for NSX VPC networking |
| VPC Transit Gateway Block | N/A | Private transit-gateway subnets |

> **NSX Host TEP IP pool:** size it at **2 IPs per host** — double the vMotion/vSAN ranges. A **static IP pool is recommended** (the deployment wizard defaults to DHCP). Under-sizing the TEP pool is one of the most common bring-up failures.
>
> *VPC = Virtual Private Cloud — NSX's multi-tenant networking construct.*

---

## 4. MTU / Jumbo Frames

The hard minimum for the **NSX overlay (TEP) network is MTU 1600** (1700 recommended) — below this the Installer pre-check **blocks deployment**. **Jumbo frames (MTU 9000) are the recommendation** for vMotion, vSAN and both NSX TEP networks; set MTU **end-to-end across the physical switches to at least the VDS/overlay MTU** (give a small margin, e.g. 9216 on the switches for a 9000 host MTU, to absorb Geneve encapsulation overhead) so frames aren't fragmented. In short: **1600 = floor, 9000 = ideal.**

---

## 5. Dynamic Routing (BGP)

NSX Edge nodes route traffic between the VCF virtual networks and the rest of your physical network. This needs a dynamic routing protocol — **BGP** (Border Gateway Protocol) is preferred. **If BGP is not available, tell the deployment team as early as possible.**

You will need to provide:

- 2 × IP addresses for the Edge nodes on **BGP Peer Link 1**
- 2 × IP addresses for the Edge nodes on **BGP Peer Link 2**
- 1 × **ASN** (Autonomous System Number) for VCF
- The **BGP peering password**
- **BFD** (Bidirectional Forwarding Detection) enabled, for faster failover

---

## 6. DNS Records

Each item below needs **two DNS records**: a **forward (A) record** (name → IP) *and* a **reverse (PTR) record** (IP → name). Use **lowercase** names (capitals are rejected), assign each an IP from the management subnet, and make sure they resolve from both the ESXi Management and VM Management networks. The FQDNs shown are **examples** — substitute your own domain.

> **Warning — avoid a `.local` domain:** **VCF Automation fails to install in a `.local` domain** (Broadcom KB 413394). Use a normal domain (e.g. `example.com`, `lab.net`).

> **About the example IPs below.** The addresses are a **worked example only** — a sample **VM Management** subnet `10.0.0.0/24` (and **ESXi Management** `10.0.1.0/24`). **Substitute your own**; your real values are the `TBA` worksheets in Sections 1–3. Note how the **Management Services worker `/28`** (`10.0.0.112/28` → usable `.113–.126` = 14 IPs, covering the 12-minimum from Section 1) is reserved as a single block, and the **4 service FQDNs sit *outside* that block**.

**Core components**

| Appliance | Example FQDN | Network | Example IP *(yours = `TBA`)* |
|---|---|---|---|
| VCF Installer | `vcf-installer.example.com` | VM Management | `10.0.0.10` |
| SDDC Manager | `sddc-manager.example.com` | VM Management | `10.0.0.11` |
| vCenter | `vcenter-mgmt.example.com` | VM Management | `10.0.0.12` |
| NSX Manager VIP (Virtual IP) | `nsx-mgmt.example.com` | VM Management | `10.0.0.20` |
| NSX Manager 1 / 2 / 3 | `nsx-mgmt-01 / -02 / -03.example.com` | VM Management | `10.0.0.21 / .22 / .23` |
| NSX Edge Node 1 / 2 | `nsx-edge-01 / -02.example.com` | VM Management | `10.0.0.31 / .32` |
| VCF Operations — Primary / Replica / Data Node | `vcf-ops-01 / -02 / -03.example.com` | VM Management | `10.0.0.41 / .42 / .43` |
| VCF Operations — Collector | `vcf-ops-collector.example.com` | VM Management | `10.0.0.44` |
| VCF Automation | `vcf-automation.example.com` | VM Management | `10.0.0.45` |
| VCF Operations for Networks — Platforms / Collector | `vcf-net-01…03 / vcf-net-collector.example.com` | VM Management | `10.0.0.51–.53 / .54` |
| Offline Depot (if used) | `vcf-depot.example.com` | VM Management | `10.0.0.60` |
| ESX Hosts 1–4 | `esx-01 … esx-04.example.com` | ESXi Management | `10.0.1.11–.14` |

**VCF Management Services (the new platform services)**

| Component | Example FQDN | Network | Example IP *(yours = `TBA`)* |
|---|---|---|---|
| VCF Services Runtime | `vcf-runtime.example.com` | VM Management | `10.0.0.70` |
| Fleet component | `vcf-fleet.example.com` | VM Management | `10.0.0.71` |
| Instance component | `vcf-instance.example.com` | VM Management | `10.0.0.72` |
| Identity Broker | `vcf-idb.example.com` | VM Management | `10.0.0.73` |
| License Server | `vcf-license.example.com` | VM Management | `10.0.0.74` |
| Runtime worker nodes (12–30 IPs) | named automatically | VM Management | **`10.0.0.112/28`** *(block; `.113–.126`)* |

> - **Worker nodes:** add a **reverse DNS zone** so every node IP gets a PTR; the **4 service FQDNs** sit on IPs **outside** the worker range (the License Server FQDN is separate).
> - **NSX Edges:** the Simplified Edge deploy doesn't prompt for DNS — set **`name-servers` / `search-domains`** on the Edges, or they can't resolve the depot.
> - **9.1 inputs:** the Installer takes Management Services (≥12 IPs) and VCF Automation (≥5 IPs) as **IP ranges**, not per-node FQDNs. **Log management moved into the cluster — no Logs FQDN at bring-up** (add a standalone Logs appliance later only for dedicated analytics). VCF Operations, Automation and Operations-for-Networks stay standalone and each need DNS.

---

## 7. DNS and NTP Servers

Provide **two DNS servers** and **two NTP (time) servers**, reachable from both the ESXi Management and VM Management networks, and hosted **outside** the VCF instance being deployed. **Accurate time is critical** — even small clock differences break sign-on, certificates and login tokens across VCF.

---

## 8. Management Hosts (ESX)

The physical ESX hosts that run the management domain — size and prepare them before commissioning. *(VMware renamed **ESXi** to **ESX** from 9.1 onward; both names refer to the same hypervisor.)*

**Software:** all hosts must run **ESX 9.1.0.0 (build 25370933)** — the 9.1 Bill of Materials build. **Stateless / Auto Deploy ESX is not supported** — hosts must be stateful installs.

**Per-host minimum hardware (PoC):**

| Resource | Minimum |
|---|---|
| CPU | **≥ 12 physical cores / 24 threads** |
| Memory | **≥ 128 GB RAM** |
| NICs | **≥ 1 × 10 GbE** (a single pNIC is supported); **25 GbE recommended**. The Installer pre-check **fails** sub-10 GbE (bypassable for a PoC). |
| Disks (vSAN ESA) | **1 × boot (≥ 128 GB) + NVMe** for vSAN — NVMe is required; 3+ devices is a typical ReadyNode layout but **not a hard minimum** (see Storage) |

**Host count** depends on storage and topology: **vSAN = 3 minimum / 4 recommended**; FC (VMFS) and NFS v3 = 2 minimum. Confirm against Broadcom's *Planning and Preparation Workbook* for your chosen model.

**Per-host prep (before commissioning):**

1. Set the host **FQDN, domain, DNS servers and search domains** on the default TCP/IP stack — must match the forward/reverse DNS records exactly (**lowercase, no capitals**).
2. Tag the **VM Management VLAN** on the management port group.
3. **Regenerate the host certificate** so it matches the new FQDN (`/sbin/generate-certificates`), then **reboot**.

---

## 9. Storage

Choose the **principal storage** for the management domain. VCF 9.1 supports three options for a new deployment:

| Principal storage | Notes | Min. hosts |
|---|---|---|
| **vSAN** | VMware's built-in storage; **9.1 defaults to vSAN ESA**, which needs an **NVMe controller (replaces the SCSI controller)** and NVMe devices, plus the vSAN VLAN and jumbo frames | 3 (4 rec.) |
| **Fibre Channel (VMFS)** | External FC SAN; one shared datastore presented to all management hosts; use redundant FC fabrics | 2 |
| **NFS v3** | External NAS; one shared export presented to all management hosts | 2 |

**iSCSI, SAS-attached storage, NFS 4.1, FCoE and NVMe-over-Fabrics are *supplemental* only** — usable for workloads, but **not** as the management domain's principal storage. Whichever array you choose must be on the **VCF 9.1 / vSphere 9 storage compatibility list (HCL = Hardware Compatibility List)**.

| Storage detail | Value |
|---|---|
| Principal storage type (vSAN / FC / NFS v3) | TBA |
| Array / NAS vendor and model | TBA |
| Connectivity (FC fabrics / NFS network / vSAN VLAN) | TBA |
| Datastore(s) or export shared to all management hosts | TBA |
| Multipathing policy (for block storage) | TBA |
| Usable capacity | TBA |

---

## 10. Licensing

VCF 9.1 centralises licensing in a dedicated **VCF License Server** appliance (a separate appliance installed alongside VCF Operations; **IPv4 only**; reserve its IP + FQDN — see Sections 1 and 6).

- New instances start in **evaluation mode**; apply licenses afterwards.
- **Register** the VCF Operations instance and the License Server with the **VCF Business Services console** (`vcf.broadcom.com`) to activate connected licensing.
- Supports multiple primary licenses and multiple Site IDs per vCenter; programmatic license APIs are available (including for disconnected/air-gapped sites).
- In **connected mode**, VCF 9.1 automates the previous manual 180-day licence refresh workflow: license usage data is sent to Broadcom every 24 hours, and updated license files are downloaded and applied automatically. In **disconnected mode**, the customer must still manually exchange usage and license files within the 180-day reporting cycle.

| Licensing detail | Value |
|---|---|
| Mode (connected / disconnected) | TBA |
| License keys (vSphere, vSAN, NSX, VCF) | TBA |

---

## 11. Firewall — Ports and URLs

**Authoritative reference:** the complete VCF 9.1 port-and-protocol list (with plain-language source/destination names and a network diagram) is on the VMware Ports & Protocols portal — **[ports.broadcom.com](https://ports.broadcom.com/)** (filter by **VCF 9.1**). The list of internet addresses to allow is **[KB 327186 – Public URL list for VCF Products](https://knowledge.broadcom.com/external/article/327186/)**.

**Outbound internet addresses to allow** (all **TCP 443 / HTTPS**):

| Address | Purpose |
|---|---|
| `dl.broadcom.com` | Main software depot — all VCF downloads |
| `vcf.broadcom.com` | Licensing and downloads |
| `eapi.broadcom.com` | vSAN compatibility and licensing data |
| `vvs.broadcom.com` | Validated-solution compatibility data |
| `vsanhealth.vmware.com` | vSAN hardware compatibility |
| `vcsa.vmware.com` | Telemetry |
| `auth.esp.vmware.com` | Update Manager download service |
| `storage.googleapis.com` | Depot CDN backend — *observed, not on Broadcom's published list; allow only if downloads fail* |

Apply this allowlist to the **VCF Management Services / Fleet** egress as well as to SDDC Manager. Remove the retired legacy addresses if present: `depot.vmware.com`, `hostupdate.vmware.com`, `vapp-updates.vmware.com`.

---

## 12. Software and Depot

**Online (has internet):** download the **VCF Installer** appliance (~2 GB) from the Broadcom Support Portal. Everything else is then pulled automatically from the online depot (`dl.broadcom.com`).

**Offline (air-gapped / no internet):** build an **offline depot** — a copy of the software repository on a local server. Use a dedicated VM (OS disk plus a ~500 GB second disk; expect ~150 GB of content). For VCF 9.1 the **VCF Download Tool is the only supported way** to populate it — do not use third-party tools. You then point VCF at it under *VCF Operations → Fleet Management → Lifecycle → Binary Management*.

> **Validate first (9.1):** you can deploy just the **VCF Installer appliance**, connect it to your depot (online or offline) and run the **full pre-check / validation** set — DNS forward+reverse, NTP, NIC speed, disk eligibility, ports — **before** committing to a full build. It also exports the config to JSON for reuse at real deploy time.

**Reference —** Broadcom guide: [Binary Management & Offline Depot (VCF 9.1)](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/lifecycle-management/binary-management-for-vmware-cloud-foundation/download-bundles-to-an-offline-depot.html)

---

## 13. Certificates

VCF manages most service certificates itself, but a few things must be right up front:

- Every appliance FQDN given to the Installer must be **lowercase** and resolvable **forward + reverse** — certificates are issued against these names, and **capital letters in FQDNs are rejected**.
- If you use an **HTTPS offline depot**, its certificate **must include a Subject Alternative Name (SAN) for the depot FQDN and/or IP** (include both to be safe) — missing SANs cause repeated trust prompts and "software depot" sync failures.
- If a proxy inspects SSL, load its **root CA** into the VCF Installer trust store (see Section 2).

---

## 14. Jump Host

A dedicated management workstation with access to the VLANs above:

- **4 vCPU, 6 GB RAM**
- SSH client (PuTTY / Terminal), **Google Chrome**, WinSCP
- Internet access recommended (makes the build easier)

> **Remote access (optional).** If your security policy permits, providing the deployment team with remote access to the jump host (RDP/VNC, or a VPN / bastion into the management network) can reduce scheduling overhead and speed up troubleshooting. Any such access should follow your organisation's security and approval processes.

---

## 15. Naming Standard

Agree a naming convention before you start, applied to the datacenter, clusters, port groups, datastores, switches, etc. Example pattern `org-sddc-m01-<object>`:

`org-sddc-m01-vds-01` · `org-sddc-m01-san01` · `org-sddc-m01-uplink-pg-1`

**All names must be lowercase — no capital letters anywhere** (hosts, vCenters, every endpoint), or bring-up validation fails.

---

## 16. Passwords

VCF 9.1 enforces strong passwords. Each must have:

- **At least 15 characters**
- At least **1 uppercase**, **1 lowercase**, **1 number**, and **1 special character**
- A **broad set of special characters is accepted**, but a few components exclude specific symbols — check the per-component policy below before standardising on one password scheme
- No username fragments or dictionary words

**Reference —** exact per-component policy: [Default Password Requirements for VCF Components](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/fleet-management/manage-passwords/default-password-requirements-for-vcf-components.html)

---

*VCF 9.1 details validated against Broadcom TechDocs (Deploy VCF Management Services; Management Services Models; Default Password Requirements; MTU Guidance; 9.1 Bill of Materials), KB 327186 (public URL list) and community deployment guides. Verify against your exact 9.1.x build before deployment.*
