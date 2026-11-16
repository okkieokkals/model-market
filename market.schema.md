# model.json — Model's market file (schema version 2)

Two separate files, each bot writes only its own and only *reads* the other:

- **model.json** — written only by **grok-model** (this Grok Bot, daily ~08:30 CET). This schema.
- **hermes.json** — written only by the **Hermes model-info bot** on the user's PC. grok-model never creates, edits, or overwrites it.

Neither file may contain API keys, tokens, `.env` contents, account ids or billing exports. Measured costs and summaries only.

`market.json` (schema `market.v1`) is retired. Do not write it.

## Timestamps
- Top-level `generated_at`: ISO 8601 with a Europe/Amsterdam offset, e.g. `2026-11-16T14:06:41+01:00`. Updated every time grok-model writes the file.
- Every record in `picks`, `routes`, `deals`, and `settings` (and the `method` object) has:
  - `last_updated` — when this record's *values* last changed. If a rebuild produces the same values as the previous `model.json`, the old timestamp is carried forward.
  - `checked_at` — when grok-model last sourced / verified the record (updates every run).
  - `source` — where the values came from, when known (OpenRouter endpoints, Respan, Nous, a deal `src`, "user (chat)", …).

## Top-level fields (grok-model)
| field | type | meaning |
|---|---|---|
| `schema_version` | `2` | bump on a breaking change |
| `generated_at` | ISO 8601 with offset | when this file was written (Europe/Amsterdam) |
| `data_date` | `YYYY-MM-DD` | date of the market data |
| `owner` | `"grok-model"` | who may write this file |
| `counterpart_file` | string | reminder that Hermes writes `hermes.json` separately |
| `page` | URL | the published Model Market page |
| `public_url` | URL or null | stable no-auth URL Hermes should fetch |
| `method` | object | cost formulas, rules, `tb_profiles`, `ac_ref`, `sources`, plus `last_updated` |
| `picks` | list | tiers: `tier, model, effort, score, cost, route, hermes, opencode, why`, plus stamp fields |
| `routes` | list | one per model + effort + route, see below |
| `deals` | list | `title, status (Free/Discount/Subscription/Preview/Ended), dates, detail, src`, plus stamp fields |
| `settings` | list | user settings as records (`name`, `value`, `note`, stamp fields). Today: `respan_daily_cap_usd` = 4 (the user's own adjustable budget cap, not a Respan limit) |
| `expiring_soon` | list | `what, ends (ISO), days_left, source_text`; anything ending within 45 days |
| `changes_since_last` | object | `against` (previous data date), `price_moves`, `new_promos`, `ended_promos`, `new_models`, `frontier_changes`, `notes` (human text) |

There is **no** `hermes_section`. Hermes keeps that data in `hermes.json`.

### `routes[]`
| field | type | meaning |
|---|---|---|
| `model`, `effort`, `route` | string | `route` is the page label, e.g. `OpenRouter Ionstream Flex`, `Nous undated deepseek-v4.1-flash (your fleet standard)`, `Nous 1017 snapshot (deepseek-v4.1-flash-1017)` |
| `provider` | string | `respan`, `openrouter`, `nous`, `opencode-zen`, `xai-oauth`, … (`a\|b` = several) |
| `pin` | string or null | OpenRouter provider tag needed for that price, e.g. `ionstream/flex`, `openai/flex` |
| `price_in`, `price_out`, `cache_read` | number or null | USD per 1M tokens; null = not published |
| `agent_cost` | number or null | USD per agent task (TB 4.1 profile × route price); null = not measured |
| `agent_cost_formula`, `agent_cost_source` | | the plain-formula value and where `agent_cost` came from (`acRef` = Hermes registry value) |
| `aa_cost` | number or null | USD per Artificial Analysis eval task (short eval) |
| `scores` | `{aa, tb, hermes}` | AA Intelligence Index (this effort), Terminal-Bench 4.1 % (only the effort TB ran), Hermes Index. Never converted or mapped scores |
| `latency`, `throughput` | null | OpenRouter publishes none today (all null on 16 Nov) |
| `uptime` | number or null | OpenRouter 24h uptime % for that endpoint |
| `kind` | `paid`/`discount`/`free` | |
| `apps` | string | `H` Hermes, `O` OpenCode |
| `capped` | bool | route counts toward the user's own Respan budget cap |
| `status_agent` | string | agent-basis status: `frontier`, `best uncapped route` (beaten only by a capped Respan route), `dominated`, `not measured`, `free route` |
| `end_date`, `notes` | string or null | |
| `source`, `checked_at`, `last_updated` | | stamp fields, see above |

## hermes.json (Hermes writes; suggested shape)

Use the same stamp convention: `generated_at` on the file, and `last_updated` / `checked_at` on each record (carry `last_updated` forward when values are unchanged). grok-model will *read* this file when it is reachable; it will never write it.

```json
{
  "schema_version": 2,
  "generated_at": "2026-11-17T09:00:00+01:00",
  "owner": "hermes-model-info",
  "counterpart_file": "model.json",
  "registry": {"models": 1041, "routes": 2001, "scored_models": 45, "last_updated": "...", "checked_at": "..."},
  "fleet": [{"profile": "pal", "model": "deepseek/deepseek-v4.1-flash", "provider": "nous", "pin": null, "fallbacks": ["..."], "last_updated": "...", "checked_at": "..."}],
  "measured_costs": [{"model": "...", "route": "...", "usd_per_task": 0.0, "n_tasks": 0, "cache_hit": 0.86, "window": "2026-11-16", "last_updated": "...", "checked_at": "..."}],
  "respan_spend_today_usd": 0.0,
  "probes": [{"route": "ionstream/flex", "ttft_s": 0.0, "total_s": 0.0, "runs": 5, "at": "...", "last_updated": "...", "checked_at": "..."}],
  "price_disagreements": [{"route": "...", "grok_price": [0,0,0], "hermes_price": [0,0,0], "gap_pct": 0, "last_updated": "...", "checked_at": "..."}],
  "proposals": ["free text for grok-model to act on next run"]
}
```

grok-model reads `hermes.json` each run when available: it flags any `price_disagreements` > 5 %, uses `probes` for the latency column, and uses `measured_costs` as a check on `agent_cost`.
