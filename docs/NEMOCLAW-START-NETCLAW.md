# Start NetClaw on a NemoClaw sandbox

This is the attach path: NetClaw on the Alma Linux host, OpenClaw already
running inside the NemoClaw / OpenShell sandbox. Default sandbox name:
`dcloud-nemoclaw`.

Do **not** install a second OpenClaw on the host. Do **not** create an
OpenShell sandbox named `netclaw`. Do **not** start
`openclaw-gateway.service` on the host. The LLM stays inside the sandbox.

For a dead `https://inference.local` doctor check, use
[NEMOCLAW-INFERENCE-LOCAL.md](NEMOCLAW-INFERENCE-LOCAL.md) first. That name
is sandbox-only. The HUD never uses it.

## What must already be up

```bash
SANDBOX="${NEMOCLAW_SANDBOX:-dcloud-nemoclaw}"

nemoclaw "$SANDBOX" doctor
nemoclaw "$SANDBOX" dashboard-url
```

Doctor should report the sandbox `Ready` and OpenShell connected to the
`nemoclaw` gateway. `dashboard-url` should resolve to the published OpenClaw
gateway (this lab: `http://127.0.0.1:18789`, also reachable on the host
data-plane address, for example `http://198.18.134.13:18789`).

If inference is the only `[fail]`, fix that with the inference runbook, then
come back here. If the sandbox is not `Ready`:

```bash
nemoclaw "$SANDBOX" recover
```

## 1. Point the host HUD at the sandbox gateway

The HUD reads `OPENCLAW_GATEWAY_URL` and `OPENCLAW_GATEWAY_TOKEN` from the
environment or from `~/.netclaw/gateway.env` (mode `0600`). Never put the
token in git, source, chat, or a world-readable file.

```bash
mkdir -p ~/.netclaw
umask 077
# Create or edit ~/.netclaw/gateway.env with an editor. Two lines only:
#   OPENCLAW_GATEWAY_URL=http://127.0.0.1:18789
#   OPENCLAW_GATEWAY_TOKEN=<sandbox gateway.auth.token>
chmod 0600 ~/.netclaw/gateway.env
```

Copy `gateway.auth.token` from the **sandbox** OpenClaw config (the value
`nemoclaw "$SANDBOX" exec -- openclaw config get gateway.auth.token` prints).
Do not paste that value into tickets, commits, or chat.

Leave the URL on `http://127.0.0.1:18789`. Do not use
`https://inference.local/v1`.

A NemoClaw rebuild rotates this token. Rewrite `gateway.env` and
`chmod 0600` again before starting the HUD.

## 2. Enable chat completions on the sandbox gateway

The HUD posts to `${OPENCLAW_GATEWAY_URL}/v1/chat/completions`. TUI works
without this; the HUD does not.

```bash
SANDBOX="${NEMOCLAW_SANDBOX:-dcloud-nemoclaw}"

nemoclaw "$SANDBOX" exec -- \
  openclaw config set gateway.http.endpoints.chatCompletions.enabled true
```

If the sandbox gateway was already running, restart only that gateway
(NemoClaw recover / sandbox gateway restart). Do not start a host OpenClaw
gateway.

## 3. Confirm first-wave skills (once per sandbox)

If you already attached with the installer, skip this.

```bash
cd /home/cisco/netclaw
NETCLAW_RUNTIME=nemoclaw \
NEMOCLAW_SANDBOX="${NEMOCLAW_SANDBOX:-dcloud-nemoclaw}" \
  ./scripts/install.sh --profile recommended
```

That installs skills and personas **into the sandbox**, writes the host
manifest under `~/.netclaw/`, and does not run `openclaw onboard` or
`npm install -g openclaw`.

## 4. Start the HUD (foreground)

```bash
cd /home/cisco/netclaw/ui/netclaw-visual
# first time on this checkout only:
#   npm install
npm run dev
```

That starts two listeners on `0.0.0.0`:

| Process | Port | Role |
|---|---|---|
| Vite | 3000 | HUD UI (`/`), Canvas chat (`/canvas.html`) |
| `server.js` | 3001 | API + WebSocket + chat proxy to the sandbox gateway |

Override bind/port if needed:

```bash
export HUD_BIND=0.0.0.0
export HUD_PORT=3001
```

`server.js` loads `~/.netclaw/gateway.env` itself. You do not need to
`source` it in the shell.

## 5. Open it

On the host:

- HUD: `http://127.0.0.1:3000/`
- Canvas chat: `http://127.0.0.1:3000/canvas.html`

From another machine on this lab network, use the host address (this dCloud
box: `http://198.18.134.13:3000/`). `127.0.0.1:3000` on your laptop is not
the Alma host.

If the TCP connect fails but `npm run dev` is running, the host firewall is
usually rejecting the port. This lab’s `firewalld` public zone needs TCP
`3000` and `3001` (and `18789` if you reach the gateway directly):

```bash
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --permanent --add-port=3001/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

The chat badge should read **LIVE**, not **LOCAL**. LOCAL plus HTTP 408
means the sandbox gateway timed out on that turn, not that the HUD process
is down. Retry a short prompt. If every request is 401 after a rebuild, the
token in `gateway.env` is stale — rewrite it (step 1).

## 6. Start the HUD in the background (optional)

Keep using `~/.netclaw/gateway.env`. Do not put the token in a unit file.

Example user unit `~/.config/systemd/user/netclaw-hud.service`:

```ini
[Unit]
Description=NetClaw Visual HUD
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/cisco/netclaw/ui/netclaw-visual
EnvironmentFile=-%h/.netclaw/gateway.env
Environment=HUD_BIND=0.0.0.0
Environment=HUD_PORT=3001
ExecStart=/usr/bin/npm run dev
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now netclaw-hud.service
systemctl --user status netclaw-hud.service
```

Do **not** `After=` / `Wants=` `nemoclaw-openshell-gateway.service` if that
gateway unit is `After=default.target` — the same `WantedBy=default.target`
cycle deletes the job. See [NEMOCLAW-INFERENCE-LOCAL.md](NEMOCLAW-INFERENCE-LOCAL.md).

To stop it:

```bash
systemctl --user stop netclaw-hud.service
```

## 7. Quick health after start

```bash
SANDBOX="${NEMOCLAW_SANDBOX:-dcloud-nemoclaw}"

# sandbox + inference
nemoclaw "$SANDBOX" doctor

# published gateway (no token printed)
curl -sS -o /dev/null -w '%{http_code}\n' --max-time 5 \
  http://127.0.0.1:18789/v1/models

# HUD UI
curl -sS -o /dev/null -w '%{http_code}\n' --max-time 5 \
  http://127.0.0.1:3000/
```

Gateway `/v1/models` may return `401` without a bearer token and still be
fine. The HUD injects the token from `gateway.env`. A `000` connect failure
means the sandbox gateway is not published on the host.

## After a NemoClaw rebuild

A rebuild wipes sandbox skills, personas, `chatCompletions`, the gateway
token, and `/etc/hosts`. In order:

1. Pin AI Defense DNS and re-run doctor
   ([NEMOCLAW-INFERENCE-LOCAL.md](NEMOCLAW-INFERENCE-LOCAL.md)).
2. Rewrite `~/.netclaw/gateway.env` with the new token (`chmod 0600`).
3. Re-enable `chatCompletions` (step 2).
4. Re-run the recommended installer if skills/personas are gone (step 3).
5. Restart the HUD so it reloads `gateway.env`.

## What not to do

- `npm install -g openclaw`
- `openclaw onboard`
- `systemctl --user start openclaw-gateway.service` (host)
- `openclaw mcp set` on the host for this path
- Copy host `config/openclaw.json` stdio MCP servers into the sandbox
- Point `OPENCLAW_GATEWAY_URL` at `https://inference.local/v1`
- Commit `~/.netclaw/gateway.env` or print the token into a ticket

## Related

- [Fix `https://inference.local`](NEMOCLAW-INFERENCE-LOCAL.md)
- [Visual HUD](../ui/netclaw-visual/README.md)
- Root [README.md](../README.md) — NemoClaw runtime table
