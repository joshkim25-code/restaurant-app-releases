# Restaurant App Releases

Release artifacts and update manifests for the Restaurant App launcher's auto-update
mechanism. This repo intentionally holds **no source code** — the app itself lives in a
private repository. This one only ever contains:

- Built release zips (published as GitHub Release assets)
- `latest.json` — the manifest the launcher fetches (anonymously, no auth) to check for updates

## `latest.json`

```json
{
  "version": "1.0.0",
  "releasedAt": "2026-09-23",
  "notes": "What changed in this release",
  "downloadUrl": "https://github.com/joshkim25-code/restaurant-app-releases/releases/download/v1.0.0/restaurant-app-1.0.0.zip",
  "sha256": "...",
  "minLauncherVersion": "1.0.0"
}
```

The launcher fetches this file directly from `main` via `raw.githubusercontent.com` and
compares `version` against the currently installed app's own `package.json` version.
