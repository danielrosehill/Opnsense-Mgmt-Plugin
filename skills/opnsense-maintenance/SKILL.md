---
name: opnsense-maintenance
description: Use when the user wants to debug their local network, inspect firewall rules, or perform maintenance on their OPNsense router via SSH or the OPNsense API. Targets OPNsense 26.7.x (checked against the 26.7.8 release notes; hotfix revisions such as 26.7.8_2 are not listed there). Reads connection details from `$CLAUDE_USER_DATA/opnsense-mgmt/config.json` (populated by the `onboard` skill in this plugin). Triggers on phrases like "opnsense maintenance", "check opnsense", "debug the network", "firewall rules".
---

# opnsense-maintenance

Inspect and maintain an OPNsense router/firewall. All host- and credential-specific values come from the plugin's config — never hard-code them.

Written for the OPNsense 26.7 series ("Xenial Xenops", FreeBSD 15.1). Older releases (25.x and below) differ in places — see [Version notes](#version-notes-267).

## Pre-flight

Resolve the plugin data directory (`${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/opnsense-mgmt/`) and load `config.json`. If it doesn't exist or is incomplete, tell the user to run the `onboard` skill first and stop.

Available fields after load:

- `host`, `ssh_user`, `ssh_port`, `ssh_key_path`
- `web_url`
- `api.enabled`, `api.key_ref`, `api.secret_ref`

Then check what you are talking to — behaviour below assumes 26.7.x:

```bash
opnsense-version
```

If the major version is older than 26.7, say so and treat the endpoint names and log paths below as unverified.

## Access

- **SSH** (always available if onboarded):
  ```bash
  ssh ${SSH_KEY_PATH:+-i "$SSH_KEY_PATH"} -p "$SSH_PORT" "$SSH_USER@$HOST"
  ```
  A non-root `ssh_user` needs a real login shell set on the OPNsense user and `sudo` rights for escalation; otherwise commands below will fail or land in the console menu.
- **OPNsense API** (only if `api.enabled` is true): resolve the key/secret from their `*_ref` pointer at runtime — 1Password item, env var, or file path. Do not log the secret.

## Typical tasks

- Local network debugging (connectivity, DNS, DHCP leases)
- Review firewall rules, NAT, port forwards
- Inspect interface status and traffic counters
- Check system logs (`/var/log/<app>/`)
- Review VPN status (WireGuard / OpenVPN if configured)
- Dump configuration backups

Useful one-liners (run via SSH):

```bash
# Exact OPNsense version
opnsense-version

# Interface status
ifconfig

# Active ruleset as loaded into pf (includes rules not visible to the rules API)
pfctl -sr

# Firewall log tail (plain-text logs, one file per day; adjust the date)
tail -n 50 /var/log/filter/filter_$(date +%Y%m%d).log

# List available log directories
ls /var/log/

# Configuration dump — contains password hashes and keys; do not paste it anywhere
head -200 /conf/config.xml
```

### DHCP leases

ISC DHCP has reached end of life; OPNsense recommends Kea or Dnsmasq instead. Find out which service the router actually runs (`core/service/search`), then read leases from that service's API:

| Service | Request |
|---|---|
| Kea DHCP | `GET /api/kea/leases/search` |
| Dnsmasq DNS & DHCP | `GET /api/dnsmasq/leases/search` |

Only if the legacy ISC DHCP plugin is still in use are leases in `/var/dhcpd/var/db/dhcpd.leases` (IPv4) and `/var/dhcpd/var/db/dhcpd6.leases` (IPv6).

## API patterns (when `api.enabled`)

URL pattern: `<web_url>/api/<module>/<controller>/<command>[/<param>...]`. Authenticate with HTTP basic auth, key as username and secret as password. `GET` reads data; `POST` creates/updates data or runs an action. Bodies are JSON.

```bash
# Read
curl -k --user "$API_KEY:$API_SECRET" \
  "$WEB_URL/api/diagnostics/interface/search_arp"

# Action (POST with an empty JSON body)
curl -k --user "$API_KEY:$API_SECRET" \
  -X POST -H 'Content-Type: application/json' -d '{}' \
  "$WEB_URL/api/core/firmware/check"
```

The key's owning user must hold the privilege for each endpoint, otherwise the call is rejected. `-k` is for the default self-signed certificate; prefer `--cacert` when the user has a trusted one.

### Endpoints (read-only unless marked)

| Path | Method | Purpose |
|---|---|---|
| `core/firmware/info` | GET | Firmware/product information |
| `core/firmware/check` | POST | Check for updates (does not install) |
| `core/system/status` | GET | System status |
| `core/service/search` | GET | List services and their state |
| `diagnostics/system/system_information` | GET | Hostname, version, uptime |
| `diagnostics/system/system_resources` | GET | Memory/resource usage |
| `diagnostics/interface/search_arp` | GET | ARP table (`get_arp` also listed) |
| `diagnostics/interface/get_routes` | GET | Routing table |
| `diagnostics/firewall/pf_states` | GET | pf state table |
| `diagnostics/firewall/log` | GET | Firewall log entries |
| `interfaces/overview/interfaces_info` | GET | Interface state |
| `firewall/filter/searchRule` | GET | List firewall rules |
| `firewall/filter/getRule/<uuid>` | GET | One rule in detail |
| `firewall/source_nat/searchRule` | GET | Source NAT rules (replaces outbound NAT) |
| `firewall/d_nat/searchRule` | GET | Port forwards / destination NAT |
| `firewall/alias/searchItem` | GET/POST | List aliases |
| `firewall/filter/apply` | POST | **Write** — applies pending rule changes |
| `kea/leases/search` | GET | Kea leases |
| `dnsmasq/leases/search` | GET | Dnsmasq leases |

Notes on naming: the official reference lists newer controllers (diagnostics, interfaces, kea, dnsmasq) in `snake_case` and older ones (firewall filter/alias, firmware) in `camelCase`, and does not say whether both spellings are accepted. Use the spelling from the table; if a call returns 404, retry with the other spelling before concluding the endpoint is gone.

## Version notes (26.7)

- **Firewall rules are MVC/API-first.** The legacy PHP rule pages moved to the `os-firewall-legacy` plugin; existing rules keep working and a migration assistant is available. If the API listing and the GUI or pf disagree, trust `pfctl -sr` and `/conf/config.xml`, and mention the migration assistant.
- **Outbound NAT is now source NAT** (`firewall/source_nat/*`); the old outbound NAT page moved to a legacy plugin.
- **Rule listing:** the 26.7 API reference documents `searchRule` and `getRule` for `firewall/filter`, and no `get`. The previous version of this skill used `firewall/filter/get`; use `searchRule` instead.
- **Interface assignments, VLANs, bridges and interface settings are in the API** (`interfaces/assignment/*`, `interfaces/vlan_settings/*`, `interfaces/bridge_settings/*`, `interfaces/settings/*`). Saved changes are queued for reconfiguration and are discarded by a reboot without an apply, so check `interfaces/assignment/pending` before assuming a change is live.
- **Logs are plain text**, one file per day under `/var/log/<app>/<app>_YYYYMMDD.log`. `clog` circular logs are no longer the default (plain files since 20.7), so do not use `clog -f`.
- **User/group privileges were merged:** `page-system-groupmanager` and `page-system-usermanager-addprivs` are now a single `page-system-groupmanager`. Account for this when creating a dedicated API user.
- **SSH host keys (26.7.8):** OpenSSH 10.6 rejects ML-DSA host keys created by 10.5, and the experimental ML-DSA host keys were removed; keys are regenerated on reboot or SSH restart. A "host key changed" warning after upgrading can be expected — confirm with the user, then `ssh-keygen -R <host>`, rather than bypassing host key checking.
- **Strict security mode (26.7.8):** no longer marked experimental; changing it recommends a reboot. Privilege-separated pages (route deletion, interface apply, monit status via configd) may behave differently for non-root users.
- **Kea logs:** the log directory `/var/log/kea` is now created by a setup script before the service starts.
- **Firewall live log search:** individual filters are limited to 255 characters.
- **Hotfix revisions** (e.g. `26.7.8_2`) are not described in the public release notes; `opnsense-version` shows the exact revision.

## Guidelines

- **Prefer the API for structured rule/config queries; prefer SSH for deeper diagnostics.**
- **Never modify firewall rules, routes, or interface config without explicit confirmation** — these can break LAN access to the router itself, including the SSH session you're using.
- Before any approved change, ask the user to take a configuration backup (System ‣ Configuration ‣ Backups) or copy `/conf/config.xml` aside, so there is something to revert to.
- When debugging LAN issues, correlate with DHCP leases and ARP tables before suggesting fixes.
- Log significant changes back to the user — don't silently apply.
- If a command requires root and the configured `ssh_user` isn't root, escalate via `sudo` rather than reconnecting as root.
