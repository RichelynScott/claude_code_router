# Routing the Hermes Agent through Claude Code Router

This recipe shows how to point [`hermes-agent`](https://github.com/RichelynScott/hermes-agent)
(an AIBC self-improving agent) at a locally-running Claude Code Router (CCR)
instance. CCR then handles multi-provider routing, transformer-based request
normalisation, and per-task model selection (background / longContext / think /
webSearch / image) on Hermes' behalf.

This is one of two supported ways to give Hermes a transformer-aware backend
in the AIBC platform. The other path — `cc_gateway_project` — is
Anthropic-only and applies identity normalisation; pick **one** per Hermes
instance, not both.

## Protocol note (read this first)

CCR exposes the **Anthropic Messages API** (`POST /v1/messages`), not the
OpenAI Chat Completions API. Hermes must therefore be configured to call CCR
via its Anthropic-compatible backend (Anthropic Agent SDK / `@anthropic-ai/sdk`
configured with a custom `baseURL`), **not** its OpenAI backend. If your
Hermes deployment is currently wired only to `OPENAI_BASE_URL`, switch it to
the Anthropic backend before continuing — otherwise requests will 404 against
`/v1/chat/completions`.

The exact env vars depend on Hermes' release; the canonical set CCR emits via
`ccr activate` is `ANTHROPIC_AUTH_TOKEN` + `ANTHROPIC_BASE_URL`, which any
Anthropic-SDK-based client (including Hermes' Anthropic backend) will pick up.

## Prerequisites

- CCR installed locally (`npm i -g @musistudio/claude-code-router` or run from
  this repo with `pnpm build && node packages/cli/dist/cli.js`).
- A configured `~/.claude-code-router/config.json` with at least one Provider
  and a `Router.default` entry. See the root `README.md` for the full schema.
- `APIKEY` set in the CCR config (required if `HOST` is anything other than
  `127.0.0.1`; strongly recommended even on loopback so Hermes authenticates
  explicitly rather than relying on bind-address security).
- A Hermes-agent build whose backend can be pointed at a custom Anthropic
  base URL.

## 1. Start CCR

```bash
ccr start
# CCR listens on http://127.0.0.1:3456 by default
ccr status
```

Confirm the server is reachable and your `APIKEY` is accepted:

```bash
curl -s http://127.0.0.1:3456/api/config -H "x-api-key: $CCR_APIKEY" | head
```

## 2. Point Hermes at CCR

The cleanest way is to use CCR's own activation helper, which exports the
right Anthropic-SDK environment variables for the current shell:

```bash
eval "$(ccr activate)"
# This sets:
#   ANTHROPIC_AUTH_TOKEN  -- mirrors your CCR APIKEY
#   ANTHROPIC_BASE_URL    -- e.g. http://127.0.0.1:3456
#   NO_PROXY=127.0.0.1
#   DISABLE_TELEMETRY=1
#   DISABLE_COST_WARNINGS=1
```

Then launch Hermes from that same shell. Any Anthropic-SDK call Hermes makes
will hit CCR; CCR will run the configured transformers (`anthropic`,
`openrouter`, `tooluse`, etc.) and forward to the resolved provider.

If you prefer not to use `ccr activate` (e.g. you launch Hermes from a
systemd unit), export the same two variables directly:

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:3456"
export ANTHROPIC_AUTH_TOKEN="$CCR_APIKEY"
```

Both `Authorization: Bearer …` and `x-api-key: …` are accepted by CCR — the
Anthropic SDK uses the latter, which is the recommended path here.

## 3. Optional: Hermes-aware preset

For reproducibility, save a CCR preset that maps Hermes' task categories to
current-generation Anthropic models (or whatever your provider catalogue
offers). Drop this into
`~/.claude-code-router/presets/hermes-anthropic/manifest.json`:

```json
{
  "name": "hermes-anthropic",
  "version": "0.1.0",
  "description": "Hermes-agent -> CCR -> Anthropic current-gen models",
  "author": "AIBC",
  "keywords": ["hermes", "anthropic", "aibc"],
  "schema": [
    {
      "id": "anthropicKey",
      "type": "password",
      "label": "Anthropic API key",
      "prompt": "Enter your Anthropic API key",
      "placeholder": "sk-ant-...",
      "required": true
    }
  ],
  "template": {
    "Providers": [
      {
        "name": "anthropic",
        "api_base_url": "https://api.anthropic.com/v1/messages",
        "api_key": "{{anthropicKey}}",
        "models": [
          "claude-opus-4-7",
          "claude-sonnet-4-6",
          "claude-haiku-4-5"
        ],
        "transformer": { "use": ["Anthropic"] }
      }
    ],
    "Router": {
      "default": "anthropic,claude-sonnet-4-6",
      "background": "anthropic,claude-haiku-4-5",
      "think": "anthropic,claude-opus-4-7",
      "longContext": "anthropic,claude-sonnet-4-6",
      "longContextThreshold": 60000
    }
  }
}
```

Then install with:

```bash
ccr preset install ~/.claude-code-router/presets/hermes-anthropic
ccr restart
```

The `Anthropic` transformer is a passthrough — it preserves the original
Anthropic request/response shape, which is exactly what we want when the
target provider IS Anthropic. Model names above are illustrative for the
2026-Q2 generation; replace with whatever your provider currently exposes.

## 4. Verify the round-trip

From the shell that has `ANTHROPIC_BASE_URL` set, drive Hermes through one
short task and then inspect CCR's application-level log:

```bash
hermes run --once "Summarise your routing path in one sentence."
tail -n 20 ~/.claude-code-router/claude-code-router.log
```

You should see a routing-decision line indicating which scenario
(`default` / `background` / `think` / …) was selected, plus the resolved
`provider,model` tuple. Server-level HTTP logs live separately in
`~/.claude-code-router/logs/ccr-*.log`.

## Security notes

- **Always set `APIKEY`** in the CCR config when Hermes is the consumer.
  Hermes runs unattended and will happily reuse whatever endpoint it is
  given; an unauthenticated CCR on localhost is still reachable by any
  process on the host.
- **Do not bind CCR to `0.0.0.0`** for this integration unless Hermes is on a
  different host *and* `APIKEY` is set. CCR forces `HOST` back to `127.0.0.1`
  when no `APIKEY` is configured — relying on that behaviour as your only
  control is fragile.
- **Do not chain CCR through `cc_gateway_project`.** The gateway expects to
  terminate at `api.anthropic.com`; CCR may rewrite to a non-Anthropic
  provider, which the gateway will reject as a protocol mismatch. Pick one.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Hermes gets 401 from CCR | `ANTHROPIC_AUTH_TOKEN` doesn't match CCR's `APIKEY`. |
| Hermes gets 404 on `/v1/chat/completions` | Hermes is still wired to its OpenAI backend; switch it to the Anthropic backend (CCR does not expose `/v1/chat/completions`). |
| Wrong model used for long contexts | `longContextThreshold` higher than the request's estimated token count; lower it or set `Router.longContext` explicitly. |
| Tool calls dropped | Provider transformer missing `tooluse` — add `"tooluse"` to the provider's `transformer.use` array. |
| Requests hang in unattended runs | Set `"NON_INTERACTIVE_MODE": true` in `config.json` so CCR doesn't wait on stdin. |

## Related

- Root `README.md` — full CCR configuration reference, including the
  `ccr activate` mechanism this recipe relies on.
- `CLAUDE.md` — sibling-repo map and AIBC platform context.
- `cc_gateway_project` — alternative Anthropic-only path with identity
  normalisation.
