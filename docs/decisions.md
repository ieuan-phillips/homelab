# Decisions

Short records of choices and the reasoning behind them. Newest last.

## 001 - Proxmox VE on the desktop
**Date:** 2026-11  
**Decision:** Install Proxmox VE directly on the i7-6700 desktop and run every service as a VM or container, not on the host itself.  
**Why:** A type-1 hypervisor mirrors how enterprises run infrastructure (VMware, Nutanix). Keeping services in guests means each can be backed up, snapshotted, rebuilt or destroyed without risking the host.

## 002 - Plain Debian on the laptop
**Date:** 2026-11  
**Decision:** Debian 13 with no desktop, instead of Proxmox.  
**Why:** 4 GB RAM. A hypervisor plus guests would leave almost nothing usable. Plain Debian is good for learning SSH, systemd, nginx and hardening directly.

## 003 - Proxmox no-subscription repository
**Date:** 2026-11-11  
**Decision:** Disable the enterprise repositories and enable `pve-no-subscription`.  
**Why:** The enterprise repos need a paid key, so `apt update` fails without one. No-subscription receives the same packages slightly earlier and with less testing. That's acceptable for a lab. No third-party "nag removal" scripts were used.

## 004 - Treat forge's disk as failed
**Date:** 2026-11-11  
**Decision:** Use forge only for disposable learning; no important data or relied-on services.  
**Why:** The SMART extended self-test failed with a read error, even though the overall SMART status still says PASSED. See the [incident write-up](incidents/2026-11-11-forge-disk-failure.md).

## 005 - Keep the ISP router in charge for now
**Date:** 2026-11  
**Decision:** No OPNsense on the live network yet; no port forwarding.  
**Why:** The goal is a working local media server first. Firewall and segmentation work will start on an isolated virtual lab network, so mistakes can't take the household offline.

## 006 - Use the Intel iGPU for Jellyfin; remove the GTX 745
**Date:** 2026-11-11  
**Decision:** Remove the discrete NVIDIA GTX 745 and use the i7-6700's Intel HD 530 (Quick Sync) for hardware transcoding.  
**Why:** The iGPU uses an open driver built into the kernel and is easy to share with an LXC container. It handles H.264 and HEVC 8-bit (the GTX 745's NVENC is H.264 only). The proprietary NVIDIA driver for Maxwell is near end of support. The card adds idle power draw around the clock. The main client (Chromecast with Google TV 4K) can play most formats directly anyway, so transcoding demand will be low.

## 007 - Jellyfin in an unprivileged LXC container
**Date:** 2026-11-12  
**Decision:** Run Jellyfin from the official Jellyfin apt repository in an unprivileged Debian 13 LXC (CT 100), with the iGPU's render node passed through and media on a separate 500 GB volume.  
**Why:** A container shares the host kernel, so it uses far less RAM than a VM on an 8 GB machine and needs no PCI passthrough for the iGPU. Unprivileged means root inside the container maps to an unprivileged user on the host, which limits the damage if Jellyfin is compromised. The media volume is separate from the system disk, starts small and can be grown later (thin volumes are easy to grow, hard to shrink). Login is by SSH key only.


## 008 - Hardening before more services
**Date:** 2026-11-12  
**Decision:** TOTP 2FA (plus recovery keys) on the Proxmox web UI; key-only SSH; a weekly `vzdump` job for guest system disks (keep 2 latest + 4 weekly); push alerts to a phone through a Proxmox webhook target (ntfy); scheduled SMART self-tests (short weekly, long monthly).  
**Why:** A server should tell me when it is failing instead of me finding out later. Email alerts were tried first, but mail sent directly from a home IP is rejected by large providers, so a push service is simpler and needs no mail relay. Backups currently share the single HDD, so they protect against mistakes, not disk failure; a second disk is a Phase 2 item.

## 009 - AdGuard Home as the network's DNS filter, per device first
**Date:** 2026-11-12  
**Decision:** Run AdGuard Home in a small unprivileged LXC with encrypted upstream DNS (Quad9, DoH/DoT, DNSSEC) and local `home.arpa` names. Point devices at it individually rather than through DHCP.  
**Why:** The ISP router cannot hand out a custom DNS server. Taking over DHCP would make the whole household depend on one machine with no failover yet, so devices opt in one at a time; whole-network DNS comes later, with a fallback plan. `home.arpa` is the name reserved for home networks (RFC 8375).
