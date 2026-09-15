# Design: Cost Control & Budget Guardrails

**Date:** 2026-09-16
**Status:** Approved design — ready for implementation planning
**Sub-project:** A of 3 (Cost Control / Reliability Patterns / Monitoring & Chaos Testing)
**Independent** of the other two specs — touches different files, can be built
in parallel with either.

## Motivation

Adapting "cost control" (per-request/customer cost tracking, budget alerts, a
kill-switch, measuring cache ROI) from an AI Evals & Reliability Engineer
curriculum. waku-agent supports 15+ swappable providers (including free-tier
models) across several messaging gateways (CLI, Telegram, WhatsApp, Discord,
voice) — a single install can rack up API spend across several channels
without anyone noticing quickly, even though it's ultimately one person's
wallet, not a multi-tenant SaaS bill.

## Current State

- **Ledger exists and works**: `waku/ops/tracing.py`'s `_record_usage`
  appends one line per LLM call to `.waku/usage.jsonl` —
  `{ts, provider, model, kind:"loop", in, out}`. It deliberately stores no
  cost and no session/gateway identity. `waku/ops/pricing.py`'s
  `usage_summary` reads it back and derives dollars at *read* time via
  `price_for()`, grouped by day/provider only.
- **No per-session or per-gateway field anywhere in the ledger.** The closest
  thing to gateway identity is `source` (cli/telegram/voice/dashboard), passed
  into `Waku.respond(user_message, source=...)` and stored in session/history
  metadata — but never propagated to the usage ledger.
- **Real per-call cost data exists but is unused at write time**:
  `waku/ops/pricing.py`'s `remember_price` shows OpenRouter's catalog reports
  exact per-model $/M rates, cached process-lifetime. That's a *rate*, not a
  computed *per-response cost* — the loop itself only ever reads
  `input_tokens`/`output_tokens` from the response, never a literal `cost`
  field even when the provider includes one.
- **No budget/kill-switch config anywhere** — nothing in `waku/config.py` or
  `.env.example` resembling a spend cap.
- **No prompt caching anywhere** in the codebase — so "measure cache ROI" has
  nothing to measure yet. Out of scope until a cache exists.
- **The one choke point for every LLM-initiated turn** is `Waku.respond()` —
  it already knows `source` (gateway) before dispatching into either the
  normal loop or the graph's `quick_reply` fast path. This is the only place
  that sees every gateway's turns before any tokens are spent, and a
  kill-switch placed anywhere downstream of it (e.g. inside the loop) would
  leak past the graph's fast path.

## Decisions

### Ledger extension

Extend the existing ledger rather than replace it: add `gateway` (renaming
`source` semantically) and a per-turn `session_id` to the `_record_usage`
payload. Extend `usage_summary` with `by_gateway` and `today` breakdowns.
Old rows lack these fields — `usage_summary` must tolerate missing keys via
`.get()`, since `usage.jsonl` is treated as permanent, append-only data (never
rewritten or backfilled).

Rejected alternative: an in-process running counter instead of re-reading the
ledger per call. Rejected because it drifts across multiple concurrent
gateways/processes (Telegram, CLI, and the dashboard each may run as separate
processes) and resets on restart — dishonest for a "daily spend" figure. The
file stays the source of truth, consistent with this project's local-first
philosophy.

Also rejected: literal "per-customer" cost accounting as named in the source
curriculum. There is one local user; `state.db` has no multi-tenant concept.
The closest honest analog is per-gateway + per-day budgets, and this spec
names it that way rather than forcing a tenant model that doesn't exist here.

### Budget defaults

- **Off by default** — opt-in via `WAKU_DAILY_BUDGET_USD`. Free-tier models
  cost $0 regardless, so an unset budget is harmless for the common case.
- **Scope: global** (one cap across all gateways sharing a `.waku/` home), not
  per-gateway, for the first version — matches the reality of one owner with
  one wallet. The `budget_scope` config field is still added so per-gateway
  caps can be turned on later without a schema change.
- **When tripped: block all gateways sharing that `.waku/` home**, not just
  the one whose message triggered the trip. Blocking only the triggering
  gateway would let, say, Telegram silently keep spending past a cap the user
  thought was total — confusing and defeats the point of a *daily* budget.
- **Reset time: local time**, not UTC. This is a single local user reasoning
  about "my day," not a distributed service with a canonical timezone.

### Kill-switch behavior

A `BudgetExceeded` condition is checked in `Waku.respond()` **before**
dispatching to either the loop or the graph fast path. It must never silently
drop the user's message — it raises a typed exception that `respond()` catches
and turns into a `LoopResult(reply=...)` explaining the block (e.g. "daily
budget of $X reached; raise `WAKU_DAILY_BUDGET_USD` or wait until tomorrow").
No tokens are spent, and the user is told why.

### Cache ROI

Deferred entirely to a follow-up phase, after prompt caching is actually
built somewhere in the codebase. Measuring ROI on a cache that doesn't exist
is not a real deliverable — this spec does not build a cache.

## Files to Touch

- `waku/ops/tracing.py` — `_record_usage`: add `gateway`/`session_id` fields.
- `waku/ops/pricing.py` — `usage_summary`: add `by_gateway`, `today_cost`.
- `waku/ops/budget.py` (new) — `check_budget(home, cap, scope)` raising
  `BudgetExceeded`.
- `waku/config.py` — add `daily_budget_usd: float | None` and
  `budget_scope: str` ("global" | "per_gateway", default "global").
- `waku/app.py` — `respond()`: call `check_budget` before running the turn;
  catch `BudgetExceeded` and return the explanatory `LoopResult` instead of
  invoking the loop or the graph.
- `.env.example` — document `WAKU_DAILY_BUDGET_USD`.
- `waku/ops/dashboard.py` (+ static JS) — surface today's spend vs. cap. A
  single-user local app needs an on-screen indicator, not an external
  alerting channel (email/Slack/PagerDuty would be over-engineering here).

## Test Strategy

New `evals/deterministic/test_budget.py`, following the `ScriptedClient`/
`make_waku`/`response()` pattern from `test_tool_trigger.py`:

- Pre-seed `tmp_path/home/usage.jsonl` with synthetic rows totaling over a
  cap.
- `make_waku(tmp_path / "home", client=ScriptedClient([...]), daily_budget_usd=1.0)`.
- Call `app.respond("hi")`.
- Assert `result.reply` contains the budget-exceeded message and
  `result.tool_calls == []`.
- Critically, assert the `ScriptedClient`'s script was **never consumed** — no
  LLM call was made — proving the check happens strictly before the loop, not
  as an after-the-fact log entry.

## Sequencing Note

Independent of the other two specs — different files
(`tracing.py`/`pricing.py`/`budget.py`/`config.py`/`app.py` vs.
`calendar.py`/`agent.py`/`models.py`). Can be implemented in parallel with
either [Monitoring & Chaos Testing](2026-09-16-monitoring-chaos-testing-design.md)
or [Reliability Patterns](2026-09-16-reliability-patterns-design.md).
