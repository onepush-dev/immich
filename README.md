# Immich — self-hosted Google Photos alternative (v3.3.3, server + machine learning + VectorChord Postgres + Valkey). Deploy on Railway with one click.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/TEMPLATE_CODE)

[Immich](https://immich.app) ([immich-app/immich](https://github.com/immich-app/immich), AGPL-3.0) is a high-performance, self-hosted photo and video library: automatic backup from the iOS and Android apps, a timeline, albums and shared links, a map, memories, partner sharing, multiple users, and on-device-free AI features (CLIP smart search, face recognition, OCR, duplicate detection) that run on your own server. This template runs the **official** v3.3.3 images, pinned by tag **and** digest, on the upstream `docker-compose.yml` layout: **immich-server**, **immich-machine-learning** (CPU), Immich's **VectorChord Postgres** and **Valkey**. It needs **no deploy-time input**. Open the URL and create the admin account.

## Why this template
- **Pinned, current release.** Immich `v3.3.3` (released 2026-10-14) for both server and ML, pinned by tag and digest. The Postgres and Valkey images use the exact digests from the release's `docker-compose.yml`. Nothing floats on `release` or `latest`, so a redeploy never runs migrations you didn't plan for.
- **The right database.** Immich needs Postgres with the **VectorChord** (`vchord`) extension for smart search and face embeddings. Plain Railway Postgres won't work. This template uses `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`, with `--data-checksums` as upstream sets it.
- **AI search works out of the box.** The CPU machine-learning service is wired in over the private network (`IMMICH_MACHINE_LEARNING_URL`). It downloads models on first use into a persistent `/cache` volume, so they survive redeploys.
- **Everything persists.** Separate volumes for the photo library (`/data`), the database, the model cache and Valkey.
- **Generated secrets, private by default.** The Postgres and Valkey passwords are generated per deploy (alphanumeric, as Immich requires). Only `immich-server` gets a public domain. ML, Postgres and Valkey are reachable only on `*.railway.internal`.

## Services
| Service | Image | Port | Volume | Notes |
|---|---|---|---|---|
| immich-server | `ghcr.io/immich-app/immich-server:v3.3.3@sha256:bae7664ff3aa4c2c1fcbdcdd86b40b105643c5f861f3e6a770fc760089f25862` | `2283` (public) | `/data` | API, web UI and background workers. Health `/api/server/ping` (600 s). No start command |
| immich-machine-learning | `ghcr.io/immich-app/immich-machine-learning:v3.3.3@sha256:73915f07e1f4289ba45b174c58443db74cef47ddf502613f475fc5e52458ea42` | `3003` (private) | `/cache` | CPU inference (CLIP, buffalo_l faces, PP-OCR). Health `/ping`. No start command |
| Postgres | `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf63357191b76a916ae5eb93464d65c07511da41e3bf7a8416db519b40b1c23` | `5432` (private) | `/var/lib/postgresql/data` (`PGDATA=/var/lib/postgresql/data/pgdata`) | Upstream image and digest. No start command |
| Valkey | `docker.io/valkey/valkey:9.1.2@sha256:c123e3715db63d06d4ad6964884037aa0d5d4d703939b9929954112889708e1d` | `6379` (private) | `/data` | BullMQ job queues. Start command below |

**Valkey start command** (also in `railway/valkey-start-command.txt`). Railway replaces an image's ENTRYPOINT when a start command is set, so it calls the upstream entrypoint itself:
```
/bin/sh -c "rm -rf /data/lost+found && exec docker-entrypoint.sh valkey-server --requirepass $REDIS_PASSWORD --save 60 1 --dir /data --maxmemory-policy noeviction --bind '* -::*'"
```

## Variables
Every variable is in [`env.example`](env.example) with its template value.

### immich-server
| Variable | Template value | Notes |
|---|---|---|
| PORT | `2283` | port Railway routes the domain/healthcheck to |
| IMMICH_PORT | `2283` | port Immich listens on |
| DB_HOSTNAME | `${{Postgres.RAILWAY_PRIVATE_DOMAIN}}` | |
| DB_PORT | `5432` | |
| DB_USERNAME | `${{Postgres.POSTGRES_USER}}` | |
| DB_PASSWORD | `${{Postgres.POSTGRES_PASSWORD}}` | |
| DB_DATABASE_NAME | `${{Postgres.POSTGRES_DB}}` | |
| REDIS_HOSTNAME | `${{Valkey.RAILWAY_PRIVATE_DOMAIN}}` | |
| REDIS_PORT | `6379` | |
| REDIS_PASSWORD | `${{Valkey.REDIS_PASSWORD}}` | |
| IMMICH_MACHINE_LEARNING_URL | `http://${{immich-machine-learning.RAILWAY_PRIVATE_DOMAIN}}:3003` | |
| TZ | `Etc/UTC` | change to your timezone if you like |

### immich-machine-learning
| Variable | Template value |
|---|---|
| PORT | `3003` |
| IMMICH_HOST | `[::]` |
| IMMICH_PORT | `3003` |
| MACHINE_LEARNING_CACHE_FOLDER | `/cache` |
| MACHINE_LEARNING_WORKERS | `1` |
| MACHINE_LEARNING_MODEL_TTL | `300` |

### Postgres
| Variable | Template value |
|---|---|
| POSTGRES_USER | `postgres` |
| POSTGRES_PASSWORD | `${{secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` |
| POSTGRES_DB | `immich` |
| POSTGRES_INITDB_ARGS | `--data-checksums` |
| PGDATA | `/var/lib/postgresql/data/pgdata` |
| DB_STORAGE_TYPE | `SSD` |
| PGHOST / PGPORT / PGUSER / PGPASSWORD / PGDATABASE | `${{RAILWAY_PRIVATE_DOMAIN}}` / `5432` / `${{POSTGRES_USER}}` / `${{POSTGRES_PASSWORD}}` / `${{POSTGRES_DB}}` |
| DATABASE_URL | `postgresql://${{POSTGRES_USER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{POSTGRES_DB}}` |

### Valkey
| Variable | Template value |
|---|---|
| REDIS_PASSWORD | `${{secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` |
| REDISHOST / REDISPORT / REDISUSER / REDISPASSWORD | `${{RAILWAY_PRIVATE_DOMAIN}}` / `6379` / `default` / `${{REDIS_PASSWORD}}` |
| REDIS_URL | `redis://default:${{REDIS_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:6379` |

## How to use
1. Click **Deploy on Railway** and wait for all four services to go green (first boot runs DB migrations and a geodata import, about a minute).
2. Open the `immich-server` URL. The **first account you create becomes the admin**, so do it right away.
3. Under **Administration → Settings → Server**, set the **External domain** to your Railway URL (or custom domain) so shared links use it.
4. Install the Immich mobile app, enter the server URL, log in and turn on backup.

## Notes
- **Resources.** The ML service uses about 1.8 GB RAM once the CLIP, face and OCR models are loaded (it unloads them after 5 minutes idle). The server peaks around 1.5 GB during import and thumbnailing. Plan for roughly 4 GB RAM across the project. On small plans you can turn off Smart Search, Facial Recognition and OCR in Administration → Settings → Machine Learning.
- **Storage.** Photos and videos live on the `immich-server` `/data` volume. Grow it in the Railway dashboard as your library grows. The model cache needs about 1–2 GB.
- **Backups.** Immich writes nightly database dumps to `/data/backups` on the library volume. Copy them off-site, or use Railway volume backups.
- **Upgrading.** Read the [Immich release notes](https://github.com/immich-app/immich/releases) first, then change the server **and** ML image tag+digest together. Keep the Postgres image unless the release notes say otherwise.
- **Hardware acceleration** (CUDA/OpenVINO) isn't available on Railway, so this template uses the CPU ML image and CPU video transcoding.

Configs in this repo (`railway.toml`, `railway/*.toml`) document each service's Railway settings. Railway deploys the official images directly, and the marketplace template is generated from a live project. Immich is © its authors and licensed under AGPL-3.0.
