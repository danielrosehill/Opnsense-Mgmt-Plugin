---
name: opnsense-maintenance
description: Use when the user wants to debug their local network, inspect firewall rules, or perform maintenance on their OPNsense router via SSH or the OPNsense API. Reads connection details from `$CLAUDE_USER_DATA/opnsense-mgmt/config.json` (populated by the `onboard` skill in this plugin). Triggers on phrases like "opnsense maintenance", "check opnsense", "debug the network", "firewall rules".
---

# opnsense-maintenance

Inspect and maintain an OPNsense router/firewall. All host- and credential-specific values come from the plugin's config — never hard-code them.

## Pre-flight

Resolve the plugin data directory (`${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/opnsense-mgmt/`) and load `config.json`. If it doesn't exist or is incomplete, tell the user to run the `onboard` skill first and stop.

Available fields after load:

- `host`, `ssh_user`, `ssh_port`, `ssh_key_path`
- `web_url`
- `api.enabled`, `api.key_ref`, `api.secret_ref`

## Access

- **SSH** (always available if onboarded):
  ```bash
  ssh ${SSH_KEY_PATH:+-i "$SSH_KEY_PATH"} -p "$SSH_PORT" "$SSH_USER@$HOST"
  ```
- **OPNsense API** (only if `api.enabled` is true): resolve the key/secret from their `*_ref` pointer at runtime — 1Password item, env var, or file path. Do not log the secret.

## Typical tasks

- Local network debugging (connectivity, DNS, DHCP leases)
- Review firewall rules, NAT, port forwards
- Inspect interface status and traffic counters
- Check system logs (`/var/log/`)
- Review VPN status (WireGuard / OpenVPN if configured)
- Dump configuration backups

Useful one-liners (run via SSH):

```bash
# Interface status
ifconfig

# Active DHCP leases
cat /var/dhcpd/var/db/dhcpd.leases

# Firewall log tail
clog -f /var/log/filter.log | head -50

# Configuration dump
cat /conf/config.xml | head -200
```

## API patterns (when `api.enabled`)

Base URL: `<web_url>/api/<module>/<controller>/<command>`. Authenticate with `--user "$API_KEY:$API_SECRET"`.

```bash
curl -k --user "$API_KEY:$API_SECRET" \
  "$WEB_URL/api/firewall/filter/get"
```

Common endpoints:

| Module | Path | Purpose |
|---|---|---|
| `core/firmware` | `/status` | Firmware/update status |
| `firewall/filter` | `/get` | Pull current rule set |
| `firewall/alias` | `/searchItem` | List aliases |
| `interfaces/overview` | `/interfacesInfo` | Interface state |
| `diagnostics/interface` | `/getArp` | ARP table |

## Guidelines

- **Prefer the API for structured rule/config queries; prefer SSH for deeper diagnostics.**
- **Never modify firewall rules, routes, or interface config without explicit confirmation** — these can break LAN access to the router itself, including the SSH session you're using.
- When debugging LAN issues, correlate with DHCP leases and ARP tables before suggesting fixes.
- Log significant changes back to the user — don't silently apply.
- If a command requires root and the configured `ssh_user` isn't root, escalate via `sudo` rather than reconnecting as root.
