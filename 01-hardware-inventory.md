# Hardware Inventory

**Last Updated:** January 15, 2026
**Status:** Current State Documentation

## Purpose
This document tracks all physical and virtual hardware in the homelab environment. Maintaining accurate inventory is critical for capacity planning, disaster recovery, and security management.

---

## Physical Hardware

### Proxmox Hypervisor (HP Z440)

**Model:** HP Z440 Workstation
**Role:** Primary hypervisor running Proxmox VE
**Location:** [Specify location]
**Purchase Date:** [Date]
**Warranty Status:** [Status]

#### Specifications

| Component | Details |
|-----------|---------|
| **CPU** | Intel Xeon E5-2620 v3 @ 2.40GHz |
| **Cores/Threads** | 6 cores / 12 threads |
| **RAM** | 64GB DDR4 ECC (4x 16GB Micron) |
| **RAM Speed** | 2133 MT/s |
| **RAM Type** | DIMM, Synchronous Registered (Buffered) |

#### Network Interfaces

| Interface | Type | Speed | MAC Address | Current Usage | Notes |
|-----------|------|-------|-------------|---------------|-------|
| eno1 | Intel I218-LM (Onboard) | 1 Gbit/s | a0:8c:fd:c1:25:1c | Proxmox Management | Primary management interface |
| enp8s0f0 | Intel X540-AT2 (PCIe) | 10 Gbit/s | [MAC] | Available for passthrough | VFIO-PCI configured |
| enp8s0f1 | Intel X540-AT2 (PCIe) | 10 Gbit/s | [MAC] | Available for passthrough | VFIO-PCI configured |

**NIC Configuration Notes:**
- Dual 10Gbit Intel X540-AT2 card is already configured for PCIe passthrough (vfio-pci driver)
- 1Gbit onboard NIC can remain dedicated to Proxmox management
- This configuration enables either physical router or properly isolated virtualized routing

#### Storage

**Primary Drive:** Micron M500 960GB SATA SSD (894.3GB usable)

| Partition/Volume | Size | Type | Mount Point | Usage |
|------------------|------|------|-------------|-------|
| sda1 | 1MB | Partition | - | BIOS boot |
| sda2 | 1GB | Partition | /boot/efi | EFI boot |
| sda3 | 893GB | LVM PV | - | Proxmox LVM |
| └─ pve-swap | 8GB | LVM | [SWAP] | Swap space |
| └─ pve-root | 96GB | LVM | / | Proxmox OS |
| └─ pve-data | 758GB | LVM thin pool | - | VM/CT storage |

**Optical:** HP PLDS DVDRW DU8AESH

**Storage Notes:**
- LVM thin provisioning enabled for efficient VM storage
- ~104GB currently allocated to VMs/CTs
- ~654GB free in thin pool for expansion

---

### Network Equipment

#### Primary Router

**Current Device:** Verizon Router  
**Model:** [Specify exact model]  
**Firmware:** [Version]  
**Role:** WAN gateway, DHCP, basic routing  
**IP Address:** [WAN IP - REDACTED for GitHub]  
**LAN Subnet:** [Current subnet]

**Limitations:**
- No VLAN routing support
- Limited firewall capabilities
- Basic management interface

**Future Replacement:** Dedicated OPNsense appliance (see Planning section)

#### Managed Switch

**Model:** TP-Link TL-SG1428PE
**Port Count:** 28 ports (24x 1G PoE+, 2x 1G SFP, 2x Combo)
**PoE Capable:** Yes (PoE+)
**PoE Budget:** 250W
**Firmware:** [Version]
**IP Address:** 192.168.1.166 (current) → 10.0.1.2 (target)

**Port Allocation:**

| Port | Device | VLAN | PoE | Speed | Notes |
|------|--------|------|-----|-------|-------|
| 1 | Uplink to Router | 1 | No | 1G | WAN connection |
| 2 | Proxmox (eno1) | 1 | No | 1G | Management |
| 3 | Proxmox (10G) | TBD | No | 10G | Future VLAN trunk |
| 4-8 | PoE IP Cameras | TBD | Yes | 1G | Camera VLAN |
| 9 | PlayStation 5 | TBD | No | 1G | IoT VLAN |
| 10-11 | Firestick (via AP) | TBD | No | - | Via WiFi AP |
| [Additional] | [Device] | [VLAN] | [Y/N] | [Speed] | [Notes] |

**Current PoE Usage:**
- Camera 1: [Watts]
- Camera 2: [Watts]
- Camera 3: [Watts]
- Camera 4: [Watts]
- Camera 5: [Watts]
- **Total:** [Total Watts] / [Total Budget] W

#### Wireless Access Point

**Model:** TP-Link BE11000 (Ceiling Mount)
**Standard:** WiFi 7 (802.11be)
**Speed:** Up to 11 Gbps (Tri-Band)
**Role:** Wireless access point only (not routing)
**Firmware:** [Version]
**Connection:** Connected to managed switch
**IP Address:** 192.168.1.168 (current) → 10.0.1.3 (target)
**Mount:** Ceiling mounted
**SSIDs Configured:**
- [Primary SSID] - Main network (VLAN 10)
- [IoT SSID] - IoT devices (VLAN 30)
- [Guest SSID] - Guest network (VLAN 50)

**WiFi 7 Features:**
- Multi-link operation (MLO)
- 320 MHz channels
- 4K-QAM modulation
- Tri-band (2.4GHz + 5GHz + 6GHz)

---

### Endpoint Devices

#### Security Cameras (5x PoE IP)

**Model:** [Brand/Model]  
**Resolution:** [1080p/4K/etc]  
**Power:** PoE (802.3af/at)  
**Storage:** [NVR/Cloud/SD Card]

| Camera | Location    | IP Address  | Notes                     |
| ------ | ----------- | ----------- | ------------------------- |
| CAM-01 | Front Door  | 10.0.40.101 | Anpviz PTZIP46620ED-SA-US |
| CAM-02 | Front Porch | 10.0.40.102 | Anpviz IPC-D3243W-S       |
| CAM-03 | Driveway    | 10.0.40.103 | Anpviz IPC-D3243W-S       |
| CAM-04 | Backyard    | 10.0.40.104 | Anpviz IPC-D3243W-S       |
| CAM-05 | Basement    | 10.0.40.105 | Anpviz IPC-D3243W-S       |

**Camera Network Requirements:**
- MUST be isolated from other networks (security)
- NO internet access (unless cloud recording required)
- Only accessible from management VLAN or dedicated NVR
- Recording destination: [Local NVR / Cloud / Both]

#### Entertainment/IoT Devices

| Device | Model | IP Address | VLAN | Notes |
|--------|-------|------------|------|-------|
| PlayStation 5 | Sony PS5 | 10.0.30.101 | IoT (30) | Gaming console |
| Firestick 1 | Amazon Fire TV | 10.0.30.102 | IoT (30) | Living room |
| Firestick 2 | Amazon Fire TV | 10.0.30.103 | IoT (30) | Bedroom |

**IoT Security Requirements:**
- Restricted outbound access (internet only)
- NO access to servers, cameras, or management
- DNS via Pi-hole for ad blocking

#### Trusted Workstations (VLAN 10)

| Device | Hostname | IP Address | VLAN | Notes |
|--------|----------|------------|------|-------|
| ASUS_Monster | admin-desktop | 10.0.10.100 | Trusted (10) | Primary admin/gaming PC |
| WorkLT | admin-laptop | 10.0.10.101 | Trusted (10) | Work laptop |

##### ASUS_Monster (Primary Workstation)

**Role:** Primary admin workstation / Gaming PC
**Hostname:** ASUS_Monster
**Target IP:** 10.0.10.100
**Target VLAN:** 10 (Trusted)

| Component | Details |
|-----------|---------|
| **CPU** | Intel Core i9-12900KF @ 3.20 GHz (12th Gen) |
| **Cores/Threads** | 16 cores (8P + 8E) / 24 threads |
| **RAM** | 64 GB |
| **GPU** | NVIDIA GeForce RTX 4070 SUPER |
| **System Type** | 64-bit, x64-based processor |
| **Device ID** | 4EDB7304-696E-4769-AE1C-653DB50B4B09 |
| **Input** | Pen support |

**Network Access Requirements:**
- Full access to Management VLAN (infrastructure administration)
- Full access to Server VLAN (service management)
- Camera viewing access (RTSP streams)
- Internet access
- DNS via Pi-hole

##### WorkLT (Work Laptop)

**Role:** Mobile admin workstation / Work laptop
**Hostname:** WorkLT
**Target IP:** 10.0.10.101
**Target VLAN:** 10 (Trusted)
**Serial Number:** 5CD350B378

| Component | Details |
|-----------|---------|
| **CPU** | Intel Core i5-1334U @ 1.30 GHz (13th Gen) |
| **Cores/Threads** | 10 cores (2P + 8E) / 12 threads |
| **RAM** | 16 GB |
| **Storage** | WD PC SN740 512GB NVMe SSD |
| **OS** | Windows 11 Pro 25H2 |
| **System Type** | 64-bit, x64-based processor |
| **Device ID** | 3C3EBE67-4D65-4FFD-AA2F-4B74ECC7FF74 |

**Network Access Requirements:**
- Full access to Management VLAN (infrastructure administration)
- Full access to Server VLAN (service management)
- Camera viewing access (RTSP streams)
- Internet access
- DNS via Pi-hole

---

## Virtual Machines & Containers

### Current Inventory

| ID | Type | Name | OS | vCPU | RAM | Disk | Current IP | Target IP | Purpose | Status |
|----|------|------|----|------|-----|------|------------|-----------|---------|--------|
| 100 | VM | TrueNAS | TrueNAS SCALE | 6 | 8GB | 32GB | 192.168.1.177 | 10.0.20.10 | Network storage | Running |
| 101 | VM | docker-host | Ubuntu 24.04 | 6 | 12GB | 32GB | 192.168.1.176 | 10.0.20.20 | Docker/Twingate/Frigate/Plex | Running |
| 102 | LXC | pihole | Debian/Ubuntu | 1 | 512MB | 8GB | 192.168.1.254 | 10.0.20.25 | DNS/Ad blocking | Running |
| 103 | VM | claude-server | Ubuntu 24.04 | 2 | 4GB | 32GB | 192.168.1.190 | 10.0.20.30 | Claude AI | Running |
| - | VM | opnsense | FreeBSD | 2 | 5GB | 32GB | - | 10.0.1.1 | Router/Firewall + Suricata IPS | Planned |
| - | VM | security-onion | Ubuntu | 8 | 16GB | 200GB | - | 10.0.99.10 | IDS/SIEM | Planned |

### Templates

| ID | Name | OS | Base Disk | Purpose |
|----|------|----|-----------|---------|
| 104 | ubuntu-2204 | Ubuntu 22.04 | cloud-init | Cloud-init template |
| 900 | ubuntu-2404 | Ubuntu 24.04 | 32GB | Cloud-init template |
| 901 | ubuntu-2504 | Ubuntu 25.04 | cloud-init | Cloud-init template |

### Resource Utilization

| Resource | Allocated | Available | Usage |
|----------|-----------|-----------|-------|
| vCPU | 12 | 12 threads | 100% |
| RAM | 20.5GB | 64GB | 32% |
| Storage | ~104GB | 758GB pool | 14% |

#### VM 100: TrueNAS

**Purpose:** Network-attached storage, media server, backup target
**OS:** TrueNAS SCALE
**VM ID:** 100
**Resources:**
- vCPU: 6
- RAM: 8GB
- Disk: 32GB

**Current IP:** 192.168.1.177
**Target IP:** 10.0.20.10 (VLAN 20 - Servers)

**Shares/Datasets:** See [CIFS Shares](06-cifs-shares.md)
- WS1 (`/mnt/S1/WS1`) - TV shows, coding projects
- WS2 (`/mnt/S2/WS2`) - Movies, music

**Backup Strategy:** [Document backup approach]

#### VM 101: Docker Host

**Purpose:** Container orchestration (Twingate, Frigate NVR, Plex Media Server)
**OS:** Ubuntu Server 24.04 LTS
**VM ID:** 101
**Resources:**
- vCPU: 6
- RAM: 12GB
- Disk: 32GB

**Current IP:** 192.168.1.176
**Target IP:** 10.0.20.20 (VLAN 20 - Servers)
**MAC Address:** bc:24:11:25:70:ef

**Attached Hardware:**
- Google Coral USB Accelerator (USB passthrough from Proxmox host)

**Docker Containers:**

| Container | Image | Purpose | Ports | Status |
|-----------|-------|---------|-------|--------|
| twingate | twingate/connector | Zero-trust network access | - | Running |
| frigate | ghcr.io/blakeblackshear/frigate:stable | NVR with AI detection | 5000, 8554, 8555 | Planned |
| plex | plexinc/pms-docker:latest | Media server | 32400 (host mode) | Planned |

**Storage Mounts:**

| Mount Point | Source | Purpose |
|-------------|--------|---------|
| /mnt/truenas/frigate | TrueNAS S2:/frigate | Frigate recordings |
| /mnt/truenas/media | TrueNAS S2:/WS2 | Plex media library |

#### CT 102: Pi-hole (LXC Container)

**Purpose:** DNS server and ad blocking
**OS:** Debian/Ubuntu (LXC)
**Container ID:** 102
**Resources:**
- vCPU: 1
- RAM: 512MB
- Disk: 8GB

**Current IP:** 192.168.1.254
**Target IP:** 10.0.20.25 (VLAN 20 - Servers)

**Configuration:**
- Upstream DNS: Cloudflare (1.1.1.1) or Quad9 (9.9.9.9)
- Serves DNS to all VLANs
- Ad blocking enabled

#### VM 103: Claude Server

**Purpose:** Running Claude AI instance
**OS:** Ubuntu Server 24.04 LTS
**VM ID:** 103
**Resources:**
- vCPU: 2
- RAM: 4GB
- Disk: 32GB

**Current IP:** 192.168.1.190
**Target IP:** 10.0.20.30 (VLAN 20 - Servers)
**MAC Address:** bc:24:11:e3:71:08

**Access:** [How is this accessed?]
**Configuration:** [Document setup details]

---

## External Services

### N8N Automation Platform

**Hosting:** VPS (External)
**Provider:** Hostinger
**Location:** [Data center region]
**IP Address:** 82.29.152.83 [REDACT BEFORE SHARING]
**Resources:**
- vCPU: 2 (AMD EPYC 9354P)
- RAM: 8 GB
- Storage: 96 GB (19 GB used)

**Purpose:** Workflow automation and integration platform
**Access:** [URL - REDACT BEFORE SHARING]
**Backup:** [Backup strategy]

**Key Workflows:**
- [Workflow 1 description]
- [Workflow 2 description]

**Security Configuration:**
- **Authentication:** PKI (certificate-based)
- **Management Access:** ASUS_Monster only (currently)
- **Future:** Add WorkLT for server management

**Security Considerations:**
- Ensure N8N can securely connect to homelab via Twingate
- API keys stored in environment variables
- Regular backup of workflow configurations
- PKI certificates for secure authentication

---

## Planned Hardware Additions

### Google Coral USB Accelerator

**Purpose:** AI/ML inference acceleration for Frigate NVR object detection
**Status:** To Purchase
**Estimated Cost:** $30-60

**Specifications:**
- Edge TPU coprocessor
- USB 3.0 interface
- 4 TOPS (int8) inference performance
- Supported models: TensorFlow Lite

**Installation:**
1. Connect to Proxmox host USB port
2. Pass through to docker-host VM (101)
3. Frigate auto-detects via `/dev/bus/usb` passthrough

**Vendor ID:** 1a6e:089a (Global Unichip) or 18d1:9302 (Google)

---

### Dedicated Router/Firewall Appliance

**Purpose:** Replace Verizon router with OPNsense for VLAN routing and advanced firewall  
**Target Specs:**
- CPU: Multi-core Intel (4+ cores recommended)
- RAM: 8GB minimum, 16GB preferred
- Storage: 64GB SSD minimum
- NICs: 3-4 Intel NICs (NOT Realtek)
- Form Factor: Small form factor / fanless preferred

**Recommended Options:**
1. **Protectli VP4670** - 6 port, fanless, Intel NICs
2. **Dell OptiPlex Micro 7050** + Intel NIC card
3. **HP EliteDesk 800 G3 Mini** + USB/PCIe NIC

**Budget:** $200-400  
**Priority:** High (required for VLAN implementation)

### Network Video Recorder (NVR)

**Purpose:** Dedicated recording for IP cameras
**Solution:** Frigate NVR running as Docker container on docker-host (VM 101)

**Implementation Details:**
- Software: Frigate NVR (ghcr.io/blakeblackshear/frigate:stable)
- AI Acceleration: Google Coral USB TPU
- Storage: TrueNAS S2 pool via SMB mount
- Cameras: 5x Anpviz (VLAN 40)
- Retention: 7 days motion, 30 days events

**Storage Requirements:**
- 5 cameras × 1080p × 7 days retention
- Estimated: 500GB - 1TB (motion-only recording)
- Location: /mnt/S2/frigate on TrueNAS

---

## Resource Utilization

### Current Proxmox Resource Usage

**CPU:**
- Total Available: 12 threads
- Currently Allocated: [#] vCPUs
- Current Usage: [%]

**Memory:**
- Total Available: 64GB
- Currently Allocated: [GB]
- Current Usage: [GB] ([%])

**Storage:**
- Total: [TB]
- Used: [TB]
- Free: [TB]

**Headroom for Growth:** [Assessment]

---

## Maintenance Schedule

| Task | Frequency | Last Performed | Next Due | Owner |
|------|-----------|----------------|----------|-------|
| Firmware updates (switch) | Quarterly | [Date] | [Date] | [Name] |
| Proxmox updates | Monthly | [Date] | [Date] | [Name] |
| VM OS updates | Monthly | [Date] | [Date] | [Name] |
| Backup verification | Weekly | [Date] | [Date] | [Name] |
| UPS battery test | Quarterly | [Date] | [Date] | [Name] |
| Physical cleaning | Annually | [Date] | [Date] | [Name] |

---

## Notes

### Power Outage Lessons Learned

**Issue:** Virtualized pfSense failed to restore network connectivity after power outage  
**Root Causes Identified:**
1. Boot order - Proxmox started before pfSense VM initialized
2. Network dependency - Couldn't access Proxmox web UI without network
3. NIC passthrough - VFIO-PCI didn't reinitialize properly

**Solutions for Future Virtual Router:**
1. Dedicate 1Gbit NIC to Proxmox management (never passed through)
2. Configure pfSense/OPNsense VM to auto-start with highest priority
3. Add startup delay for other VMs to ensure router is ready
4. Document out-of-band access procedures
5. Consider physical router as alternative

### GitHub Sanitization Notes

**Before sharing this document publicly:**
- Replace all internal IP addresses with placeholders (10.x.x.x → 10.0.X.X)
- Redact MAC addresses
- Remove serial numbers and warranty information
- Genericize hostnames
- Remove purchase dates and locations
- Use template values for passwords/keys (never commit real ones)

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-01-14 | Initial inventory creation | [Your Name] |
| 2026-01-15 | Added ASUS_Monster (i9-12900KF, RTX 4070 SUPER) to Trusted Workstations | [Your Name] |
| 2026-01-15 | Updated switch model to TP-Link TL-SG1428PE | [Your Name] |
| 2026-01-15 | Updated WiFi AP to TP-Link BE11000 ceiling mount | [Your Name] |
| 2026-01-15 | Updated Proxmox host to HP Z440, added storage details | [Your Name] |
| 2026-01-15 | Updated VM/CT inventory with IDs, specs, IPs (Pi-hole is LXC not Docker) | [Your Name] |
| 2026-01-15 | Added WorkLT (i5-1334U work laptop) to Trusted Workstations | [Your Name] |
| 2026-01-16 | Updated docker-host (VM 101) to 6 vCPU/12GB RAM for Frigate/Plex | [Your Name] |
| 2026-01-16 | Added Google Coral USB Accelerator to planned hardware | [Your Name] |
| 2026-01-16 | Added Frigate NVR and Plex containers to docker-host | [Your Name] |
