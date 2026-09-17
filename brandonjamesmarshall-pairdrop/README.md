# PairDrop

Browser-based, cross-platform AirDrop. Files move browser-to-browser over
WebRTC; the container is only a signaling server and never sees the data.

Packaged from the official `ghcr.io/schlagmichdoch/pairdrop` image (not
the linuxserver wrapper), digest-pinned and Renovate-managed.

## How devices find each other

PairDrop groups peers by public IP. Because the server sits *inside* your
LAN, it collapses every RFC1918 client address into one room, so every
device on the home network sees every other one — through the umbrelOS
app proxy or the direct port alike.

Devices on different networks (phone on cellular, laptop at home) don't
share a room. Link them with **Pair Devices** (6-digit code / QR); the
pairing persists across sessions.

## Ports

| Port | Purpose |
| --- | --- |
| 8223 | umbrelOS app proxy (requires umbrelOS login) |
| 18223 | Direct, no auth — use this for phones, TVs, guests |

The direct port is intentional: an AirDrop replacement that first demands
an umbrelOS login on every guest's phone isn't one. PairDrop has no login
of its own; the only thing exposed is a list of device nicknames, and file
transfers still require the receiver to accept.

## Remote / cross-network use

Paired devices on different networks need a WebRTC path between them.
PairDrop ships Google's public STUN server, which works when neither side
is behind a symmetric NAT. If transfers between paired devices hang at
"connecting", you need a TURN server:

1. Run one (coturn, or a hosted TURN) and write an `rtc_config.json`:
   ```json
   {
     "sdpSemantics": "unified-plan",
     "iceServers": [
       { "urls": "stun:stun.l.google.com:19302" },
       { "urls": "turn:turn.example.com:3478", "username": "u", "credential": "p" }
     ]
   }
   ```
2. Mount it into the container and set `RTC_CONFIG=/path/to/rtc_config.json`.

If you expose PairDrop through a reverse proxy / tunnel, it must forward
`X-Forwarded-For` and WebSocket upgrades. Serve it over HTTPS: the PWA
install and clipboard features require a secure context, and browsers
increasingly restrict WebRTC on plain HTTP outside localhost.

`WS_FALLBACK=true` makes the server relay files for clients whose WebRTC is
blocked (some VPNs). It's off by default so nothing is ever proxied through
the box; turn it on only if you need it.

## Alternatives considered

- **LocalSend** — native app, mDNS discovery, no server at all. Better if
  every device can install an app; nothing to host, so not an Umbrel app.
- **Snapdrop** (official Umbrel store) — PairDrop's upstream; unmaintained
  since 2023 and lacks pairing/rooms.
