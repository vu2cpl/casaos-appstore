# casaos-appstore

A one-app [CasaOS](https://casaos.zimaspace.com/) app store that keeps
**Immich** current, so CasaOS's own **Update** button works for it.

The official CasaOS store's Immich has been stuck on v2.7.2 since before
Immich 3.0, and third-party stores lag weeks behind. This store republishes
the official Immich `docker-compose.yml` soon after each Immich release,
with safety checks, as the store app `vu2cpl-immich`.

## Add the store to CasaOS

App Store → **More apps** → add source:

```
https://github.com/vu2cpl/casaos-appstore/releases/latest/download/casaos-appstore.zip
```

or from the CasaOS host itself:

```bash
curl -X POST "http://127.0.0.1:$(sudo cat /var/run/casaos/app-management.url | sed 's#.*:##')/v2/app_management/appstore?url=https://github.com/vu2cpl/casaos-appstore/releases/latest/download/casaos-appstore.zip"
```

Use the **release asset URL**, not the repo's "Download ZIP" link. CasaOS only
re-downloads a store when a HEAD request reports a different
`Content-Length`, and GitHub's branch archives report none, so a store served
that way never refreshes after the first download. Release assets report a
real size, and the build makes sure every release's zip differs in size from
the last.

## How updates reach an installed Immich

CasaOS shows **Update** when the image tag of an installed app's main service
(`immich-server`) differs from the store copy's. Clicking it replaces only the
`image:` lines in the installed compose file, so the install's volumes,
passwords and paths are untouched.

An existing Immich install can be linked to this store by adding
`store_app_id: vu2cpl-immich` to its top-level `x-casaos` block, as long as its
service names are the official ones (`immich-server`,
`immich-machine-learning`, `redis`, `database`).

## Release checks

[`update-immich.yml`](.github/workflows/update-immich.yml) runs every 6 hours
and on demand. [`scripts/update_immich.py`](scripts/update_immich.py) compares
the latest Immich release with the last published one:

| Result | When | What happens |
|---|---|---|
| publish | only images or healthchecks changed | new store release; CasaOS shows Update within ~10 min |
| hold | new major version; a "Breaking Changes" section in any release since the last published; any other compose or `example.env` change | GitHub issue with the reasons and diff, nothing published |
| blocked | service names changed | issue; CasaOS can't update across this, so manual migration |

A held release is published after review from **Actions → Update Immich store
app → Run workflow** with `approve` set to the Immich tag (e.g. `v3.3.0`).
Do any manual compose edits the issue lists first, because a CasaOS update only
swaps images.

Optional Telegram notices: set the repo secrets `TELEGRAM_BOT_TOKEN` and
`TELEGRAM_CHAT_ID`.

## Layout

| Path | What |
|---|---|
| `Apps/vu2cpl-immich/docker-compose.yml` | the published store app (generated) |
| `templates/immich/x-casaos.yml` | CasaOS metadata merged into it |
| `upstream/immich/` | Immich's compose + example.env as of the last publish (the comparison baseline) |
| `state/immich.json` | last published Immich tag, store release tag, zip size |
| `scripts/update_immich.py` | the check/build script |

Test locally without publishing anything (it writes into the working tree, so
use a scratch copy):

```bash
cp -R . /tmp/store-test && cd /tmp/store-test
GH_TOKEN=$(gh auth token) uv run --with pyyaml python3 scripts/update_immich.py
```

## Credits

Immich is by the Immich team, and the app definition here is their official
compose file, republished unchanged apart from CasaOS metadata. Upstream is at
[immich-app/immich](https://github.com/immich-app/immich). Not affiliated with
Immich or CasaOS.
