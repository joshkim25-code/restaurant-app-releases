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

**`launcher/RestaurantAppLauncher-1.0.12.zip`** — the launcher itself (`RestaurantLauncher.exe`
plus its `scripts/` and `tools/` folders, which it needs alongside it to work). Direct
download:

```
https://raw.githubusercontent.com/joshkim25-code/restaurant-app-releases/main/launcher/RestaurantAppLauncher-1.0.12.zip
```
SHA256: `453F194D625F2E195E2D8A73B23E3CC44C10B94B444938FD427B37BB23AB5867`

**1.0.12** (2026-09-25): 1.0.11's explicit password-reset never actually ran against the right
instance. `Find-PgBin`/`Find-PgDataDir` searched `C:\Program Files\PostgreSQL` and picked
whichever version directory sorted highest - with a leftover PostgreSQL 17 still present on the
same real device (from an attempt before this script switched to installing 16), that always
resolved to 17, not the 16 this script had just installed and actually needed to fix. The
password reset was silently running against the wrong, unrelated instance every time. Also
fixed the same-class bug in the post-install service lookup, which used a `postgresql-x64-*`
wildcard that matches both coexisting services at once and would have passed an invalid,
multi-name value to `Restart-Service`. Both now derive the exact target deterministically from
`$PostgresVersion` (the version this script itself just installed), never searching or
guessing among whatever else happens to be on the machine.

**1.0.11** (2026-09-25): the `psql: FATAL: password authentication failed for user "postgres"`
bug came back a *third* time on the same real device, this time on a genuinely fresh, isolated
PostgreSQL 16 install (1.0.10's fix) with no other explanation available - `--superpassword`
has now proven unreliable across three separate real-machine failures for reasons that don't
fully reduce to any single root cause. Rather than continuing to chase why, the installer's
`--superpassword` is no longer trusted at all: after install, `provision-machine.ps1` now
explicitly sets the superuser password itself via a standard, well-established Postgres
recovery technique - temporarily allow passwordless ("trust") local connections in
`pg_hba.conf`, use that to run a normal `ALTER USER ... WITH PASSWORD`, then restore
`pg_hba.conf` exactly as it was. This works unconditionally, independent of whatever
`--superpassword` does or doesn't do.

**1.0.10** (2026-09-25): 1.0.9's dedicated-port fix (5433) wasn't the whole story on the same
real device with an existing POS-owned PostgreSQL 17 install - the installer reported success
but nothing ended up listening on 5433, because it detected the pre-existing PostgreSQL 17
(left over from an earlier attempt before 1.0.9, same major version this script always
installs) and didn't create a genuinely separate instance, even with a different `--serverport`.
Standard PostgreSQL installers don't reliably support two side-by-side instances of the *same*
major version. This app now installs **PostgreSQL 16** instead of 17 - a different major
version installs as a fully independent instance (own directory, own service, own everything)
regardless of what else is already on the machine, sidestepping the whole class of problem
without needing to know or touch whatever else is there.

**1.0.9** (2026-09-25): fixed a real port collision, found on a real device that already ran
a POS system with its own separate PostgreSQL install. `provision-machine.ps1` previously
always installed on Postgres's default port (5432) and only checked whether a service literally
named `postgresql-x64-*` already existed before deciding to install fresh — a check that can't
tell "we already set this up" apart from "something else entirely installed Postgres for its
own reasons." This app's dedicated PostgreSQL instance now always installs on its own port
(5433, not configurable via the UI, but a `-DbPort` script parameter) so it can never collide
with anything else already on the machine, and the "already installed" check now looks at
whether *that specific port* is already listening, not at service names, which multiple
unrelated Postgres installs can share.

**1.0.8** (2026-09-25): the `psql: FATAL: password authentication failed for user "postgres"`
bug (thought fixed in 1.0.3) came back on a real device running 1.0.6. Root cause: 1.0.4's
`--debugtrace`/`--debuglevel` flags, added to the Postgres installer call for diagnostics, were
only ever confirmed against EDB's docs for their commercial EPAS product — never for the plain
community PostgreSQL installer this script actually uses. Most likely explanation: those
unrecognized flags, sitting right after `--superpassword` in the argument list, corrupted how
the installer parsed that value. Removed. Also added real SHA256 verification for the Postgres
installer download itself (there wasn't one before) — a corrupted/truncated download was
another live possibility given how unreliable the connection to that server can be, confirmed
on the same real device (severely throttled: ~350-500 kbps against a 74 Mbps connection, traced
to that specific server/host, not the device or antivirus).

**1.0.7** (2026-09-25): a freshly-provisioned machine's generated `.env` defaulted
`VOICE_PUBLIC_URL` to `http://localhost:3100` — silently broken on every fresh install, since
voice now runs cloud-hosted (Railway) for every restaurant, not locally. Billing pages and the
new auto-restore-on-login both depend on this being correct. `provision-machine.ps1` now bakes
in the real production URL (a public HTTPS endpoint, not a credential — safe to include, unlike
`CLOUD_DATABASE_URL`, which stays blank).

`1.0.0` through `1.0.11` have been removed rather than kept for reference — use `1.0.12`.

**1.0.6** (2026-09-25): setup failures now show the real reason directly in the launcher's own
status text, instead of a generic "check provision-logs" message pointing at a log file the
person then had to go find and read. `provision-machine.ps1` writes a clean single-message
`last-error.txt` next to its full transcript on any failure; `MainForm.cs` reads it
(`MachineProvisioner.ReadLastError`) and shows it directly. Two explicit small fixes to stale
partial downloads came along with this: `provision-machine.ps1` and `lib/install-app.ps1` now
delete any leftover partial file before retrying a download, rather than relying on
`Invoke-WebRequest`'s implicit overwrite.

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
