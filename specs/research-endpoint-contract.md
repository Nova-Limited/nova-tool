# /research Endpoint Contract

Reference doc for building the Group 4 nova-tool UI that calls the Layer 3 research endpoint. Captured from `~/nova-proxy/worker.js` commit `e86b144` on Mon 25 May 2026.

---

## Endpoint basics

| | |
|---|---|
| URL | `https://nova-ch-proxy.jamie-b41.workers.dev/research` |
| Method | `POST` only (other methods return `405 method_not_allowed`) |
| Auth | None at the proxy (proxy holds `ANTHROPIC_KEY` secret server-side) |
| Body | JSON, see below |
| CORS | Open (handled by `CORS_HEADERS` and preflight) |

---

## Request body

```json
{
  "companyName": "Grant & Stone Limited",
  "companyNumber": "01987538",
  "registeredAddress": "Unit 14 Aylesbury Road, Aston Clinton, Aylesbury HP22 5AH",
  "sector": "Builders Merchant"
}
```

| Field | Required | Validation | Notes |
|---|---|---|---|
| `companyName` | yes | non-empty, max 200 chars | trimmed server-side |
| `companyNumber` | yes | non-empty | cache key, see below |
| `registeredAddress` | no | string | substituted as `(not provided)` if absent |
| `sector` | no | string | substituted as `(not provided)` if absent |

Returns `400 missing_input` if `companyName` or `companyNumber` missing. `400 invalid_input` if `companyName` exceeds 200 chars. `400 bad_request` if the body isn't valid JSON.

---

## Caching

- Cache key: `research-cache:{companyNumber}` in the `PLACES_CACHE` KV namespace (yes, reused — the namespace is shared with /places, separated by key prefix)
- TTL: **14 days**
- Cache hit overlays `_meta.cacheHit: true` and `_meta.creditsUsed: 0` onto the stored payload, preserving the original `researchedAt`
- Negative results are still cached (the only thing not cached is upstream error responses)

---

## Quotas

Two counters, both must pass:

| Window | Cap | KV key |
|---|---|---|
| Daily | 20 calls | `research-quota:YYYY-MM-DD` (48h TTL) |
| Monthly | 200 calls | `research-quota-month:YYYY-MM` (35d TTL) |

Returns `429 daily_cap_exceeded` or `429 monthly_cap_exceeded` with current count in `detail`.

Counters bump **after** the Anthropic call regardless of success or failure (a failed call still costs Anthropic credit). Cache hits do **not** bump quotas.

---

## Success response (200)

```json
{
  "buyingTriggers":       { "value": "...", "confidence": "high|medium|low" },
  "decisionMaker":        { "value": "...", "confidence": "high|medium|low" },
  "financialHealth":      { "value": "...", "confidence": "high|medium|low" },
  "sustainabilityStance": { "value": "...", "confidence": "high|medium|low" },
  "recommendedAngle":     { "value": "...", "confidence": "high|medium|low" },
  "_meta": {
    "researchedAt":    "2026-05-25T10:43:00.000Z",
    "model":           "claude-sonnet-4-6",
    "webSearchesUsed": 3,
    "tokensUsed":      4521,
    "cacheHit":        false,
    "creditsUsed":     1
  }
}
```

Every one of the five fields is guaranteed to be present, an object, with `value` and `confidence` keys, with `confidence` being exactly one of `high`/`medium`/`low`. If the model returns anything else, the proxy returns `502 research_lookup_failed` rather than passing bad data through.

`value` can be `null` (model couldn't find anything for that field). UI should render this as "No data" or similar, not as empty string.

`_meta.cacheHit: true` means the data is up to 14 days old. Worth showing the user.

---

## Error responses

| Status | error code | Cause |
|---|---|---|
| 400 | `bad_request` | Body isn't valid JSON |
| 400 | `missing_input` | companyName or companyNumber missing |
| 400 | `invalid_input` | companyName over 200 chars |
| 405 | `method_not_allowed` | GET/PUT/etc instead of POST |
| 429 | `daily_cap_exceeded` | 20 calls already today |
| 429 | `monthly_cap_exceeded` | 200 calls already this month |
| 500 | `misconfigured` | PLACES_CACHE KV unbound or ANTHROPIC_KEY secret missing |
| 502 | `research_lookup_failed` | Various upstream/parse/validation failures |

The 502s deserve careful UI handling. Sub-causes in `detail`:

- `Anthropic returned {status}: {first 300 chars of body}` — direct upstream failure
- `invalid_json_from_anthropic` — Anthropic returned 200 but body wasn't JSON
- `response_truncated_at_max_tokens` — model hit 2000 token cap before finishing JSON (fail loud rather than try to parse a half-JSON)
- `no_text_block_in_response` — model returned only tool_use blocks
- `model_returned_invalid_json: {first 200 chars}` — JSON.parse failed on extracted text
- `model_returned_invalid_shape: {field} {reason}. Raw parsed sample: {first 500 chars}` — required field missing or confidence not in {high,medium,low}

All error responses are JSON with `{ error, detail }`.

---

## Under the hood (informational)

- **Model:** `claude-sonnet-4-6`
- **Max tokens:** 2000
- **Tool:** `web_search_20250305` (Anthropic's server-side web_search)
- **Prompt caching:** System prompt marked `cache_control: ephemeral` (5-min TTL, ~80% saving on back-to-back calls)

The system prompt instructs the model to fill five fields about a UK company, each as `{value, confidence}`. Sector-specific guidance is baked in for warehousing/logistics, manufacturing, retail/supermarkets, builders merchants/distribution. Other sectors fall back to general indicators.

---

## UI build implications

When wiring the Group 4 nova-tool UI:

1. **Trigger:** "Research" button next to a saved/searched company. Requires `companyName` and `companyNumber` already known (Use this match flow has both).
2. **Loading state:** Anthropic + web_search round-trip takes 15-45 seconds typically. Show a non-blocking progress indicator; don't just hang the button.
3. **Result display:** Five cards/sections, one per field. Confidence as a coloured pill (green/amber/grey for high/medium/low). `null` values rendered explicitly ("No data found").
4. **Meta strip:** "Researched 25 May 2026, 3 web searches" or "Cached result from 18 May 2026" (if `cacheHit: true`, show date from `researchedAt`).
5. **Error handling:**
   - 429 → toast "Daily limit reached, try tomorrow" or "Monthly limit reached"
   - 502 → toast "Research failed, see console" + log full detail for inspection
   - 500 → toast "Worker misconfigured, contact admin" (shouldn't happen in prod)
6. **Save:** Consider a "Save to CRM" button that pastes the brief into a new field on Companies (multilineText). Field doesn't exist yet — design decision pending.

---

## Test call (paste-ready curl)

```bash
curl -X POST https://nova-ch-proxy.jamie-b41.workers.dev/research \
  -H "Content-Type: application/json" \
  -d '{
    "companyName": "Grant & Stone Limited",
    "companyNumber": "01987538",
    "registeredAddress": "Unit 14 Aylesbury Road, Aston Clinton, Aylesbury HP22 5AH",
    "sector": "Builders Merchant"
  }'
```

First call is live (~15-45s, costs 1 quota credit + Anthropic credits). Second call within 14 days is a cache hit (instant, free).

---

*Source: nova-proxy worker.js, commit e86b144, captured Mon 25 May 2026.*
