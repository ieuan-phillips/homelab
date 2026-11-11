# Runbook: Proxmox VE Host Updates (No-Subscription)

| | |
|---|---|
| **Applies to** | `pve` (Proxmox VE 9.x, Debian 13 "trixie") |
| **Frequency** | Weekly, or when a security advisory is published |
| **Downtime** | None for package installation; a short outage if a kernel update requires a reboot |
| **Last performed** | 2026-11-11 (212 packages, kernel 7.0.2-6 → 7.0.14-28) |

## Purpose
Apply Debian and Proxmox package updates to the hypervisor without a paid subscription. Do it safely, and confirm the host recovers after a reboot.

## Prerequisites
- Repository configuration (one-time, under *Node → Updates → Repositories*):
  - `pve-enterprise` and `ceph` enterprise repositories **disabled** (they need a subscription key and return `401 Unauthorized` without one)
  - `pve-no-subscription` repository **enabled**
  - Debian `trixie`, `trixie-updates` and `trixie-security` enabled
- Root shell access (web UI *Shell* or SSH)
- Local console access available for the first reboot after a kernel update

## Procedure
1. Refresh package indexes:
   ```bash
   apt update
   ```
   Expected: no `Err:` lines and no requests to `enterprise.proxmox.com`.

2. Review and apply updates:
   ```bash
   apt full-upgrade
   ```
   Before confirming, check the summary. `Removing` should be 0. **Abort** if `proxmox-ve`, `pve-manager` or `qemu-server` are listed for removal.

   `full-upgrade` is required, not `upgrade`. Proxmox releases routinely add new packages, such as versioned kernels, and `upgrade` will not install them.

3. During installation:
   - `apt-listchanges` notices (`press q to quit`): read, then press `q`.
   - Configuration file prompts: keep the locally installed version unless the change is understood.

4. Verify nothing is left half-configured:
   ```bash
   dpkg --audit            # expect no output
   apt full-upgrade        # expect "0 upgraded, 0 newly installed"
   ```

5. If a new `proxmox-kernel-*` package was installed, reboot from the web UI.

## Verification
```bash
uname -r                        # new kernel version
ip -br addr show vmbr0          # UP with 192.168.1.10/24
```
Confirm the web UI (`https://192.168.1.10:8006`) is reachable from another machine.

## Recovery
| Symptom | Action |
|---|---|
| Web shell disconnects mid-upgrade (services restarting) | Reconnect, run `dpkg --configure -a`, then `apt full-upgrade` |
| Host unreachable after reboot | Use the local console. Check `ip a` and `/etc/network/interfaces`. Select the previous kernel from the GRUB menu if needed |
| `401 Unauthorized` from `enterprise.proxmox.com` | Disable the enterprise repositories (see Prerequisites) |
