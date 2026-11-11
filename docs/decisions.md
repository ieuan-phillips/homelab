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
