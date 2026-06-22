# Full Chain Walkthrough — m1 → m2 → t1 → t2 → t3 → t4

> **Audience**: an integrator who wants one copy-pasteable run from zero to a finished
> reading. Every request below is a real, runnable call. Protocol details (12 tools,
> error codes, billing) are canonical in [`../SKILL.md`](../SKILL.md) — this file is a
> **happy-path transcript only**.
>
> Base URL: `https://fortunehub.lighttune.com.au` · Auth: `x-api-key: <your-key>` (see
> [`../SKILL.md`](../SKILL.md) §2 Step 0 for how to get a key).

## Two things to know before you start

1. **Success shapes differ by transport.** MCP returns your payload as a **JSON string**
   inside `result.content[0].text` — you must `JSON.parse` it. REST returns a real object
   under `data`. (Full detail: [`../SKILL.md`](../SKILL.md) §3 "Success response shape".)
2. **Treat t1–t3 outputs as opaque.** You do **not** parse the internals of `base_context`
   / `pattern_result` / `year_analysis_result` — you pass the **whole object** straight
   into the next step. The truncated shapes below are illustrative; never hand-build them.

The example person below: born **1985-07-15 14:30, male, Beijing**, asking about **2026**.

---

## m1 — `get_skill_info` (free, always call first)

```bash
curl -X POST https://fortunehub.lighttune.com.au/api/universal/meta/get_skill_info \
  -H "x-api-key: $KEY" -H "content-type: application/json" -d '{}'
```

```jsonc
// REST success
{
  "success": true,
  "data": {
    "skill": "fortune-hub",
    "version": "0.2.0",
    "fortune_pricing": [ { "tool_name": "bazi_basic_analysis", "credit_cost": 1 }, "..." ],
    "tools": [ "...12 entries..." ]
  },
  "credits_deducted": 0
}
```

Use the live `fortune_pricing` here — do not hardcode the numbers shown.

## m2 — `get_user_credits` (free, call before any paid tool)

```bash
curl -X POST https://fortunehub.lighttune.com.au/api/universal/meta/get_user_credits \
  -H "x-api-key: $KEY" -H "content-type: application/json" -d '{}'
```

```jsonc
{
  "success": true,
  "data": {
    "balance": 25,
    "free_remaining": [ { "tool_name": "bazi_basic_analysis", "remaining": 1 } ],
    "pricing": [
      { "tool_name": "bazi_basic_analysis",       "credit_cost": 1, "min_balance": 1 },
      { "tool_name": "bazi_pattern_analysis",      "credit_cost": 4, "min_balance": 4 },
      { "tool_name": "bigluck_year_analysis",      "credit_cost": 1, "min_balance": 1 },
      { "tool_name": "bigluck_year_fortune_eval",  "credit_cost": 6, "min_balance": 6 }
    ]
  },
  "credits_deducted": 0
}
```

**Gate**: if `balance < sum(pricing.credit_cost)` for the steps you intend to run, stop and
ask the user to top up. (Numbers above are illustrative — read your live values.)

## t1 — `bazi_basic_analysis` (live `credit_cost`; may be covered by a free quota — see `free_remaining[]`)

```bash
curl -X POST https://fortunehub.lighttune.com.au/api/universal/fortune/bazi_basic_analysis \
  -H "x-api-key: $KEY" -H "content-type: application/json" -d '{
    "birth_year": 1985, "birth_month": 7, "birth_day": 15,
    "birth_hour": 14, "birth_minute": 30, "gender": "male",
    "location": { "city_name": "Beijing" }
  }'
```

```jsonc
{
  "success": true,
  "data": {
    "base_context": {            // ← opaque; keep the whole object for t3
      "four_pillars": { "year": "乙丑", "month": "癸未", "day": "甲子", "hour": "辛未" },
      "five_elements": { "...": "..." },
      "major_luck": { "...": "..." }
    }
  },
  "credits_deducted": 0          // 0 if a free quota covered it (per `free_remaining`); else live `credit_cost`
}
```

Save `data.base_context`. Sub-second, pure algorithm.

## t2 — `bazi_pattern_analysis` (paid, ~25s LLM)

Same birth arguments as t1.

```bash
curl -X POST https://fortunehub.lighttune.com.au/api/universal/fortune/bazi_pattern_analysis \
  -H "x-api-key: $KEY" -H "content-type: application/json" -d '{
    "birth_year": 1985, "birth_month": 7, "birth_day": 15,
    "birth_hour": 14, "birth_minute": 30, "gender": "male",
    "location": { "city_name": "Beijing" }
  }'
```

```jsonc
{
  "success": true,
  "data": {
    "pattern_result": {          // ← opaque; keep the whole object for t3 AND t4
      "pattern_type": "...",
      "useful_gods": [ "..." ],
      "five_elements": { "...": "..." }
    }
  },
  "credits_deducted": 4
}
```

Save `data.pattern_result` — you reuse it in **both** t3 and t4.

## t3 — `bigluck_year_analysis` (paid, sub-second)

Pass the t1 + t2 outputs through; add `target_year`.

```bash
curl -X POST https://fortunehub.lighttune.com.au/api/universal/fortune/bigluck_year_analysis \
  -H "x-api-key: $KEY" -H "content-type: application/json" -d '{
    "base_context":   { "...from t1 data.base_context..." },
    "pattern_result": { "...from t2 data.pattern_result..." },
    "target_year":    2026
  }'
```

```jsonc
{
  "success": true,
  "data": {
    "year_analysis_result": { "...": "..." }   // ← opaque; keep for t4
  },
  "credits_deducted": 1
}
```

## t4 — `bigluck_year_fortune_eval` (paid, ~30–90s LLM — set timeout ≥120s)

Takes t3's `year_analysis_result` + t2's `pattern_result` (**not** `base_context`).

```bash
curl -X POST https://fortunehub.lighttune.com.au/api/universal/fortune/bigluck_year_fortune_eval \
  --max-time 130 \
  -H "x-api-key: $KEY" -H "content-type: application/json" -d '{
    "year_analysis_result": { "...from t3 data.year_analysis_result..." },
    "pattern_result":       { "...from t2 data.pattern_result..." }
  }'
```

```jsonc
{
  "success": true,
  "data": {
    "final_result": {
      "score": 78,
      "level": "...",
      "narrative": "2026 是你的……（the natural-language reading the user came for）"
    }
  },
  "credits_deducted": 6
}
```

This `final_result` is what you present to the user.

## Same run over MCP JSON-RPC

Identical semantics, different envelope. Each step is one `tools/call`; remember to
**parse the string** in `result.content[0].text`:

```bash
curl -X POST https://fortunehub.lighttune.com.au/api/mcp \
  -H "x-api-key: $KEY" -H "content-type: application/json" -d '{
    "jsonrpc": "2.0", "id": 10, "method": "tools/call",
    "params": {
      "name": "bazi_basic_analysis",
      "arguments": {
        "birth_year": 1985, "birth_month": 7, "birth_day": 15,
        "birth_hour": 14, "birth_minute": 30, "gender": "male",
        "location": { "city_name": "Beijing" }
      }
    }
  }'
```

```jsonc
{
  "jsonrpc": "2.0",
  "id": 10,
  "result": {
    "content": [
      { "type": "text", "text": "{\"base_context\":{ ... }}" }  // ← JSON.parse this
    ]
  }
}
```

## Follow-up turns — don't re-run t1/t2

If the user then asks "what about **2027**?", reuse the `base_context` and `pattern_result`
you already hold: re-call **only t3** (`target_year: 2027`) then **t4**. Re-running t1/t2
re-charges credits for the same person. See [`../usage/chain-patterns.md`](../usage/chain-patterns.md).

## If a step fails

OneShot has no resume token. Re-run **that step only**, reusing the earlier outputs you
hold (e.g. re-call t4 with the same `year_analysis_result` + `pattern_result`) — never
restart from t1. Full recovery rules: [`../usage/error-handling.md`](../usage/error-handling.md).

---

*Numbers (pricing, balance, score) and the truncated payload fields above are illustrative.
Read live pricing from m2 and treat all chain payloads as opaque pass-through objects.*
