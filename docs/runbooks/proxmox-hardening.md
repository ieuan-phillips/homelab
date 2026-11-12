# Runbook: Proxmox host hardening basics

**Host:** Proxmox VE 9.2, single node. **Last run:** 2026-11-12.

## 1. SSH key access (verify the host key first)
On the host (web Shell): `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`.
On the admin machine: `ssh-keyscan -t ed25519 <host> | ssh-keygen -lf -`. Only trust the key if the fingerprints match, then add it to `~/.ssh/known_hosts` and append your public key to `/root/.ssh/authorized_keys`.
Look up hosts with `ssh-keygen -F <host>`, not `grep` (`192.168.1.10` also matches `192.168.1.109`).

## 2. 2FA on the web UI
Datacenter > Permissions > Two Factor > Add > TOTP for `root@pam`, then Add > Recovery Keys and store them in a password manager. Log out and back in to test. To change authenticator app: add a second TOTP, test it, then remove the old one.

## 3. Guest backups
```bash
pvesh create /cluster/backup --id weekly-guests --schedule "sun 02:30" --vmid <ids> \
  --storage local --mode snapshot --compress zstd --prune-backups "keep-last=2,keep-weekly=4"
vzdump <id> --storage local --mode snapshot --compress zstd   # test run
```
Mount points with `backup=0` (e.g. a media volume) are skipped by design.

## 4. Alerts to a phone (ntfy)
```bash
b(){ printf '%s' "$1" | base64 -w0; }
pvesh create /cluster/notifications/endpoints/webhook --name ntfy --method post \
  --url 'https://ntfy.sh/{{ secrets.topic }}' \
  --header "name=Title,value=$(b '{{ title }}')" \
  --header "name=Tags,value=$(b '{{ severity }},proxmox')" \
  --header "name=Priority,value=$(b '{{#if (eq severity "error")}}high{{else}}default{{/if}}')" \
  --body "$(b '{{ message }}')" --secret "name=topic,value=$(b '<random-topic>')"
pvesh set /cluster/notifications/matchers/default-matcher --target ntfy
pvesh create /cluster/notifications/targets/ntfy/test
```
- The API expects header values, body and secrets **base64-encoded**.
- Use a long random topic and treat it like a password (anyone with it can read the alerts).
- Mail to root (smartd, cron) reaches the same targets via `/root/.forward` -> `proxmox-mail-forward`.

## 5. SMART monitoring
In `/etc/smartd.conf`, add a self-test schedule to the `DEVICESCAN` line:
`-s (S/../../6/03|L/../15/./04)` (short test Saturdays 03:00, long test on the 15th at 04:00), then `systemctl restart smartd`.
Test the alert path with a one-off config containing `/dev/sda -d sat -H -m root -M test -M exec /usr/share/smartmontools/smartd-runner` and `smartd -q onecheck -c <file>`.
