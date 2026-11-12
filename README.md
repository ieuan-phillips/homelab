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
        ISP router (gateway, DHCP)
             192.168.1.0/24
     /              |               \
 anvil (pve)      forge            clients
 Proxmox VE 9     Debian 13        laptop, PC: DNS via flux
 i7-6700, 8 GB    lab box          TV: Chromecast + Jellyfin
 2 TB HDD         (disk failing)
   |
   +-- CT 100 jellyfin  Jellyfin, Intel Quick Sync (iGPU passthrough), Samba share for media
   +-- CT 101 flux      AdGuard Home: DNS filtering, encrypted upstream (Quad9), local home.arpa names
   +-- CT 102 tongs     Gatus: service monitoring, alerts to phone (ntfy)
   +-- CT 103 bellows   Caddy reverse proxy: https://<service>.home.arpa with a local CA
```

Names follow a blacksmith theme: the Proxmox host is **anvil** (hostname `pve`), and each service is a tool of the forge.

| Host / guest | Role | Platform | Status |
|---|---|---|---|
| `anvil` (`pve`) | Hypervisor for all services | Proxmox VE 9 (Debian 13) | Updated, 2FA on web UI, key-only SSH, weekly guest backups, SMART tests and phone alerts |
| `jellyfin` (CT 100) | Media server | Unprivileged LXC, Debian 13 | Direct play and hardware transcoding (~12x real time) verified |
| `flux` (CT 101) | DNS filter | Unprivileged LXC, Debian 13 | ~400k blocking rules; opt-in per device |
| `tongs` (CT 102) | Monitoring | Unprivileged LXC, Debian 13 | 7 checks, config in [`config/tongs/`](config/tongs/) |
| `bellows` (CT 103) | Reverse proxy | Unprivileged LXC, Debian 13 | HTTPS for all web UIs, config in [`config/bellows/`](config/bellows/) |
| `forge` | Linux admin practice box | Debian 13 (no desktop) | Disk failing; offline / lab use only |

## Roadmap
- [x] **Phase 0 - Foundations:** Proxmox and Debian installed, static addressing, SSH key authentication, update repositories configured, disk health testing
- [x] **Phase 0 hardening:** TOTP two-factor on the hypervisor web UI, key-only SSH, disk-failure alerting to a phone, scheduled SMART self-tests, weekly guest backups *(still to do: SHA256 verification of install media)*
- [x] **Phase 1 - Media:** Jellyfin in an LXC container with Intel Quick Sync hardware transcoding, local playback
- [ ] **Phase 2 - Storage and backups:** separate media and backup storage, scheduled Proxmox backups, 3-2-1 plan
- [ ] **Phase 3 - Services:** ~~DNS sinkhole~~ (AdGuard Home done), ~~reverse proxy with local TLS~~ (Caddy done), Immich photo backup
- [ ] **Phase 4 - Network security:** OPNsense firewall on an isolated lab network, segmentation, IDS (Suricata), packet capture
- [ ] **Phase 5 - Monitoring:** ~~uptime monitoring~~ (Gatus done), metrics, Wazuh SIEM
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
  - [Proxmox host hardening basics](docs/runbooks/proxmox-hardening.md)
  - [Jellyfin LXC with Intel Quick Sync](docs/runbooks/jellyfin-lxc.md)
  - [AdGuard Home DNS filter](docs/runbooks/adguard-home-lxc.md)
  - [Gatus monitoring](docs/runbooks/gatus-monitoring.md)
  - [Caddy reverse proxy with local HTTPS](docs/runbooks/caddy-reverse-proxy.md)
- Incidents
  - [2026-11-11 - forge disk failed SMART extended test](docs/incidents/2026-11-11-forge-disk-failure.md)

## Principles
- Reuse existing hardware. Buy only when a real limitation is demonstrated.
- Prefer free and open-source software.
- Storage is not a backup.
- No secrets in this repository: no passwords, keys, MAC addresses, serial numbers or host fingerprints.
