# OPNsense Management Plugin

A Claude Code plugin for managing an [OPNsense](https://opnsense.org/) router/firewall. SSH-based diagnostics with optional OPNsense API support.

Per-host details (IP, SSH user, API credential references) are stored **outside the plugin** at `$CLAUDE_USER_DATA/opnsense-mgmt/config.json`, so the same install works against any number of OPNsense routers and survives plugin updates.

## Skills

- `onboard` — interactive first-run setup. Captures host, SSH user, SSH key path, web URL, and (optionally) OPNsense API credential references. Writes `config.json`.
- `opnsense-maintenance` — inspect firewall rules, NAT, interfaces, DHCP leases, ARP, system logs, and VPN status via SSH or the OPNsense API. Reads from `config.json`.

## Installation

```
claude plugins install opnsense-mgmt@danielrosehill
```

## Quick start

1. Install the plugin.
2. Run the `onboard` skill — Claude will interview you for the connection details and write them to `$CLAUDE_USER_DATA/opnsense-mgmt/config.json`.
3. Ask Claude things like *"check OPNsense"* or *"show the firewall rules"* — it'll read the config and connect.

## Storage convention

This plugin follows the [`rudder:plugin-data-storage`](https://github.com/danielrosehill/Claude-Rudder) convention:

- **Plugin code** lives at the install path (read-only, replaced on update).
- **User config** lives at `${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/opnsense-mgmt/config.json`.
- **Secrets** (API key/secret) are never stored in the config — only a *reference* to where they live (1Password item, env var, file path). Skills resolve the reference at runtime.

## License

MIT
