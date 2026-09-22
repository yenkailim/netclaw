# Fix `https://inference.local` on NemoClaw

Doctor fail this runbook covers:

```text
[fail] Inference route (gateway): Inference gateway unreachable on
       https://inference.local/v1/models from inside the sandbox.
       DNS may have failed or the agent gateway / auth proxy is not running.
```

This is a **sandbox-internal** route. Do not point the host, the HUD, or
`~/.netclaw/gateway.env` at `https://inference.local/v1`. The LLM stays inside
the NemoClaw sandbox. The host HUD talks to the published OpenClaw gateway on
`http://127.0.0.1:18789` only.

## What `inference.local` actually is

Inside the sandbox (`dcloud-nemoclaw` by default):

1. Processes call `https://inference.local/v1/...`.
2. That name is **not** public DNS. `getent hosts inference.local` is often empty.
3. Traffic goes through the in-sandbox auth proxy (`HTTPS_PROXY=http://10.200.0.1:3128`).
4. The proxy presents an OpenShell-issued certificate for `CN=inference.local`
   (issuer `OpenShell Sandbox CA`). That CA lives at
   `/etc/openshell-tls/ca-bundle.pem` inside the sandbox.
5. The proxy then reaches the configured compatible endpoint (AI Defense / NIM)
   using the pinned AI Defense hostname.

If the AI Defense hostname is missing from the container `/etc/hosts`, the
proxy hop fails and doctor reports `inference.local` unreachable — even when
OpenClaw itself is `Ready`.

Docker regenerates `/etc/hosts` from `ExtraHosts` on every container start.
The Docker driver has **no** persistent `hosts-list` API, so the pin must be
reapplied after boot, restart, recover, and rebuild.

## Confirm the failure

Replace `dcloud-nemoclaw` if your sandbox name differs.

```bash
SANDBOX="${NEMOCLAW_SANDBOX:-dcloud-nemoclaw}"

nemoclaw "$SANDBOX" doctor
```

The exact probe doctor uses (must succeed with HTTP `2xx`–`4xx`):

```bash
nemoclaw "$SANDBOX" exec -- sh -c '
CA_BUNDLE="${CURL_CA_BUNDLE:-${SSL_CERT_FILE:-}}"
echo "CA_BUNDLE=$CA_BUNDLE HTTPS_PROXY=$HTTPS_PROXY"
HTTP_CODE=$(/usr/bin/curl -q -s -o /dev/null -w "%{http_code}" \
  --cacert "$CA_BUNDLE" --connect-timeout 3 --max-time 8 \
  https://inference.local/v1/models) || HTTP_CODE=000
echo "HTTP_CODE=$HTTP_CODE"
'
```

A host-side `curl https://inference.local/v1/models` is the wrong test. That
name is only valid **inside** the sandbox, through the sandbox proxy and CA.

## Manual fix (do this first)

This lab pins the AI Defense hostname to a VIP that presents a valid
IdenTrust certificate. Defaults match `~/.local/bin/nemoclaw-pin-ai-defense-dns`:

| Variable | Default on this host |
|---|---|
| `NEMOCLAW_SANDBOX` | `dcloud-nemoclaw` |
| `NEMOCLAW_AI_DEFENSE_HOST` | `ai-def-gw-ai-defense-onprem.apps.rtp-ai2-ucs.svpod.dc-01.com` |
| `NEMOCLAW_AI_DEFENSE_IP` | `100.65.1.21` |

### 1. Pin once

```bash
/home/cisco/.local/bin/nemoclaw-pin-ai-defense-dns --once
```

Expected log line: `pinned <host> -> 100.65.1.21 in container <id>`.

### 2. Confirm the pin inside the sandbox

```bash
SANDBOX="${NEMOCLAW_SANDBOX:-dcloud-nemoclaw}"
HOST="${NEMOCLAW_AI_DEFENSE_HOST:-ai-def-gw-ai-defense-onprem.apps.rtp-ai2-ucs.svpod.dc-01.com}"

nemoclaw "$SANDBOX" exec -- getent hosts "$HOST"
```

The first column must be `100.65.1.21`. Then re-run the doctor curl probe
above. You want `HTTP_CODE=200` (or another `2xx`–`4xx`).

### 3. Recover, then doctor

```bash
SANDBOX="${NEMOCLAW_SANDBOX:-dcloud-nemoclaw}"

nemoclaw "$SANDBOX" recover
nemoclaw "$SANDBOX" doctor
```

Healthy inference row:

```text
[ok] Inference route (gateway): https://inference.local/v1/models reachable
```

`[info] Provider health (upstream): Endpoint URL is not known` is expected
when the endpoint is withheld. `[info] Ollama` and `[info] cloudflared` are
not this failure — this path uses the compatible endpoint, not local Ollama.

## Keep the pin after restarts

`nemoclaw-ai-defense-dns.service` (user systemd) runs
`nemoclaw-pin-ai-defense-dns --watch` and re-pins when the sandbox container
starts.

```bash
systemctl --user status nemoclaw-ai-defense-dns.service
systemctl --user start nemoclaw-ai-defense-dns.service
```

If the unit is `enabled` but `inactive (dead)`, check for an ordering cycle:

```bash
journalctl --user -b --no-pager | grep -F 'ordering cycle'
```

Typical bad cycle:

```text
default.target
  → After=nemoclaw-ai-defense-dns.service   (WantedBy=default.target)
    → After=nemoclaw-openshell-gateway.service
      → After=default.target                (NVIDIA-managed unit)
```

systemd then deletes the DNS job: `Job nemoclaw-ai-defense-dns.service/start deleted to break ordering cycle`.

Fix: the DNS unit must **not** `After=` / `Wants=` `nemoclaw-openshell-gateway.service`.
The pin script already waits for the container. Use:

```ini
[Unit]
Description=Pin AI Defense DNS inside the NemoClaw sandbox
After=network-online.target

[Service]
Type=simple
ExecStart=/home/cisco/.local/bin/nemoclaw-pin-ai-defense-dns --watch
Restart=always
RestartSec=3
Environment=NEMOCLAW_SANDBOX=dcloud-nemoclaw
Environment=NEMOCLAW_AI_DEFENSE_HOST=ai-def-gw-ai-defense-onprem.apps.rtp-ai2-ucs.svpod.dc-01.com
Environment=NEMOCLAW_AI_DEFENSE_IP=100.65.1.21
Environment=NEMOCLAW_TRUSTED_PRIVATE_HOSTS=ai-def-gw-ai-defense-onprem.apps.rtp-ai2-ucs.svpod.dc-01.com

[Install]
WantedBy=default.target
```

Then:

```bash
systemctl --user daemon-reload
systemctl --user enable --now nemoclaw-ai-defense-dns.service
systemctl --user is-active nemoclaw-ai-defense-dns.service
```

Do not edit the NVIDIA-managed `nemoclaw-openshell-gateway.service`
(`After=default.target`) to break the cycle. Change only the DNS unit.

## After a NemoClaw rebuild

A rebuild recreates the container and wipes `/etc/hosts`. The watch service
should re-pin on the Docker `start` event. If doctor fails again:

```bash
systemctl --user restart nemoclaw-ai-defense-dns.service
/home/cisco/.local/bin/nemoclaw-pin-ai-defense-dns --once
nemoclaw dcloud-nemoclaw recover
nemoclaw dcloud-nemoclaw doctor
```

Rebuild also rotates the sandbox gateway token. Refresh
`~/.netclaw/gateway.env` as described in
[NEMOCLAW-START-NETCLAW.md](NEMOCLAW-START-NETCLAW.md) — that is a HUD
auth problem, not this DNS problem.

## What not to do

- Do not set `OPENCLAW_GATEWAY_URL=https://inference.local/v1` on the host.
- Do not add `inference.local` to the host `/etc/hosts`.
- Do not run `nemoclaw <sandbox> hosts-list` expecting a persistent alias.
  The Docker driver does not support that API.
- Do not use `curl -k` on the host against `inference.local`.
- Do not start a second OpenClaw on the host to “fix inference.”

## Related

- [Start NetClaw on a NemoClaw sandbox](NEMOCLAW-START-NETCLAW.md)
- `nemoclaw dcloud-nemoclaw connect` can attempt an inference-route repair if
  the pin is in place and the route is still `BROKEN`.
