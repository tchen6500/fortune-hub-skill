# Billing Rules — Reference

> **For protocol-level summary**, see [`../SKILL.md`](../SKILL.md) §4.
>
> **All credit values are fetched at runtime from the pricing database**. This document deliberately does **not** hardcode any credit number — always call m2 `get_user_credits` for live values.

## Core rules

1. **Deduct-on-success**: credit is only deducted when the tool call succeeds. For a single tool call, no refund is needed on failure. For a chain (t1→t2→t3→t4), each step deducts independently — no chain-level rollback needed.
2. **Free quota (t1 only)**: `bazi_basic_analysis` is the only tool with a free quota. When the free quota is available, the call returns `creditsDeducted: 0, fromFreeQuota: true`. t2–t4 currently have no free quota (db-configurable). Free quota is per-user, one-time, non-resetting.
3. **Live `credit_cost` from db**: numbers change. Always read from the pricing database at runtime. m1 `get_skill_info.fortune_pricing` and m2 `get_user_credits.pricing[]` both expose live values; the agent should prefer m2 for the most current state (m1 may be cached briefly).
4. **No separate LLM charge**: `credit_cost` for t2 / t4 includes the LLM cost. There is no token line item.
5. **Chain pattern (OneShot)**: t1→t2→t3→t4 are 4 separate `tools/call` (not a single batched call). Each step deducts independently on success.
6. **New-account starter allowance**: the hub may grant new users an initial credit balance and/or free tool quota. The exact amounts are hub-configured and **subject to change** — do not hardcode or assume them, and do not assume a new user starts at zero. The only reliable source of truth is the live m2 `get_user_credits` response (`balance` for credits, `free_remaining[]` for per-tool free quota).

## Recommended agent pattern

```
1. m2 get_user_credits           → confirm balance ≥ sum(fortune_pricing)
2. t1 bazi_basic_analysis        → base_context  (0 credits if free quota available, else credit_cost)
3. t2 bazi_pattern_analysis      → pattern_result  (1 LLM call, ~25s)
4. t3 bigluck_year_analysis      → year_analysis_result  (uses t1 + t2)
5. t4 bigluck_year_fortune_eval  → final reading  (1 LLM call, ~30–90s)
6. m3 get_usage_history          → confirm charges match expectations
```

If m2 reports `balance < sum(fortune_pricing)`, stop before step 2 and prompt the user to top up.

## What you do NOT need to know

- You do **not** need to read or call any internal "deduct" function — the hub handles it on the server side.
- You do **not** need to know about any legacy "full-analysis" deduction helper — the OneShot pattern (4 separate calls) is the canonical approach.
- You do **not** need to query any internal billing or audit store directly — m3 `get_usage_history` returns the relevant audit log.

## See also

- [`../usage/chain-patterns.md`](../usage/chain-patterns.md) — full chain pseudocode + failure recovery
- [`../usage/error-handling.md`](../usage/error-handling.md) — what to do when `INSUFFICIENT_CREDITS` fires
