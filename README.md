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

## How to add this store to umbrelOS

1. Open umbrelOS → App Store.
2. Click the menu in the top-right → **Community App Stores** → **Add Store**.
3. Paste the URL of this repo:
   ```
   https://github.com/brandonjamesmarshall/umbrel-community-app-store
   ```
4. Apps appear in the new "Brandon's" store.

## Media stack: one-time setup

The media stack (Plex, Sonarr, Radarr, Prowlarr, SABnzbd, Transmission)
all share a single NFS mount from a Synology NAS at `/data` inside each
container. That single mount point is what makes
[TRaSH-style hardlinks and atomic moves](https://trash-guides.info/Hardlinks/Hardlinks-and-Instant-Moves/)
work — Sonarr/Radarr import via `link(2)` instead of copying gigabytes.

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
- Inside the share, make sure these subfolders exist:
  - `Download/incomplete/`
  - `Download/tv/`
  - `Download/movies/`

### 2. Mount the NFS share on the Umbrel host

SSH into your Umbrel and add a permanent NFS mount at `/mnt/plex-media`
(the path every media app's `docker-compose.yml` binds to `/data`):

```bash
sudo mkdir -p /mnt/plex-media
echo '192.168.X.Y:/volume3/Plex\040Media   /mnt/plex-media   nfs4   defaults,_netdev,rw,hard,noatime   0 0' | sudo tee -a /etc/fstab
sudo mount -a
ls /mnt/plex-media    # should show Books/ Download/ Movies/ Music/ Photos/ TV/
```

Notes:

- Replace `192.168.X.Y` with your NAS's LAN IP.
- The `\040` is the octal escape for the space in `Plex Media`. `fstab`
  requires this — a literal space breaks parsing.
- `_netdev` tells systemd to wait for the network before mounting (avoids
  boot-time race).
- If you ever rename the Synology share to a no-space name, drop the
  `\040` escape.

### 3. Install the apps from umbrelOS

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

Each app's `umbrel-app.yml` description has the specific paths and host
names to paste during setup.

## Transmission + iVPN secrets

Transmission runs inside [gluetun](https://github.com/qdm12/gluetun)'s
network namespace, so it can only reach the internet through your iVPN
WireGuard tunnel — if the tunnel drops, Transmission is offline (kill-switch
by construction).

Credentials are **never** in this repo. After installing the Transmission
app, on the Umbrel:

```bash
sudo mkdir -p ~/umbrel/app-data/brandonjamesmarshall-transmission/data
sudo nano ~/umbrel/app-data/brandonjamesmarshall-transmission/data/gluetun.env
```

Paste the values from
[gluetun.env.example](./brandonjamesmarshall-transmission/gluetun.env.example).
Get `WIREGUARD_PRIVATE_KEY` and `WIREGUARD_ADDRESSES` from your iVPN
account area (WireGuard tab → add key → download config → grab the values
from the `.conf` file).

Verify the kill-switch:

```bash
docker exec brandonjamesmarshall-transmission_gluetun_1 wget -qO- https://ifconfig.me
# must return an iVPN IP, NOT your home IP
```

## Newt secrets (existing)

Same pattern as Transmission. See
[brandonjamesmarshall-newt/newt.env.example](./brandonjamesmarshall-newt/newt.env.example).

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
