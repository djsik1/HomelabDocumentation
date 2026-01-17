# IP Addressing Scheme

**Last Updated:** January 15, 2026
**Status:** Design Standard

## Overview

This document defines the complete IP addressing scheme for the homelab environment. Consistent, well-organized addressing is critical for troubleshooting, security, and documentation.

**Address Space:** 10.0.0.0/16 (Private Class A)  
**Subnetting Strategy:** /24 per VLAN (254 usable addresses each)  
**Allocation Method:** Structured hierarchy with room for growth

---

## Address Space Allocation

### VLAN Subnet Summary

| VLAN | Subnet | Mask | Usable IPs | Gateway | Broadcast |
|------|--------|------|------------|---------|-----------|
| 1 | 10.0.1.0 | /24 | 10.0.1.1 - 10.0.1.254 | 10.0.1.1 | 10.0.1.255 |
| 10 | 10.0.10.0 | /24 | 10.0.10.1 - 10.0.10.254 | 10.0.10.1 | 10.0.10.255 |
| 20 | 10.0.20.0 | /24 | 10.0.20.1 - 10.0.20.254 | 10.0.20.1 | 10.0.20.255 |
| 30 | 10.0.30.0 | /24 | 10.0.30.1 - 10.0.30.254 | 10.0.30.1 | 10.0.30.255 |
| 40 | 10.0.40.0 | /24 | 10.0.40.1 - 10.0.40.254 | 10.0.40.1 | 10.0.40.255 |
| 50 | 10.0.50.0 | /24 | 10.0.50.1 - 10.0.50.254 | 10.0.50.1 | 10.0.50.255 |
| 99 | 10.0.99.0 | /24 | 10.0.99.1 - 10.0.99.254 | 10.0.99.1 | 10.0.99.255 |

### Reserved Ranges

| Range | Purpose | Assignment Method |
|-------|---------|-------------------|
| 10.0.0.0/24 | Reserved (future use) | - |
| 10.0.X.1 | VLAN gateway (OPNsense) | Static |
| 10.0.X.2-99 | Infrastructure/servers | Static only |
| 10.0.X.100-200 | DHCP pool | Dynamic (with reservations) |
| 10.0.X.201-254 | Reserved for static expansion | Static only |

---

## VLAN 1: Management Network (10.0.1.0/24)

### Addressing Philosophy
- All static assignments
- No DHCP (prevents unauthorized devices)
- DNS via Pi-hole: 10.0.20.25
- Secondary DNS: 10.0.1.1 (OPNsense)

### Infrastructure Assignments

| IP | Hostname | Device Type | Interface | Notes |
|----|----------|-------------|-----------|-------|
| 10.0.1.1 | opnsense-mgmt | OPNsense | LAN/VLAN1 interface | Primary gateway, firewall management |
| 10.0.1.2 | switch-mgmt | TP-Link Switch | Management IP | Web UI, CLI access |
| 10.0.1.3 | ap-mgmt | WiFi AP | Management IP | Access point configuration |
| 10.0.1.10 | proxmox | Proxmox Host | eno1 (1Gbit) | Hypervisor management (CRITICAL) |
| 10.0.1.11 | idrac | iDRAC/iLO/IPMI | OOB port | Out-of-band management (if available) |
| 10.0.1.12-19 | reserved | Reserved | - | Future infrastructure devices |
| 10.0.1.20 | ups-mgmt | UPS Network Card | Ethernet | UPS management (if network-attached) |

### Reserved Blocks
- 10.0.1.21-49: Future network equipment
- 10.0.1.50-99: Emergency use (documented exceptions)
- 10.0.1.100-254: NEVER USED (management is static only)

---

## VLAN 10: Trusted Network (10.0.10.0/24)

### Addressing Philosophy
- Mix of static and DHCP
- Admin workstations get static DHCP reservations
- DNS: Pi-hole (10.0.20.25), fallback OPNsense (10.0.10.1)

### Gateway and DNS
- Gateway: 10.0.10.1 (OPNsense)
- Primary DNS: 10.0.20.25 (Pi-hole)
- Secondary DNS: 10.0.10.1 (OPNsense)

### Static Assignments

| IP | Hostname | Device | MAC Address | Notes |
|----|----------|--------|-------------|-------|
| 10.0.10.100 | ASUS_Monster | i9-12900KF Gaming PC | [MAC] | Primary admin workstation (64GB RAM) |
| 10.0.10.101 | WorkLT | i5-1334U Work Laptop | [MAC] | Mobile admin device (16GB RAM, 512GB NVMe) |
| 10.0.10.102 | admin-tablet | Tablet | [MAC] | Trusted tablet |
| 10.0.10.103-109 | reserved | Reserved | - | Future admin devices |

### DHCP Configuration
- Range: 10.0.10.110-200
- Lease Time: 24 hours
- Domain Name: homelab.local (or custom domain)
- NTP Server: 10.0.1.1 (OPNsense) or pool.ntp.org

### Reserved
- 10.0.10.201-254: Future static assignments

---

## VLAN 20: Server Network (10.0.20.0/24)

### Addressing Philosophy
- All static assignments
- No DHCP (servers should have predictable IPs)
- VMs grouped by function

### Gateway and DNS
- Gateway: 10.0.20.1 (OPNsense)
- Primary DNS: 10.0.20.25 (Pi-hole)
- Secondary DNS: 10.0.20.1 (OPNsense)

### Core Infrastructure VMs/Containers

| IP | Hostname | ID | Type | vCPU | RAM | Services |
|----|----------|----|----|------|-----|----------|
| 10.0.20.10 | truenas | 100 | VM | 6 | 8GB | NAS, SMB, NFS, iSCSI |
| 10.0.20.20 | docker-host | 101 | VM | 6 | 12GB | Docker engine, Twingate, Frigate, Plex |
| 10.0.20.25 | pihole | 102 | LXC | 1 | 512MB | DNS, ad blocking |
| 10.0.20.30 | claude-server | 103 | VM | 2 | 4GB | Claude AI instance |

### Docker Containers (on docker-host: 10.0.20.20)

| Container | Host Port | Internal Port | Service |
|-----------|-----------|---------------|---------|
| twingate | - | - | VPN connector (no exposed ports) |
| frigate | 5000 | 5000 | NVR Web UI |
| frigate | 8554 | 8554 | RTSP restream |
| frigate | 8555 | 8555 | WebRTC |
| plex | 32400 | 32400 | Media server (host network mode) |

**Service URLs:**
- Frigate NVR: http://10.0.20.20:5000
- Plex Media Server: http://10.0.20.20:32400/web

**Note:** Pi-hole runs as a dedicated LXC container (10.0.20.25), not in Docker.

### Future Server Allocations

| IP | Purpose | Status |
|----|---------|--------|
| 10.0.20.40 | NVR VM (if deployed) | Planned |
| 10.0.20.41 | Home Assistant (if deployed) | Planned |
| 10.0.20.42 | Additional service VM | Reserved |
| 10.0.20.43-49 | Application servers | Reserved |
| 10.0.20.50-69 | Database servers | Reserved |
| 10.0.20.70-89 | Development/testing VMs | Reserved |
| 10.0.20.90-99 | Temporary/scratch VMs | Reserved |

### Reserved
- 10.0.20.100-254: Future expansion

---

## VLAN 30: IoT Network (10.0.30.0/24)

### Addressing Philosophy
- Primarily DHCP with static reservations for important devices
- Predictable IPs for streaming/gaming (better firewall logging)

### Gateway and DNS
- Gateway: 10.0.30.1 (OPNsense)
- Primary DNS: 10.0.20.25 (Pi-hole)
- Secondary DNS: 10.0.30.1 (OPNsense)

### Static DHCP Reservations

| IP | Hostname | Device | MAC Address | Type |
|----|----------|--------|-------------|------|
| 10.0.30.101 | ps5 | PlayStation 5 | xx:xx:xx:xx:xx:10 | Gaming console |
| 10.0.30.102 | firestick-living | Firestick 1 | xx:xx:xx:xx:xx:11 | Streaming |
| 10.0.30.103 | firestick-bedroom | Firestick 2 | xx:xx:xx:xx:xx:12 | Streaming |
| 10.0.30.104 | smart-tv | Smart TV | xx:xx:xx:xx:xx:13 | Television (if applicable) |
| 10.0.30.105 | smart-speaker | Smart Speaker | xx:xx:xx:xx:xx:14 | Voice assistant (if applicable) |
| 10.0.30.106-110 | reserved | Future IoT | - | Smart home devices |

### DHCP Configuration
- Range: 10.0.30.111-200
- Lease Time: 12 hours (shorter for transient devices)
- DNS: Pi-hole for ad blocking
- NTP: OPNsense

### Reserved
- 10.0.30.201-254: Avoid (DHCP exhaustion buffer)

---

## VLAN 40: Camera Network (10.0.40.0/24)

### Addressing Philosophy
- Static IPs for all cameras (required for NVR software)
- Small DHCP pool for testing/troubleshooting
- Sequential numbering by physical location

### Gateway and DNS
- Gateway: 10.0.40.1 (OPNsense)
- DNS: 10.0.20.25 (Pi-hole) - cameras should not need DNS
- Secondary: 10.0.40.1

### IP Camera Assignments

| IP | Hostname | Location | MAC Address | Model | PoE Port |
|----|----------|----------|-------------|-------|----------|
| 10.0.40.101 | cam-front-door | Front door | xx:xx:xx:xx:xx:20 | [Brand/Model] | Port 4 |
| 10.0.40.102 | cam-backyard | Backyard | xx:xx:xx:xx:xx:21 | [Brand/Model] | Port 5 |
| 10.0.40.103 | cam-garage | Garage | xx:xx:xx:xx:xx:22 | [Brand/Model] | Port 6 |
| 10.0.40.104 | cam-driveway | Driveway | xx:xx:xx:xx:xx:23 | [Brand/Model] | Port 7 |
| 10.0.40.105 | cam-sideyard | Side yard | xx:xx:xx:xx:xx:24 | [Brand/Model] | Port 8 |

### NVR and Management

| IP | Hostname | Device | Purpose |
|----|----------|--------|---------|
| 10.0.40.10 | nvr | NVR VM/Appliance | Video recording (if deployed) |
| 10.0.40.11-20 | reserved | Reserved | Future cameras or NVR failover |

### DHCP Configuration (Limited)
- Range: 10.0.40.100-110 (small range for testing)
- Lease Time: 1 hour
- Purpose: Testing new cameras before static assignment

### Reserved
- 10.0.40.111-254: Future camera expansion

---

## VLAN 50: Guest Network (10.0.50.0/24)

### Addressing Philosophy
- Pure DHCP (guests don't need static IPs)
- Large pool for many simultaneous guests
- Short lease times for turnover

### Gateway and DNS
- Gateway: 10.0.50.1 (OPNsense)
- Primary DNS: 10.0.20.25 (Pi-hole) - ad blocking for guests
- Secondary DNS: 10.0.50.1

### DHCP Configuration
- Range: 10.0.50.100-254 (154 available addresses)
- Lease Time: 4 hours
- Maximum Leases: 154 (effectively unlimited for home use)

### Reserved
- 10.0.50.1-99: Reserved (static if ever needed, unlikely)

---

## VLAN 99: Security Monitoring (10.0.99.0/24)

### Addressing Philosophy
- All static assignments
- Security-critical infrastructure
- Predictable IPs for firewall rules

### Gateway and DNS
- Gateway: 10.0.99.1 (OPNsense)
- Primary DNS: 10.0.20.25 (Pi-hole)
- Secondary DNS: 10.0.99.1

### Security Infrastructure

| IP | Hostname | Service | VM Specs | Purpose |
|----|----------|---------|----------|---------|
| 10.0.99.10 | security-onion | Security Onion | 8vCPU, 16GB RAM | IDS/IPS, SIEM, monitoring |
| 10.0.99.11 | syslog | Syslog Server (optional) | 2vCPU, 4GB RAM | Centralized logging |
| 10.0.99.20 | [Reserved] | Future security tool | - | Expansion |

### Reserved
- 10.0.99.21-99: Security tools expansion
- 10.0.99.100-254: Avoid

---

## DNS Configuration

### DNS Hierarchy

**Primary DNS Server:** Pi-hole (10.0.20.25)
- Blocks ads and trackers
- Forwards to upstream: Cloudflare (1.1.1.1) or Quad9 (9.9.9.9)
- Provides local DNS resolution for homelab.local domain

**Secondary DNS:** OPNsense VLAN gateways
- VLAN 1: 10.0.1.1
- VLAN 10: 10.0.10.1
- VLAN 20: 10.0.20.1
- VLAN 30: 10.0.30.1
- VLAN 40: 10.0.40.1
- VLAN 50: 10.0.50.1
- VLAN 99: 10.0.99.1

### Local DNS Records (Pi-hole)

**A Records:**
- opnsense.homelab.local → 10.0.1.1
- proxmox.homelab.local → 10.0.1.10
- truenas.homelab.local → 10.0.20.10
- docker.homelab.local → 10.0.20.20
- pihole.homelab.local → 10.0.20.25
- security.homelab.local → 10.0.99.10

**CNAME Records (optional):**
- nas.homelab.local → truenas.homelab.local
- firewall.homelab.local → opnsense.homelab.local
- frigate.homelab.local → docker.homelab.local
- plex.homelab.local → docker.homelab.local

### Reverse DNS Zones

Configured on OPNsense for PTR lookups:
- 1.0.10.in-addr.arpa (VLAN 1)
- 10.0.10.in-addr.arpa (VLAN 10)
- 20.0.10.in-addr.arpa (VLAN 20)
- 30.0.10.in-addr.arpa (VLAN 30)
- 40.0.10.in-addr.arpa (VLAN 40)
- 50.0.10.in-addr.arpa (VLAN 50)
- 99.0.10.in-addr.arpa (VLAN 99)

---

## DHCP Server Configuration

### DHCP Servers per VLAN (All on OPNsense)

| VLAN | DHCP Enabled | Range | Lease Time | Special Options |
|------|--------------|-------|------------|-----------------|
| 1 | ❌ No | - | - | Static only |
| 10 | ✅ Yes | .110-.200 | 24h | NTP, domain name |
| 20 | ❌ No | - | - | Static only |
| 30 | ✅ Yes | .111-.200 | 12h | NTP, domain name |
| 40 | ⚠️ Limited | .100-.110 | 1h | Testing only |
| 50 | ✅ Yes | .100-.254 | 4h | Guest network |
| 99 | ❌ No | - | - | Static only |

### DHCP Options Distributed

**All VLANs:**
- Option 3 (Router): VLAN gateway IP
- Option 6 (DNS): 10.0.20.25 (Pi-hole), then gateway
- Option 15 (Domain Name): homelab.local
- Option 42 (NTP): 10.0.1.1 (OPNsense) or pool.ntp.org

**VLAN 10 (Trusted) - Additional:**
- Option 119 (Domain Search): homelab.local

**VLAN 50 (Guest) - Additional:**
- Option 252 (WPAD): None (disable auto-proxy for security)

---

## Subnet Capacity Planning

### Current Usage vs. Capacity

| VLAN | Subnet | Available | Assigned | DHCP Pool | Reserved | % Utilized |
|------|--------|-----------|----------|-----------|----------|------------|
| 1 | /24 | 254 | ~10 | 0 | 244 | 4% |
| 10 | /24 | 254 | ~3 static | 91 DHCP | 160 | 37% |
| 20 | /24 | 254 | ~3 | 0 | 251 | 1% |
| 30 | /24 | 254 | ~5 static | 90 DHCP | 159 | 37% |
| 40 | /24 | 254 | ~6 | 11 DHCP | 237 | 7% |
| 50 | /24 | 254 | 0 | 155 DHCP | 99 | 61% |
| 99 | /24 | 254 | ~2 | 0 | 252 | 1% |

**Scalability:** Plenty of headroom in all VLANs. Current design supports:
- Management: 200+ infrastructure devices
- Trusted: 90+ simultaneous devices
- Servers: 250+ VMs/services
- IoT: 90+ smart devices
- Cameras: 150+ cameras
- Guest: 155+ simultaneous guests
- Security: 250+ monitoring tools

---

## IP Address Management (IPAM)

### Tracking Method

**Documentation:**
- This markdown file (source of truth)
- OPNsense DHCP static mappings
- Spreadsheet backup (optional, for non-technical stakeholders)

**Update Procedure:**
1. Assign IP in OPNsense DHCP static mapping OR set static on device
2. Update this document
3. Update network diagram if new device type
4. Commit to Git with descriptive message

### Naming Convention

**Hostnames:**
- Lowercase only
- Hyphen-separated (no underscores, spaces, special chars)
- Format: `[function]-[location/identifier]`
- Examples: cam-front-door, firestick-living, admin-laptop

**DNS Domain:**
- Internal: homelab.local (or custom domain)
- External: N/A (not exposing services publicly)

---

## Migration Planning

### Current to Target State

**Pre-Migration:**
1. Document all current IP assignments
2. Create IP transition map (old → new)
3. Back up all device configurations

**Migration Strategy:**
- VLAN-by-VLAN approach
- Start with management (VLAN 1)
- Test thoroughly before moving to next VLAN
- Rollback plan: revert to flat network

### IP Transition Map

| Device | Current IP | Target IP | VLAN | Notes |
|--------|-----------|-----------|------|-------|
| Proxmox | [Current] | 10.0.1.10 | 1 | CRITICAL - update first |
| Switch | [Current] | 10.0.1.2 | 1 | Update via console |
| WiFi AP | [Current] | 10.0.1.3 | 1 | May lose connectivity during |
| TrueNAS | [Current] | 10.0.20.10 | 20 | Update share paths |
| Docker Host | 192.168.1.176 | 10.0.20.20 | 20 | Docker/Twingate |
| Pi-hole (LXC) | 192.168.1.254 | 10.0.20.25 | 20 | DNS server - update all clients |
| Claude | [Current] | 10.0.20.30 | 20 | Low impact |
| Cameras | [Current] | 10.0.40.101-105 | 40 | Update NVR configuration |
| PS5 | [Current] | 10.0.30.101 | 30 | Let DHCP assign |
| Firesticks | [Current] | 10.0.30.102-103 | 30 | Let DHCP assign |

---

## Troubleshooting Reference

### Quick Reference Table

| I need to... | Look here... |
|-------------|--------------|
| Find a device's current IP | This document or OPNsense DHCP leases |
| Assign a new server IP | VLAN 20 reserved range (10.0.20.41+) |
| Add a new IoT device | Let DHCP assign, then create static reservation if needed |
| Add a new camera | Next available in 10.0.40.106+ range |
| Check if IP is in use | Ping test + OPNsense DHCP leases page |

### Common IP Issues

**Issue:** IP conflict detected  
**Solution:**
1. Check this document for assignment
2. Check OPNsense DHCP leases
3. If rogue device, investigate and remove
4. Update documentation if legitimate

**Issue:** Device not getting DHCP address  
**Solution:**
1. Verify device on correct VLAN
2. Check OPNsense DHCP server is enabled for VLAN
3. Check DHCP pool not exhausted
4. Review firewall rules (DHCP uses UDP 67/68)

**Issue:** Can't resolve hostnames  
**Solution:**
1. Check DNS server configured (should be 10.0.20.25)
2. Verify Pi-hole is running
3. Test direct query: `nslookup google.com 10.0.20.25`
4. Check firewall allows DNS (UDP 53)

---

## Related Documents

- [Hardware Inventory](./01-hardware-inventory.md)
- [Network Diagram - Current](./02-network-diagram-current.md)
- [Network Diagram - Target](./03-network-diagram-target.md)
- [VLAN Design](./04-vlan-design.md)
- [OPNsense Configuration](../procedures/opnsense-configuration.md)

---

## Change Log

| Date | Change | Affected IPs | Author |
|------|--------|--------------|--------|
| 2026-01-14 | Initial IP scheme design | All | [Your Name] |
| 2026-01-16 | Added Frigate/Plex services on docker-host | 10.0.20.20 | [Your Name] |
| | | | |
