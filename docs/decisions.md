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

## 010 - Gatus for service monitoring, config in git
**Date:** 2026-11-12  
**Decision:** Monitor the lab with Gatus in a small LXC (`tongs`), built from the tagged source, configured from a YAML file kept in this repo, with alerts to the same ntfy channel as Proxmox.  
**Why:** Uptime Kuma's current release needs a newer Node.js than Debian ships and stores its settings in a database. Gatus is one Go binary, uses about 30 MB of RAM, and its whole configuration is a text file that can be reviewed and versioned ("monitoring as code"). Secrets (the ntfy topic) stay out of the repo in an environment file.

## 011 - Caddy reverse proxy with a local certificate authority
**Date:** 2026-11-12  
**Decision:** Put the web UIs behind Caddy in a small LXC (`bellows`) with HTTPS from Caddy's internal CA, under `*.home.arpa` names served by AdGuard Home. Devices trust the CA's root certificate once. Direct IP:port access stays available for clients that can't use the names yet (Chromecast, Samba).  
**Why:** One front door with clean names and TLS everywhere, without exposing anything to the internet. Public CAs can't issue certificates for `home.arpa`, and buying a domain only for this isn't justified yet. Caddy verifies Proxmox's own certificate against the Proxmox cluster CA rather than skipping verification.  
**Risk accepted:** whoever holds the CA's private key could impersonate any site to devices that trust it. Mitigations: the key never leaves `bellows`, which runs nothing else and allows key-only SSH; it is included in guest backups. A future improvement is a CA with name constraints limited to `home.arpa`.

## 012 - OPNsense lab on an internal-only bridge
**Date:** 2026-11-19  
**Decision:** Run OPNsense as a VM (`hearth`, VM 200) with WAN on the home network (`vmbr0`) and LAN on a new bridge `vmbr1` that has no physical port and no host IP. Lab machines sit only on `vmbr1` (`10.10.10.0/24`). The first lab client is a small unprivileged LXC (`lab-client`, CT 104), not a full VM. The lab is IPv4 only for now.  
**Why:** With no port and no host IP on `vmbr1`, the only way in or out of the lab is through the firewall, so rules can be tested without touching the household network (see 005). An LXC proves DHCP, NAT and rules just as well as a VM, with no OS install and a quarter of the RAM on an 8 GB host. IPv6 is left out until IPv4 rules are understood.  
**Rules so far:** only the admin laptop may reach the firewall's web GUI from the home network (port 443), and the lab may not reach `192.168.1.0/24` but keeps internet access. Steps and test results: [OPNsense lab runbook](runbooks/opnsense-lab.md).

## 013 - Suricata IDS on the lab interface, alert only
**Date:** 2026-11-20  
**Decision:** Run Suricata on `hearth` in IDS mode (PCAP, alert only) on the **LAN** interface, with a small set of ET Open rules updated daily, and **Home networks = `10.10.10.0/24`** (the lab only). SSH to the firewall is key-only and only from the admin laptop.  
**Why:** Alert-only cannot break the lab while rules are being learnt. On LAN, alerts show the real lab host (on WAN everything is NATed to one address). Home networks should list what the firewall protects; the home network is outside `hearth`, so it counts as external. A test showed this setting decides whether scan rules fire at all. Only five rulesets, because the VM has 2 GB RAM. Steps and test results: [OPNsense lab runbook](runbooks/opnsense-lab.md).

## 014 - DMZ segment for lab servers
**Date:** 2026-11-21  
**Decision:** Add a second lab segment, `DMZ` (`10.10.20.0/24`, bridge `vmbr2`, hearth `vtnet2`), for servers. The lab LAN may reach the DMZ web server on port 80 only; the DMZ may use the firewall for DNS and reach the internet, but not the firewall itself, the lab LAN or the home network. Suricata watches both LAN and DMZ, with Home networks kept to the lab LAN.  
**Why:** Servers that accept connections are the likeliest to be compromised. Keeping them in their own zone with default deny means a compromised server cannot reach clients or the home network, and the IDS records the attempt (tested: a scan from the DMZ server was blocked and raised ET SCAN alerts on the DMZ interface). Servers get static addresses so rules can name them. Steps and results: [OPNsense lab runbook](runbooks/opnsense-lab.md).

