# Runbook: AdGuard Home in an LXC (network DNS filter)

**Host:** Proxmox VE 9.2. **Last run:** 2026-11-12 (AdGuard Home v0.107.80).

## 1. Container
```bash
pct create 101 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname flux --unprivileged 1 --features nesting=1 \
  --cores 1 --memory 512 --swap 256 --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=<IP>/24,gw=<GATEWAY>,firewall=1 \
  --nameserver <GATEWAY> --searchdomain home.arpa --onboot 1 --startup order=1 \
  --ssh-public-keys <key.pub> --ostype debian
```
`--startup order=1` starts DNS before the other guests.

## 2. Install (checksum-verified)
```bash
V=v0.107.80; cd /tmp
curl -fsSLO https://github.com/AdguardTeam/AdGuardHome/releases/download/$V/AdGuardHome_linux_amd64.tar.gz
curl -fsSLO https://github.com/AdguardTeam/AdGuardHome/releases/download/$V/checksums.txt
grep " ./AdGuardHome_linux_amd64.tar.gz$" checksums.txt | sha256sum -c -
tar -C /opt -xzf AdGuardHome_linux_amd64.tar.gz
/opt/AdGuardHome/AdGuardHome -s install
chmod 700 /opt/AdGuardHome
```
Run the setup wizard at `http://<IP>:3000`: admin UI on port 80, DNS on port 53, all interfaces.

## 3. Configuration (in the UI, or `AdGuardHome.yaml` with the service stopped)
- Upstreams: `https://dns.quad9.net/dns-query`, `tls://dns.quad9.net`; bootstrap `9.9.9.9`, `149.112.112.112`; DNSSEC on.
- Private reverse DNS: the router.
- Blocklists: AdGuard DNS filter; HaGeZi Multi NORMAL (`https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/multi.txt`); Perflyst Smart TV (`https://raw.githubusercontent.com/Perflyst/PiHoleBlocklist/master/SmartTV-AGH.txt`).
- DNS rewrites for local names under `home.arpa`.
- Query log and statistics retention: 7 days.
- Validate with `AdGuardHome --check-config -c AdGuardHome.yaml -w /opt/AdGuardHome`.

## 4. Pointing clients at it
- **NetworkManager + systemd-resolved (Linux laptop, home network only):**
  `nmcli con mod <home-ssid> ipv4.ignore-auto-dns yes ipv4.dns <IP> ipv4.dns-search "home.arpa,~." ipv6.ignore-auto-dns yes`
  (`~.` routes all lookups to this link's DNS). On Omarchy, first run `omarchy dns DHCP` to remove its global DNS override; that command resets every profile's DNS, so apply the line above afterwards.
- **Windows:** network adapter > IPv4 > preferred DNS = `<IP>`; clear IPv6 DNS or set it to automatic only if the router does not advertise its own.
- **Check:** a blocked domain (e.g. `doubleclick.net`) returns `0.0.0.0`.

## Limits
DNS filtering can't block ads served from the same domain as the content (e.g. YouTube). Devices with hard-coded DNS (some streaming sticks) or an active VPN bypass it.
