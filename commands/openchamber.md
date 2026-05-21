---
description: Start OpenChamber with Cloudflare quick tunnel and print the URL
---

Start OpenChamber for the current project and expose it through a Cloudflare quick tunnel.

Default UI password: value of `OPENCHAMBER_UI_PASSWORD`, or command arguments if provided.

If command arguments are provided, treat `$ARGUMENTS` as the UI password override. Otherwise use `OPENCHAMBER_UI_PASSWORD`. If neither is set, fail instead of starting a public tunnel with a hardcoded password.

Run the following shell logic exactly, then output only the final Cloudflare `connectUrl` and no extra explanation:

```bash
set -euo pipefail

PORT="${OPENCHAMBER_PORT:-3000}"
DEFAULT_PASSWORD="${OPENCHAMBER_UI_PASSWORD:-}"
ARG_PASSWORD='$ARGUMENTS'

if [ -n "$ARG_PASSWORD" ]; then
  PASSWORD="$ARG_PASSWORD"
else
  PASSWORD="$DEFAULT_PASSWORD"
fi

if [ -z "$PASSWORD" ]; then
  echo "OPENCHAMBER_UI_PASSWORD or command argument is required" >&2
  exit 1
fi

openchamber stop >/dev/null 2>&1 || true
openchamber --port "$PORT" --ui-password "$PASSWORD" >/dev/null

COOKIE="$(mktemp)"
cleanup() {
  rm -f "$COOKIE"
}
trap cleanup EXIT

curl -fsS -c "$COOKIE" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -X POST "http://127.0.0.1:${PORT}/auth/session" \
  --data "{\"password\":\"${PASSWORD}\",\"trustDevice\":true}" >/dev/null

RESPONSE="$(curl -fsS -b "$COOKIE" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -X POST "http://127.0.0.1:${PORT}/api/openchamber/tunnel/start" \
  --data '{"provider":"cloudflare","mode":"quick","connectTtlMs":1800000,"sessionTtlMs":28800000}')"

python3 -c 'import json, sys; data=json.load(sys.stdin); url=data.get("connectUrl") or data.get("url"); print(url if url else ""); sys.exit(0 if url else 1)' <<< "$RESPONSE"
```
