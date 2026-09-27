# mealie-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [Mealie](https://github.com/mealie-recipes/mealie).

## What this is

A four-file TOS Docker application archive (`mealiedocker.tar.gz`) that installs
Mealie on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `mealiedocker`)
- `mealiedocker.lang` — 23-language superset (TOS requires 14)
- `mealiedocker.svg` — application icon (460 bytes, 2 SVG elements)
- `docker-compose.yml` — one service, image `hkotel/mealie:v3.28.0`

## How it works

| | |
|---|---|
| Container image | `hkotel/mealie:v3.28.0` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8010/` (container port 9000) |
| Runs as | non-root, `user: "1000:1000"` |
| Persistent data | `/app/data` → `/Volume1/DockerAppData/mealiedocker/data` |
| Logs | `docker logs mealiedocker` (stdout/stderr) |

Everything Mealie stores lives in one directory: `mealie.db` (the SQLite database with recipes, users, meal plans and shopping lists), `recipes/` (uploaded and scraped images), `backups/` (automatic backups Mealie creates itself), `mealie.log` and the two key files `.secret` and `.session_secret` that protect saved integrations and login sessions.

## Design highlights

- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container reports uid 1000).
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no `network_mode: host`, and no Docker socket, no `cap_add`.
- **Health check** probing the service on container port 9000.
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip without a timestamp, plus the review-standards self-check above.
- **Runs as the non-root application user.** The upstream start script only switches to a dedicated user when it is started as root; with `user: "1000:1000"` Mealie serves directly, verified on real hardware.
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no `network_mode: host`, no Docker socket.
- **Health check** probing `/api/app/about` on container port 9000, the same endpoint the upstream image uses.
- **One data directory.** The whole application state is `/app/data`, mounted from the application data folder, so copying that folder is a complete backup.

## Runtime file manifest

Everything the application writes on the NAS lives inside its own data
directory; the container-internal `/tmp` is discarded with the container. The
table lists what is created and how it behaves. Nothing else is written.

| Path | Purpose | Created when | Growth bound | Lifecycle |
|---|---|---|---|---|
| `data/mealie.db` | recipes, ingredients, users, meal plans and shopping lists (SQLite database) | first start | grows with every recipe and plan | never overwritten; back it up before uninstalling with data removal |
| `data/.secret` | key that encrypts saved integration secrets (mail, OIDC, webhooks) | first start | static | never overwritten; without it saved secrets can no longer be read |
| `data/.session_secret` | key used to sign login sessions | first start | static | never overwritten; changing it logs everyone out |
| `data/recipes` | images uploaded with recipes | first start | grows with every recipe that has a photo | managed by Mealie |
| `data/backups` | automatic backups created by Mealie | on the configured backup schedule | bounded by Mealie's retention setting | managed by Mealie; safe to copy elsewhere |
| `data/mealie.log` | application log | first start | appended; rotate or delete if it gets large | safe to delete |
| `/Volume1/DockerAppData/mealiedocker/data` | Mealie application data (everything the container writes outside /tmp) | on first start / on demand | bounded by your own data | persistent; kept on uninstall unless you ask to delete it |

## Build

```bash
scripts/build.sh                  # -> out/mealiedocker.tar.gz + .sha256
scripts/build.sh 3.28.0-2   # packaging iteration bump
scripts/build.sh 3.28.0-1 aarch64
```

The build stages the four files, substitutes the version/platform placeholders,
removes CRLF/BOM/AppleDouble, then runs
`tools/verify.py`, a review-standards self-check (archive layout, JSON validity,
required fields, 14+ language coverage, version consistency, Docker-Hub-only
images, fixed tag, reserved/recommended host ports, non-root user, per-service
healthcheck/TZ/restart, data path below the application data root, never the
`/Volume*` wildcard, comment-free sequence blocks, `x-app-meta` position, no
literal secrets and the SVG size/element limits).

## Local verification

```bash
./TOSAppSelfTestingTool -c out/mealiedocker.tar.gz   # official spec check
bash test/install-and-verify.sh                  # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

## Store submission checklist

1. Public GitHub repo (code + README only).
2. Release **tag `v3.28.0-1`**, assets `mealiedocker.tar.gz` +
   `mealiedocker.tar.gz.sha256` (Docker asset naming: `<app_id>.tar.gz`, **no
   platform suffix**).
3. Developer platform → Add Application: ID `mealiedocker`, package type Docker,
   repository URL, architecture `x86_64`.
4. Version Management → Add Version `3.28.0-1` → automated validation → review.

## License / branding

Mealie is distributed by its upstream authors under **AGPL-3.0**. The container image is the official image from Docker Hub (`hkotel/mealie:v3.28.0`) and is used unmodified; no upstream source code is redistributed here. This packaging is an independent community submission by Moechz and is not affiliated with the upstream project or TerraMaster.

## Limitations

The first login uses the built-in administrator `changeme@example.com` with the password `MyPassword`. Change the e-mail address and the password immediately, otherwise anyone on your network can sign in.
Recipe import depends on the source website. Pages that block automated access or hide their content behind a login cannot be imported automatically; Mealie lets you enter those recipes by hand.
The app is served over plain HTTP on the LAN. Do not expose the published port to the internet without TLS.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
