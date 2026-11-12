# Runbook: Gatus monitoring (tongs)

**Host:** LXC 102 on Proxmox, Debian 13. **Last run:** 2026-11-12 (Gatus v5.38.0, Go 1.26).
Config: [`config/tongs/gatus-config.yaml`](../../config/tongs/gatus-config.yaml), unit: [`config/tongs/gatus.service`](../../config/tongs/gatus.service).

## Build (Gatus publishes container images, not binaries)
Gatus needs a newer Go than Debian 13 has, so use the official toolchain and remove it afterwards.
```bash
# Go from go.dev, checksum from https://go.dev/dl/?mode=json
curl -fsSLO https://go.dev/dl/<go1.26.x.linux-amd64.tar.gz>
echo "<sha256>  <file>" | sha256sum -c -
tar -C /usr/local -xzf <file>
git clone --depth 1 --branch v5.38.0 https://github.com/TwiN/gatus.git /root/build/gatus
cd /root/build/gatus
GOPATH=/root/build/gopath GOCACHE=/root/build/gocache GOTMPDIR=/root/build \
  CGO_ENABLED=0 /usr/local/go/bin/go build -p 1 -trimpath -ldflags="-s -w" -o /usr/local/bin/gatus .
rm -rf /root/build /usr/local/go
```
**Gotcha:** in these containers `/tmp` is a RAM-backed tmpfs. Building there filled the container's memory and the compiler was OOM-killed. Keep build files on disk and give the container ~2 GB RAM for the build only (256 MB afterwards).

## Install
```bash
useradd --system --no-create-home --shell /usr/sbin/nologin gatus
install -d -m 750 -o root -g gatus /etc/gatus
install -m 640 -o root -g gatus gatus-config.yaml /etc/gatus/config.yaml
printf 'NTFY_TOPIC=%s\n' '<topic>' > /etc/gatus/gatus.env && chgrp gatus /etc/gatus/gatus.env && chmod 640 /etc/gatus/gatus.env
cp gatus.service /etc/systemd/system/ && systemctl daemon-reload && systemctl enable --now gatus
```
Dashboard: `http://<ip>:8080`. Gatus reloads the config file automatically when it changes.

## Test alerts
Add a temporary endpoint pointing at a closed port (e.g. `tcp://<ip>:9`) with `failure-threshold: 1`, wait for the ntfy notification, then remove it.
