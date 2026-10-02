<p align="center">
  <img src="assets/netban-logo.svg" width="220" alt="netban — subnet IP range ban">
</p>

<p align="center">
  Manual IPv4 network bans for Linux, built on nftables.<br>
  Temporary bans that expire on their own. Permanent bans that survive reboots.
</p>

<p align="center">
  <img alt="Bash" src="https://img.shields.io/badge/bash-4%2B-2EF5A0?logo=gnubash&logoColor=white">
  <img alt="nftables" src="https://img.shields.io/badge/firewall-nftables-2EF5A0">
  <img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-2EF5A0">
</p>

---

`netban` is a single Bash script for blocking whole networks (for example a
`/24`) on a Linux host, either for a fixed time or permanently. It's meant as
a **manual fallback** next to automated defences such as Wazuh active
response, Suricata, fail2ban or CrowdSec: you use it when those miss something
or when you need to stop traffic *right now* without editing firewall configs.

```
$ netban 203.0.113.57/24
banned 203.0.113.0/24 for 24h
```

## Why

A typical case: a server exposes SSH with key-only auth, and a bot keeps
hammering it from a rotating pool of IPs inside one subnet:

```
sshd: Invalid user taow from 203.0.113.239 port 50612
sshd: Invalid user admin from 203.0.113.12 port 41877
sshd: Invalid user oracle from 203.0.113.190 port 39020
```

It isn't a real threat, because passwords are off. It is still noise: it fills
the logs, triggers SIEM alerts and wastes CPU. Banning single IPs is pointless
when the bot rotates, but banning the whole range for a day stops it.

Why netban instead of a quick `ufw deny` or `iptables -I`:

- **Self-expiring:** temporary bans are removed by the kernel when the timer
  runs out, so there are no forgotten rules lingering for months.
- **Isolated:** it lives in its own nftables table (`inet netban`). It never
  touches UFW, firewalld, `/etc/nftables.conf` or anyone else's rules, and
  removing it is a single command.
- **Ordered correctly:** it hooks `input` at priority `-10`, before UFW and
  firewalld, so banned traffic is dropped whatever allow rules exist. A
  plain `ufw deny` gets appended after `allow 22` and never matches.
- **Auditable:** permanent bans are kept in a plain text file with the date
  and an optional note.
- **Drop counters:** you can see whether the ban is actually doing anything.
- **Forgiving input:** paste `203.0.113.57/24` straight from an alert and the
  host bits are masked to `203.0.113.0/24`.

## How it works

```
table inet netban
 ├─ set perm   (interval)            permanent networks
 ├─ set temp   (interval, timeout)   temporary networks, auto-expire
 └─ chain input  hook input priority -10
      ip saddr @perm counter drop
      ip saddr @temp counter drop
```

- **Temporary bans** exist only in the kernel set, each with its own timeout.
- **Permanent bans** are added to the kernel set and to
  `/etc/netban/permanent.list`. A systemd oneshot unit (`netban.service`)
  runs `netban restore` at boot.
- `netban.service` is `PartOf=nftables.service`, so the permanent bans are
  reloaded if nftables is restarted and the ruleset flushed.

## Requirements

- Linux with nftables (`nft`): Debian 10+, Ubuntu 20.04+, RHEL / Rocky /
  Alma 8+, Fedora, Arch, …
- Bash 4+
- root or sudo (the script re-executes itself with `sudo` if needed)
- systemd, optional, only for restoring permanent bans at boot

It works alongside UFW and firewalld.

## Install

```bash
curl -fsSLO https://raw.githubusercontent.com/<you>/netban/main/netban
sudo bash netban --install
```

Or from a clone:

```bash
git clone https://github.com/<you>/netban.git
sudo bash netban/netban --install
```

This installs:

```
/usr/local/bin/netban                  the tool
/etc/netban/permanent.list             permanent ban list
/etc/systemd/system/netban.service     restores permanent bans at boot
```

Re-running `--install` is safe and also works as an update.

## Usage

### Interactive

```
$ netban
netban — type a network to ban it (default 24h).
commands: <net> [dur|perm] [note] | unban <net> | list | restore | purge | help | quit
netban> 203.0.113.0/24
banned 203.0.113.0/24 for 24h
netban> 198.51.100.7/24 6h
banned 198.51.100.0/24 for 6h
netban> 192.0.2.0/24 perm ssh user-enum bot
banned 192.0.2.0/24 permanently
netban> unban 198.51.100.0/24
removed temporary ban 198.51.100.0/24
netban> quit
```

### One-liners

```bash
netban 203.0.113.0/24                          # temporary, 24h default
netban 203.0.113.0/24 2d12h                    # temporary, custom duration
netban 203.0.113.0/24 perm "ssh rotation bot"  # permanent, with a note
netban 203.0.113.7                             # single host (/32)
netban -l                                      # list bans + drop counters
netban -u 203.0.113.0/24                       # unban (temp and/or perm)
netban restore                                 # reload permanent list
netban --purge                                 # remove all runtime bans
netban -h                                      # help
```

Durations: `30m`, `6h`, `1d`, `2d12h`, or `perm`. Banning a network that's
already banned resets its timer.

### Example `netban -l`

```
--- temporary ---
		elements = { 203.0.113.0/24 timeout 1d expires 23h41m12s }
--- permanent (live) ---
		elements = { 192.0.2.0/24 }
--- permanent (/etc/netban/permanent.list) ---
192.0.2.0/24  # added 2026-10-01 - ssh user-enum bot
--- drops ---
@perm counter packets 1843 bytes 110580
@temp counter packets 212 bytes 12720
```

## From a SIEM alert to a ban

Given an alert such as Wazuh rule `5710` (*sshd: attempt to login using a
non-existent user*) with `srcip: 203.0.113.239`, run this on the affected host:

```bash
netban 203.0.113.239/24
```

If the range keeps coming back after the ban expires:

```bash
netban 203.0.113.0/24 perm "returns daily, ssh user enum"
```

To verify it's working, the counter climbs and the log lines stop:

```bash
netban -l
sudo journalctl -u ssh -f
```

## Limitations

- **IPv4 only.**
- **Manual by design.** It doesn't watch logs. Automated per-IP banning
  belongs in fail2ban, CrowdSec, Wazuh active response or similar.
- **Host-level.** Packets still reach the machine before they're dropped.
  Blocking at the edge router or cloud firewall saves bandwidth and covers
  every host behind it.
- **No overlapping entries.** nftables interval sets reject a network that
  overlaps one already in the same set. Unban the larger range first.
- **Exact match for unban.** Use the network exactly as `netban -l` shows it.
- **Temporary bans don't survive** a reboot or an nftables restart. Permanent
  bans do.
- **Not a security control on its own.** It reduces noise and load. Real SSH
  protection is key-only auth and limiting who can reach port 22.

## Uninstall

```bash
sudo systemctl disable --now netban.service
sudo netban --purge
sudo rm -f /usr/local/bin/netban /etc/systemd/system/netban.service
sudo rm -rf /etc/netban
sudo systemctl daemon-reload
```

## License

[MIT](LICENSE)
