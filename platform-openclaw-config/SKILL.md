---
name: openclaw-config
description: Configure OpenClaw gateway settings including agents, channels, heartbeat, and model defaults via the CLI.
triggers:
  - "openclaw config"
  - "openclaw set default model"
  - "openclaw heartbeat model"
  - "openclaw gateway restart"
---

# OpenClaw Config Skill

CLI-based config management for OpenClaw via `openclaw config` subcommands.

## Verify Gateway Status

```bash
openclaw gateway status
```
- Checks systemd service, RPC probe, running channels
- Log file at `/tmp/openclaw/openclaw-YYYY-MM-DD.log`
- Common issue: `signal-cli` not installed → Signal channel fails with ENOENT

## Config Management

### Get config
```bash
openclaw config get agents.defaults.model
openclaw config get agents.defaults.heartbeat
openclaw config get channels
```

### Set config
```bash
openclaw config set agents.defaults.model.primary minimax/minimax-m2.7
openclaw config set agents.defaults.heartbeat.model openrouter/nvidia/nemotron-3-super-120b-a12b:free
```

### Config file
- Location: `~/.openclaw/openclaw.json`
- Backed up automatically with `.bak` on each change

## Gateway Restart

```bash
openclaw gateway stop && openclaw gateway start
```
or via systemd:
```bash
systemctl --user restart openclaw-gateway.service
```

## Schema Discovery

**Note**: `openclaw.dev` is parked/squatted — use `openclaw config schema` instead of web docs.

Config schema has many nested keys. To find a specific setting:
```bash
openclaw config schema | python3 -c "
import json,sys
d=json.load(sys.stdin)
# search for heartbeat model key
..."
```

Note: `openclaw config get agents.defaults.heartbeat` returns empty if not explicitly set, even though the schema supports it.

## Disable a Channel

```bash
openclaw config set channels.signal.enabled false
openclaw gateway stop && openclaw gateway start
```

## Common Log Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `spawn signal-cli ENOENT` | signal-cli not installed | `apt install signal-cli` or disable channel |
| `channel exited: signal daemon exited` | Signal disabled but not cleanly | `openclaw config set channels.signal.enabled false` |

## Key Config Paths

| Path | Purpose |
|------|---------|
| `agents.defaults.model.primary` | Default model for new sessions |
| `agents.defaults.model.fallbacks` | Fallback model chain |
| `agents.defaults.heartbeat.model` | Model used for heartbeat checks |
| `agents.defaults.heartbeat.every` | Heartbeat frequency (e.g. "15m") |
| `agents.defaults.heartbeat.activeHours` | Restrict heartbeats to hours |
| `agents.list[].model` | ⚠️ **Overrides** `agents.defaults.model.primary` — if wrong, agent uses unexpected model |
| `agents.defaults.workspace` | Workspace directory |
| `channels.whatsapp.enabled` | WhatsApp channel |
| `channels.signal.enabled` | Signal channel |

## Direct Provider Configuration

If setting a direct provider model (e.g. `minimax/minimax-m2.7`), the provider must be configured in `models.providers`. Otherwise the gateway agent times out with no error message.

### Provider Config Structure

The provider config at `models.providers.<name>` requires:

```json
{
  "baseUrl": "https://api.minimax.io/anthropic",
  "apiKey": "your-api-key",
  "api": "anthropic-messages",
  "models": [
    {
      "id": "MiniMax-M2.7",
      "name": "MiniMax M2.7",
      "input": ["text", "image"],
      "contextWindow": 1000000,
      "maxTokens": 8192,
      "thinking": {
        "type": "enabled",
        "budgetTokens": 64000
      }
    }
  ]
}
```

**CRITICAL — correct MiniMax provider config**:
1. baseUrl MUST be `https://api.minimax.io/anthropic` — the `/anthropic` suffix is required (API returns 404 without it)
2. api MUST be `anthropic-messages` — not `openai-completions`
3. Model id is `MiniMax-M2.7` (capital M, dash before 2.7) — not `minimax-m2.7`
4. MiniMax `/v1/models` returns 404 — the `models` array MUST be populated to avoid enumeration failure
5. Provider config validation is SILENT — invalid keys/values are silently ignored with NO error. Gateway falls back to OpenRouter routing, causing confusing `timeout` or `no healthy upstream` errors

**Setting provider config** (use JSON string):

```bash
openclaw config set models.providers.minimax '{"baseUrl":"https://api.minimax.io/anthropic","apiKey":"YOUR-KEY","api":"anthropic-messages","models":[{"id":"MiniMax-M2.7","name":"MiniMax M2.7","input":["text","image"],"contextWindow":1000000,"maxTokens":8192,"thinking":{"type":"enabled","budgetTokens":64000}}]}'
```

**Hot reload caveat**: Changing `models.providers` requires a **full gateway restart** — `openclaw gateway stop && openclaw gateway start` does a hot reload that may not reinitialize the provider. If direct provider calls still timeout after config changes, try `systemctl --user restart openclaw-gateway.service`.

### Provider Config Validation is Silent

**Critical**: If `models.providers.<name>` has invalid keys or wrong values, OpenClaw **silently ignores the provider** at startup — no error is logged. The gateway then routes `provider/model` through OpenRouter instead of the direct provider. This causes confusing errors:

- `model_not_found` — OpenRouter doesn't have that model
- `timeout` / `no healthy upstream` — routing to wrong endpoint

**Look for these log errors to diagnose**:
```
Error: Config validation failed: models.providers.minimax: Unrecognized key: "providerName"
Error: Config validation failed: models.providers.minimax.models.0.api: Invalid option
```

**Always validate your provider config** before restarting the gateway:
```bash
# Check the backup to retrieve the actual API key
APIKEY=$(python3 -c "import json; d=json.load(open('/root/.openclaw/openclaw.json.bak')); print(d['models']['providers']['minimax']['apiKey'])")

# Test direct API manually first
curl -s -X POST "https://api.minimax.io/v1/text/chatcompletion_v2" \
  -H "Authorization: Bearer $APIKEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"minimax-m2.7","messages":[{"role":"user","content":"hi"}],"max_tokens":10}'
```

**Useful commands**:
```bash
# Get the schema for valid provider config structure
openclaw config schema | python3 -c "import json,sys; d=json.load(sys.stdin); print(json.dumps(d['properties']['models']['properties']['providers']['additionalProperties'], indent=2))"
```

### Debugging Direct Provider Calls

If a direct provider model times out but OpenRouter routing works:
1. Test the API directly: verify baseUrl and auth work before blaming OpenClaw
2. If direct API works (HTTP 200) but OpenClaw times out → provider not fully initialized by gateway
3. If direct API also fails → API key, model name, or network issue

### Fallback Chain Always Works

Even if the direct provider is misconfigured or down, the fallback chain ensures the agent never fails:

```bash
openclaw config set agents.defaults.model '{"primary":"minimax/MiniMax-M2.7","fallbacks":["openrouter/nvidia/nemotron-nano-12b-v2-vl:free","openrouter/nvidia/nemotron-3-super-120b-a12b:free"]}'
```

Gateway logs show failover decisions with `failoverReason`: `rate_limit`, `timeout`, `model_not_found`.

## Model Format

OpenClaw accepts:
- Direct provider: `minimax/minimax-m2.7` — **requires provider entry in `models.providers`**
- OpenRouter: `openrouter/minimax/minimax-m2.7` — uses existing OpenRouter auth
- OpenRouter free tier: `openrouter/nvidia/nemotron-3-super-120b-a12b:free`
- `openrouter/auto` — let OpenRouter pick best

## Skills

- `openclaw-imports/*` — imported OpenClaw skills for Google Workspace, Gmail, Calendar, etc.
