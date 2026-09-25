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

**`launcher/RestaurantAppLauncher-1.0.5.zip`** — the launcher itself (`RestaurantLauncher.exe`
plus its `scripts/` and `tools/` folders, which it needs alongside it to work). Direct
download:

```
https://raw.githubusercontent.com/joshkim25-code/restaurant-app-releases/main/launcher/RestaurantAppLauncher-1.0.5.zip
```
SHA256: `AC7BFE7626E5BE6B5EB9688EF7261DE24F9437D70C2BB15B37BBA1F1AADB6737`

**1.0.5** (2026-09-25): fixed setup appearing to hang for over an hour on a real device,
stuck downloading the ~350MB Postgres installer. Root cause: Windows PowerShell 5.1's
`Invoke-WebRequest` renders a progress bar by default, and updating it per chunk has severe
overhead on large files — a well-documented issue that can make a multi-hundred-MB download
take dramatically longer than a normal browser download. `provision-machine.ps1` now sets
`$ProgressPreference = "SilentlyContinue"` before any download.

**1.0.4** (2026-09-24): the Postgres installer itself started failing with a bare "exited with
code 1" on a third real device (1.0.3's password fix worked - this is a separate failure,
further into the same step). EDB's installer doesn't say why on its own, so
`provision-machine.ps1` now also passes `--debugtrace`/`--debuglevel 4` to get its own trace
log written to `provision-logs\postgres-installer-<timestamp>.log` alongside the main
transcript.

**1.0.3** (2026-09-24): fixed `provision-machine.ps1`'s fresh-install path failing with
`psql: FATAL: password authentication failed for user "postgres"` — the randomly generated
superuser/app passwords (Base64, could contain `+`/`/`/`=`) didn't always survive intact through
the Postgres installer's own command-line argument parsing. Passwords generated for that path
are now alphanumeric-only. **If you hit this exact error on 1.0.2 or earlier**: PostgreSQL was
actually installed on your machine before the failure, just under a password nobody knows —
uninstall it via Settings → Apps first (and delete `C:\Program Files\PostgreSQL` if anything's
left over) before retrying, so it gets a genuinely fresh install.

**1.0.2** (2026-09-24): fixed the "Set up this machine" button still not appearing after 1.0.1
on a real second Windows 10 device — turned out to be a z-order/paint-over issue (a Dock.Fill
status label sitting over it), not just a position issue.

`1.0.0` through `1.0.4` have been removed rather than kept for reference — use `1.0.5`.

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
