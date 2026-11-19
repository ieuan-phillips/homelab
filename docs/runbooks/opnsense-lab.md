# Runbook: OPNsense firewall lab (hearth)

**Host:** VM 200 on Proxmox. **Last run:** 2026-11-19 (OPNsense 26.7).
Lab network: Proxmox bridge `vmbr1`, `10.10.10.0/24`. Test client: CT 104 `lab-client`.

```
 home network 192.168.1.0/24 (vmbr0)
            |
   WAN vtnet0  192.168.1.60/24, gw 192.168.1.254, DNS 192.168.1.53
   [ hearth - OPNsense 26.7 ]
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

## Gotchas
- **Block private networks** is on for WAN by default. In this lab, WAN *is* a private network (`192.168.1.0/24`), so it must be off before any WAN rule can match.
- **reply-to on WAN rules.** OPNsense adds `reply-to (vtnet0 <gateway>)` to WAN pass rules, so replies are sent to the gateway instead of back to the client. Fine for internet traffic, but a client on the *same* subnet as WAN (the laptop) never gets the reply: the connection times out even though the rule matches. Fix: tick **Disable reply-to** on that rule only. Seen in `pfctl -sr` as `pass in log quick on vtnet0 reply-to (vtnet0 192.168.1.254) ...`.
- **Rule order.** First matching `quick` rule wins. A new rule is added at the bottom, below *Default allow LAN to any*, where it never matches. Move it up (tick it, then "Move selected rule before this rule") and check with `pfctl -sr`, not just the GUI.
- Locked out of the GUI? From the console shell, `pfctl -d` turns the packet filter off until the next rule reload. Use it only briefly.
- Typing into the console through Proxmox's `sendkey` API uses US key codes, but the console keymap is UK: `|` is `shift-less`, `"` is `shift-2`.
- The WAN picked up a public IPv6 address from the ISP router's announcements even with DHCP6 off. IPv6 type **None** removes it (only the `fe80::` link-local address stays).
- New installs of OPNsense 26.x use Dnsmasq for DHCP and Unbound for DNS.
