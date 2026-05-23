# Brandon's Umbrel Community App Store

A personal [Umbrel](https://umbrel.com) community app store with a couple of
self-hosted apps that aren't (yet) in the official store.

## Apps

| App | What it is |
| --- | --- |
| [Resilio Sync](./brandonjamesmarshall-resilio-sync) | Peer-to-peer file sync (BitTorrent Sync) using the `linuxserver/resilio-sync` image. |
| [Newt](./brandonjamesmarshall-newt) | Userspace WireGuard tunnel client for [Pangolin](https://docs.pangolin.net). |

## How to add this store to umbrelOS

1. Open umbrelOS → App Store.
2. Click the menu in the top-right → **Community App Stores** → **Add Store**.
3. Paste the URL of this repo:
   ```
   https://github.com/brandonjamesmarshall/umbrel-community-app-store
   ```
4. Install Resilio Sync or Newt from the new "Brandon's" store.

## Newt secrets

Newt needs your Pangolin endpoint, NEWT_ID, and NEWT_SECRET. These are
**never** stored in this repo. After installing the Newt app, SSH into your
Umbrel and create:

```
~/umbrel/app-data/brandonjamesmarshall-newt/data/newt.env
```

Use [brandonjamesmarshall-newt/newt.env.example](./brandonjamesmarshall-newt/newt.env.example)
as a template, then restart the app from umbrelOS.

A `.gitignore` rule blocks `.env` / `*.env` files in this repo so a real
`newt.env` cannot be accidentally committed.

---

Forked from [getumbrel/umbrel-community-app-store](https://github.com/getumbrel/umbrel-community-app-store).
