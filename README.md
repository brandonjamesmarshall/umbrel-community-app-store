# Brandon's Umbrel Community App Store

A personal [Umbrel](https://umbrel.com) community app store with self-hosted
apps that aren't (yet) in the official store, plus a full TRaSH-style media
stack with hardware-transcoded Plex, NFS-backed library on a Synology NAS,
and a VPN-isolated torrent client.

## Apps

| App | What it is |
| --- | --- |
| [Resilio Sync](./brandonjamesmarshall-resilio-sync) | Peer-to-peer file sync (BitTorrent Sync). |
| [Newt](./brandonjamesmarshall-newt) | Userspace WireGuard tunnel client for [Pangolin](https://docs.pangolin.net). |
| [Plex](./brandonjamesmarshall-plex) | Media server with N100 hardware transcoding. |
| [Sonarr](./brandonjamesmarshall-sonarr) | TV collection manager. |
| [Radarr](./brandonjamesmarshall-radarr) | Movie collection manager. |
| [Prowlarr](./brandonjamesmarshall-prowlarr) | Indexer manager for the *arr stack. |
| [SABnzbd](./brandonjamesmarshall-sabnzbd) | Usenet download client. |
| [Transmission (via iVPN)](./brandonjamesmarshall-transmission) | Torrent client tunneled through iVPN with kill-switch. |
| [Tautulli](./brandonjamesmarshall-tautulli) | Plex stats and history. |
| [Maintainerr](./brandonjamesmarshall-maintainerr) | Rule-based library cleanup. |
| [LazyLibrarian](./brandonjamesmarshall-lazylibrarian) | Book & audiobook collection manager (the "arr" for books). |
| [Bookshelf](./brandonjamesmarshall-bookshelf) | Book & audiobook manager — a Readarr revival; nicer-UI alternative to LazyLibrarian. |
| [Calibre-Web Automated](./brandonjamesmarshall-calibre-web-automated) | eBook library, web reader, and Send-to-Kindle. |
| [GitHub Actions Runner](./brandonjamesmarshall-github-runner) | Self-hosted CI runner with Docker support (for Vibewatch CI). |

## How to add this store to umbrelOS

1. Open umbrelOS → App Store.
2. Click the menu in the top-right → **Community App Stores** → **Add Store**.
3. Paste the URL of this repo:
   ```
   https://github.com/brandonjamesmarshall/umbrel-community-app-store
   ```
4. Apps appear in the new "Brandon's" store.

## Media stack: one-time setup

The media apps (Plex, Sonarr, Radarr, SABnzbd, Transmission, LazyLibrarian,
Bookshelf, Calibre-Web Automated) each declare a **Docker-managed NFS volume** in their
`docker-compose.yml` pointing at the Synology share
(`192.168.50.111:/volume3/Plex Media`). Most mount the whole share at `/media`;
Calibre-Web Automated instead mounts two sub-folders of it
(`Books/CalibreLibrary` and `Books/Ingest`) at the paths it expects.

Why this approach over a host-level NFS mount:

- Docker uses the **in-kernel NFS module** (part of the umbrelOS firmware
  image), so the mount survives every reboot, freeze recovery, and A/B
  firmware update — no `apt install nfs-common` to redo, no `/etc/fstab`
  to maintain, no `~/scripts/` you have to remember.
- One mount point per app means [TRaSH-style hardlinks and atomic
  moves](https://trash-guides.info/Hardlinks/Hardlinks-and-Instant-Moves/)
  work natively — Sonarr/Radarr import via `link(2)` instead of copying
  gigabytes.

The NAS IP (`192.168.50.111`) is hard-coded in the compose files. It's
an RFC1918 LAN address with zero exposure outside the network — your NAS
isn't internet-facing, and a private IP is useless to anyone not already
on your LAN.

### 1. Enable NFS on the Synology

- Control Panel → File Services → NFS → **Enable NFS service**, choose **NFSv4.1**.
- File Station → right-click your media share (e.g. `Plex Media`) →
  **Properties → NFS Permissions → Create**:
  - Hostname/IP: your Umbrel's LAN IP (or subnet, e.g. `192.168.1.0/24`).
  - Privilege: Read/Write.
  - Squash: **Map all users to admin** (simplest; containers running as
    PUID=1000 will write as the NAS admin user).
  - Security: `sys`. Async: on. Allow non-privileged ports: on. Allow
    access to mounted subfolders: on.

### 2. Install the apps from umbrelOS

Order is mostly free, but a sensible flow:

1. **Plex** → claim at plex.tv, enable HW transcoding (Settings → Transcoder
   → "Use hardware acceleration when available"). Requires Plex Pass.
2. **Transmission** → set up gluetun first (see below) before launching.
3. **SABnzbd** → wire up your Usenet provider, configure folders/categories.
4. **Prowlarr** → add your indexer accounts.
5. **Sonarr / Radarr** → add root folders (`/data/TV`, `/data/Movies`),
   wire to Transmission + SABnzbd in Download Clients, then go to Prowlarr
   → Settings → Apps and connect them so Prowlarr can sync indexers.
6. **Tautulli** → point at Plex.
7. **Maintainerr** → point at Plex + Sonarr + Radarr.
8. **Books (LazyLibrarian + Calibre-Web Automated)** → see below.

Each app's `umbrel-app.yml` description has the specific paths and host
names to paste during setup.

### Books stack (LazyLibrarian + Calibre-Web Automated)

The book flow mirrors the *arr stack: **LazyLibrarian** searches your Prowlarr
indexers and hands downloads to SABnzbd/Transmission, then drops finished
books into a shared **ingest** folder. **Calibre-Web Automated (CWA)**
auto-imports from ingest into a Calibre library, serves a web reader/OPDS,
and sends books to your **Kindle** by e-mail.

**Before installing CWA**, create two folders on the NAS share (alongside
`TV`, `Movies`, `Download`):

```
Books/CalibreLibrary    ← CWA's Calibre library (metadata.db lives here)
Books/Ingest            ← LazyLibrarian writes here; CWA imports & empties it
```

CWA mounts those two sub-folders directly (`/calibre-library`,
`/cwa-book-ingest`) and runs with `NETWORK_SHARE_MODE=true`, which switches
its file watcher from inotify to ~5s polling — required because inotify
events don't cross NFS clients, so CWA would otherwise never see the books
LazyLibrarian writes. CWA seeds an empty library on first run if
`CalibreLibrary` is empty.

Then: install **LazyLibrarian** (wire to Prowlarr + your download clients,
set eBook destination `/media/Books/Ingest`), install **CWA**, log in
(`admin` / `admin123` — change it immediately), and configure the SMTP/Send-
to-Kindle e-mail server under Admin → Edit Basic Configuration.

> **Heads-up on updates:** LinuxServer tags LazyLibrarian by git commit hash
> (no semver), so it's pinned to `latest@sha256` and Renovate refreshes only
> the digest — umbrelOS won't show a version bump for it (the same "invisible
> digest update" behaviour noted below). CWA uses proper semver, so it
> surfaces updates normally. If you want visible LazyLibrarian updates, the
> Transmission `+vpnN` auto-bump workflow can be generalised to it.

**Alternative front-end — Bookshelf:** if LazyLibrarian feels clunky,
[**Bookshelf**](./brandonjamesmarshall-bookshelf) (a community revival of
Readarr, with the Sonarr/Radarr UI) is packaged here as a drop-in replacement
for the *grabber* half of this flow. Point its root folder at the same
`/media/Books/Ingest` and wire it to Prowlarr + your download clients exactly
like Sonarr — CWA picks up its output identically. Turn on **Unmonitor
Deleted Books** in its Media Management settings so the post-import deletion
CWA performs doesn't make Bookshelf re-grab the same titles. It runs the
rolling `hardcover` tag (Hardcover.app metadata) pinned by digest, so it has
the same "invisible digest update" behaviour as LazyLibrarian above. The two can run
side by side; just manage a given book in one or the other to avoid double
grabs.

## Per-app `.env` files (Newt + Transmission only)

Two apps need an `.env` at `~/umbrel/app-data/<app-id>/.env`, loaded via
the `env_file:` directive in their compose. **Pre-create the .env before
clicking Install** — otherwise the install fails (Newt because env_file
short-form requires the file to exist; Transmission because gluetun
can't start without a private key, which blocks the dependent server
container).

- **Newt**: `PANGOLIN_ENDPOINT`, `NEWT_ID`, `NEWT_SECRET`. See
  [brandonjamesmarshall-newt/.env.example](./brandonjamesmarshall-newt/.env.example).
- **Transmission**: iVPN WireGuard credentials. See
  [brandonjamesmarshall-transmission/.env.example](./brandonjamesmarshall-transmission/.env.example).

The other 5 apps (Plex/Sonarr/Radarr/SABnzbd/Resilio Sync) need no
`.env` — they get the media library from the Docker-managed NFS volume
declared in their compose (see "Media stack: one-time setup" above).

Verify the Transmission kill-switch once configured:

```bash
docker exec brandonjamesmarshall-transmission_gluetun_1 wget -qO- https://ifconfig.me
# must return an iVPN IP, NOT your home IP
```

## Image pinning policy

Every app's `docker-compose.yml` references its image by **both a tag and a
`@sha256:` digest**. Docker will refuse to pull anything else.

## Automated updates via Renovate (14-day delay)

[`renovate.json`](./renovate.json) configures Mend Renovate to:

- Scan every `docker-compose.yml` for new image versions.
- Scan every `umbrel-app.yml`'s `# renovate: depName=...` comment to keep
  the user-visible `version:` field in lockstep with the Docker tag.
- **Wait 14 days** after a publisher tags a new version before opening a PR.
  Mitigates supply-chain attacks: a maliciously-pushed image landed inside
  14 days never reaches this repo.
- Auto-merge the PR once the wait elapses (GitHub-side: zero touch).
- umbrelOS then surfaces "Update available" on your installed apps next
  time it polls. You click Update at your leisure.

One-time setup: install the
[Mend Renovate](https://github.com/apps/renovate) GitHub App on this repo.
Free for public repositories. After that, look for the auto-created
"Dependency Dashboard" issue to see all known updates and where each is in
the wait window.

Tunables in [`renovate.json`](./renovate.json):

- `minimumReleaseAge` — bump higher (e.g. `"30 days"`) for more caution.
- Per-image `automerge: false` in `packageRules` if you want hands-on
  review of certain apps (security-critical ones, for example).

---

Forked from [getumbrel/umbrel-community-app-store](https://github.com/getumbrel/umbrel-community-app-store).
