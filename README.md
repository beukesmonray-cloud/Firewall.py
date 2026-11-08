# firewall.py

A simple host-based firewall manager for windows 11, built on top of
`iptables`. Default-deny inbound, with an allow-list you control, plus
a watch mode that flags IPs hammering your blocked ports (basic
scan / brute-force detection).

## Requirements

- Kali Linux (or any Debian-based distro with iptables)
- Run as root (it edits firewall rules)
- Python 3 (no extra packages needed — standard library only)

## Quick start

```bash
# 1. Set up default-deny inbound (do this first)
sudo python3 firewall.py init

# 2. Allow what you actually need, e.g. SSH on 22
sudo python3 firewall.py allow --proto tcp --port 22

# 3. Allow a specific trusted IP full access (e.g. your other machine)
sudo python3 firewall.py allow --ip 192.168.1.50

# 4. See what's active
sudo python3 firewall.py list

# 5. In a separate terminal, watch for scan/brute-force attempts
sudo python3 firewall.py watch
```

## Commands

| Command  | Purpose |
|----------|---------|
| `init`   | Flush rules, set INPUT/FORWARD to DROP, OUTPUT to ACCEPT, allow loopback + established connections, log everything else |
| `allow`  | Add an ACCEPT rule (`--ip`, `--port`, `--proto`) |
| `block`  | Add a DROP rule at the top of the chain (useful for explicitly blocklisting an IP) |
| `remove` | Remove allow/block rules for an IP |
| `list`   | Show live iptables rules + everything added via this tool |
| `status` | Dump raw iptables policy |
| `watch`  | Tail the kernel log for blocked packets; alerts when one IP crosses a rate threshold |
| `reset`  | Emergency escape hatch — flushes everything back to allow-all if you lock yourself out |

## How `watch` works

`init` adds a rate-limited `LOG` rule before packets get dropped, tagged
with the prefix `FWTOOL-BLOCK:`. `watch` tails `/var/log/kern.log`,
extracts the source IP of each blocked packet, and keeps a sliding
window of timestamps per IP. If an IP crosses `--threshold` hits within
`--window` seconds (default: 10 hits / 60s), it prints an alert and
suggests a block command. This is a simple signal, not a full IDS —
good for noticing obvious scans or repeated connection attempts, not a
replacement for something like Suricata or Fail2Ban in production.

## Notes / limitations

- **Rules don't survive reboot** by default. To persist them, install
  `iptables-persistent` (`sudo apt install iptables-persistent`) and
  run `sudo netfilter-persistent save` after configuring your rules.
- This protects the machine it's running on — it does not hide or
  change your public IP address. For that you'd want a VPN/Tor, which
  is a separate concern from firewalling.
- If you get locked out of something you need, run
  `sudo python3 firewall.py reset` to flush everything.
- Test changes on a non-critical machine/VM first, especially before
  running `init` on a box you're remotely connected to (if you SSH in,
  allow port 22 immediately after `init` or you'll lock yourself out).
