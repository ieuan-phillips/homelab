# Hardware

All reused hardware. Nothing bought yet.

## pve - Proxmox host (old gaming desktop)
| Part | Detail |
|---|---|
| Motherboard | MSI Z270 Gaming Plus |
| CPU | Intel Core i7-6700, 4C/8T, VT-x and VT-d enabled |
| iGPU | Intel HD Graphics 530 (Quick Sync). Active since the GTX 745 was removed; passed into the Jellyfin container. Board video out is DisplayPort only; the server runs headless |
| dGPU | None. GTX 745 removed 2026-11-12 (see decision 006) |
| RAM | 8 GB DDR4 (board supports up to 64 GB) |
| Storage | 1 x 2 TB 5400-class HDD (OS + guests, for now) |
| PSU | EVGA SuperNOVA 550 G3 (80+ Gold), ECO (semi-passive fan) mode on |
| Network | 1 x Gigabit Ethernet |

**Disk health (2026-11-11):** SMART attributes clean (0 reallocated, pending or uncorrectable sectors; empty error log; about 4,300 power-on hours). All data partitions are 4K-aligned. Extended self-test **completed without error** (2026-11-12). `Load_Cycle_Count` is being watched, because WD 5400-rpm drives park their heads aggressively and that adds up under 24/7 server use.

**Known limits:** 8 GB RAM will constrain running Jellyfin, Immich and a SIEM together. The single disk means no redundancy and nowhere local for backups.

## forge - Debian server (old laptop)
| Part | Detail |
|---|---|
| CPU | Intel Core i7-2670QM, 4C/8T |
| RAM | 4 GB |
| Storage | 500 GB 5400 RPM HDD - **failed SMART extended test**, see [incident](incidents/2026-11-11-forge-disk-failure.md) |
| Firmware | Legacy BIOS |

Chose plain Debian over Proxmox here because 4 GB RAM leaves little for guests after the hypervisor.

## Network
- ISP router handles routing, DHCP and DNS. It does not support VLANs.
- Servers use addresses outside the DHCP pool (static or reserved).
- One Ethernet run to the lab location; an unmanaged switch is the first planned network purchase.
