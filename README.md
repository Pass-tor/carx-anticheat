# CarX Anti-Cheat

**Version:** 2.3.0 · **License:** AGPL-3.0 · **Stack:** FXServer · OneSync · oxmysql · Lua 5.4

A FiveM server anti-cheat with an in-game admin **Control Center**, server-authoritative
detections, **Safe Events** (signed client → server events, installable into all resources)
and a **resource whitelist**.

> All punishment decisions are made on the **server**. The client only reports evidence.

---

## Features

- **Safe Events** — signed events for every protected resource, one-command installer
- **Resource whitelist** — exempt resources from Safe Events or from chosen detections
- **Server-authoritative checks** — health, armour, godmode, weapon, ped model, teleport, speed, damage range / rate, net-event spam, entity density
- **Client checks** — godmode, noclip, invisibility, spectate, super jump, blacklisted weapons / peds / tasks / animations / plates and more
- **Signed heartbeat** and **client resource integrity** (flags injected or missing resources)
- **HARD / SOFT confidence**, **shadow mode** and **threat score** to avoid false bans
- **Honeypot events**, `entityCreating` blacklist, explosion / taze / ped-task limits
- **Ban evasion check** (all hardware tokens), ban ledger, appeal text, progressive ban durations
- **Replay evidence** (last ~30 s of position / weapon / health) attached to detections
- **Control Center (NUI)** — live feed, history, false-positive button, module quality table, live config
- Discord webhooks, txAdmin integration, ESX / QBCore / standalone

---

## Installation

1. Put `carx-anticheat` in your `resources` folder.
2. Import `sql/database.sql` (optional — tables are also created / migrated automatically on start).
3. Add to `server.cfg`. **The anticheat must start first** (after oxmysql):

```cfg
ensure oxmysql
ensure carx-anticheat
# ...all your other resources below

add_principal identifier.license:YOUR_LICENSE group.admin
add_ace group.admin command allow
```

4. Add the same staff in **txAdmin → Admin Manager** and open the txAdmin in-game menu once.
5. Edit `config/webhooks.lua` (Discord) and `config/server_config.lua` (server-side options).
6. Restart. Open the Control Center with **F9** or `/carxacmenu`.

---

## Safe Events

Every `TriggerServerEvent` sent by a protected resource carries a signed token. The server
verifies it and **removes it before your handler runs**, so handler arguments never change.
Events sent by a cheat menu are unsigned or forged.

### Console commands

Run these in the **txAdmin live console or server terminal only**. They are not available to
game clients because they write into other resources' folders.

| Command | Purpose |
|---|---|
| `carxac safeevents install dry` | Preview which resources would be patched |
| `carxac safeevents install` | Add the loader to every eligible `fxmanifest.lua` (backup: `fxmanifest.lua.carx-backup`) |
| `carxac safeevents uninstall` | Remove the loader again |
| `carxac safeevents status` | Mode, coverage, unsigned / forged events seen |
| `carxac safeevents enforce on` / `off` | `off` = monitor (log only, default) · `on` = drop and punish |
| `carxac safeevents exclude <resource>` / `include <resource>` | Safe Events resource whitelist |
| `carxac safeevents exempt <resource> <event>` / `unexempt …` | Exempt a single event |
| `carxac honeypot scan` | Re-arm honeypot events |
| `carxac help` | List all commands |

### Recommended rollout

1. `carxac safeevents install dry` → check the list.
2. `carxac safeevents install` → restart the server (or each patched resource).
3. Play in **monitor** mode (default). Nothing is blocked.
4. `carxac safeevents status` → for each legitimate unsigned event run `exempt`, or `exclude` the resource.
5. When the report is clean: `carxac safeevents enforce on`.

### What gets skipped

The anticheat itself, `oxmysql`, system resources (`[cfx-default]`, `[system]`, `[managers]`),
resources without scripts, and anything in `SafeEvents.ExcludeResources`.
JavaScript / C# resources and some escrowed resources may need to be excluded.

---

## Resource whitelist

Manage it in **Control Center → Resource Whitelist** or from the console.

| List | Meaning | Console |
|---|---|---|
| Safe Events exclusions | Resource is never signed or checked | `carxac safeevents exclude <res>` |
| Detection whitelist | While the resource is **started**, chosen detections (or all) are ignored | `carxac whitelist add <res> [all \| Reason,Reason]` |
| Event exemptions | One event of a resource is not checked | `carxac safeevents exempt <res> <event>` |
| Client resource whitelist | Resources the server does not have are flagged on the client | `ResourceIntegrity` in `server_config.lua` |

Example: `carxac whitelist add qb-ambulancejob Anti Godmode,Anti Teleport`

Runtime changes are stored in `data/rules.json` (server only). A detection whitelist applies to
**every player** while the resource runs, so for zones (safezones, hospitals) prefer the
per-player exports:

```lua
exports['carx-anticheat']:GrantGrace(playerId, durationMs, 'reason')
exports['carx-anticheat']:SuppressDetections(playerId, { 'Anti Godmode', 'Anti Teleport' }, durationMs)
exports['carx-anticheat']:ClearGrace(playerId)
```

Restrict who may call these in `CarXAC.ExportCallers` (`Enforce = true` + `Allowed = { 'my-safezone' }`).

---

## Configuration

| File | Loaded by | Contains |
|---|---|---|
| `config/config.lua` | server + client | Detection toggles, limits, punishments, messages |
| `config/server_config.lua` | **server only** | Threat score, confidence, shadow mode, heartbeat, server checks, honeypots, Safe Events, ban evasion, appeals, retention |
| `config/resource_whitelist.lua` | **server only** | Detection whitelist and framework preset catalog |
| `config/webhooks.lua` | server only | Discord webhooks |
| `data/runtime.json` | server only | Live overrides saved from the Control Center |
| `data/rules.json` | server only | Resource whitelist rules saved at runtime |

Punishments are `WARN`, `KICK` or `BAN`. Bans use txAdmin (`txaBan`) by default
(`BanSystem.Provider = "txadmin" | "internal" | "both"`).

### Avoiding false bans

- **Confidence** — in `balanced` mode a single client claim is SOFT until the server confirms it.
  SOFT detections only warn / add threat score and can never permanently ban on their own.
- **Shadow mode** — `ShadowMode = { Enable = true, Modules = { ["Anti Teleport"] = true } }` logs without punishing. Use `"*"` for everything.
- **Threat score** — detections add points; thresholds escalate WARN → KICK → BAN. Score decays over time.
- **False-positive button** — Detection History → *False positive*. The rate per module appears under Protection → *Detection quality*.

---

## Control Center

Open with **F9** / `/carxacmenu`. Shortcuts: `Esc` close, `Ctrl+K` command palette.

Dashboard · Players · Live Detections · Detection History · Protection · **Safe Events** ·
**Resource Whitelist** · Bans · Whitelist · Blacklist · Logs · Health · Configuration · Admin Tools

Admin access is checked on the server for every request: txAdmin staff, then `group.admin` /
`group.god` / `admin`, then the optional `carxac_admin` table.

---

## Exports

```lua
-- server
exports['carx-anticheat']:BanPlayer(targetId, reason, issuer)
exports['carx-anticheat']:UnbanPlayer(banId, issuer)
exports['carx-anticheat']:GetThreatScore(playerId)
exports['carx-anticheat']:GetPlayerDossier(playerId)
exports['carx-anticheat']:GrantGrace(playerId, durationMs, reason)
exports['carx-anticheat']:SuppressDetections(playerId, reasons, durationMs)
exports['carx-anticheat']:ClearGrace(playerId)

-- client / shared style
exports['carx-anticheat']:CarXAC_CHANGE_TEMP_WHITELIST(status, durationMs)
exports['carx-anticheat']:CarXAC_CHECK_TEMP_WHITELIST()
```

---

## Folder structure

```
carx-anticheat/
├── fxmanifest.lua
├── config/            config.lua · server_config.lua · resource_whitelist.lua · webhooks.lua
├── shared/            hash.lua · schema.lua
├── client/            main.lua · features.lua · session.lua · menu.lua
├── server/            main.lua · features.lua · guard.lua · session.lua · store.lua · safeevents.lua · control.lua
├── safeevents/        loader.lua            (added to other resources by the installer)
├── tables/            blacklists (weapons, peds, vehicles, events, words, ...)
├── data/              runtime.json · rules.json   (server only)
├── sql/               database.sql
├── ui/                Control Center (index.html, css, js, assets)
├── tests/             run.lua · loader_test.lua
└── tools/             syntax_check.lua
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| A resource fails to start after install | `ensure carx-anticheat` must come first. Or `carxac safeevents exclude <res>` and `uninstall` |
| Legit events show as unsigned | `carxac safeevents exempt <res> <event>`, or exclude the resource. Stay in monitor mode meanwhile |
| Players kicked for a detection after a revive / teleport script | Add the resource to the detection whitelist or use `GrantGrace` in that script |
| Control Center won't open | Check admin via txAdmin or `group.admin`, and `AdminMenu.Enable` |
| No history / stats | Start `oxmysql`; tables are created automatically on resource start |
| Console commands do nothing in game | They are console-only by design |

---

## Limits

Lua clients can be modified. Signing keys live in the client, so Safe Events stops generic menus,
replayed and forged events, but **not** a custom injector that reads the key. It is one layer;
the server-side checks do not trust the client at all. Tune new modules in shadow / monitor mode first.

---

## Tests

Run from the project root with Lua 5.3 / 5.4:

```
lua tests/run.lua           # hash, manifest patching, rules, key and token verification, helpers
lua tests/loader_test.lua   # loader strips tokens and drops forged events
lua tools/syntax_check.lua .
```

---

## Changelog

### 2.3.0
- Safe Events with console installer, resource whitelist, client resource integrity
- Signed challenge heartbeat, HARD / SOFT confidence, shadow mode, ban safety
- Server samplers, damage validation, net-event limits, `entityCreating`, entity density, honeypots
- Evidence buffer, ban ledger and evasion check, appeals, persisted audit, retention, DB migration
- Control Center: Safe Events and Resource Whitelist pages, false-positive button, module quality table
- Fixed: Control Center toasts and live feed never rendered
- Fixed: startup file check used a wrong path
- Server-only config split; `runtime.json` is no longer sent to clients

### 2.1.0
- Control Center redesign, command palette, confirm modals, About page
