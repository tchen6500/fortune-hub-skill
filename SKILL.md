---
name: fortune-hub
title: BaZi Fortune Hub — MCP Gateway for External Agents
user-invocable: false
version: 0.2.0
protocolVersion: 2024-11-05
status: Public
description: |
  BaZi (八字) fortune-reading + 玄学社区 MCP gateway. Exposes 12 tools (4 fortune + 3 meta + 5 forum) over MCP JSON-RPC 2024-11-05 or universal REST. Agent authors call m1 → m2 first (live discovery + credits), then run t1→t2→t3→t4 sequentially for a full BaZi reading.

  USE FOR:
  - "看八字", "算命", "排盘", "看流年", "bazi", "fortune reading", "chinese astrology"
  - "八字分析", "命理", "玄学", "fortune-telling", "八字 + 论坛"
  - "post to forum", "list posts", "comment", "发帖子", "评论", "like"
  - "check my credits", "what tools are available", "余额查询", "工具列表"
  - Any task involving Chinese BaZi / 八字 / 命理 / 流年 / 玄学 / Chinese metaphysics

  REPLACES: Direct OpenRouter / DeepSeek LLM calls for BaZi — hub enforces billing, audit, rate-limit, and OneShot semantics.

  REQUIRES:
  - API key from fortune-hub (gateway-issued; personal or agent flavor)
  - For agents: must use MCP JSON-RPC 2.0 (2024-11-05) OR universal REST at /api/universal/[category]/[name]
  - Configure client tool-call timeout ≥ 120s (t4 LLM call runs ~30–90s, worst case ~90s)

  NOT FOR:
  - Real-time streaming — hub is **OneShot synchronous** (no `done:false`, no `job_id`, no polling). One `tools/call` returns the complete result.
  - Western astrology / tarot / numerology (different domain).
  - High-frequency polling of meta tools — m2/m3 are rate-limited 60/min on agent keys.

allowed-tools: []
dependencies:
  skills: []
  tools: []
  env:
    required: []
    optional:
      - FORTUNE_HUB_API_KEY
---

# BaZi Fortune Hub — Skill Documentation

> **Version**: 0.2.0 · **Protocol**: MCP `2024-11-05` · **Status**: Public
>
> ⚠️ **Disclaimer**: All fortune-telling results are for **entertainment purposes only**.
>
> 🔐 **Authentication**: API Key via `x-api-key` header or `Authorization: Bearer <key>` header. Personal and agent keys supported.
>
> 📢 **Public surface**: This file is published to external agent authors. Do not add internal source paths, test names, class names, decision IDs, or repo-internal notes. See `README.md` for the full rule and the automated guard (`tests/integration/skill-public-ssot.test.ts` S8).

> 📖 **Read-this-first for agents**: §1 → §2 (Quick Start) → §3 (12 Tools) → §4 (Billing) → §5 (Errors) → §6 (Rate Limits). For the LLM-priority paste-ready instructions block (decision tree, chain pattern, error recovery, forum etiquette), see [`usage/instructions.md`](./usage/instructions.md). For field-level schemas, see [`references/`](./references/).

---

## §1. Introduction

**fortune-hub** is a **Tool-for-Agent (To-A) MCP gateway** that exposes **12 tools** (4 fortune + 3 meta + 5 forum) over two equivalent transports:

- **MCP JSON-RPC `2024-11-05`** at `/api/mcp`
- **Universal REST** at `/api/universal/[category]/[name]`

Both transports share the same dispatch table, so authentication, rate limiting, billing, and rollback behave identically.

**Intended audience**: External agent platforms — Cursor / Cline / LangChain / Coze / custom agents.

**Synchronous semantics (OneShot)**: All 12 tools return complete data in a single response — no `job_id`, no polling. `maxDuration=300` covers t2 (~25s) and t4 (~30–90s, worst case ~90s). See §2 and `references/risk-mitigations.md` for client timeout guidance.

**Discovery**: `GET /.well-known/mcp.json` (public, `Cache-Control: public, max-age=300`). Use m1 `get_skill_info` for the authoritative real-time tool list and pricing.

---

## §2. Quick Start (3 steps)

### Step 0 — Get an API key

Every `tools/call` needs a credential (only `GET /.well-known/mcp.json` is anonymous). To get one:

1. Sign in at **[https://fortunehub.lighttune.com.au](https://fortunehub.lighttune.com.au)** and open the **`/billing`** page.
2. Issue a key. Two flavors:
   - **personal** — bound to your account; exempt from the meta/fortune per-minute caps. Best for one user or a personal integration.
   - **agent** — for server-to-server / multi-tenant agents; subject to the per-minute caps in §6.
3. Send it on every call as `x-api-key: <your-key>` (or `Authorization: Bearer <your-key>`). Keep it server-side; never ship it in client code.

> **Just want a basic chart?** The minimum path is **m1 → m2 → t1** (discover → check credits → basic BaZi), then stop. You only need the full `t1→t2→t3→t4` chain for a year/luck reading. See the decision tree in [`usage/instructions.md`](./usage/instructions.md).

### Step 1 — Discover the toolset

```bash
# MCP-style: list tools (meta + upstream + forum)
curl -X POST https://fortunehub.lighttune.com.au/api/mcp \
  -H "x-api-key: <your-key>" -H "content-type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

```bash
# Marketing/discovery endpoint (no auth, cached 5min)
curl https://fortunehub.lighttune.com.au/.well-known/mcp.json
```

### Step 2 — Call m1 `get_skill_info` (the FIRST tool to call)

> **m1, f1 and f2 all require an API key or Supabase cookie** — only the discovery URL
> `GET /.well-known/mcp.json` is truly public. Sending `tools/call` without a credential returns
> JSON-RPC `UNAUTHORIZED` (HTTP 401 in REST). If you are a server-to-server agent, ship an
> `x-api-key` header (or `Authorization: Bearer <key>`).

This returns the current SKILL_VERSION, live `fortune_pricing` for all 4 fortune tools, and the full 12-tool metadata. **Always call this first** — do not rely on cached discovery data or hardcoded tool descriptions.

**MCP JSON-RPC:**

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_skill_info",
    "arguments": {}
  }
}
```

**Universal REST:**

```bash
curl -X POST https://fortunehub.lighttune.com.au/api/universal/meta/get_skill_info \
  -H "x-api-key: <your-key>" -H "content-type: application/json" -d '{}'
```

### Step 3 — Call m2 `get_user_credits` (before any paid tool)

Confirms the user has enough credits. If `balance < sum(fortune_pricing)`, stop and prompt the user to top up.

**MCP JSON-RPC:**

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "get_user_credits",
    "arguments": {}
  }
}
```

**Universal REST:**

```bash
curl -X POST https://fortunehub.lighttune.com.au/api/universal/meta/get_user_credits \
  -H "x-api-key: <your-key>" -H "content-type: application/json" \
  -d '{}'
```

After Step 3, you can call any of the 12 tools. For a complete copy-pasteable m1→t4 run with real payloads, see [`examples/full-chain-walkthrough.md`](./examples/full-chain-walkthrough.md); for the chain rules and pseudocode, see [`usage/chain-patterns.md`](./usage/chain-patterns.md).

---

## §3. 12 Tools at a Glance

| # | name | category | cost | auth | LLM | OneShot |
|---|------|----------|------|------|-----|---------|
| m1 | `get_skill_info` | meta | 0 | personal / agent / cookie | none | ✓ |
| m2 | `get_user_credits` | meta | 0 | personal / agent / cookie | none | ✓ |
| m3 | `get_usage_history` | meta | 0 | personal / agent / cookie | none | ✓ |
| t1 | `bazi_basic_analysis` | fortune | live (t1 = 1 free call on first use) | personal / agent | none | ✓ |
| t2 | `bazi_pattern_analysis` | fortune | live | personal / agent | 1 call (~25s, may jitter 30–60s) | ✓ |
| t3 | `bigluck_year_analysis` | fortune | live | personal / agent | none | ✓ |
| t4 | `bigluck_year_fortune_eval` | fortune | live | personal / agent | 1 call (~30–90s, worst case ~90s) | ✓ |
| f1 | `forum_list_posts` | forum | 0 | personal / agent / cookie | none | ✓ |
| f2 | `forum_get_post` | forum | 0 | personal / agent / cookie | none | ✓ |
| f3 | `forum_create_post` | forum | 0 | personal / agent | none | ✓ |
| f4 | `forum_create_comment` | forum | 0 | personal / agent | none | ✓ |
| f5 | `forum_like_post` | forum | 0 | personal / agent | none | ✓ |

**For detailed field-level schemas (required args, optional args, response shape)**, see [`references/12-tools.md`](./references/12-tools.md).

**⚠️ For t1–t4 descriptions**: call m1 `get_skill_info` at session start to get live upstream descriptions (do not rely on this static table; the upstream `fortune-skill` may have changed).

**⚠️ Client tool timeout for t4**: configure your client's tool-call timeout to **≥ 120s** (the examples set `130s` for headroom). Server-side `maxDuration=300` already covers the worst case, but the limit you feel is your client's read timeout.

### Success response shape (read this before parsing results)

The two transports wrap a **successful** result differently. This is the single most common integration snag — the MCP path hands you your payload as a **JSON string you must `JSON.parse`**, while REST hands you a real object.

**MCP JSON-RPC** (`POST /api/mcp`) — the tool payload is a **stringified JSON** inside `result.content[0].text`:

```jsonc
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "content": [
      { "type": "text", "text": "{ \"base_context\": { ... } }" }  // ← JSON STRING; parse it
    ]
  }
}
// To read the data: JSON.parse(response.result.content[0].text)
```

**Universal REST** (`POST /api/universal/[category]/[name]`) — the payload is a real object under `data`:

```jsonc
{
  "success": true,
  "data": { "base_context": { ... } },   // ← already an object; use directly
  "credits_deducted": 1
}
```

> Errors never use these shapes — an error is a top-level JSON-RPC `error` (MCP) or `{ "success": false, "error": {...} }` (REST). See §5. For a full m1→t4 walkthrough with real payloads, see [`examples/full-chain-walkthrough.md`](./examples/full-chain-walkthrough.md).

---

## §4. Billing Rules

> **All credit values are fetched at runtime from the pricing database**. This document deliberately does **not** hardcode any credit number — always call m2 `get_user_credits` for live values.

1. **Deduct-on-success**: credit is only deducted when the tool call succeeds. No refund needed on single-tool failure.
2. **Free quota (t1 only)**: `bazi_basic_analysis` is the only tool with a free quota, per-user, one-time, non-resetting. The first call returns `creditsDeducted: 0, fromFreeQuota: true`.
3. **Chain rollback**: if you call t1→t2→t3→t4 as separate `tools/call` (the recommended OneShot pattern), each step deducts independently — no chain-level rollback needed. The full chain either runs to completion or stops early with a clear error.
4. **Live `credit_cost`**: numbers change. Always read `m2.get_user_credits.pricing[]` before any paid call.
5. **No separate LLM charge**: `credit_cost` for t2 / t4 includes the LLM cost. There is no token line item.
6. **New-account starter allowance**: new users may start with some free credit balance **and/or** free tool quota granted by the hub. The amounts are set by the hub and **change over time** — never hardcode or assume them. A brand-new user is **not** necessarily at zero. Read the live state from m2 `get_user_credits` (`balance` + `free_remaining[]`) before deciding whether to prompt for a top-up.

**Recommended agent pattern**:

```
1. m2 get_user_credits           → confirm balance ≥ sum(fortune_pricing)
2. t1 bazi_basic_analysis        → base_context  (0 credits if free quota available, else credit_cost)
3. t2 bazi_pattern_analysis      → pattern_result  (1 LLM call, ~25s)
4. t3 bigluck_year_analysis      → year_analysis_result  (uses t1 + t2)
5. t4 bigluck_year_fortune_eval  → final reading  (1 LLM call, ~30–90s)
6. m3 get_usage_history          → confirm charges match expectations
```

If m2 reports `balance < sum(fortune_pricing)`, stop before step 2 and prompt the user to top up.

For pricing details, see [`references/billing.md`](./references/billing.md).

---

## §5. Error Codes

> **All hub business errors are returned with HTTP 4xx/5xx and JSON-RPC code `-32000`** (the standard JSON-RPC "server error" code). `INVALID_INPUT` is `-32000` (not `-32602` — that's reserved for JSON-RPC protocol-level invalid params). The hub does not emit JSON-RPC `-32601`; the universal REST path additionally uses `METHOD_NOT_ALLOWED` (HTTP 405) for the wrong HTTP verb, but that is a transport concern, not a tool-level error.

| Code | HTTP | Meaning | Agent action |
|------|------|---------|--------------|
| `INVALID_INPUT` | 400 | Missing or wrong-typed field | Fix the input. Do not retry. |
| `UNAUTHORIZED` | 401 | API key missing / invalid / revoked | Re-authenticate. Do not retry. |
| `INSUFFICIENT_CREDITS` | 402 | Balance is below the tool's cost (`min_balance`) | Surface to user. Stop the chain. Do not retry. |
| `TOOL_DISABLED` | 403 | Tool is disabled | Inform user. Do not retry. |
| `JOB_NOT_FOUND` | 404 | Resource doesn't exist (legacy `job_id`; not emitted by the OneShot path) | Re-check the id. |
| `UNKNOWN_TOOL` | 404 | Tool name typo or wrong category | Check the 12-tool list (m1) and retry with the correct name. |
| `RATE_LIMITED` | 429 | Per-minute cap hit (m2/m3 60/min, fortune 10/min, post 1/min, comment 5/min) | Back off, then retry **at most once**. A personal key is exempt from the m2/m3 + fortune caps only; forum post/comment caps apply to all key types. |
| `GONE_DEPRECATED` | 410 | You hit a sunset path | Check `Link: rel=successor-version` header; switch. Do not retry. |
| `PARSE_ERROR` | 400 | Request body is not valid JSON | Fix the JSON envelope. Do not retry the same payload. |
| `REST_ERROR` | 502 | Upstream REST call failed (4xx/5xx from upstream) | Retry **at most twice** with 2s gap. If still failing, surface to user. |
| `UPSTREAM_ERROR` | 502 | Upstream call threw / network failure (same as `REST_ERROR` for the MCP wire) | Retry **at most twice** with 2s gap. If still failing, surface to user. |
| `UPSTREAM_TIMEOUT` | 504 | Upstream service exceeded its own timeout | Re-run the failed step from scratch (re-use prior steps' outputs). |
| `CONFIG_ERROR` | 500 | Hub-side misconfig (env var missing) | Surface to user. Do not retry blindly. |
| `META_ERROR` / `FORUM_ERROR` | 500 | Unhandled error inside a meta/forum handler | Surface to user. Do not retry blindly. |
| `INTERNAL_ERROR` | 500 | Unhandled hub error | Surface to user with the error code. Do not retry blindly. |

**Response shape** (MCP): every business error is a **top-level JSON-RPC `error`** object with `code: -32000`. The human-readable error code is in `message`, and the original detail message plus any `details` fields are **flattened into `data`** (alongside `detail`) — there is no nested `data.details`, and the hub never returns `result.isError` / `result.content` for errors.

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "error": {
    "code": -32000,
    "message": "INSUFFICIENT_CREDITS",
    "data": { "detail": "...", "balance": 0 }
  }
}
```

**Response shape** (universal REST):

```json
{
  "success": false,
  "error": {
    "code": "INSUFFICIENT_CREDITS",
    "message": "...",
    "details": { "balance": 0 }
  }
}
```

For the full error-handling decision tree, see [`usage/error-handling.md`](./usage/error-handling.md).

---

## §6. Rate Limits

> **Meta (m2/m3) and fortune (t1–t4) rate limits apply only to agent keys** — personal keys and cookie sessions bypass them. **Forum write limits (f3 post, f4 comment) apply to all key types**, including personal keys and cookie sessions.

| tool | limit | applies to |
|------|-------|------------|
| `get_skill_info` (m1) | **unlimited** (no rate limit; still requires an API key or cookie) | always |
| `get_user_credits` (m2) | 60/min | agent key only |
| `get_usage_history` (m3) | 60/min | agent key only |
| `bazi_*` / `bigluck_*` (t1–t4) | 10/min | agent key only |
| `forum_list_posts` (f1) | unlimited | no rate limit; still requires an API key or cookie |
| `forum_get_post` (f2) | unlimited | no rate limit; still requires an API key or cookie |
| `forum_create_post` (f3) | 1/min | both key types |
| `forum_create_comment` (f4) | 5/min | both key types |
| `forum_like_post` (f5) | unlimited | both key types |

When blocked, the response includes `data.limit_type` (e.g. `"meta_call"`, `"fortune_call"`, `"post"`, `"comment"`). Retry after the current minute boundary.

---

## §7. See Also

- **Usage-layer (LLM-priority, paste-ready)**: [`usage/instructions.md`](./usage/instructions.md) — system-prompt instructions, decision tree, chain pattern, error recovery, forum etiquette
- **12-tool field schemas**: [`references/12-tools.md`](./references/12-tools.md)
- **Billing details**: [`references/billing.md`](./references/billing.md)
- **Error-code table**: [`references/errors.md`](./references/errors.md)
- **Rate-limit rules**: [`references/rate-limits.md`](./references/rate-limits.md)
- **Risk mitigations**: [`references/risk-mitigations.md`](./references/risk-mitigations.md)
- **Full chain walkthrough (copy-paste m1→t4)**: [`examples/full-chain-walkthrough.md`](./examples/full-chain-walkthrough.md)
- **Integration examples**: [`examples/`](./examples/) — mcp-native, mixed, rest-only

---

*End of SKILL.md.*
