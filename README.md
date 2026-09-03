# Hermes Friendli Provider

FriendliAI model provider plugin for [Hermes Agent](https://github.com/NousResearch/hermes-agent).

Fast-path profile for `api.friendli.ai/serverless/v1`: catalog-driven reasoning
control (no hardcoded per-model effort tables) and the one wire quirk that
silently breaks the generic `custom` profile on Friendli — disabling
reasoning.

## Installation

```bash
hermes plugins install Lee-Si-Yoon/hermes-friendli-provider
```

Then add your API key to `~/.hermes/.env`:

```
FRIENDLI_API_KEY=your_key_here
```

(`FRIENDLI_TOKEN` is also checked, in case that's what you already have set.)

Pick the provider with `hermes model` (FriendliAI → `zai-org/GLM-5.3` etc.)
or set it in `config.yaml`:

```yaml
model:
  provider: friendli
  default: zai-org/GLM-5.3
```

## Models

The profile discovers the live catalog via `GET /models` — no hardcoded
model list. Friendli's serverless lineup changes over time (new GLM/DeepSeek/
MiniMax releases), and so does each model's reasoning-effort vocabulary; both
are read from the catalog response at runtime instead of being pinned in
code. Notable families live as of 2026-09-03:

| Model                                 | Notes                                             |
|----------------------------------------|----------------------------------------------------|
| `zai-org/GLM-5.3`                      | Flagship GLM, 1M context, graded effort (low/high/max) |
| `zai-org/GLM-5.3-Flash`                | Fast/cheap GLM tier — this plugin's `default_aux_model` |
| `zai-org/GLM-5.2`                      | GLM flagship, effort enum (high/max) + on/off toggle |
| `zai-org/GLM-5.1`                      | GLM coding model, on/off thinking toggle only      |
| `deepseek-ai/DeepSeek-V3.2`             | On/off thinking toggle only, no graded effort enum |
| `MiniMaxAI/MiniMax-M2.5`               | Reasoning via token budget only (no toggle/effort enum) |
| `LGAI-EXAONE/K-EXAONE-2.0-750B-A37B`    | On/off thinking toggle only                        |
| `google/gemma-4-31B-it`                | On/off thinking toggle only                        |

Every listed model supports tool calling, parallel tool calls, and
structured output per Friendli's `functionality` catalog fields.

## Known API quirks handled

- **Disabling reasoning never sends `reasoning_effort: "none"`.** Friendli
  validates the top-level `reasoning_effort` field against a per-model enum
  that does not include `"none"`, and rejects it with **HTTP 422**. Hermes'
  generic `custom` profile emits exactly that field on disable, so Friendli
  traffic routed through `custom` 422s the moment a user turns reasoning off
  (desktop toggle, `/reasoning none`, `reasoning_effort: none`/`false` in
  config.yaml). This plugin instead sends
  `extra_body.chat_template_kwargs.enable_thinking: false` — Friendli's
  actual disable switch — unconditionally and before any catalog fetch has
  happened, so the fix applies on the very first request.
- **No hardcoded effort vocabulary.** Unlike the in-tree Z.AI profile
  (`GLM52_EFFORTS`/`GLM53_EFFORTS` constants), this plugin reads each model's
  `reasoning_options` from the live `/models` response and clamps
  (`agent.reasoning_effort.clamp_effort`) an explicit effort request onto
  whatever enum that model actually declares. A new GLM/DeepSeek/MiniMax
  release with a different effort ladder is picked up automatically, with no
  code change here.
- **Effort is only sent where the model declares an enum.** A model whose
  catalog entry has an `"effort"` reasoning option (e.g. GLM-5.3:
  `low`/`high`/`max`) gets a top-level `reasoning_effort` on explicit
  request. A model with only a `"toggle"` option (DeepSeek-V3.2, GLM-5.1,
  EXAONE, Gemma) has no graded levels to send, so a positive effort request
  is honored as `enable_thinking: true` instead of forwarding an unvalidated
  string that would 422. A model with neither (only `"budget_tokens"`, e.g.
  MiniMax-M2.5) gets neither field — the request omits both and lets
  Friendli's server-side default apply.
- **Models with `reasoning: false` never receive reasoning fields at all**,
  regardless of what the user's Hermes config asks for.
- **The reasoning catalog is never fetched on the request hot path.** It's
  seeded by `fetch_models()` (the `/model` picker, `hermes setup`, `hermes
  doctor`) and refreshed by a background warmer; `build_api_kwargs_extras()`
  and `supported_reasoning_efforts()` only ever read a cache. A cold cache
  falls back to omitting the effort field (never a guess that could 422) —
  except the disable path above, which is always safe to send.

## Tests

```bash
HERMES_HOME=~/.hermes hermes-agent/venv/bin/python -m pytest tests/ -v
```

20 tests: provider discovery, the disable-path bug fix, catalog-driven
effort clamping across effort/toggle/budget-only/non-reasoning models, and
the `supported_reasoning_efforts` tri-state contract. No live network calls
— the catalog fetch is mocked by seeding the in-memory cache directly.

## License

MIT
