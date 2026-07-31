# Buzz on Umbrel — setup guide

The umbrelOS app-store description can't be copy-pasted from, so the commands live
here. The app is install-first: clicking Install generates
`~/umbrel/app-data/brandonjamesmarshall-buzz/.env` with fresh auto-minted secrets
(mode 600), and the relay crash-loops **closed** (the auth/membership policy is
hardcoded in `docker-compose.yml` and cannot be weakened from `.env`) until you fill
in the two values only you know.

## 1. Fill in your identity + domain

```bash
sudo nano ~/umbrel/app-data/brandonjamesmarshall-buzz/.env
```

- `RELAY_OWNER_PUBKEY` → your Nostr pubkey, **64-char hex** (not `npub…`; convert if
  needed).
- Replace `CHANGE_ME.example.com` with your Pangolin domain **everywhere** (5 lines —
  in nano `Ctrl+\` does replace-all). Leave the generated secrets alone.

Then **Stop and Start the app** in umbrelOS (env changes only apply on a full
stop/start, not a plain restart) and verify:

```bash
curl http://umbrel.local:3777/_liveness
```

## 2. Pangolin resource

Point a Pangolin resource at this device, port `3777`, with WebSocket support
enabled. Clients (Buzz desktop/mobile, buzz-cli) connect to `wss://<your-domain>`.

## Managing members

The relay is closed-membership; the owner (you) is bootstrapped automatically and
cannot be removed. Add and inspect members with the `buzz-admin` binary inside the
relay container:

```bash
sudo docker exec brandonjamesmarshall-buzz_relay_1 /usr/local/bin/buzz-admin add-member --pubkey npub1... --role member
```

```bash
sudo docker exec brandonjamesmarshall-buzz_relay_1 /usr/local/bin/buzz-admin list-members
```

(`remove-member` mirrors `add-member`. When adding several members in a loop,
sleep 1s between adds — same-second roster events collide. Agent identities
created from the Buzz desktop app may be added automatically; use `list-members`
to check before adding by hand.)

## Notes

- **Backups:** Postgres (workspace history + audit chain), git data, media, and the
  `.env` all live under this app's data directory, so umbrelOS backups cover them.
- **Updates:** the relay image is pinned by digest and tracks upstream `main`
  (pre-1.0, no releases yet); `BUZZ_AUTO_MIGRATE=true` is set so image bumps
  self-migrate the database.
- **Reset:** deleting the app's `.env` and stop/starting regenerates it with new
  secrets — the relay's own keypair changes, so treat that as a re-install, not a
  routine operation.
