# Routing the Hermes Agent through Claude Code Router

This recipe shows how to point [`hermes-agent`](https://github.com/RichelynScott/hermes-agent)
(an AIBC self-improving agent that speaks the OpenAI Chat Completions API) at a
locally-running Claude Code Router (CCR) instance. CCR then handles
multi-provider routing, transformer-based request normalisation, and per-task
model selection (background / longContext / think / webSearch / image) on
Hermes' behalf.

This is one of two supported ways to give Hermes a transformer-aware backend
in the AIBC platform. The other path — `cc_gateway_project` — is
Anthropic-only and applies identity normalisation; pick **one** per Hermes
instance, not both.

## Prerequisites

- CCR installed locally (`npm i -g @musistudio/claude-code-router` or run from
  this repo with `pnpm build && node packages/cli/dist/cli.js`).
- A configured `~/.claude-code-router/config.json` with at least one Provider
  and a `Router.default` entry. See the root `README.md` for the full schema.
- `APIKEY` set in the CCR config (required when listening on anything other
  than localhost; strongly recommended even on localhost so Hermes
  authenticates explicitly rather than relying on bind-address security).
- Hermes-agent v0.4+ (any version that supports configuring a custom
  `OPENAI_BASE_URL`).

## 1. Start CCR

```bash
ccr start
# CCR listens on http://127.0.0.1:3456 by default
ccr status
```

Confirm the server is up:

```bash
curl -s http://127.0.0.1:3456/health -H "Authorization: Bearer $CCR_APIKEY"
```

## 2. Point Hermes at CCR

Hermes reads OpenAI-compatible env vars. Set them in the shell that launches
the agent (or in Hermes' own `.env`):

```bash
export OPENAI_BASE_URL="http://127.0.0.1:3456/v1"
export OPENAI_API_KEY="$CCR_APIKEY"
# Tell Hermes which logical model to ask for. CCR will route this to whatever
# provider/model your Router.default (or a custom router) resolves to.
export HERMES_MODEL="claude-opus-4-7"
```

Hermes will now send Chat Completions requests to CCR; CCR will run the
configured transformers (`anthropic`, `openrouter`, `tooluse`, etc.) and
forward to the chosen provider.

## 3. Optional: Hermes-aware preset

For reproducibility, save a CCR preset that maps Hermes' task categories to
current-generation Anthropic models (or whatever your provider catalogue
offers). Drop this into
`~/.claude-code-router/presets/hermes-anthropic/manifest.json`:

```json
{
  "name": "hermes-anthropic",
  "version": "0.1.0",
  "description": "Hermes-agent → CCR → Anthropic current-gen models",
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
        "transformer": { "use": ["anthropic"] }
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

Model names above are illustrative for the 2026-Q2 generation; replace with
whatever your provider currently exposes.

## 4. Verify the round-trip

From inside the Hermes runtime:

```bash
hermes run --once "Summarise your routing path in one sentence."
```

Then tail the CCR application log to confirm Hermes' request was scored,
routed, and transformed:

```bash
tail -n 20 ~/.claude-code-router/claude-code-router.log
```

You should see a routing-decision line indicating which scenario
(`default` / `background` / `think` / …) was selected, plus the resolved
`provider,model` tuple.

## Security notes

- **Always set `APIKEY`** in the CCR config when Hermes is the consumer.
  Hermes runs unattended and will happily reuse whatever endpoint it is given;
  an unauthenticated CCR on localhost is still reachable by any local
  process.
- **Do not bind CCR to `0.0.0.0`** for this integration unless Hermes is on a
  different host *and* `APIKEY` is set. CCR explicitly disables auth when no
  `APIKEY` is configured.
- **Do not chain CCR through `cc_gateway_project`.** The gateway expects to
  terminate at `api.anthropic.com`; CCR may rewrite to a non-Anthropic
  provider, which the gateway will reject as a protocol mismatch. Pick one.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Hermes gets 401 from CCR | `OPENAI_API_KEY` doesn't match CCR's `APIKEY`. |
| Hermes gets 404 on `/v1/chat/completions` | `OPENAI_BASE_URL` missing `/v1`. |
| Wrong model used for long contexts | `longContextThreshold` higher than the request's estimated token count; lower it or set `Router.longContext` explicitly. |
| Tool calls dropped | Provider transformer missing `tooluse` — add `"tooluse"` to the provider's `transformer.use` array. |

## Related

- Root `README.md` — full CCR configuration reference
- `CLAUDE.md` — sibling-repo map and AIBC platform context
- `cc_gateway_project` — alternative Anthropic-only path with identity
  normalisation
