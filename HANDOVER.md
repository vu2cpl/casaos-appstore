# casaos-appstore — HANDOVER

Last updated: 2026-09-17

## What this is

A single-app CasaOS store (`vu2cpl-immich`) that republishes the official
Immich compose soon after each release, so CasaOS's native **Update** button
keeps the shack's Immich current. Consumer: `macmini-server`
(192.168.1.213), whose installed app `immich`
(`/var/lib/casaos/apps/immich/docker-compose.yml`) carries
`store_app_id: vu2cpl-immich`. Public repo (Manoj's explicit choice,
2026-09-17), since CasaOS can't download from a private one.

## Why it exists (2026-09-17 findings)

- Official CasaOS store Immich = v2.7.2, never moved to v3 (v3.0.0 came
  out 2026-07-02). Big Bear = v3.1.0, last bumped 2026-08-02. The shack runs
  3.2.2, so neither store could run its database (Immich won't downgrade).
- CasaOS itself is effectively unmaintained (last release v0.4.15,
  2024-12-19; AppManagement last commit 2025-04-16). The box runs
  casaos-app-management v0.4.5.
- CasaOS source facts this design relies on (CasaOS-AppManagement
  `service/appstore_management.go`, `service/compose_app.go`,
  `service/appstore.go`):
  - update available = main service image **tag** differs from the store
    copy's (digest comparison only for `:latest`); cached 1 h, purged on a
    store refresh;
  - `Update()` requires identical service names in both directions, then
    replaces **only `image:`** per service, marshals the compose back
    and pulls/applies. Volumes, env and paths are preserved;
  - store refresh every 10 min, **skipped unless HEAD Content-Length
    changed** → GitHub branch archives (no Content-Length) never refresh
    after the first download; release assets work;
  - store app ID = compose `name:`; duplicate IDs across stores resolve
    in random map order, hence `vu2cpl-immich` rather than `immich`.

## Current state

- Workflow `update-immich.yml`: every 6 h (`17 */6 * * *`) + manual
  dispatch (`approve`, `force` inputs).
- Store URL:
  `https://github.com/vu2cpl/casaos-appstore/releases/latest/download/casaos-appstore.zip`
- Telegram secrets **not set** (notices skipped); GitHub issues are the
  hold channel.

## Operating it

- **Held release** → issue titled `Immich vX.Y.Z hold: needs review`.
  Read it, do any listed manual edits on the server's compose file, then
  Run workflow with `approve=vX.Y.Z`. Publishing closes open review
  issues.
- **Changing `templates/immich/x-casaos.yml` or the script** changes the
  template hash, so the next run republishes the same Immich version as
  `immich-vX.Y.Z-r2`, `-r3`, …
- **Manual CasaOS store refresh** isn't needed; the 10-minute poll picks up
  a new release. Restarting `casaos-app-management` forces it.

## Open items

- [ ] Optionally set `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID` repo
      secrets for publish/hold notices (Manoj to set; shack bot creds).
- [ ] First real upstream release after 3.2.2: confirm the tile shows
      Update and that clicking it lands cleanly (4/4 healthy, version
      bump). Recorded in macmini-server HANDOVER too.
