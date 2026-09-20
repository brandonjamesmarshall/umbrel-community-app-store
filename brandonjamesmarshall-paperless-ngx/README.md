# Paperless-ngx

Document archive with OCR, auto-tagging and full-text search, packaged from
the official `ghcr.io/paperless-ngx/paperless-ngx` image with the Postgres
stack from upstream's `docker-compose.postgres.yml` (webserver + Postgres +
Valkey), digest-pinned and Renovate-managed.

Gotenberg and Tika are not included. They add Office-document and email-body
parsing and cost two more always-on containers; add them later if you start
feeding it `.docx` files.

## Where the data lives

| What | Where | Why |
| --- | --- | --- |
| Search index, classifier, logs | `app-data/.../data/data` (local SSD) | tantivy does a lot of small random IO |
| Postgres | `app-data/.../data/postgres` (local SSD) | a database over NFS is a corruption story |
| Valkey AOF | `app-data/.../data/valkey` (local SSD) | task queue, rebuilt on demand |
| Originals, archive PDFs, thumbnails | NAS media share, `Documents/` (NFS) | the archive belongs with the rest of your files |
| Consume dropbox | NAS consume share (NFS) | the scanner already writes there |
| Export staging | `app-data/.../data/export` (local SSD) | migration in/out |

Paths are at the bottom of [docker-compose.yml](docker-compose.yml).

## Before installing

On the NAS:

1. Create the documents folder inside the media share (`<media share>/Documents`)
   and **put a file in it** — `touch .keep`. A Docker NFS volume pointing at an
   empty directory fails with `failed to chmod ... operation not permitted`,
   which umbrelOS shows as an install stuck at 1%.
2. Give both that share and the consume share an NFS permission rule for the
   Umbrel's IP (NFSv4.1, read/write, squash mapping to admin is fine —
   the container runs as UID 1000 and the server maps it).

## Ports

| Port | Purpose |
| --- | --- |
| 8000 | umbrelOS app proxy (requires umbrelOS login; `/api/*` is whitelisted for token clients) |
| 18000 | Direct — Paperless's own login is the only gate. Use this for the mobile app, the scanner and any Pangolin resource |

## First login

There is no default account. Either migrate (below), which brings your users
and passwords across, or create one:

```bash
docker exec -it brandonjamesmarshall-paperless-ngx_webserver_1 python3 manage.py createsuperuser
```

## Optional config

Anything user-specific goes in `~/umbrel/app-data/brandonjamesmarshall-paperless-ngx/.env`
(create it; it is optional and not overwritten):

```sh
PAPERLESS_URL=https://docs.example.com      # your public URL — required for CSRF once exposed
PAPERLESS_TIME_ZONE=America/New_York
PAPERLESS_OCR_LANGUAGE=eng
```

Stop and Start the app afterwards. Infrastructure settings (database, Redis,
paths, consume polling) are set in `environment:` in the compose file and
deliberately can't be overridden from `.env`.

## Migrating from an existing Paperless

Use Paperless's own exporter — not a file copy. The export is a directory of
original files plus a `manifest.json`, and the importer replays it into a
fresh install, so it survives different Paperless versions (target must be
the same or newer), a different database engine, and a different filename
format. A raw `media/` + database copy requires all of those to match.

**1. Export on the old machine.** Into a folder on a share you can reach
later, e.g. the consume share:

```bash
docker exec -it <old-paperless-container> document_exporter ../export --no-progress-bar
```

Add `--delete` if you re-run it and want stale files cleaned up. The export
is roughly the size of your archive; check free space first.

**2. Copy it to the Umbrel.** From the Umbrel, with the export folder
reachable over NFS/SMB — or straight over SSH from the old box:

```bash
rsync -a --info=progress2 <old-host>:/path/to/export/ ~/umbrel/app-data/brandonjamesmarshall-paperless-ngx/data/export/
```

**3. Import.** With the app running and its database empty (a fresh install —
the importer refuses to run over existing documents):

```bash
docker exec -it brandonjamesmarshall-paperless-ngx_webserver_1 document_importer ../export --no-progress-bar
```

**4. Rebuild the search index** (the export carries documents, not the
index):

```bash
docker exec -it brandonjamesmarshall-paperless-ngx_webserver_1 document_index reindex
```

**5. Check.** Log in with your old credentials, confirm the document count
matches, spot-check a few PDFs and a full-text search. Then delete the
export folder — it is a full second copy of the archive.

Point the scanner at port 18000 (or keep dropping files in the consume
share) and decommission the old instance.

## Updates

Renovate opens a PR when a new Paperless release is 14 days old, bumping both
the image digest and `version:` in `umbrel-app.yml`, and merges it
automatically; umbrelOS then offers the update. Postgres major bumps are
gated to a human PR — a major refuses to start on the previous major's data
directory, so that PR also has to carry a dump/restore migration (the `buzz`
app in this store has one to copy).
