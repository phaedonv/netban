## netban v1.0.0

First public release. netban is a single Bash script for manual IPv4 network bans with nftables.

### Features
- Temporary bans with per-entry timeouts (`30m`, `6h`, `1d`, `2d12h`) that the kernel expires automatically
- Permanent bans stored in `/etc/netban/permanent.list` with date and note, restored at boot by `netban.service`
- Runs in its own `inet netban` table at priority -10, before UFW/firewalld, without touching existing rules
- Interactive prompt and one-liner CLI
- Accepts IPs or CIDRs and masks host bits (`203.0.113.57/24` → `203.0.113.0/24`)
- Drop counters for both temporary and permanent sets
- `--install` sets up the binary, state file and systemd unit; re-running it updates in place

### Requirements
Linux with nftables, Bash 4+, root/sudo. systemd optional (needed only for restoring permanent bans at boot).

### Install
```bash
curl -fsSLO https://raw.githubusercontent.com/phaedonv/netban/v1.0.0/netban
sudo bash netban --install
```
