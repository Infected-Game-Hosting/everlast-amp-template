# Everlast: Undying Tale — AMP deployment template

An [AMP](https://cubecoders.com/AMP) GenericModule template for the **Everlast: Undying
Tale** dedicated server (Steam app `4268160`). Native Linux, no Wine or Proton.

Maintained by [Infected Game Hosting](https://infectedgamehosting.com). Shared with the
AMP community after working the template out against the game's own server build.

## What this gives you

- Native Linux server via anonymous SteamCMD (`4268160`), so nothing is redistributed
- **Explicit query port.** This is the important one — see *Query port* below
- The full 23 settings the server actually exposes, grouped by category, with the
  difficulty presets the game ships
- Tail-based readiness and join/leave detection from the server log
- Docker-capable, but works without a container too

## Files

| File | Purpose |
|---|---|
| `everlast.kvp` | Template root: metadata, executable, command line, console regexes, limits |
| `everlastconfig.json` | Settings manifest — the 23 server settings exposed in the AMP UI |
| `everlastmetaconfig.json` | Tells AMP which files are importable/auto-mapped |
| `everlastports.json` | UDP game port (`9003`) and query port (`27019`) |
| `everlaststart.json` | Pre-start stage that writes `queryPort` into `appsettings.json` |
| `everlastupdates.json` | SteamCMD update definition |

## Install

Drop these six files into your deployment templates directory, so that
`everlast.kvp`, `everlastconfig.json`, `everlastmetaconfig.json`, `everlastports.json`,
`everlaststart.json` and `everlastupdates.json` sit side by side:

```
<AMP-ADS>/Plugins/ADSModule/DeploymentTemplates/<your-templates-repo>/
```

Refresh the template list in the ADS and `Everlast` appears as a deployable app. The
template expects a Linux target with `libicu` present and roughly 1 GB of free disk for
the server install.

To pick up the template without a controller restart, use ADS's *Configuration →
Instance Deployment → Refresh Template List*.

## Query port — read this

The game's own README describes the Steam query port as **derived** from the game port:

```
27016 + (gamePort % 5)
```

For the default game port `9003` that yields **`27019`**, which collides with another
game's default query port. Two Everlast servers on one machine, or Everlast beside that
other title, can end up fighting over the same query port.

**`NetworkConfig.queryPort` can be set explicitly and the server honours it.** Newer
server builds added it as a first-class setting (live on Steam since 2026-10-28). This
template therefore sets it explicitly and never relies on the derived value:

- `everlastports.json` declares `QueryPort` on `27019`
- `everlaststart.json` runs a pre-start stage that writes
  `NetworkConfig.queryPort` into `appsettings.json` from AMP's assigned port, so AMP's
  port allocation always wins
- the settings manifest exposes it for manual override

If you are on an older server build without `NetworkConfig.queryPort`, the derived value
still applies and you should avoid stacking servers whose ports would collide.

## Verified measurements

Taken on a Ryzen 9 7950X3D, containerised, reading the game process directly. Useful for
sizing plans and for sanity-checking your own install.

| State | CPU | RSS | Threads |
|---|---|---|---|
| Empty world, no players | **0.38–0.43 of a core** | ~1.5–1.8 GB | 27–29 |

The world simulation — NPCs, physics, chunk streaming — is essentially the entire idle
cost. A single connected player does not move CPU measurably past that baseline.

The developer's own live-server figures show CPU scaling with player count rather than
RAM:

| Players | Cores | RAM |
|---|---|---|
| 0–2 | 0.70 | 2.3–2.8 GB |
| 11 | 1.00 | 2.6 GB |
| 25 | 1.25 | 2.8 GB |
| 45 | 1.65 | 2.9 GB |
| ~70 | — | ~3.0 GB |

**RAM is nearly flat; CPU is what scales.** Roughly 0.019 cores per player between 11 and
45 players. A 1 vCPU quota is not enough — an empty world already uses ~0.40 of a core,
so the entry point should be 2 vCPU. Their measurements were on a JIT build on a Ryzen 5
5600X, described as somewhat heavier than the standalone build this template runs.

## Notes and known quirks

- **`App.ForceUpdate=True`** keeps the SteamCMD validation stage running on every start.
  It lives in the app definition, not in template `SettingMappings`, so writing it to a
  template does nothing.
- **`ContainerCPUs` is a half-step.** One unit is 2 vCPU. A quota of `1` is one core.
  Getting this wrong doubles or halves the CPU available, silently.
- **Difficulty is fixed at world creation.** The setting is read when a world is created
  and is not re-read afterwards, so changing it on an existing world has no effect. The
  setting's description says so; a new world is required to change difficulty.
- **`MaxUsers` is 16 by default** in `everlast.kvp`. The game supports far more; raise it
  if you are selling larger slots. This template ships the stock default deliberately.
- The server writes `Worlds/<WorldName>/server.log`, which the readiness and join/leave
  regexes tail.

## Licence and credit

The Everlast dedicated server is distributed by the game's developers through Steam. This
template contains no game binaries and does not redistribute them — it configures
SteamCMD to download the public server build anonymously.

Please do not present servers built from this template as official Everlast servers.

Template authored by Infected Game Hosting. Issues and improvements welcome.
