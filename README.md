# Everlast: Undying Tale — AMP deployment template

An [AMP](https://cubecoders.com/AMP) GenericModule template for the **Everlast: Undying
Tale** dedicated server (Steam app `4268160`). Native Linux, no Wine or Proton.

Maintained by [Infected Game Hosting](https://infectedgamehosting.com). Shared with the
AMP community after working the template out against the game's own server build.

## What this gives you

- Native Linux server via anonymous SteamCMD (`4268160`), so nothing is redistributed
- **Explicit query port.** This is the important one — see *Query port* below
- The full 31 settings the server actually exposes, grouped by category, with the
  difficulty presets the game ships, including respawn scaling and server region
- Tail-based readiness and join/leave detection from the server log
- Docker-capable, but works without a container too

## Files

| File | Purpose |
|---|---|
| `manifest.json` | Lets AMP add this repo by URL under *Configuration → Instance Deployment* |
| `everlast.kvp` | Template root: metadata, executable, command line, console regexes, limits |
| `everlastconfig.json` | Settings manifest — the 31 server settings exposed in the AMP UI |
| `everlastmetaconfig.json` | Tells AMP which files are importable/auto-mapped |
| `everlastports.json` | UDP game port (`9003`) and query port (`27019`) |
| `everlaststart.json` | Pre-start stage that writes `queryPort` into `appsettings.json` |
| `everlastupdates.json` | SteamCMD update definition |

## Install

There are two routes. Pick whichever suits you.

### Option A — add this repo by URL (easiest)

In AMP: **Configuration → Instance Deployment → Configuration Repositories → Add**, then
give it:

```
https://github.com/Infected-Game-Hosting/everlast-amp-template
```

AMP reads `manifest.json` from the repo and pulls the templates automatically. Fetch and
refresh the template list afterwards and `Everlast` appears as a deployable app.

### Option B — copy the files in

Drop the six template files into your deployment templates directory so they sit side by
side:

```
<AMP-ADS>/Plugins/ADSModule/DeploymentTemplates/<your-templates-repo>/
```

Then refresh the template list in the ADS.

Either way, use *Configuration → Instance Deployment → Refresh Template List* to pick the
template up without restarting the controller.

### Requirements

| Requirement | Detail |
|---|---|
| **`python3` at `/usr/bin/python3`** | **Required.** The pre-start stage that writes `queryPort` is a Python one-liner. Verified on `cubecoders/ampbase:debian` (3.13) and on a bare Debian host (3.12). If your image lacks it, either install `python3` or delete the pre-start stage and set `queryPort` by hand. |
| Linux target, x86-64 | The server is a native Linux AOT build. No Wine or Proton. |
| ~1 GB free disk | Server install is ~950 MB before worlds. |
| `libicu` | Required by the .NET runtime. Present on standard Debian/Ubuntu images. |

If `python3` is missing and you leave the pre-start stage in place, AMP will fail the
start stage rather than silently boot with a wrong query port — which is the failure mode
you want, but it does mean the dependency is not optional.

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

Taken on a Ryzen 9 7950X3D (16 cores / 32 threads), containerised, reading the game
process directly. Useful for sizing plans and for sanity-checking your own install.

| State | CPU used | Memory (RSS) |
|---|---|---|
| Empty world, no players | **0.38–0.43 of one core** | ~1.5–1.8 GB |

Read that first number carefully: **under half of a single core**, roughly 1.2% of a
32-thread machine. For planning, 2 vCPU (one core) is a sensible floor for an empty
world, not because the server needs two cores but because it needs a full core with
room to breathe.

> **Thread count is not core usage.** You will see this process holding around 27–29 OS
> threads, which can look alarming on a 32-thread machine. It is not using them. In a
> 10-second sample on an empty world, only **3 threads consumed any CPU at all** and the
> busiest single thread accounted for 0.099 of a core. The rest are parked pool threads
> the .NET runtime allocates up front. Judge capacity from the CPU figure, never the
> thread count.

The world simulation — NPCs, physics, chunk streaming — is essentially the entire idle
cost. A single connected player does not move CPU measurably past that baseline.

### Scaling with players

The developer's figures from live servers, on a JIT build which they describe as heavier
than the standalone build this template runs:

| Players | CPU | RAM |
|---|---|---|
| 0–2 | 0.70 cores | 2.3–2.8 GB |
| 11 | 1.00 cores | 2.6 GB |
| 25 | 1.25 cores | 2.8 GB |
| 45 | 1.65 cores | 2.9 GB |
| 95 (peak observed) | 2.00 cores | 3.4–3.6 GB |

**RAM is nearly flat; CPU is what scales.** Two things worth noting:

- **Marginal cost per player falls as the server fills.** It runs ~0.018–0.020 cores per
  player between 11 and 45 players, then ~0.014 between 70 and 95. Scaling is sublinear,
  so sizing from the steep early slope is conservative.
- **A full 100-player server draws about 2.0–2.1 cores.** Budget one full core per
  50 players plus roughly 0.4 for the world itself.

A 1 vCPU quota is not enough — an empty world already uses ~0.40 of a core, so the entry
point should be 2 vCPU. Memory stays near 3.5 GB even at 95 players (with a one-off
~4.4 GB spike at startup), so a modest RAM allocation goes a long way; spending on cores
is what buys capacity on this title.


## Privacy: crash reporting

**This template defaults crash reporting OFF.** The game ships with Sentry crash reporting
enabled; the settings manifest here sets it to `false`, so a server built from this
template does not send anything to the developers unless you turn it on.

Worth knowing if you do enable it — the server's own config in this build is:

```json
"SentryConfig": {
  "Enabled": true,
  "SendDefaultPii": false,
  "AttachStacktrace": true,
  "TracesSampleRate": 0.2
}
```

`SendDefaultPii: false` means stack traces go without player personal data. That is the
game's setting, not a guarantee this template makes, and it can change between server
builds. Treat crash reports as data leaving your machine and decide accordingly — some
hosts will want them on to help the developers fix crashes, others will not.

## Notes and known quirks

- **`App.ForceUpdate=True`** keeps the SteamCMD validation stage running on every start.
  It lives in the app definition, not in template `SettingMappings`, so writing it to a
  template does nothing.
- **`ContainerCPUs` is a half-step.** One unit is 2 vCPU. A quota of `1` is one core.
  Getting this wrong doubles or halves the CPU available, silently.
- **Difficulty is fixed at world creation.** The setting is read when a world is created
  and is not re-read afterwards, so changing it on an existing world has no effect. The
  setting's description says so; a new world is required to change difficulty.
- **`MaxUsers` is 100** in `everlast.kvp`, matching the game's own `MaxPlayers` default and
  the 1–1000 range the server accepts. These are two separate numbers: `App.MaxUsers` is the
  panel's cap, `MaxPlayers` in `worldsettings.json` is the game's. Change both together, and
  note that the panel field writes the game file — it does not move `App.MaxUsers`.
- **Log level can flood the disk.** `Serilog.MinimumLevel.Default` at `Debug` or `Verbose`
  grows `server.log` quickly. Leave it at `Information` outside active diagnosis.
- The server writes `Worlds/<WorldName>/server.log`, which the readiness and join/leave
  regexes tail.

## Settings added in config version 4

Verified against the published server build and the vendor's own `templates/` defaults.

| Setting | Default | Notes |
|---|---|---|
| `NetworkConfig.Location` | `Usa` | Cosmetic browser region. Does not affect latency or who can join |
| `LaunchConfig.AdvertiseServer` | `true` | Off = private server, direct connect only |
| `RespawnScalingConfig.Enabled` | `true` | Server builds from 29 Oct 2026 |
| `RespawnScalingConfig.RadiusMeters` | `50` | 1–500 |
| `RespawnScalingConfig.PerPlayerSpeedup` | `0.5` | 0–10 |
| `RespawnScalingConfig.MinMultiplier` | `0.25` | 0.05–1 |
| `TransferExceptionIds` | empty | Comma-separated SteamID64s allowed in despite transfers being off |
| `Serilog.MinimumLevel.Default` | `Information` | Verbose/Debug/Information/Warning/Error/Fatal |

**`Max Players` now defaults to `100`**, not `16`. The old default produced a fresh server
that quietly accepted a sixth of the players the game allows.

**`AfkConfig.ThresholdSeconds` is deliberately not exposed.** The vendor documented it in the
README commit of 3 November 2026, but the public server build (`26380401`, published 2 November
2026 20:00 UTC) does not contain the setting — the binary has no `AfkConfig` type. Adding a
field for it would look configured and do nothing. Expose it when a build that contains it ships.

## Licence and credit

The Everlast dedicated server is distributed by the game's developers through Steam. This
template contains no game binaries and does not redistribute them — it configures
SteamCMD to download the public server build anonymously.

Please do not present servers built from this template as official Everlast servers.

Template authored by Infected Game Hosting. Issues and improvements welcome.
