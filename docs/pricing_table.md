# CopilotScope — Pricing Table Reference

## File Location

```
shared/pricing_table.json
```

This file ships with the installer and is copied to `[INSTALLFOLDER]\pricing_table.json`. The daemon hot-reloads it on a file-system change notification (no restart required).

---

## Schema

```json
{
  "_updated": "YYYY-MM-DD",
  "_source": "URL(s) to pricing pages used to populate this table",
  "models": {
    "<model-id>": {
      "input_per_1m":       <float>,  // USD per 1,000,000 input tokens
      "output_per_1m":      <float>,  // USD per 1,000,000 output tokens
      "cache_read_per_1m":  <float>,  // USD per 1,000,000 cache read tokens (0 if N/A)
      "cache_write_per_1m": <float>   // USD per 1,000,000 cache write tokens (0 if N/A)
    }
  }
}
```

The `model-id` must exactly match what the API returns in the `model` field of its response (or what `vscode.lm.selectChatModels` returns in `model.id`).

---

## Updating Prices

### Manual Update

1. Open `shared/pricing_table.json` in any text editor.
2. Find the model entry to update (or add a new one).
3. Update the per-million token costs.
4. Change `_updated` to today's date.
5. Save the file — the daemon picks up the change within ~5 seconds.

### Automatic Update (future)

The GUI's **Price Calc** panel has a "Check for updates" button that will (in a future version) fetch current pricing from a public endpoint and propose an update.

---

## Current Pricing Table (as of 2026-03-11)

| Model | Input/1M | Output/1M | Cache Read/1M | Cache Write/1M |
|-------|----------|-----------|--------------|----------------|
| `gpt-4o` | $2.50 | $10.00 | $1.25 | — |
| `gpt-4o-mini` | $0.15 | $0.60 | $0.075 | — |
| `o1` | $15.00 | $60.00 | $7.50 | — |
| `o3-mini` | $1.10 | $4.40 | $0.55 | — |
| `claude-3-5-sonnet-20241022` | $3.00 | $15.00 | $0.30 | $3.75 |
| `claude-sonnet-4-5` | $3.00 | $15.00 | $0.30 | $3.75 |
| `claude-3-7-sonnet-20250219` | $3.00 | $15.00 | $0.30 | $3.75 |
| `claude-opus-4` | $15.00 | $75.00 | $1.50 | $18.75 |
| `gemini-2.0-flash` | $0.10 | $0.40 | $0.025 | — |
| `gemini-2.0-pro` | $1.25 | $5.00 | $0.31 | — |

Sources:
- OpenAI: https://openai.com/pricing
- Anthropic: https://www.anthropic.com/pricing
- Google: https://cloud.google.com/vertex-ai/generative-ai/pricing

---

## Adding a New Model

```json
"my-new-model-v1": {
    "input_per_1m":       1.50,
    "output_per_1m":      6.00,
    "cache_read_per_1m":  0.75,
    "cache_write_per_1m": 0.00
}
```

The `model-id` key is matched **case-sensitively** against the API response. Check the exact string with the Tokens panel ("Model" column).

---

## Unknown Models

If the daemon receives a request for a model ID not in the pricing table, it:
1. Calculates $0.00 cost for that request.
2. Marks the breakdown with `model_found = false`.
3. Logs a `CS-N004`-style informational note (if token count discrepancy is also present).

No alert is fired for unknown models — it's normal during model rollout periods.

---

## Cache Pricing Notes

- **Cache read** (`cache_read_per_1m`): Applies to Anthropic prompt caching and OpenAI cached input tokens. Set to `0` for models that don't support caching.
- **Cache write** (`cache_write_per_1m`): Applies to Anthropic `cache_control` prompt caching writes. Set to `0` for OpenAI (cache writes are free / included in input price).

The daemon's token ledger tracks `input_reported`, `input_actual`, `output_reported`, `output_actual` but **does not** currently separately track cache read/write token counts from API usage fields. Cache cost in the Price Calc panel is therefore estimated at zero unless the proxy captures explicit usage fields from the API response.
