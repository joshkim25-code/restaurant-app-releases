# Restaurant App Releases

Release artifacts and update manifests for the Restaurant App launcher's auto-update
mechanism, plus the launcher itself. This repo intentionally holds **no source code** — the
app itself lives in a private repository. This one only ever contains:

- Built app release zips, under `releases/` (published as raw files - no real GitHub Release
  objects, since publishing here happens without GitHub API/CLI access)
- `latest.json` — the manifest the launcher fetches (anonymously, no auth) to check for updates
- The launcher itself, under `launcher/` (see below)

## `latest.json`

```json
{
  "version": "1.0.0",
  "releasedAt": "2026-09-23",
  "notes": "What changed in this release",
  "downloadUrl": "https://raw.githubusercontent.com/joshkim25-code/restaurant-app-releases/main/releases/restaurant-app-1.0.0.zip",
  "sha256": "...",
  "minLauncherVersion": "1.0.0"
}
```

The launcher fetches this file directly from `main` via `raw.githubusercontent.com` and
compares `version` against the currently installed app's own `package.json` version.

## Downloading the launcher

**`launcher/RestaurantAppLauncher-1.0.2.zip`** — the launcher itself (`RestaurantLauncher.exe`
plus its `scripts/` and `tools/` folders, which it needs alongside it to work). Direct
download:

```
https://raw.githubusercontent.com/joshkim25-code/restaurant-app-releases/main/launcher/RestaurantAppLauncher-1.0.2.zip
```
SHA256: `1329A02CA879966231AE603CE0F57D5E4CE226D1512A094D90AE02D4368FD2DF`

**1.0.2** (2026-09-24): fixes the "Set up this machine" button still not appearing after 1.0.1
on a real second Windows 10 device — turned out to be a z-order/paint-over issue (a Dock.Fill
status label sitting over it), not just a position issue. `1.0.0`/`1.0.1` were both broken on
this specific path and have been removed rather than kept for reference — use `1.0.2`.

Unzip it anywhere and run `RestaurantLauncher.exe`.

**⚠️ Not yet code-signed.** This build is unsigned. On a machine with Windows 11's Smart App
Control enforced (increasingly the default on new installs), it will be **blocked outright,
with no "Run anyway" override** — this is a known, real, currently-unsolved gap, not a bug to
report. A real Authenticode certificate is needed to fix this for real distribution; until
then, this only reliably runs on machines where Smart App Control is off or still in its
"Evaluation" phase. See the main app repo's `launcher/README.md` ("Code signing" section) for
the full explanation.

Assumes Postgres and the core Windows Services are already set up on the target machine
(`provision-machine.ps1`, bundled inside, can set these up from scratch on a genuinely blank
machine - the launcher offers this automatically if it finds none of the services registered).
