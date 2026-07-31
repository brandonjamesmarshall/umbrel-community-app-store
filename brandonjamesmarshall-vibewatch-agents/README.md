# Vibewatch agent host on Umbrel — setup guide

Runs the Vibewatch resident agents (headless Claude Code + buzz-cli) against the
Buzz relay on this box. Source and architecture:
[vibewatch-agents/agent-host](https://github.com/Vibewatch-io/vibewatch-agents/tree/main/agent-host).

Install-first: clicking Install generates
`~/umbrel/app-data/brandonjamesmarshall-vibewatch-agents/.env` (mode 600) and the
host waits, restarting, until the two CHANGE_ME values are filled.

## 1. Get the two secrets

- **`CLAUDE_CODE_OAUTH_TOKEN`** — on your Mac:

  ```bash
  claude setup-token
  ```

  Approve in the browser; paste the token it prints. It draws on your Claude
  subscription (no API billing) and expires yearly.

- **`BUZZ_NSEC_ECHO`** — in the Buzz desktop app connected to this relay: create
  an agent named **Echo** (Agents sidebar → create). Copy the `nsec…` it mints.
  Then add Echo as a member of the `#agent-dev` channel (channel → members → add).

## 2. Fill the env

```bash
sudo nano ~/umbrel/app-data/brandonjamesmarshall-vibewatch-agents/.env
```

Replace both CHANGE_ME values. `BUZZ_RELAY_URL` defaults to the Buzz app on this
same Umbrel via the Docker host gateway — leave it unless your relay lives
elsewhere. `GH_TOKEN` is optional until an agent needs the product repos (Echo
doesn't).

Then **Stop and Start the app**. Logs should show
`echo: online as <pubkey>… in agent-dev`.

## 3. Smoke test

Say anything in `#agent-dev`. Echo replies in a thread within a few seconds
(it's instructed to always work the word "buzz" in). Ask "what did I say
earlier?" to prove session persistence. Say "silence test" and it should not
reply at all.
