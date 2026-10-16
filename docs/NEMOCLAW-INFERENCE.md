# NemoClaw inference route (`dcloud-nemoclaw`)

How chat from this host reaches Nemotron, and how to tell a route failure from a request-body failure. Recorded 2026-09-28 after `nemoclaw dcloud-nemoclaw doctor` reported HTTP 503 and the HUD then reported HTTP 404.

The AI Defense connection URL is a credential. It lives only in `~/bin/aid-nonstream-proxy.py`. Do not copy it into this file, chat, or git history.

## Path

```text
HUD or OpenClaw client
  POST http://127.0.0.1:18789/v1/chat/completions
    OpenClaw in sandbox dcloud-nemoclaw
      POST https://inference.local/v1/chat/completions
        OpenShell supervisor (inside the sandbox netns)
          http://host.openshell.internal:11435/v1
            host proxy ~/bin/aid-nonstream-proxy.py
              AI Defense connection (URL in that script)
                nvidia/nemotron-3-super-120b-a12b
```

`host.openshell.internal` resolves to the `openshell-docker` bridge gateway (currently `172.18.0.1`) from inside the sandbox. The supervisor dials that address from the container network, where it has `172.18.0.2`. Loopback in that network is the container, not the host.

The proxy is `aid-nonstream-proxy.service` (systemd user). It listens on `127.0.0.1:11435` for host checks and on the bridge gateway so the sandbox can reach it. It does not listen on `0.0.0.0`: callers are not authenticated. It forces `stream` to false because AI Defense corrupts SSE, and it removes `stream_options` because vLLM rejects that field unless `stream` is true.

Provider state is OpenShell provider `compatible-endpoint`, config key `OPENAI_BASE_URL`, model `nvidia/nemotron-3-super-120b-a12b`.

## What failed, and what changed

### Doctor: HTTP 503, "inference service unavailable"

`nemoclaw dcloud-nemoclaw doctor` probes `https://inference.local/v1/models` from inside the sandbox. The route answered HTTP 503 with `{"error":"inference service unavailable"}`.

The provider base URL was `http://127.0.0.1:11435/v1`, and the proxy listened only on the host loopback. The supervisor cannot use that socket. This is not an upstream outage: `curl http://127.0.0.1:11435/v1/models` on the host was already HTTP 200.

Changes:

- `~/bin/aid-nonstream-proxy.py` also binds the `openshell-docker` gateway address. Restart with `systemctl --user restart aid-nonstream-proxy.service`.
- `openshell provider update compatible-endpoint --config OPENAI_BASE_URL=http://host.openshell.internal:11435/v1`

After that, the same doctor check is HTTP 200 and the models list includes `nvidia/nemotron-3-super-120b-a12b`.

### HUD: HTTP 404, "Routing to: pyATS"

The HUD posts to the sandbox gateway at `/v1/chat/completions`. On failure it prints `OpenClaw rejected the chat request (HTTP 404)` and then a local activation line such as `Routing to: pyATS`. That line means the model reply was empty. It does not mean pyATS ran.

Gateway log `/tmp/gateway.log` inside the sandbox showed the request did arrive. Each attempt returned HTTP 400 in about 150–200 ms:

```text
Stream options can only be defined when `stream=True`. (parameter=stream_options)
```

OpenClaw labels that `model_not_found` and returns HTTP 404 to the HUD. The NemoClaw network policy is not what blocked these calls. A policy deny stops the connection before the model returns a validation error. The log line `tool policy removed 3 tool(s) ... cron, gateway, nodes` only hides those owner-only tools. The model fetch still runs.

The proxy was setting `stream` to false and leaving `stream_options` in the JSON body. It now drops `stream_options` in the same rewrite. A body that still contains both fields is HTTP 200 from `http://127.0.0.1:11435/v1/chat/completions` after the service restart.

## Next diagnosis

Read the sandbox gateway log first:

```bash
nemoclaw dcloud-nemoclaw exec -- sh -c 'tail -n 80 /tmp/gateway.log'
```

Look for `[model-fetch] response ... status=`.

| What you see | Meaning | Check |
|---|---|---|
| HTTP 503 `inference service unavailable` within a few milliseconds | Supervisor cannot dial the provider URL | `ss -ltn` shows `11435` on `127.0.0.1` and on the docker bridge, not only on loopback. `openshell provider get compatible-endpoint` must not be a `127.0.0.1` base URL. `systemctl --user status aid-nonstream-proxy.service` is active. |
| HTTP 400 mentioning `stream_options` | Proxy rewrite is incomplete or an old process is still bound | Restart `aid-nonstream-proxy.service`. Confirm the running file is `~/bin/aid-nonstream-proxy.py` and that it pops `stream_options`. |
| HTTP 404 on the HUD only | OpenClaw's translation of the model failure above | Use the gateway log status, not the HUD code. |
| `Routing to: pyATS` (or another skill name) | HUD fallback after an empty model reply | Same log. pyATS was not invoked. |
| No `model-fetch` line | Request never reached OpenClaw | Dashboard publish `18789`, `gateway.http.endpoints.chatCompletions.enabled` is `true`, HUD env `OPENCLAW_GATEWAY_URL` points at that gateway. |

Host models probe, which does not print the upstream URL:

```bash
curl -sS -m 8 -o /dev/null -w '%{http_code}\n' http://127.0.0.1:11435/v1/models
```

`nemoclaw dcloud-nemoclaw doctor` is the sandbox-side check. HTTP 200 there means the bridge path works. It does not prove a chat body with `stream_options` will be accepted.
