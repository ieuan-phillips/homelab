# Runbook: Jellyfin LXC on Proxmox with Intel Quick Sync

**Purpose:** Build (or rebuild) the Jellyfin media server container with hardware transcoding.
**Host:** Proxmox VE 9.2, Intel HD 530 iGPU. **Last run:** 2026-11-12.

## Prerequisites
- iGPU visible on the host: `lspci -nn | grep -Ei 'vga|display'` shows Intel, and `/dev/dri/renderD128` exists.
- A free static IP on the LAN, outside the router's DHCP range.
- An SSH public key to put in the container.

## 1. Create the container (on the Proxmox host)
```bash
pveam update
pveam download local debian-13-standard_13.6-1_amd64.tar.zst
pct create 100 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname jellyfin --unprivileged 1 --features nesting=1 \
  --cores 2 --memory 2048 --swap 512 \
  --rootfs local-lvm:16 --mp0 local-lvm:500,mp=/media,backup=0 \
  --net0 name=eth0,bridge=vmbr0,ip=<IP>/24,gw=<GATEWAY>,firewall=1 \
  --nameserver <DNS> --timezone host --onboot 1 \
  --ssh-public-keys <path/to/key.pub> --ostype debian
pct start 100
```
- `--unprivileged 1`: container root is not host root.
- `--mp0 ...,backup=0`: separate media volume, left out of container backups (too big; media needs its own backup plan).
- No root password is set. Log in with the SSH key, or `pct enter 100` from the host.
- Verify the container's SSH host key from the host before trusting it: `pct exec 100 -- ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`.

## 2. Install Jellyfin (inside the container)
```bash
apt update && apt full-upgrade -y && apt install -y curl gpg ca-certificates
install -d -m 755 /etc/apt/keyrings
curl -fsSL https://repo.jellyfin.org/jellyfin_team.gpg.key | gpg --dearmor -o /etc/apt/keyrings/jellyfin.gpg
cat > /etc/apt/sources.list.d/jellyfin.sources <<'SRC'
Types: deb
URIs: https://repo.jellyfin.org/debian
Suites: trixie
Components: main
Architectures: amd64
Signed-By: /etc/apt/keyrings/jellyfin.gpg
SRC
apt update && apt install -y jellyfin
getent group render        # note the GID (was 992)
id jellyfin                # should already include render
```
Check the key fingerprint: `gpg --show-keys /etc/apt/keyrings/jellyfin.gpg` should show `4918 AABC 486C A052 358D 778D 4902 3CD0 1DE2 1A7B`.

## 3. Pass the iGPU through (on the host)
```bash
pct shutdown 100
pct set 100 --dev0 /dev/dri/renderD128,gid=<container render GID>
pct start 100
```
`gid=` is the **container's** render group, so the device appears as `root:render` inside and the `jellyfin` user can open it.

## 4. Media folders and checks (inside the container)
```bash
mkdir -p /media/movies /media/tv && chown -R jellyfin:jellyfin /media
/usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
```
Expect `Intel iHD driver` with H.264 and HEVC Main `VLD` (decode) and `EncSlice` (encode). Skylake has no HEVC 10-bit or hardware VP9 decode.
The `chown` warning about `lost+found` is harmless.

## 5. Web setup
Browse to `http://<IP>:8096`, run the setup wizard, then **Dashboard > Playback > Transcoding**:
- Hardware acceleration: **Intel QuickSync (QSV)**, device `/dev/dri/renderD128`.
- Hardware decoding: H264, HEVC, MPEG2, VC1, VP8. Leave HEVC 10-bit, VP9 and AV1 off (unsupported on HD 530).
- Leave the Intel low-power encoders off (they need extra GuC/HuC firmware setup on Skylake).

## Rollback
`pct stop 100 && pct destroy 100` removes the container **and its media volume**. Detach or back up `mp0` first if it holds anything.

## 6. Verify playback
- Run **Scan All Libraries** once after creating the libraries; real-time monitoring only starts watching folders after the first scan.
- **Direct play:** play something on the client, then run `pgrep -x ffmpeg` in the container. No process = direct play (full quality, almost no server load).
- **Hardware transcode:** set the client's max bitrate low (e.g. 3 Mbps) and check the newest `/var/log/jellyfin/FFmpeg.Transcode-*.log` for `h264_qsv` and the `speed=` figure.
- Result on 2026-11-12 (Sintel, 1080p H.264 to 2.5 Mbit/s): VA-API decode + `h264_qsv` encode at about 296 fps / **12x real time**, negligible CPU.
