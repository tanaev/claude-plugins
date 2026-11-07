---
name: home-assistant
description: Query and control the user's Home Assistant (smart home) — lights, climate, sensors, switches, covers, automations, dashboards, history, energy, "what do I have", "is X on", "turn on/off", temperatures. Use whenever the user mentions Home Assistant, HA, their home/house/apartment devices, rooms, or any smart-home entity.
---

# Home Assistant

The `ha` command (on PATH from this plugin) talks to the user's Home Assistant over its REST and WebSocket APIs, and edits YAML over SSH.

Everything the user owns lives in the workspace `~/HomeAssistant` (or `$HA_HOME`):
- `data/summary.md` — **read first**: a small overview of areas, devices and automations.
- `data/inventory.md` — the full entity list. It can be 100+ KB, so never read it whole; grep it or use `ha find`.
- `notes.md` — the user's own facts: aliases ("бойлер" = switch.x), what devices really do, quirks, preferences. Read it before resolving names. When the user says "remember…", or you learn a non-obvious mapping or quirk, append it there.
- `config/` — a local mirror of HA's YAML. `dashboards/` holds pulled dashboard JSON. `data/backups/` keeps the remote copies saved before each push.

**If `ha` says "not configured" or "no token saved", or `ha doctor` shows failures, run the `setup` skill of this plugin (`/home-assistant:setup`).** Never ask the user to paste a token into the chat.

## Workflow
1. Resolve names with `ha find <words> [-a area] [-d domain]` (offline cache) or with notes.md.
2. Get live values with `ha state <entity_id...>`, or `ha state -a <area>` / `-d <domain>`. Add `-v` for attributes such as brightness or hvac modes.
3. Act with `ha call <domain.service> <entity_id...> [--data '{json}']`. If unsure about fields, run `ha services <domain>` first. Add `--response` for services that return data.
4. History and debugging: `ha history <id> --hours 48`, `ha logbook [id]`, `ha template '{{ ... }}'`.
5. Escape hatches: `ha api GET|POST /api/<path> [json]` for REST, and `ha ws <type> [json]` for WebSocket. WebSocket covers the entity/device/area registries, helpers (`input_boolean/create`…), config entries, long-term statistics (`recorder/statistics_during_period`) and themes.

The cache auto-refreshes when older than 24h. Run `ha sync` if an entity is missing or the user says they added devices; it prints the added and removed entities.

For history over many days, prefer `ha ws recorder/statistics_during_period` (5minute/hour/day mean/min/max) over raw `/api/history`: raw history for many entities over weeks is huge and times out.

## Editing YAML (automations, scripts, scenes, configuration.yaml, packages, templates)
Needs SSH (see setup). `config/` is a local mirror; never edit files on HA directly over ssh.
1. `ha pull` downloads fresh copies. Always pull first: the HA UI also writes `automations.yaml`, `scripts.yaml` and `scenes.yaml`. If pull refuses because of unpushed edits, show `ha diff` to the user.
2. Edit files in `~/HomeAssistant/config/` with the normal Read/Edit tools. Keep the existing style. Give new automations a unique `id:` so they stay editable in the UI. Use entity ids from the inventory, and verify them with `ha find`.
3. `ha diff` shows the change. Show it to the user and get a confirmation before pushing.
4. `ha push` refuses if the file changed on HA since the pull. Otherwise it backs up the remote copies, uploads, and runs HA's config check, restoring the old files automatically if the check fails. It reloads automations, scripts, scenes and groups itself. For other files it prints what to run next: `homeassistant.reload_all` is usually enough, a restart needs the user's confirmation.
- A new integration added in YAML (a new top-level key) usually needs a restart. Many integrations (history_stats, template, utility_meter, derivative…) can instead be created as UI helpers through the config-flow REST API (`/api/config/config_entries/flow`), which needs no restart.
- `secrets.yaml` and `zigbee2mqtt/` are intentionally never pulled. Reference secrets with `!secret name`, and ask the user to add new secret values themselves.
- `.storage/` is JSON owned by HA. Never edit it over ssh; use the WebSocket API instead.

Without SSH, automations and scripts can still be managed through the REST config API: `ha api GET /api/config/automation/config/<id>`, and `POST` the same path with the full config, then `automation.reload`.

## Editing UI dashboards (cards, views)
Dashboards go through the WebSocket API, not files, so they need no SSH.
1. `ha dashboard list` shows the dashboards. `ha dashboard pull [name]` saves one to `dashboards/<name>.json`; the default Overview is `lovelace`.
2. Edit that JSON. Before wiring a card, check that its entity ids exist and carry the right units (`ha state ...`). Custom cards must be installed (HACS) and registered as resources (`ha ws lovelace/resources`).
3. Run `ha dashboard diff [name]`, show it, and get confirmation. Then `ha dashboard push [name]`: it refuses if the dashboard changed in the UI since the pull, and backs up the old version.

## Rules
- Confirm before anything with physical risk or that is hard to undo: locks, garage doors and gates, alarms, heating setpoints far from current values, `ha push`, `ha dashboard push`, restarting HA, deleting entities or devices.
- Turning lights or media on and off, and reading data, needs no confirmation.
- Don't install third-party code (HACS repositories, add-ons) on the user's HA yourself; tell them what to install and verify afterwards.
- Reply with the result in the user's language, not with raw dumps.
