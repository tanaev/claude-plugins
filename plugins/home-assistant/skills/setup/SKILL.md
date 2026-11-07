---
name: setup
description: Connect Claude to the user's Home Assistant step by step — URL, access token, optional SSH for YAML editing, permissions, first sync. Use when the user asks to set up / connect / configure Home Assistant access, when `ha` reports "not configured" or "no token saved", or when `ha doctor` shows failures.
---

# Home Assistant setup

Walk the user through connecting Claude to their Home Assistant. Talk in the user's language, one step at a time, and wait for them where a step needs their hands. Skip steps that `ha doctor` already shows as `ok`.

The workspace is `~/HomeAssistant` (override with `$HA_HOME`). It holds config, cache, notes and the YAML mirror. The token never goes into the chat or into a file Claude reads.

## 0. Check the current state
Run `ha doctor`. If everything is `ok`, say so and stop (offer step 4 or 5 if they're missing). If `ha` is not found, the plugin isn't enabled: tell the user to run `/plugin` and enable `home-assistant`, then restart Claude Code.

Python 3.8+ is required: check `python3 --version`.

## 1. Home Assistant URL
Ask how they reach HA in a browser. Typical answers:
- At home: `http://homeassistant.local:8123` or `http://<ip>:8123`.
- From anywhere: Nabu Casa (`https://xxxx.ui.nabu.casa`) or their own domain behind a reverse proxy.

Check it: `curl -s -o /dev/null -w '%{http_code}\n' <url>/api/`. **401 means HA is reachable** (it just wants a token). Anything else (000, timeout, 404) means wrong address or not reachable from this computer: help fix that first. A LAN-only address won't work away from home unless they have a VPN.

## 2. Access token
Tell the user (adapt to their language):
1. Open Home Assistant in the browser and click your user name at the bottom of the left sidebar → **Security** tab.
2. Scroll to **Long-lived access tokens** → **Create token**, name it `Claude`.
3. **Copy** the token (it's shown only once). **Don't paste it into the chat**, just say when it's copied.

The token has the same rights as that user. Use an **administrator** account: config checks, registries, helpers and dashboards need admin.

When they confirm, run:
```
ha setup --url <url> --token-from-clipboard
```
It validates the token against HA before saving it to the OS keyring (macOS Keychain, or libsecret on Linux; otherwise `~/HomeAssistant/.token` with mode 600), then does the first sync and creates `notes.md`.
- `token rejected` → they copied something else or the token was deleted; create a new one.
- Clipboard unavailable (e.g. a remote shell): ask them to run `! ha setup --url <url>` in this prompt, or `ha setup --url <url>` in a separate terminal, and paste the token at the hidden prompt.

## 3. SSH for YAML editing (optional, recommended)
Without SSH, Claude can still control devices, read history, and edit dashboards and helpers. SSH is needed only to pull and push YAML (`automations.yaml`, `configuration.yaml`, packages…). Ask if they want it.

Find the install type: run `ha api GET /api/config` and check whether `"hassio"` is in `components`. If it is, it's **HA OS / Supervised** (add-ons available). If not, it's **Container/Core**, and SSH goes to the host machine.

### HA OS / Supervised
1. Key: if `~/.ssh/id_ed25519.pub` doesn't exist, ask, then run `ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -C claude-ha`. Show the user the content of the `.pub` file (it's public, safe to show).
2. The user installs the add-on: **Settings → Add-ons → Add-on Store → "Advanced SSH & Web Terminal"** (community; preferred) or **"Terminal & SSH"** (official).
3. Add-on **Configuration** tab:
   - Advanced SSH: under `ssh:` set `username: hassio` and add the public key to `authorized_keys`; leave `password` empty.
   - Terminal & SSH: add the key to `authorized_keys`; the user is `root`. In the **Network** section, set the SSH port to `22` (it's disabled by default).
   Save, **Start** the add-on, and enable "Start on boot".
4. With the user's OK, add a host alias to `~/.ssh/config` (create the file with mode 600 if missing; don't touch existing hosts):
   ```
   Host homeassistant
       HostName <ip or hostname of HA, without http/port>
       User hassio            # root for "Terminal & SSH"
       Port 22
       IdentityFile ~/.ssh/id_ed25519
   ```
5. Test: `ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new homeassistant ls /config/configuration.yaml`.
6. Save it: `ha setup --ssh-host homeassistant` (the token is kept), then `ha pull`.

### Container / Core
SSH to the machine running HA (any user that can read and write the config dir). Set it up the same way from step 4, then `ha setup --ssh-host <alias> --config-dir <path of the mounted config dir, e.g. /opt/homeassistant/config>`. `ha pull` needs `find` and `tar` on that host.

SSH usually works only on the home network (or over VPN), while the API URL may work everywhere. That's fine: pull and push just fail away from home.

## 4. Permissions (optional)
Offer to allow read-only commands without prompts. With their OK, merge these into `permissions.allow` in `~/.claude/settings.json` (read it first; keep existing entries; valid JSON):
```
"Bash(ha find:*)", "Bash(ha state:*)", "Bash(ha sync:*)", "Bash(ha services:*)",
"Bash(ha history:*)", "Bash(ha logbook:*)", "Bash(ha check:*)", "Bash(ha doctor:*)",
"Bash(ha pull:*)", "Bash(ha diff:*)", "Bash(ha ls:*)",
"Bash(ha dashboard list:*)", "Bash(ha dashboard pull:*)", "Bash(ha dashboard diff:*)",
"Read(~/HomeAssistant/**)", "Edit(~/HomeAssistant/notes.md)",
"Edit(~/HomeAssistant/config/**)", "Edit(~/HomeAssistant/dashboards/**)"
```
`ha call`, `ha push`, `ha dashboard push`, `ha api` and `ha ws` stay behind a prompt on purpose.

## 5. Finish
1. `ha doctor`: everything they set up should be `ok`.
2. Read `~/HomeAssistant/data/summary.md` and give a short overview: HA version, areas, how many lights, sensors and automations.
3. Ask them to tell you house-specific facts worth remembering (what an oddly named switch controls, nicknames for rooms), and put them in `notes.md`.
4. Show a few things to try, in the user's language, e.g. "what's on right now?", "what's the temperature in the bedroom?", "make an automation: hallway light on motion after sunset", "add a card with … to the dashboard".

## Changing things later
- New token or new URL: `ha setup --url <url> --token-from-clipboard`.
- Remove access: delete the token in HA (profile → Security), and the keyring entry (macOS: `security delete-generic-password -s home-assistant -a ~/HomeAssistant` with the full path).
