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
- Replace `CHANGE_ME.example.com` with your Pangolin domain **everywhere** (6 lines —
  in nano press `Ctrl+\`, enter the old and new text, then answer `A` to replace
  All; the pairing line becomes `pair.<your-domain>`, or set any other host you
  prefer). Leave the generated secrets alone.

Then **Stop and Start the app** in umbrelOS (env changes only apply on a full
stop/start, not a plain restart) and verify:

```bash
curl http://umbrel.local:3777/_liveness
```

## 2. Pangolin resource

Point a Pangolin resource at this device, port `3777`, with WebSocket support
enabled. Clients (Buzz desktop/mobile, buzz-cli) connect to `wss://<your-domain>`.

## 3. Mobile pairing (optional)

The Buzz mobile app pairs by QR (NIP-AB): the desktop transfers your identity to
the phone over an end-to-end-encrypted channel, confirmed by a code shown on both
screens. Because the main relay is closed-membership, an unpaired phone can't
reach it — pairing goes through the app's dedicated **open** pairing sidecar
(`buzz-pair-relay`, host port `3778`). It is a publicly reachable,
unauthenticated endpoint by design: what's protected is the *content* (only
ephemeral NIP-44 ciphertext between throwaway session keys ever transits it,
and it stores nothing), while the endpoint itself carries the same
resource-exposure surface as any public WebSocket — the app caps it with a
container memory limit, and Pangolin sits in front if you want rate limiting.
Without this setup, desktop Settings → Mobile shows
`WebSocket connection failed: HTTP error: 404 Not Found` — that's "pairing not
configured", nothing is broken.

1. Add a second Pangolin resource: same device, port `3778`, WebSocket support
   on, its own hostname (any works — `pair.<your-domain>`,
   `buzz-pair.<apex>`, …). Both the desktop and the phone connect to it, so it
   must be publicly reachable.
2. Make sure `.env` has the matching line (fresh installs get a template line;
   **existing installs add it by hand** — `.env` is never regenerated):

   ```bash
   BUZZ_PAIRING_RELAY_URL=wss://<your-pairing-host>
   ```

3. Stop and Start the app, then in Buzz desktop: Settings → Mobile → scan the QR
   with the phone and confirm the matching code on both screens.

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
