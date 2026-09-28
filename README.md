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

**`launcher/RestaurantAppLauncher-1.0.25.zip`** — the launcher itself (`RestaurantLauncher.exe`
plus its `scripts/` and `tools/` folders, which it needs alongside it to work). Direct
download:

```
https://raw.githubusercontent.com/joshkim25-code/restaurant-app-releases/main/launcher/RestaurantAppLauncher-1.0.25.zip
```
SHA256: `7C510048760E79D4D7D2B0C2DEE74197D55FEB29AB589BB6086B9A90E29C0D4E`

**1.0.25** (2026-09-28): following 1.0.24's fix, a real device that already had its core services
registered and running from an earlier successful setup had its `.next-prod\BUILD_ID` deleted by
hand (the documented recovery step) to force a clean rebuild - and `npm ci` failed with `EPERM`
unlinking a native `.node` file, because the still-running `RestaurantApp` service had it loaded.
This script only ever assumed a genuinely blank machine with no services registered yet; it never
accounted for a hybrid state where services already exist but a rebuild is still needed. The
rebuild path now stops any already-registered core services first (releasing file locks) before
touching `node_modules`, and `Install-CoreService` now (re)starts a service even when it finds
one already registered, instead of assuming "already registered" still means "currently running."

**1.0.24** (2026-09-28): the actual root cause behind the entire "password stopped verifying
after being set" mystery from 1.0.19-1.0.23, and separately a corrupted `DATABASE_URL` that broke
login after a real device finally got all the way through setup. Confirmed by direct testing in
this environment: when a function's result is captured (`$x = Some-Function`), PowerShell
captures *every* object written to the success/output stream during that call - not just the
value passed to `return`. Every `Write-Output "..."` progress message inside `Ensure-
PostgresInstalled`, `Uninstall-OwnPostgres`, `Get-VerifiedRelease`, `Ensure-AppDatabase`, and
`Get-LatestManifest` was silently becoming part of their captured return values
(`$superPassword`, `$databaseUrl`, `$manifest`) - garbling `$superPassword` into a huge string of
concatenated log messages plus the real password, and `$databaseUrl` into log messages plus the
real connection string. This explains BOTH earlier mysteries at once: the password genuinely
looked like it kept "reverting" because `Ensure-AppDatabase` was comparing a garbled password
against a clean one Postgres actually had - the self-heal added in 1.0.19 "fixed" it only by
resetting Postgres's real password to match the garbled value, which is also exactly how a
garbled `DATABASE_URL` (containing the literal substring "data*base*") ended up in `.env`,
producing Prisma's `"Can't reach database server at 'base'"` on login. Fix: every progress
message in these functions now uses `Write-Host` instead, which bypasses the success stream
entirely (still visible on screen and still captured by the transcript) and can never leak into
a captured return value. Verified live: reproduced the exact corruption and confirmed the fix.
**Existing installs with an already-built app need a clean re-run** - delete
`C:\ProgramData\RestaurantApp\app\.next-prod\BUILD_ID` before retrying "Set up this machine", so
setup regenerates `.env` with a correct `DATABASE_URL` instead of reusing the corrupted one (the
`.next-prod\BUILD_ID` check is what makes it skip that step otherwise).

**1.0.23** (2026-09-28): setup got further still on a real device - all the way to `prisma
migrate deploy`, which failed with only `"prisma migrate deploy failed with exit code 1"` and no
indication of what prisma itself actually reported. This whole phase (npm ci, prisma generate,
npm run build, prisma migrate deploy) had never run end-to-end on a real device before either,
and every one of those calls only ever checked `$LASTEXITCODE`, discarding the command's real
output entirely - the same diagnostic gap already fixed once for psql calls. `Invoke-
NativeAllowingStderr` moved from `provision-machine.ps1` into the shared `lib/install-app.ps1`
(dot-sourced by both `provision-machine.ps1` and `apply-update.ps1`), alongside a new
`Invoke-CheckedCommand` helper that captures a native command's real output, streams it live to
the transcript as before, and includes it in the thrown message on a non-zero exit. Applied to
all four of `Install-AppDependencies`'s calls (`npm ci`, `prisma generate`, `npm run build`) plus
both scripts' own `prisma migrate deploy` calls - so whatever prisma is actually reporting shows
up directly in `last-error.txt` the next time this fails, instead of a bare exit code.

**1.0.22** (2026-09-28): 1.0.20's line-number diagnostic paid off immediately - the exact failing
line came back: `$roleExists = (Invoke-Psql @(...) -join "").Trim()`, a genuinely new "You
cannot call a method on a null-valued expression." Root cause, confirmed by direct testing in
this environment: `(Func @(args) -join "")` does NOT apply `-join` to `Func`'s return value the
way it visually appears to. Once the parser commits to command-invocation syntax (a bare
function name as the first token inside the parens), every token after it - including `-join`
and `""` - gets parsed as MORE ARGUMENTS to that function call, not as an operator applied
afterward. `Invoke-Psql` doesn't read `$args`, so those extra tokens were always silently
swallowed and `-join` never actually ran; the parens' value was always just `Invoke-Psql`'s own
raw return value. This bug has existed since this line was first written and was completely
invisible as long as `Invoke-Psql` happened to return a single non-null string (any real query
match) - it only crashes on `.Trim()` when the query returns nothing at all, which is exactly
what a `-tAc` role/database-existence check returns on a genuinely fresh install (the ordinary,
common case, not an edge case). Both occurrences (`$roleExists`, `$dbExists`) now wrap the
`Invoke-Psql` call in its own parens first - `((Invoke-Psql @(...)) -join "")` - forcing it to
resolve to a value before `-join` (now unambiguously an operator) is applied. Verified directly:
the broken pattern reproducibly throws the exact same error against a null-returning function,
and the double-parens form fixes it.

**1.0.21** (2026-09-28): the TLS fix from 1.0.15 came back on a real device - a fresh Postgres
installer download failed with the same `"underlying connection was closed"` error even with
`SecurityProtocol` forced to TLS 1.2. Switched from a flat assignment (replace the whole enabled
protocol set with only Tls12) to `-bor` (add Tls12 to whatever the OS already had enabled) in
both `provision-machine.ps1` and `apply-update.ps1` - a flat assignment can strip out
protocol/negotiation state some endpoints' handshake quirks still depend on even when TLS 1.2 is
the one actually selected. Separately, `get.enterprisedb.com`'s installer download specifically
has now shown three different flaky-network symptoms across independent real-device incidents
(severe throttling in 1.0.8, TLS negotiation failure in 1.0.15, a mid-download connection reset
here) - `Get-VerifiedRelease` (`lib/install-app.ps1`, shared by both scripts) now retries a
failed download or checksum mismatch up to 3 times with a 5s delay before giving up, instead of
failing outright on the first transient network hiccup.

**1.0.20** (2026-09-28): 1.0.19 got further than any prior version ever has on a real device —
far enough to hit `"You cannot call a method on a null-valued expression"`, a bare PowerShell
runtime error with no indication of which line. This is genuinely uncharted territory: no real
device has ever gotten past the password-auth step before, so the entire rest of the script
(app fetch/build/migrate) had never actually been exercised end-to-end. Rather than guess which
of several `.Method()` call sites is the culprit, the top-level error handler now appends the
failing line number and the actual source line text (from `$_.InvocationInfo`) to
`last-error.txt`, turning "something failed somewhere in this 600-line script" into an exact,
actionable line the moment it happens next, instead of costing another real-device round trip
just to find out where to even look.

**1.0.19** (2026-09-28): 1.0.18's real-error surfacing paid off immediately — the *actual* psql
text came back on the same real device: `FATAL: password authentication failed for user
"postgres"`. That's a real, reproducible rejection, not a timing gap — reaching that exact error
means the server was already up and pg_hba.conf was already asking for a password, and the
password `Reset-PostgresSuperuserPassword`'s own check had just verified moments earlier was now
being rejected. Something external changes the stored password in that window; EDB's Windows
installer (a BitRock-style bootstrapper) is the leading suspect, since `-Wait` on its launcher
process doesn't guarantee an async post-install child has actually finished touching the
instance. Rather than chase down exactly which process does it, `Ensure-AppDatabase` now
self-heals: if its first connection check fails, it re-runs the same unconditional,
password-independent trust-mode reset used right after install (`Reset-
PostgresSuperuserPassword`, now via a shared `Get-PgServiceName`) immediately before it actually
needs the connection, rather than giving up on the first failure.

**1.0.18** (2026-09-28): 1.0.17's `Invoke-NativeAllowingStderr` fix worked exactly as intended —
tested minutes later on a real device, setup now correctly reaches `Reset-
PostgresSuperuserPassword`'s own retry loop and lets it finish (it self-verified successfully),
but then failed with a *new, different, coded* error from `Ensure-AppDatabase`'s own connection
check moments later: `"Could not connect as postgres with the password this script just set,
even after 5 attempts."` That message itself was a dead end, though — it was a guess written
before this script could ever see real psql error text (every previous attempt died to 1.0.16's
underlying EAP bug before reaching this code path at all). `Wait-ForPostgresAuth` now captures
and surfaces the *actual* psql output/exit code from the last failed attempt in both places that
call it, instead of a generic "something is wrong" message, and its retry budget is raised from
5 to 8 attempts as a reasonable safety margin now that retries are confirmed to genuinely run.
If this exact error reappears, the new message will say what psql is actually reporting
(connection refused vs. a real auth mismatch vs. something else) instead of requiring another
guess-and-ship round.

**1.0.17** (2026-09-28): found the actual, sole root cause behind every "password
authentication failed" failure chased since 1.0.3 — confirmed directly by live-testing
PowerShell's own behavior, not guessed. This script sets `$ErrorActionPreference = "Stop"`
globally, and with a native command's stderr redirected via `2>&1` (used in every `psql` call
this script makes), PowerShell 5.1 throws a *terminating* exception on the native command's
very first stderr line — carrying that raw line as the exception message — regardless of the
command's real exit code, and *before* `$LASTEXITCODE` is ever checked. Every retry/
verification loop built across 1.0.11 through 1.0.16 (`Reset-PostgresSuperuserPassword`'s own
check, then the shared `Wait-ForPostgresAuth`) was correctly written but never actually got to
run past the very first failed attempt — the first failing psql call always threw straight out
of the function, skipping the retry loop entirely. This is why the exact same raw, unwrapped
`psql: FATAL: password authentication failed` error kept appearing on 1.0.16 despite it
supposedly retrying 5 times. Fix: a new `Invoke-NativeAllowingStderr` wraps every `psql` call,
temporarily relaxing `$ErrorActionPreference` to `"Continue"` around just that native call (the
standard, documented fix for this exact PowerShell 5.1 behavior) so `$LASTEXITCODE` checks and
retry loops actually run as written.

**1.0.16** (2026-09-25): the password-auth failure showed up once more, but this time from
`Ensure-AppDatabase`'s own connection, not `Reset-PostgresSuperuserPassword`'s (which now has
its own verification, and would show a different, specific error if it were the culprit) -
meaning the reset likely verified fine, but the very next connection attempt moments later
still failed once, a genuine transient/timing gap right at that boundary. The retry-with-delay
logic from 1.0.13 is now shared (`Wait-ForPostgresAuth`) between both `Reset-
PostgresSuperuserPassword`'s own check and `Ensure-AppDatabase`'s first connection, rather than
assuming one successful check earlier means every later connection is automatically safe too.

**1.0.15** (2026-09-25): fixed a fresh Windows machine failing setup with a generic
`"The underlying connection was closed: An unexpected error occurred on a send."` - Windows
PowerShell 5.1's default enabled TLS protocol set can leave TLS 1.2 disabled even on an
up-to-date Windows install, and every host this script talks to (nodejs.org,
get.enterprisedb.com, raw.githubusercontent.com) requires it. `provision-machine.ps1` and
`apply-update.ps1` now both force `[Net.ServicePointManager]::SecurityProtocol =
[Net.SecurityProtocolType]::Tls12` before any web request. Every download/fetch also now
includes the actual URL in its error message if it fails, instead of a bare, unhelpful .NET
exception message with no indication of which of several requests this script makes was the
one that failed.

**1.0.14** (2026-09-25): no more manual cleanup needed between retries. `provision-machine.ps1`
now unconditionally removes its own dedicated PostgreSQL instance (exact major version, exact
install directory, exact service name - never anything else on the machine) before every fresh
install, via a new `Uninstall-OwnPostgres`, rather than requiring the person to uninstall it by
hand in Settings between attempts. Safe specifically because of when this script ever runs at
all - only when core services aren't registered yet - so no real restaurant data can exist in
this instance regardless of how far a previous attempt got.

**1.0.13** (2026-09-25): the password-auth failure came back yet again on a *fourth* real
device, a genuinely clean one this time (only ever had PostgreSQL 16 on it, ruling out 1.0.12's
wrong-coexisting-instance bug). `Reset-PostgresSuperuserPassword` now verifies its own work
immediately - reconnecting with the new password, retrying up to 5 times with a short delay -
rather than trusting that the `ALTER USER` inside it "succeeded" and finding out from a
downstream symptom later. Covers the case where the final restart (reverting `pg_hba.conf` back
to requiring a password) genuinely needs longer than a few seconds to be ready on some
machines; if the password still doesn't verify after 5 attempts, it now fails with a specific
error naming this exact step, instead of the same generic `Ensure-AppDatabase` connection
failure seen every time so far.

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

`1.0.0` through `1.0.24` have been removed rather than kept for reference — use `1.0.25`.

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
