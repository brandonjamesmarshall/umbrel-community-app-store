# Paperless-ngx

Document archive with OCR, auto-tagging and full-text search, packaged from
the official `ghcr.io/paperless-ngx/paperless-ngx` image with the Postgres
stack from upstream's `docker-compose.postgres.yml` (webserver + Postgres +
Valkey), digest-pinned and Renovate-managed.

Gotenberg and Tika are not included. They add Office-document and email-body
parsing and cost two more always-on containers; add them later if you start
feeding it `.docx` files.

## Where the data lives

Everything a person might want to open without Paperless lives in one NAS
share; everything that is derived state Paperless can rebuild lives on the
Umbrel's SSD.

| What | Where | Why |
| --- | --- | --- |
| Scanner dropbox | `<share>/Inbox` (NFS) | the scanner writes here; files are moved out as they're consumed |
| Originals, archive PDFs, thumbnails | `<share>/Documents` (NFS) | the archive itself — browsable and backed up by the NAS |
| `document_exporter` output | `<share>/Export` (NFS) | a portable copy of the whole archive, on storage that already has backups |
| Search index, classifier, logs | `app-data/.../data/data` (local SSD) | tantivy does a lot of small random IO |
| Postgres | `app-data/.../data/postgres` (local SSD) | a database over NFS is a corruption story |
| Valkey AOF | `app-data/.../data/valkey` (local SSD) | task queue, rebuilt on demand |

`Documents/` uses Paperless's own subfolder layout and the database tracks
those paths — read it, copy from it, but don't reorganize it by hand.

Paths are at the bottom of [docker-compose.yml](docker-compose.yml).

## Before installing

On the NAS:

1. Create `Inbox`, `Documents` and `Export` in the share, each with a file in
   it — `touch .keep`. A Docker NFS volume pointing at an empty directory
   fails with `failed to chmod ... operation not permitted`, which umbrelOS
   shows as an install stuck at 1%.
2. Give the share an NFS permission rule for the Umbrel's IP (NFSv4.1,
   read/write; squash mapping to admin is fine — the container runs as UID
   1000 and the server maps it).

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

**1. Export on the old machine.**

```bash
docker exec -it <old-paperless-container> document_exporter ../export --delete --no-progress-bar
```

The export is roughly the size of your archive; check free space first.

**2. Move it into the share's `Export` folder.** If the old install is on the
same NAS, this is a local move — no copy over the network:

```bash
mv /path/to/old/export/* /volume<n>/<share>/Export/
```

Otherwise rsync it there. Either way it lands at `/usr/src/paperless/export`
inside the new container, because that folder is mounted.

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
matches, spot-check a few PDFs and a full-text search. Keep or clear the
export as you like — it is a full second copy of the archive, and re-running
`document_exporter` on a schedule is a reasonable backup in its own right.

Point the scanner at port 18000 (or keep dropping files in the consume
share) and decommission the old instance.

## Updates

Renovate opens a PR when a new Paperless release is 14 days old, bumping both
the image digest and `version:` in `umbrel-app.yml`, and merges it
automatically; umbrelOS then offers the update. Postgres major bumps are
gated to a human PR — a major refuses to start on the previous major's data
directory, so that PR also has to carry a dump/restore migration (the `buzz`
app in this store has one to copy).
