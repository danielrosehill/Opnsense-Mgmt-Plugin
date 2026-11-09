---
name: onboard
description: First-run onboarding for the opnsense-mgmt plugin. Interview the user about their OPNsense router — host/IP, SSH user, whether the OPNsense API is enabled, and where API credentials live — and persist the result to `$CLAUDE_USER_DATA/opnsense-mgmt/config.json`. Run this before any other skill in this plugin, or whenever the connection details change. Triggers on phrases like "set up opnsense", "onboard opnsense", "configure opnsense plugin".
---

# opnsense-mgmt: onboard

Establish the persistent connection profile for an OPNsense host. The same plugin install can serve multiple environments — only the values in `config.json` are environment-specific. No host details should ever be hard-coded into other skills in this plugin.

## Workspace resolution

Resolve the plugin's data directory as `$CLAUDE_USER_DATA/opnsense-mgmt/` if `CLAUDE_USER_DATA` is set; otherwise `$XDG_DATA_HOME/claude-plugins/opnsense-mgmt/` if `XDG_DATA_HOME` is set; otherwise `~/.local/share/claude-plugins/opnsense-mgmt/`. Create the directory if it doesn't exist. See the canonical convention in the `rudder:plugin-data-storage` skill.

Shell form:

```bash
PLUGIN_DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/opnsense-mgmt"
mkdir -p "$PLUGIN_DATA_DIR"
```

The config file is `$PLUGIN_DATA_DIR/config.json`.

## When to use

- User says "set up opnsense", "onboard opnsense", "configure opnsense plugin".
- Any other opnsense-mgmt skill finds no `config.json` at the resolved path — offer to run onboarding first.
- The user changes routers, IPs, or SSH credentials.

## Procedure

1. **Load** existing config from `$PLUGIN_DATA_DIR/config.json` if present. Show the current values back. Offer **Update** vs **Replace** vs **Cancel**.

2. **Interview** the user. Ask each question; default sensibly when the user just hits enter:

   | Field | Prompt | Notes |
   |---|---|---|
   | `host` | "What's the LAN address of your OPNsense router?" | Default: `10.0.0.1`. Accept hostname or IP. |
   | `ssh_user` | "SSH username?" | Default: `root` (OPNsense default admin). Accept anything. |
   | `ssh_port` | "SSH port?" | Default: `22`. |
   | `ssh_key_path` | "Path to the SSH private key (or leave blank for default agent)?" | Optional. Expand `~`. |
   | `web_url` | "Web UI URL?" | Default: `https://<host>`. |
   | `api_enabled` | "Is the OPNsense API enabled? (y/n)" | If yes, ask for the next two. |
   | `api_key_ref` | "Where is the API key stored? (e.g. `1password://OPNsense API/key`, env var name, or file path)" | Store the **reference**, not the secret itself. |
   | `api_secret_ref` | "Where is the API secret stored?" | Same — reference only. |
   | `notes` | "Any free-text notes about this router?" | Optional. |

3. **Test connectivity** (best-effort, don't block on failure):

   ```bash
   ssh -o ConnectTimeout=5 -o BatchMode=yes -p "$SSH_PORT" "$SSH_USER@$HOST" "opnsense-version; uname -a" 2>&1
   ```

   `opnsense-version` prints the exact OPNsense release (e.g. `26.7.8_2`). If it is older than 26.7, tell the user the `opnsense-maintenance` endpoint names and log paths may not match. If the command fails, surface the error to the user but still write the config — they may want to fix SSH separately.

4. **Write** `$PLUGIN_DATA_DIR/config.json` with this schema (pretty-printed, 2-space indent):

   ```json
   {
     "schema_version": 1,
     "host": "10.0.0.1",
     "ssh_user": "root",
     "ssh_port": 22,
     "ssh_key_path": null,
     "web_url": "https://10.0.0.1",
     "api": {
       "enabled": false,
       "key_ref": null,
       "secret_ref": null
     },
     "notes": null,
     "updated_at": "<ISO-8601 timestamp>"
   }
   ```

5. **Summarise**: print every field and its final value. Show the path to `config.json`. List the sibling skills the user can now invoke (`opnsense-maintenance`).

## Re-running

If `config.json` already exists, never silently overwrite. Always confirm. If the user picks **Replace**, back up the existing file with `.bak` suffix first.

## Hard rules

- **Never store the API key/secret itself in `config.json`.** Only store a reference (1Password item path, env var name, file path). The seed skill resolves the reference at runtime.
- **Never hard-code IPs, hostnames, or usernames into other skills.** Every operational skill must read from `config.json`.
- **Default values are suggestions, not assumptions.** Always confirm with the user — defaults exist to make the prompt less tedious, not to silently encode someone else's setup.
