# VLAN Design and Segmentation

**Last Updated:** January 14, 2026  
**Status:** Design Document

## Executive Summary

This document defines the Virtual LAN (VLAN) segmentation strategy for the homelab environment. The design follows enterprise security best practices including defense-in-depth, least privilege access, and network segmentation to minimize attack surface and contain potential security breaches.

**Key Objectives:**
- Isolate untrusted IoT devices from critical infrastructure
- Protect IP camera privacy and prevent unauthorized access
- Provide secure guest network access
- Enable security monitoring without impacting production
- Maintain manageable and well-documented infrastructure

---

## VLAN Overview Table

| VLAN ID | Name | Subnet | Gateway | DHCP Range | Purpose | Trust Level |
|---------|------|--------|---------|------------|---------|-------------|
| 1 | Management | 10.0.1.0/24 | 10.0.1.1 | None (static only) | Infrastructure management | 🔴 Critical |
| 10 | Trusted | 10.0.10.0/24 | 10.0.10.1 | .100-.200 | Admin workstations, trusted devices | 🟢 High |
| 20 | Servers | 10.0.20.0/24 | 10.0.20.1 | None (static only) | VMs, containers, services | 🟡 Medium |
| 30 | IoT | 10.0.30.0/24 | 10.0.30.1 | .100-.200 | Smart devices, streaming, gaming | 🟠 Low |
| 40 | Cameras | 10.0.40.0/24 | 10.0.40.1 | .100-.110 | IP surveillance cameras | 🔴 Restricted |
| 50 | Guest | 10.0.50.0/24 | 10.0.50.1 | .100-.254 | Guest WiFi, temporary devices | ⚫ Zero Trust |
| 99 | Security | 10.0.99.0/24 | 10.0.99.1 | None (static only) | IDS/IPS, logging, monitoring | 🔴 Critical |

**Subnet Sizing:**
- All VLANs use /24 (254 usable addresses)
- Provides adequate room for growth
- Simplifies troubleshooting with clear addressing

**Address Space Reservations:**
- .1 - Gateway (OPNsense interface)
- .2-.99 - Reserved for infrastructure/static assignments
- .100-.200 - DHCP pool (where applicable)
- .201-.254 - Reserved for future static assignments

---

## VLAN 1: Management Network

**Purpose:** Administrative access to all infrastructure devices  
**Subnet:** 10.0.1.0/24  
**Gateway:** 10.0.1.1 (OPNsense)

### Design Rationale

The management VLAN is the most critical network segment. It must:
- Be accessible even if routing fails
- Never depend on services from other VLANs
- Provide out-of-band management capability
- Use separate physical interfaces where possible

### Static IP Assignments

| IP | Device | Interface | Notes |
|----|--------|-----------|-------|
| 10.0.1.1 | OPNsense | LAN interface | Primary gateway/firewall |
| 10.0.1.2 | Managed Switch | Management IP | Web UI, SSH |
| 10.0.1.3 | WiFi AP | Management IP | Web UI, configuration |
| 10.0.1.10 | Proxmox | eno1 (1Gbit) | Hypervisor web UI |
| 10.0.1.11 | iDRAC/iLO | IPMI (if available) | Out-of-band management |
| 10.0.1.20-29 | Reserved | - | Future infrastructure |

### Access Control

**Allowed Sources:**
- VLAN 10 (Trusted) - administrators can access management interfaces
- Local console access - always available

**Blocked Sources:**
- VLAN 30 (IoT) - no management access from IoT devices
- VLAN 40 (Cameras) - cameras cannot access management
- VLAN 50 (Guest) - guests cannot access infrastructure

**Services Permitted:**
- HTTPS (443) - Web UIs
- SSH (22) - CLI access
- SNMP (161/162) - Monitoring (read-only)
- NTP (123) - Time synchronization

### Security Hardening

- Strong passwords (minimum 16 characters, complexity required)
- SSH key authentication preferred over passwords
- Management interfaces not accessible from WAN
- Failed login attempt monitoring (>3 failures = alert)
- Regular firmware/OS updates
- Separate credentials per device (no shared admin passwords)

### Disaster Recovery

**Critical Requirement:** Management network must remain accessible even if:
- OPNsense VM fails
- Main routing fails
- Power outage recovery

**Implementation:**
- Proxmox eno1 (1Gbit NIC) never passed through to VMs
- Direct laptop connection capability to Proxmox
- Console access to switch via serial cable
- Documented recovery procedures

---

## VLAN 10: Trusted Network

**Purpose:** Administrator workstations and fully trusted personal devices  
**Subnet:** 10.0.10.0/24  
**Gateway:** 10.0.10.1 (OPNsense)

### Design Rationale

This VLAN hosts devices owned and managed by administrators. These devices:
- Require access to all other VLANs for management
- Run updated, patched operating systems
- Have endpoint security (antivirus, firewall)
- Are actively monitored

### DHCP Configuration

**DHCP Range:** 10.0.10.100 - 10.0.10.200  
**Lease Time:** 24 hours  
**DNS Servers:** 10.0.20.25 (Pi-hole), 10.0.10.1 (OPNsense)

### Static Reservations (DHCP)

| IP | Device | MAC Address | Notes |
|----|--------|-------------|-------|
| 10.0.10.100 | Admin Desktop | xx:xx:xx:xx:xx:xx | Primary workstation |
| 10.0.10.101 | Admin Laptop | xx:xx:xx:xx:xx:xx | Mobile admin device |
| 10.0.10.102 | Tablet | xx:xx:xx:xx:xx:xx | Trusted tablet |

### Access Permissions

**Outbound Access:**
- ✅ VLAN 1 (Management) - Full administrative access
- ✅ VLAN 20 (Servers) - SSH, HTTP/S, SMB, NFS, RDP
- ✅ VLAN 30 (IoT) - HTTP/S for device configuration
- ✅ VLAN 40 (Cameras) - RTSP, HTTP/S for viewing
- ✅ VLAN 99 (Security) - HTTPS to Security Onion dashboard
- ✅ Internet - Full access

**Inbound Access:**
- Limited - Only established/related connections
- No services listening by default

### WiFi SSID Mapping

**SSID:** "HomeNetwork" (or your primary SSID)  
**Security:** WPA3-Personal (WPA2 fallback)  
**Hidden:** No (for usability)

### Security Policies

**Endpoint Requirements:**
- OS patch level within 30 days
- Local firewall enabled
- Full disk encryption recommended
- Antivirus/EDR installed and updated
- No shared devices (each user has own device)

**Monitoring:**
- Log successful/failed authentications
- Alert on unusual traffic patterns
- Baseline normal behavior (machine learning optional)

---

## VLAN 20: Server Network

**Purpose:** Virtual machines, containers, and production services  
**Subnet:** 10.0.20.0/24  
**Gateway:** 10.0.20.1 (OPNsense)

### Design Rationale

Servers operate with moderate trust:
- Provide services to users
- May have internet access for updates
- Should not initiate connections to client devices
- Require monitoring and logging

### Static IP Assignments

| IP | Hostname | Type | Service | OS | Notes |
|----|----------|------|---------|----|----- |
| 10.0.20.10 | truenas | VM | Network storage | TrueNAS SCALE | File shares, backup target |
| 10.0.20.20 | docker-host | VM | Container platform | Ubuntu 24.04 | Twingate only |
| 10.0.20.25 | pihole | LXC | DNS/Ad blocking | Debian/Ubuntu | Serves all VLANs |
| 10.0.20.30 | claude-server | VM | Claude instance | Ubuntu 24.04 | AI assistant server |
| 10.0.20.40 | [future] | - | Available | - | Reserved |

### Docker Container Networking

**Host:** docker-host (10.0.20.20)

| Container | Internal Port | External Port | Purpose |
|-----------|---------------|---------------|---------|
| twingate | - | - | Zero-trust connector (no exposed ports) |

**Note:** Only Twingate runs in Docker. Pi-hole runs as a dedicated LXC container at 10.0.20.25.

### Access Permissions

**Outbound Access:**
- ✅ Internet - For updates, external APIs, cloud services
- ⚠️ Other VLANs - Generally BLOCKED (servers don't initiate to clients)
- ✅ VLAN 1 - For management (SSH, monitoring agents)

**Inbound Access:**
- ✅ VLAN 1 (Management) - SSH, web UIs, management protocols
- ✅ VLAN 10 (Trusted) - Service-specific ports (SMB, NFS, HTTP/S)
- ⚠️ VLAN 30 (IoT) - BLOCKED (IoT should not access servers directly)
- ✅ All VLANs - DNS to Pi-hole (10.0.20.25:53) - EXPLICIT ALLOW RULE

### Service-Specific Rules

**TrueNAS (10.0.20.10):**
- TCP 80, 443 (Web UI) - From VLAN 1, 10
- TCP 445 (SMB) - From VLAN 10
- TCP 2049 (NFS) - From VLAN 10
- TCP 22 (SSH) - From VLAN 1

**Docker Host (10.0.20.20):**
- TCP 22 (SSH) - From VLAN 1
- Docker daemon - NOT exposed outside host
- Twingate connector (outbound only)

**Pi-hole LXC (10.0.20.25):**
- UDP/TCP 53 (DNS) - From ALL VLANs (explicit allow)
- TCP 80/443 (Web UI) - From VLAN 1, 10

**Claude Server (10.0.20.30):**
- TCP 22 (SSH) - From VLAN 1
- TCP [application port] - From VLAN 10 (or as needed)

### Backup Strategy

**Backup Targets:**
- Primary: TrueNAS local storage
- Secondary: External USB drive (weekly manual)
- Offsite: N8N VPS via Twingate (encrypted)

**Backup Schedule:**
- Daily incremental (VMs)
- Weekly full (VMs)
- Real-time replication (critical data)

### Monitoring Requirements

**Metrics to Collect:**
- CPU, RAM, disk utilization
- Network throughput
- Service uptime
- Failed login attempts
- Disk SMART status (TrueNAS)

**Alerting Thresholds:**
- Disk >85% full
- Service down >5 minutes
- RAM >90% utilized
- Failed SSH >5 attempts/hour

---

## VLAN 30: IoT Network

**Purpose:** Internet of Things devices, streaming boxes, gaming consoles  
**Subnet:** 10.0.30.0/24  
**Gateway:** 10.0.30.1 (OPNsense)

### Design Rationale

IoT devices are inherently untrustworthy:
- Rarely receive security updates
- Unknown/unverifiable firmware
- Potential for compromise (botnets, etc.)
- Should NEVER access internal infrastructure

**Critical Security Posture:** Internet access ONLY, complete internal isolation

### DHCP Configuration

**DHCP Range:** 10.0.30.100 - 10.0.30.200  
**Lease Time:** 12 hours (shorter for transient devices)

### Static Reservations

| IP | Device | MAC Address | Type |
|----|--------|-------------|------|
| 10.0.30.101 | PlayStation 5 | xx:xx:xx:xx:xx:xx | Gaming console |
| 10.0.30.102 | Firestick-LivingRoom | xx:xx:xx:xx:xx:xx | Streaming |
| 10.0.30.103 | Firestick-Bedroom | xx:xx:xx:xx:xx:xx | Streaming |
| 10.0.30.104 | [Smart TV] | xx:xx:xx:xx:xx:xx | If applicable |
| 10.0.30.105 | [Smart Speaker] | xx:xx:xx:xx:xx:xx | If applicable |

### Access Permissions

**Outbound Access:**
- ✅ Internet - HTTP, HTTPS, DNS, NTP, streaming protocols
- ✅ VLAN 20 - DNS to Pi-hole ONLY (10.0.20.20:53)
- ❌ VLAN 1 (Management) - **BLOCKED**
- ❌ VLAN 10 (Trusted) - **BLOCKED**
- ❌ VLAN 20 (Servers) - **BLOCKED** (except DNS)
- ❌ VLAN 40 (Cameras) - **BLOCKED**
- ❌ VLAN 99 (Security) - **BLOCKED**

**Inbound Access:**
- Only established/related connections
- No unsolicited inbound connections

### Firewall Rules (OPNsense)

**Default IoT VLAN Policy:**
```
Action: Block
Interface: VLAN30
Source: VLAN30 net
Destination: RFC1918 (All private subnets)
Log: Yes
Description: Block IoT from accessing internal networks
```

**Explicit Allow: Internet**
```
Action: Pass
Interface: VLAN30
Source: VLAN30 net
Destination: !RFC1918 (NOT private IPs)
Log: No
Description: Allow IoT internet access
```

**Explicit Allow: DNS to Pi-hole**
```
Action: Pass
Interface: VLAN30
Source: VLAN30 net
Destination: 10.0.20.25
Protocol: UDP
Port: 53
Description: Allow DNS to Pi-hole
```

### WiFi SSID Mapping

**SSID:** "HomeNetwork-IoT" (separate SSID recommended)  
**Security:** WPA2-Personal (some IoT devices don't support WPA3)  
**Client Isolation:** Enabled (devices can't see each other)

### Security Hardening

**Device Configuration:**
- Disable UPnP on all IoT devices
- Disable remote management features
- Use separate password per device (not same as WiFi)
- Update firmware when available (carefully - test first)

**Network Controls:**
- Application-layer filtering (Sensei) to detect anomalies
- Rate limiting to prevent DDoS participation
- Block known malicious IPs/domains (CrowdSec)

### Monitoring and Alerts

**What to Watch:**
- Unusual outbound connections (C2 servers)
- Port scanning attempts
- Traffic to RFC1918 addresses (should be blocked)
- Bandwidth abuse

**Alert Triggers:**
- IoT device attempting management VLAN access
- Connection to known malware domains
- Participation in DDoS (high connection count)
- Firmware update available (if trackable)

---

## VLAN 40: Camera/Security Network

**Purpose:** IP surveillance cameras  
**Subnet:** 10.0.40.0/24  
**Gateway:** 10.0.40.1 (OPNsense)

### Design Rationale

IP cameras pose unique security/privacy risks:
- Often manufactured overseas with questionable security
- Known for backdoors and vulnerabilities
- Contain sensitive video footage
- Frequently targeted by botnets

**Critical Security Posture:** Complete isolation from internet and other VLANs

### Static IP Assignments

| IP | Camera Location | MAC Address | Model |
|----|-----------------|-------------|-------|
| 10.0.40.101 | Front Door | xx:xx:xx:xx:xx:xx | [Brand/Model] |
| 10.0.40.102 | Backyard | xx:xx:xx:xx:xx:xx | [Brand/Model] |
| 10.0.40.103 | Garage | xx:xx:xx:xx:xx:xx | [Brand/Model] |
| 10.0.40.104 | Driveway | xx:xx:xx:xx:xx:xx | [Brand/Model] |
| 10.0.40.105 | Side Yard | xx:xx:xx:xx:xx:xx | [Brand/Model] |
| 10.0.40.10 | NVR VM (optional) | - | Recording server |

### Access Permissions

**Outbound Access:**
- ❌ Internet - **COMPLETELY BLOCKED** (unless cloud recording required)
- ❌ All VLANs - **BLOCKED** by default

**Inbound Access:**
- ✅ VLAN 1 (Management) - Configuration and monitoring
- ✅ VLAN 10 (Trusted) - Viewing camera feeds (RTSP, HTTP/S)
- ✅ VLAN 20 - If NVR VM is in this VLAN
- ❌ VLAN 30 (IoT) - **BLOCKED**
- ❌ VLAN 50 (Guest) - **BLOCKED**

### Firewall Rules

**Camera Isolation (Default Deny):**
```
Action: Block
Interface: VLAN40
Source: VLAN40 net
Destination: Any
Log: Yes
Description: Block all camera outbound traffic by default
```

**Allow Management Access:**
```
Action: Pass
Interface: VLAN1
Source: VLAN1 net
Destination: VLAN40 net
Protocol: TCP
Port: 80, 443, 554 (RTSP)
Description: Allow management VLAN to access cameras
```

**Allow Trusted Viewing:**
```
Action: Pass
Interface: VLAN10
Source: VLAN10 net
Destination: VLAN40 net
Protocol: TCP
Port: 554 (RTSP), 80, 443
Description: Allow trusted users to view camera feeds
```

**Optional: Cloud Recording**
If cloud recording is required (e.g., for alerts when away):
```
Action: Pass
Interface: VLAN40
Source: VLAN40 net
Destination: [Specific cloud IP/domain]
Protocol: TCP
Port: 443
Description: Allow cameras to upload to cloud service ONLY
Log: Yes
```

### Physical Security

**Switch Port Configuration:**
- Ports 4-8: Access mode, VLAN 40 only
- PoE enabled on these ports
- Port security: MAC address locking (prevent rogue device)
- Storm control: Prevent broadcast storms

### NVR Configuration (If Using)

**Option 1: Dedicated NVR VM**
- IP: 10.0.40.10
- Software: Frigate, Blue Iris, Zoneminder, etc.
- Storage: TrueNAS iSCSI or NFS mount
- Accessible from VLAN 1 and 10 for viewing

**Option 2: TrueNAS Plugin**
- Install NVR app directly on TrueNAS
- Cameras record to local storage
- Access via TrueNAS web UI

### Recording Strategy

**Retention:**
- Continuous: 7 days
- Motion-triggered: 30 days
- Critical events: Indefinite (manual review)

**Storage Requirements:**
- 5 cameras × [resolution] × [bitrate] × retention
- Estimated: [Calculate based on camera specs]
- Plan for 2x calculated capacity

### Privacy Considerations

**Data Protection:**
- Recording notification (if legally required)
- Camera views do NOT include neighbors' property
- Footage access logged and audited
- Encryption at rest on NVR storage
- No sharing of footage without consent

---

## VLAN 50: Guest Network

**Purpose:** Temporary access for visitors  
**Subnet:** 10.0.50.0/24  
**Gateway:** 10.0.50.1 (OPNsense)

### Design Rationale

Guest devices are completely untrusted:
- Unknown security posture
- Potentially compromised
- No relationship with homelab owner

**Security Posture:** Internet-only access, zero trust for internal resources

### DHCP Configuration

**DHCP Range:** 10.0.50.100 - 10.0.50.254  
**Lease Time:** 4 hours (short lease for transient devices)  
**DNS Servers:** 10.0.20.25 (Pi-hole for ad blocking)

### Access Permissions

**Outbound Access:**
- ✅ Internet - HTTP, HTTPS, DNS
- ✅ VLAN 20 - DNS to Pi-hole ONLY (10.0.20.20:53)
- ❌ All other VLANs - **COMPLETELY BLOCKED**

**Inbound Access:**
- Only established/related connections

### Firewall Rules

**Complete Internal Isolation:**
```
Action: Block
Interface: VLAN50
Source: VLAN50 net
Destination: RFC1918
Log: Yes
Description: Block guest from all internal networks
```

**Allow Internet:**
```
Action: Pass
Interface: VLAN50
Source: VLAN50 net
Destination: !RFC1918
Protocol: Any
Description: Allow guest internet access
```

### WiFi SSID Mapping

**SSID:** "HomeNetwork-Guest"  
**Security:** WPA2-Personal  
**Password:** Shared openly with guests  
**Client Isolation:** Enabled (guests can't see each other)  
**Captive Portal:** Optional - can add terms of use

### Rate Limiting

To prevent bandwidth abuse:
- Per-device limit: 10 Mbps down, 5 Mbps up
- Total VLAN limit: 50% of WAN bandwidth
- Priority: Lowest (QoS)

### Usage Policies

**Acceptable Use:**
- Web browsing
- Email
- Streaming (with rate limits)

**Prohibited:**
- Torrenting (blocked at firewall)
- Port scanning
- Bandwidth abuse

**Monitoring:**
- Log all connections
- Alert on suspicious activity
- Automatic disconnect after 24 hours (re-auth required)

---

## VLAN 99: Security Monitoring Network

**Purpose:** Intrusion detection, logging, and security monitoring  
**Subnet:** 10.0.99.0/24  
**Gateway:** 10.0.99.1 (OPNsense)

### Design Rationale

Security infrastructure must be:
- Isolated from production traffic
- Unable to be tampered with by attackers
- Highly available and monitored
- Capable of passive traffic analysis

### Static IP Assignments

| IP | Hostname | Service | Purpose |
|----|----------|---------|---------|
| 10.0.99.10 | security-onion | IDS/IPS, SIEM | Primary security monitoring |
| 10.0.99.11 | syslog-server (optional) | Log aggregation | Centralized logging |
| 10.0.99.20 | [Reserved] | Future security tools | Expansion |

### Access Permissions

**Outbound Access:**
- ✅ Internet - For threat intelligence feeds, updates
- ⚠️ All VLANs - Read-only (receives logs, SPAN traffic)
- ✅ VLAN 1 - For management

**Inbound Access:**
- ✅ VLAN 1 (Management) - Full administrative access
- ✅ VLAN 10 (Trusted) - View dashboards, reports
- ✅ All VLANs - Syslog/SNMP/NetFlow to Security Onion
- ❌ VLAN 30, 40, 50 - Cannot access security monitoring

### Security Onion Configuration

**Monitoring Interfaces:**
- Management: 10.0.99.10 (VLAN 99)
- Sniffing: SPAN/mirror port (promiscuous mode, no IP)

**Traffic Sources:**
- Switch SPAN port - copies all inter-VLAN traffic
- OPNsense NetFlow export - flow metadata
- Syslog from all devices - event logs
- SNMP traps - hardware alerts

**Ruleset:**
- Suricata ET Open rules (free)
- Custom rules for homelab environment
- Disabled noisy rules (tune over time)

### Log Aggregation

**Log Sources:**
- OPNsense: Authentication, firewall blocks, IDS alerts
- Proxmox: VM events, resource alerts
- TrueNAS: Storage health, access logs
- Docker: Container lifecycle events
- Switches/APs: Network events

**Retention:**
- Raw logs: 30 days
- Indexed/searchable: 90 days
- Critical alerts: Indefinite

### Alerting Strategy

**Severity Levels:**
- 🔴 Critical: Immediate response (SMS/email)
- 🟠 High: Review within 1 hour
- 🟡 Medium: Review daily
- 🟢 Low: Review weekly
- ⚪ Info: Logged only

**Alert Examples:**
- Critical: Camera attempting to access internet
- High: Failed SSH login from internet
- Medium: New device on network
- Low: Firewall rule hit unusual number of times

---

## Inter-VLAN Routing Summary

### Full Access Matrix

| Source ↓ / Destination → | VLAN 1 | VLAN 10 | VLAN 20 | VLAN 30 | VLAN 40 | VLAN 50 | VLAN 99 | Internet |
|--------------------------|--------|---------|---------|---------|---------|---------|---------|----------|
| **VLAN 1 (Mgmt)** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **VLAN 10 (Trusted)** | ✅ | ✅ | ✅* | ⚠️** | ✅*** | ❌ | ✅**** | ✅ |
| **VLAN 20 (Servers)** | ✅***** | ❌ | ✅ | ❌ | ❌ | ❌ | ✅****** | ✅ |
| **VLAN 30 (IoT)** | ❌ | ❌ | ⚠️******* | ❌ | ❌ | ❌ | ❌ | ✅ |
| **VLAN 40 (Cameras)** | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ⚠️******** |
| **VLAN 50 (Guest)** | ❌ | ❌ | ⚠️******* | ❌ | ❌ | ❌ | ❌ | ✅ |
| **VLAN 99 (Security)** | ✅ | ✅********* | ✅********* | ✅********* | ✅********* | ✅********* | ✅ | ✅ |

**Legend:**
- ✅ = Allowed (all or most services)
- ⚠️ = Limited (specific services only)
- ❌ = Blocked (default deny)

**Footnotes:**
- \* SSH, HTTP/S, SMB, NFS to servers
- \*\* HTTP/S only for IoT device configuration
- \*\*\* RTSP, HTTP/S for camera viewing
- \*\*\*\* HTTPS to Security Onion dashboard
- \*\*\*\*\* Management protocols (SSH, SNMP, etc.)
- \*\*\*\*\*\* Send logs to Security Onion
- \*\*\*\*\*\*\* DNS to Pi-hole ONLY (10.0.20.20:53)
- \*\*\*\*\*\*\*\* Optional cloud recording only
- \*\*\*\*\*\*\*\*\* Passive monitoring (SPAN/logs), no active connections

---

## Switch Configuration

### VLAN Trunking

**Trunk Ports:**
- Port 1: To OPNsense (all VLANs tagged)
- Port 3: To Proxmox 10G NIC (VLAN 20 tagged, others as needed)
- Port 10: To WiFi AP (VLANs 1, 10, 30, 50 tagged)

**Access Ports:** Each port assigned to specific VLAN (untagged)

### Port VLAN Assignments

| Ports | VLAN | Mode | Purpose |
|-------|------|------|---------|
| 1 | All | Trunk | Uplink to OPNsense |
| 2 | 1 | Access | Proxmox management (eno1) |
| 3 | 20 | Trunk | Proxmox 10G (VM traffic) |
| 4-8 | 40 | Access | IP Cameras (PoE) |
| 9 | 30 | Access | PlayStation 5 |
| 10 | Multiple | Trunk | WiFi AP (SSIDs → VLANs) |
| 11 | 10 | Access | Admin workstation |
| 12-24 | 10 | Access | General use (trusted devices) |
| 25 | 99 | Access | Security Onion management |
| 26 | - | Mirror | SPAN port to Security Onion |

### SPAN/Mirror Configuration

**Source:** All ports (or specific critical ports)  
**Destination:** Port 26  
**Traffic:** Bidirectional (Ingress + Egress)

This provides Security Onion with visibility into all network traffic.

---

## Implementation Checklist

### Phase 1: Planning (Current)
- [x] Design VLAN architecture
- [x] Define IP addressing scheme
- [x] Document firewall rules
- [ ] Finalize device assignments

### Phase 2: OPNsense Deployment
- [ ] Select hardware platform
- [ ] Install OPNsense OS
- [ ] Configure VLANs on OPNsense
- [ ] Set up DHCP servers per VLAN
- [ ] Configure basic firewall rules
- [ ] Test routing between VLANs

### Phase 3: Switch Configuration
- [ ] Back up current switch config
- [ ] Create VLANs on switch
- [ ] Assign ports to VLANs
- [ ] Configure trunk ports
- [ ] Set up SPAN/mirror
- [ ] Enable port security on camera ports

### Phase 4: WiFi AP Configuration
- [ ] Configure multiple SSIDs
- [ ] Map SSIDs to VLANs
- [ ] Set WPA2/WPA3 security
- [ ] Enable client isolation on IoT/Guest
- [ ] Test VLAN tagging

### Phase 5: Device Migration
- [ ] Migrate management devices (VLAN 1)
- [ ] Migrate admin workstations (VLAN 10)
- [ ] Migrate servers/VMs (VLAN 20)
- [ ] Migrate IoT devices (VLAN 30)
- [ ] Migrate cameras (VLAN 40)
- [ ] Test guest network (VLAN 50)

### Phase 6: Security Hardening
- [ ] Implement all firewall rules
- [ ] Deploy Security Onion (VLAN 99)
- [ ] Configure logging/alerting
- [ ] Enable IDS/IPS on OPNsense
- [ ] Test access controls
- [ ] Penetration test from each VLAN

### Phase 7: Documentation
- [ ] Update network diagrams
- [ ] Document all static IPs
- [ ] Create runbooks for common tasks
- [ ] Write disaster recovery procedures
- [ ] Archive configuration backups

---

## Troubleshooting Guide

### Common Issues

**Issue:** Device can't reach internet  
**Check:**
1. Correct VLAN assignment?
2. DHCP lease obtained?
3. Gateway configured (VLAN interface IP)?
4. Firewall rule allowing outbound?
5. NAT configured on OPNsense WAN?

**Issue:** Device can't access server in different VLAN  
**Check:**
1. Firewall rule exists for this traffic?
2. Rule placed above default deny?
3. Source/destination VLAN correct in rule?
4. Service port correct?
5. Check OPNsense live firewall log

**Issue:** VLAN routing not working at all  
**Check:**
1. OPNsense VLAN interfaces created?
2. VLANs assigned to physical interface?
3. VLAN interfaces enabled?
4. Switch trunk configured to OPNsense?
5. Switch VLANs created?

---

## Related Documents

- [Network Diagram - Current State](./02-network-diagram-current.md)
- [Network Diagram - Target State](./03-network-diagram-target.md)
- [IP Addressing Scheme](./05-ip-addressing.md)
- [Firewall Rules Documentation](../security/firewall-rules.md)
- [OPNsense Configuration Guide](../procedures/opnsense-configuration.md)

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-01-14 | Initial VLAN design | [Your Name] |
| | | |
