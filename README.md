# Homelab

A self-hosted lab built from reused hardware. I use it to learn virtualisation, Linux administration, networking and security tooling, and to host my own media.

Everything here is documented as I build it: what I did, why, what went wrong, and how I fixed it.

## Goals
1. **Own my media.** Host my film library on my own hardware with Jellyfin, instead of relying on streaming licences that can disappear.
2. **Learn enterprise skills on free tools.** Proxmox in place of VMware/Nutanix, OPNsense in place of commercial firewalls, Wazuh as a SIEM, Wireshark for packet analysis.
3. **Work safely.** Nothing is exposed to the internet until it is understood, hardened and monitored.

## Current architecture

```
            Internet
                |
     ISP router (gateway, DHCP, DNS)
        192.168.1.0/24
       /        |          \
   pve        forge        admin laptop
 Proxmox VE   Debian 13    (SSH, web UI)
 i7-6700      i7-2670QM
 8 GB RAM     4 GB RAM
 2 TB HDD     500 GB HDD (failing - lab use only)
```

| Host | Role | OS | Status |
|---|---|---|---|
| `pve` | Hypervisor for all services | Proxmox VE 9 (Debian 13) | Updated, reboot-tested |
| `forge` | Linux admin practice box | Debian 13 (no desktop) | SSH key auth, nginx; disk failing |

## Roadmap
- [x] **Phase 0 - Foundations:** Proxmox and Debian installed, static addressing, SSH key authentication, update repositories configured, disk health testing
- [ ] **Phase 0 hardening:** TOTP two-factor on the hypervisor web UI, disk-failure alerting, SHA256 verification of install media
- [ ] **Phase 1 - Media:** Jellyfin in an LXC container with Intel Quick Sync hardware transcoding, local playback
- [ ] **Phase 2 - Storage and backups:** separate media and backup storage, scheduled Proxmox backups, 3-2-1 plan
- [ ] **Phase 3 - Services:** DNS sinkhole (Pi-hole/AdGuard Home), Immich photo backup, reverse proxy with local TLS
- [ ] **Phase 4 - Network security:** OPNsense firewall on an isolated lab network, segmentation, IDS (Suricata), packet capture
- [ ] **Phase 5 - Monitoring:** Wazuh SIEM, uptime and metrics
- [ ] **Phase 6 - Clustering:** second Proxmox node with a QDevice tie-breaker

### Future plans
- Self-hosted media streaming (films and music) to replace paid subscriptions
- Secure remote access over a VPN, with no services exposed directly to the internet
- Network segmentation, so untrusted devices are isolated from the rest of the network
- Security monitoring with a SIEM, and an Active Directory attack-and-detection lab
- Automated, tested backups

## Documentation
- [Hardware](docs/hardware.md)
- [Decisions](docs/decisions.md) - what I chose and why
- Runbooks
  - [Updating Proxmox without a subscription](docs/runbooks/proxmox-updates.md)
- Incidents
  - [2026-11-11 - forge disk failed SMART extended test](docs/incidents/2026-11-11-forge-disk-failure.md)

## Principles
- Reuse existing hardware. Buy only when a real limitation is demonstrated.
- Prefer free and open-source software.
- Storage is not a backup.
- No secrets in this repository: no passwords, keys, MAC addresses, serial numbers or host fingerprints.
