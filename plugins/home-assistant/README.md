# Home Assistant for Claude Code

**English** · [Українська](README.uk.md)

A plugin that lets Claude manage your Home Assistant in plain language: "what's on right now?", "what's the temperature in the bedroom?", "make an automation…", "add a card to the dashboard". You set up access once and never explain it again.

## Install

In Claude Code:
```
/plugin marketplace add tanaev/claude-plugins
/plugin install home-assistant@tanaev
```
Restart Claude Code, then type:
```
/home-assistant:setup
```
or just "connect my Home Assistant". Claude walks you through it step by step. The rest of this page covers the same ground, so you know what to expect.

## What you need

| What | Where to get it | Required? |
|---|---|---|
| **Python 3.8+** | usually already there on macOS and Linux (`python3 --version`) | yes |
| **HA address** | the one you open in a browser: `http://homeassistant.local:8123`, `http://192.168.x.x:8123`, Nabu Casa or your own domain | yes |
| **Access token** | in HA, click your name at the bottom of the sidebar → **Security** tab → **Long-lived access tokens** → **Create token** → name it `Claude` → copy it. Use an **administrator** account | yes |
| **SSH access to the config** | the **Advanced SSH & Web Terminal** add-on (Settings → Add-ons → Add-on Store) with your public SSH key. Claude sets up the key and `~/.ssh/config` itself | no, only for editing YAML |

**Don't paste the token into the chat.** Copy it and tell Claude "copied". Claude reads the token straight from the clipboard, checks it against HA and saves it in the Keychain (macOS) or libsecret (Linux). Claude never sees the token itself.

Almost everything works without SSH: states, control, history, helpers and dashboards. SSH is needed only for `automations.yaml`, `configuration.yaml` and YAML packages.

## Where things live

Everything that's yours is kept in `~/HomeAssistant` (set `HA_HOME` to use another folder):
```
~/HomeAssistant/
├── config.json     ← URL, SSH host, config path
├── notes.md        ← your notes: what things are called, device quirks (Claude reads and adds to it)
├── data/           ← device list (summary.md, inventory.md), backups taken before each push
├── config/         ← local copy of HA's YAML
└── dashboards/     ← dashboard configs
```

## Safety

- **What Claude does without asking:** reads data, turns lights and media on and off.
- **What needs your confirmation:** locks, gates, alarms, big heating changes, uploading changed config or dashboards, restarting HA. Claude Code also asks before each service call unless you've allowed it.
- **How `push` protects your config:** before uploading, it saves a copy of the files on HA. After uploading, it runs HA's config check. If the check fails, it puts the old files back automatically.
- **What never reaches Claude:** `secrets.yaml` and `zigbee2mqtt/` are never pulled.
- **How to revoke access:** delete the token in your HA profile.

## `ha` commands

Claude runs these itself, but they're handy if you want to check something by hand:

| Command | What it does |
|---|---|
| `ha doctor` | what's configured and what works |
| `ha find kitchen light` / `ha state light.x` | find devices / current state |
| `ha call light.turn_on light.x --data '{"brightness_pct":40}'` | call a service |
| `ha history sensor.x --hours 48`, `ha logbook` | history |
| `ha pull` → edit → `ha diff` → `ha push` | edit YAML |
| `ha dashboard pull/diff/push [name]` | dashboards |
| `ha api …`, `ha ws …` | any REST or WebSocket API request |

## Troubleshooting

| Symptom | What to do |
|---|---|
| `ha: command not found` | `/plugin` → enable `home-assistant`, restart Claude Code |
| `cannot reach …` | check the address; a home address won't work away from home without a VPN |
| `token rejected` / HTTP 401 | create a new token and tell Claude "update the token" |
| `ssh failed` | is the add-on running? is the key added? are you at home or on VPN? |
