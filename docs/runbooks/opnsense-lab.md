# Runbook: OPNsense firewall lab (hearth)

**Host:** VM 200 on Proxmox. **Last run:** 2026-11-20 (OPNsense 26.7.9_3).
Lab network: Proxmox bridge `vmbr1`, `10.10.10.0/24`. Test client: CT 104 `lab-client`.

```
 home network 192.168.1.0/24 (vmbr0)
            |
   WAN vtnet0  192.168.1.60/24, gw 192.168.1.254, DNS 192.168.1.53
   [ hearth - OPNsense 26.7.9, Suricata IDS on LAN ]
   LAN vtnet1  10.10.10.1/24, DHCP .100-.200
            |
 lab network 10.10.10.0/24 (vmbr1, no physical port, no host IP)
            |
   CT 104 lab-client (DHCP)
```

## 1. Get and verify the image
Download `OPNsense-<ver>-dvd-amd64.iso.bz2`, its `.sig`, `checksums-amd64.sha256` (+ `.sig`) and `OPNsense-<ver>.pub` from `pkg.opnsense.org`. Fetch the `.pub` again from a **second mirror** and compare, so one tampered server can't fool the check.
```bash
sha256sum -c --ignore-missing checksums-amd64.sha256
# each .sig is base64; decode, then verify with the public key
openssl base64 -d -in X.sig -out x.sig
openssl dgst -sha256 -verify OPNsense-<ver>.pub -signature x.sig X
bunzip2 OPNsense-<ver>-dvd-amd64.iso.bz2   # then verify the .iso signature the same way
```
Copy the ISO to `/var/lib/vz/template/iso/` on the Proxmox host.

## 2. Lab bridge (no physical port, no host IP)
In `/etc/network/interfaces` on the Proxmox host (back it up first), then `ifreload -a`:
```
auto vmbr1
iface vmbr1 inet manual
        bridge-ports none
        bridge-stp off
        bridge-fd 0
```
With no port and no IP, the only way in or out of the lab is through the firewall VM.

## 3. Create the VM
2 cores (`cpu host`), 2048 MB (ballooning off), 16 GB virtio-scsi disk (iothread, discard), `net0` virtio -> `vmbr0` (WAN), `net1` virtio -> `vmbr1` (LAN), ISO on `ide2`.

## 4. Install
1. Boot. The live system first asks to assign interfaces: answer `n` (LAGG), `n` (VLAN), WAN `vtnet0`, LAN `vtnet1`. Check the MAC addresses shown against the VM's Hardware tab so WAN and LAN aren't swapped.
2. Log in as `installer` / `opnsense`. Keymap: United Kingdom. **Install (UFS)** -> disk `da0` (not `cd0`, that's the DVD) -> confirm.
3. With 2 GB RAM the installer warns that copying the live image wants 3000 MB. "Proceed anyway" worked fine.
4. Set the root password (password manager), **Complete Install**, reboot.
5. Check it booted from disk (`Root file system: /dev/gpt/rootfs`), then remove the ISO: `qm set 200 --ide2 none,media=cdrom`.

## 5. Console setup (log in as root)
- **1) Assign interfaces:** WAN `vtnet0`, LAN `vtnet1`.
- **2) Set interface IP address -> LAN:** no DHCP, `10.10.10.1`, `/24`, no gateway, no IPv6, enable DHCP server `10.10.10.100`-`10.10.10.200`, keep HTTPS.
- **2) -> WAN:** no DHCP, `192.168.1.60`, `/24`, gateway `192.168.1.254`, don't use the gateway as DNS, DNS `192.168.1.53`, no DHCP6.
- **7) Ping host:** gateway, `9.9.9.9` and a hostname should all answer.

## 6. Test client
Unprivileged Debian 13 LXC on `vmbr1` with DHCP:
```bash
pct create 104 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname lab-client --cores 1 --memory 512 --swap 512 --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr1,ip=dhcp --unprivileged 1 --features nesting=1 \
  --ssh-public-keys ~/.ssh/authorized_keys --start 1
pct exec 104 -- sh -c 'ip -4 -br a show eth0; ip route; ping -c2 9.9.9.9'
pct exec 104 -- wget -q --no-check-certificate -S -O /dev/null https://10.10.10.1/
```
Expected: an address in `10.10.10.100-200`, default route via `10.10.10.1`, internet ping works (outbound NAT), and the GUI answers `HTTP/1.1 200 OK`. The Debian standard template has no `curl`; use `wget`.

## 7. Firewall rules
Goal: only the admin laptop can reach the GUI from the home network, and the lab cannot reach the home network at all (but keeps internet access).

**Getting in the first time.** The GUI is blocked from WAN by default. From the console shell (option 8): `pfctl -d`, then browse to `https://192.168.1.60`. The first rule *Apply* turns the filter back on.

**WAN rule - admin laptop to GUI only.** Firewall -> Rules -> WAN:

| Action | Proto | Source | Destination | Port | Log | Advanced |
|---|---|---|---|---|---|---|
| Pass | TCP (IPv4) | admin laptop `/32` | WAN address | 443 | yes | **Disable reply-to** |
| Pass | TCP (IPv4) | admin laptop `/32` | WAN address | 22 (SSH) | yes | **Disable reply-to** (clone of the rule above) |

Then Interfaces -> WAN: untick **Block private networks**, set IPv6 type **None**, Save, Apply. Save the rule first and apply both together, or the filter comes back on before the rule exists.

**LAN rule - lab cannot reach home.** Firewall -> Rules -> LAN, above *Default allow LAN to any*:

| Action | Proto | Source | Destination | Log |
|---|---|---|---|---|
| Block | any (IPv4) | LAN net | `192.168.1.0/24` | yes |

**Verify.**
```bash
# from the admin laptop: 200; port 80 and ping time out
curl -sk -o /dev/null -w '%{http_code}\n' https://192.168.1.60/
# from another home device (phone): https://192.168.1.60 times out
# from the lab client: home addresses blocked, internet + DNS + LAN-side GUI still work
pct exec 104 -- sh -c 'for h in 192.168.1.53 192.168.1.10 192.168.1.254 9.9.9.9; do ping -c1 -W2 $h >/dev/null && echo "$h ok" || echo "$h BLOCKED"; done'
# on the firewall: the order pf actually uses (first matching quick rule wins)
pfctl -sr | grep -n -e '192.168.1.0/24' -e 'inet from (vtnet1:network) to any'
```
Result on 2026-11-19: laptop 200; phone timed out; lab client blocked to `.53`, `.10`, `.254` and flux's web page, while `9.9.9.9`, DNS, `http://deb.debian.org` and `https://10.10.10.1` still worked. Before the LAN rule, the lab client could reach all of them.

## 8. SSH (key only, admin laptop only)
1. System -> Access -> Users -> root -> **Authorized keys**: paste the laptop's `~/.ssh/id_ed25519.pub`. Save.
2. System -> Settings -> Administration -> Secure Shell: **Enable**, **Permit root user login**, **Permit password login off**, port 22, Listen Interfaces **WAN** only. Save.
3. WAN rule for port 22 (table in section 7).
4. `ssh root@192.168.1.60` -> console menu -> `8` for a shell. Check the ED25519 fingerprint against the banner on the Proxmox console the first time.

## 9. Suricata IDS (alert only)
Services -> Intrusion Detection -> Administration.

**Settings.** Enabled; **Capture mode `PCAP live mode (IDS)`** (26.x has no "IPS mode" tickbox: Netmap/Divert are the inline IPS modes); Promiscuous off; Interfaces **LAN**; Pattern matcher and Detect Profile Default; **Home networks `10.10.10.0/24`**. Apply.
Why LAN: on WAN all lab traffic is already NATed to `192.168.1.60`, so alerts would not show which lab host it was.

**Rulesets (Download tab).** Enable and download ET open `emerging-attack_response`, `emerging-scan`, `emerging-malware`, `emerging-exploit`, `emerging-policy`. Kept small: hearth has 2 GB RAM (about 927 MB used with these five).
Rules tab: also enabled the four `ET SCAN NMAP -sS window ...` rules (SIDs 2000537, 2009582, 2009583, 2009584), which ET ships disabled.

**Daily update.** Schedule tab (= the cron job in System -> Settings -> Cron): enable the existing job, command *Update and reload intrusion detection rules*, `30 4 * * *`.

**Verify.**
```bash
# 1. content rule (SID 2100498 + 2019284). testmynids.org has expired, so send the string ourselves:
pct exec 104 -- wget -qO /dev/null --post-data='uid=0(root) gid=0(root) groups=0(root)' http://example.com/
# 2. scan rules (needs nmap in the lab client; scan only our own hosts)
pct exec 104 -- nmap -sS -T4 10.10.10.1
# on the firewall: what is enabled (a leading # = disabled) and in which direction
grep -h 'NMAP -sS' /usr/local/etc/suricata/opnsense.rules/*.rules | cut -c1-200
```
Alerts tab: alert-only mode shows matches as **allowed**.
Result on 2026-11-20: test 1 alerted every time. Test 2 alerted **only** when lab-client was outside Home networks (Home networks `10.10.10.1/32` for the test): `2009582` NMAP -sS window 1024, `2002910` VNC scan, `2010935` MSSQL 1433, `2010936` Oracle 1521. With Home networks covering the whole lab, the same scan was silent, because ET SCAN rules match `$EXTERNAL_NET -> $HOME_NET` only.

**Before updating the firmware:** `qm snapshot 200 <name>` on pve; roll back with `qm rollback 200 <name>`; delete it once the update has proven fine. Download a config backup afterwards (System -> Configuration -> Backups). It contains password hashes and the GUI key, so keep it in the password manager, not in git.

## Gotchas
- **Block private networks** is on for WAN by default. In this lab, WAN *is* a private network (`192.168.1.0/24`), so it must be off before any WAN rule can match.
- **reply-to on WAN rules.** OPNsense adds `reply-to (vtnet0 <gateway>)` to WAN pass rules, so replies are sent to the gateway instead of back to the client. Fine for internet traffic, but a client on the *same* subnet as WAN (the laptop) never gets the reply: the connection times out even though the rule matches. Fix: tick **Disable reply-to** on that rule only. Seen in `pfctl -sr` as `pass in log quick on vtnet0 reply-to (vtnet0 192.168.1.254) ...`.
- **Rule order.** First matching `quick` rule wins. A new rule is added at the bottom, below *Default allow LAN to any*, where it never matches. Move it up (tick it, then "Move selected rule before this rule") and check with `pfctl -sr`, not just the GUI.
- Locked out of the GUI? From the console shell, `pfctl -d` turns the packet filter off until the next rule reload. Use it only briefly.
- Typing into the console through Proxmox's `sendkey` API uses US key codes, but the console keymap is UK: `|` is `shift-less`, `"` is `shift-2`.
- The WAN picked up a public IPv6 address from the ISP router's announcements even with DHCP6 off. IPv6 type **None** removes it (only the `fe80::` link-local address stays).
- New installs of OPNsense 26.x use Dnsmasq for DHCP and Unbound for DNS.
- **Home networks decides what Suricata calls an attack.** Most ET SCAN rules are `$EXTERNAL_NET -> $HOME_NET`. Traffic between two hosts inside Home networks never matches them, even when the rules are enabled.
- Many ET rules ship **disabled** (a leading `#` in `/usr/local/etc/suricata/opnsense.rules/*.rules`). Check before assuming a rule can fire.
- Closed ports show as **filtered** to nmap, not closed: OPNsense sets `net.inet.tcp.blackhole`, so no reset is sent.
