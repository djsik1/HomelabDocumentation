# CIFS/SMB Shares

**Last Updated:** January 15, 2026
**Status:** Current State Documentation

## Purpose

This document tracks all CIFS/SMB file shares hosted on TrueNAS SCALE. These shares provide network storage for media, backups, and project files accessible to trusted devices on the network.

---

## Storage Pools Overview

| Pool | Status | Disks | Used | Available | Share |
|------|--------|-------|------|-----------|-------|
| S1 | DEGRADED | 1 | 53% | 850 GiB | WS1 |
| S2 | ONLINE | 1 | 71% | 264 GiB | WS2 |

**Storage Server:** TrueNAS SCALE (VM 100)
**Current IP:** 192.168.1.177
**Target IP:** 10.0.20.10 (VLAN 20 - Servers)

---

## Pool Details

### Pool S1

**Path:** `/mnt/S1`
**Status:** DEGRADED
**Total Disks:** 1 (data vdev)
**Used Space:** 53%
**Available Space:** 850.21 GiB

**Health Warning:** Pool is in DEGRADED state. Investigation required.
- [ ] Identify cause of degradation
- [ ] Check disk SMART status
- [ ] Plan remediation (replace disk, add mirror, etc.)

### Pool S2

**Path:** `/mnt/S2`
**Status:** ONLINE
**Total Disks:** 1 (data vdev)
**Disks with Errors:** 0
**Used Space:** 71%
**Available Space:** 264.39 GiB

---

## CIFS Shares

### WS1 - Media & Projects

| Property | Value |
|----------|-------|
| **Share Name** | WS1 |
| **Path** | `/mnt/S1/WS1` |
| **Pool** | S1 |
| **Purpose** | TV shows, coding projects |
| **Status** | Active (on DEGRADED pool) |

**Contents:**
- TV shows (media library)
- Coding/development projects

**Access:**
| Device | Access Level | VLAN | Notes |
|--------|--------------|------|-------|
| ASUS_Monster | Read/Write | 10 (Trusted) | Primary workstation |
| WorkLT | Read/Write | 10 (Trusted) | Work laptop |

**UNC Path (Current):** `\\192.168.1.177\WS1`
**UNC Path (Target):** `\\10.0.20.10\WS1`

---

### WS2 - Media Library

| Property | Value |
|----------|-------|
| **Share Name** | WS2 |
| **Path** | `/mnt/S2/WS2` |
| **Pool** | S2 |
| **Purpose** | Movies, music |
| **Status** | Active |

**Contents:**
- Movies (media library)
- Music collection

**Access:**
| Device | Access Level | VLAN | Notes |
|--------|--------------|------|-------|
| ASUS_Monster | Read/Write | 10 (Trusted) | Primary workstation |
| WorkLT | Read/Write | 10 (Trusted) | Work laptop |

**UNC Path (Current):** `\\192.168.1.177\WS2`
**UNC Path (Target):** `\\10.0.20.10\WS2`

---

## Network Access Requirements

### Current State (Flat Network)
All devices on 192.168.1.0/24 can access shares directly.

### Target State (VLAN Segmentation)

| Source VLAN | Access | Notes |
|-------------|--------|-------|
| VLAN 1 (Management) | Full | Infrastructure administration |
| VLAN 10 (Trusted) | Full | Primary media consumers |
| VLAN 20 (Servers) | Local | TrueNAS resides here |
| VLAN 30 (IoT) | Blocked | No NAS access for IoT |
| VLAN 40 (Cameras) | Blocked | Cameras isolated |
| VLAN 50 (Guest) | Blocked | No internal access |

**Firewall Rules Required:**
- Allow VLAN 10 → VLAN 20 on TCP 445 (SMB)
- Allow VLAN 10 → VLAN 20 on TCP 139 (NetBIOS)
- Block all other VLANs from accessing SMB ports

---

## Authentication

**Method:** Local TrueNAS User Account

| Property | Value |
|----------|-------|
| **Username** | priest |
| **Password** | [REDACT BEFORE SHARING] |
| **Guest Access** | Disabled |
| **AD Integration** | No |

**Security Note:** Store credentials in a password manager. Sanitize this document before public sharing.

---

## Backup Considerations

| Share | Backup Status | Backup Target | Frequency |
|-------|---------------|---------------|-----------|
| WS1 | [TBD] | [TBD] | [TBD] |
| WS2 | [TBD] | [TBD] | [TBD] |

**Note:** Single-disk pools have NO redundancy. Disk failure = data loss.

**Recommendations:**
- [ ] Implement 3-2-1 backup strategy
- [ ] Consider adding mirror vdevs to pools
- [ ] Set up offsite backup for critical data (coding projects)

---

## Performance Notes

**Protocol:** SMB3
**Connection Speed:** 1 Gbit/s (current), 10 Gbit/s available via X540

**Optimization Opportunities:**
- [ ] Enable 10G connectivity to TrueNAS VM
- [ ] Configure SMB multichannel if multiple NICs available
- [ ] Tune SMB settings for large file transfers (media)

---

## Troubleshooting

### Common Issues

**Cannot connect to share:**
1. Verify TrueNAS VM is running (Proxmox → VM 100)
2. Check network connectivity: `ping 192.168.1.177`
3. Verify SMB service is running in TrueNAS
4. Check Windows credentials manager for stale entries

**Slow transfer speeds:**
1. Check network link speed (should be 1G minimum)
2. Verify no disk errors in TrueNAS
3. Check pool health status
4. Review SMB protocol version negotiation

---

## Change Log

| Date       | Change                            | Author |
| ---------- | --------------------------------- | ------ |
| 2026-01-15 | Initial CIFS shares documentation | Priest |

