# Runbook: Caddy reverse proxy with local HTTPS (bellows)

**Host:** LXC 103 on Proxmox, Debian 13, Caddy v2.11 from the official repository. **Last run:** 2026-11-12.
Config: [`config/bellows/Caddyfile`](../../config/bellows/Caddyfile).

## Install
```bash
apt install -y curl gpg debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf https://dl.cloudsmith.io/public/caddy/stable/gpg.key | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt > /etc/apt/sources.list.d/caddy-stable.list
apt update && apt install -y caddy
```

## Configure
- `local_certs` in the global block: every site gets a certificate from Caddy's internal CA. `skip_install_trust` stops Caddy trying to edit the container's trust store.
- Proxmox upstream: copy `/etc/pve/pve-root-ca.pem` from the host and use `tls_trust_pool file ...` with `tls_server_name pve.home.arpa` (a name in the Proxmox certificate) instead of `tls_insecure_skip_verify`.
- Always `caddy fmt --overwrite` and `caddy validate` before `systemctl reload caddy`.
- Point the names at the proxy with DNS rewrites in AdGuard Home.
- Jellyfin behind a proxy: add the proxy IP to `<KnownProxies>` in `/etc/jellyfin/network.xml` so it logs real client addresses.

## Trust the root certificate on clients
The public root is at `/var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt` and is also served at `http://<proxy-ip>/ca.crt`. **Compare the SHA-256 fingerprint before installing** (`openssl x509 -in root.crt -noout -fingerprint -sha256`).
- **Arch / Omarchy:** `sudo trust anchor --store root.crt`, then restart the browser.
- **Windows:** open the file > Install Certificate > Local Machine > Trusted Root Certification Authorities.
- **Android:** Settings > Security and privacy > More security settings > Install from device storage > CA certificate (browsers trust it; most apps don't).
- **Debian (e.g. the monitoring container):** copy to `/usr/local/share/ca-certificates/*.crt` and run `update-ca-certificates`.

## Gotchas
- Devices that don't use the local DNS server can't resolve `*.home.arpa` (here: the phone and Chromecast, because the ISP router advertises its own IPv6 DNS).
- A container resolves its own hostname from `/etc/hosts`, so a health check from `tongs` to `tongs.home.arpa` bypasses the proxy.
