# Design: Monitoring & Chaos Testing (ships first)

**Date:** 2026-09-16
**Status:** Approved design — ready for implementation planning
**Sub-project:** C of 3 (Cost Control / Reliability Patterns / Monitoring & Chaos Testing)
**Ships before:** [Reliability Patterns](2026-09-16-reliability-patterns-design.md) — see Sequencing below

## Motivation

waku-agent is a local, single-user, single-process personal assistant — there is
no hosted production traffic, no multi-tenant load, no canary deployment target.
Several roadmap items under "production monitoring" and "chaos/load testing"
(from an AI Evals & Reliability Engineer curriculum) assume a hosted service and
genuinely do not apply here. This spec adapts what's left to a laptop-scale app,
and — more importantly — surfaces a real, already-reproducible data-loss bug
found while researching the gap.

## Current State

- **Tracing**: `waku/ops/tracing.py` writes JSONL always to `.waku/traces/`, plus
  optional OTel export to Phoenix/Langfuse. Every trace is one local user's own
  turns — there is no "production" traffic distinct from this.
- **Eval split**: `evals/deterministic/` (0/1, pytest) vs `evals/judge/`
  (DeepEval GEval, 0–1 scored). CLAUDE.md: "never mix" the two.
- **Dataset**: `evals/dataset.jsonl` has 11 cases today, consumed by the live
  tier of `evals/deterministic/test_tool_trigger.py` (skipped in CI — needs an
  API key).
- **CI**: `.github/workflows/validate-skills.yml` runs only the deterministic
  tier on every PR/push. The judge tier and the live-model dataset cases never
  run in CI.
- **Release gate**: `waku/ops/release_gate.py` already runs deterministic (must
  pass) → judge (if a key is present) → writes `eval_report.json` and appends
  to `eval_runs.jsonl`. This is already a scored-history mechanism, just not
  periodic.
- **Chaos coverage today**: exactly one test —
  `test_iteration_guardrail_stops_runaway_loop` (scripts 99 tool calls, checks
  `max_iterations` stops it).
- **Retrieval-gate failure coverage**: `evals/deterministic/test_retrieval_gate.py`
  already covers malformed JSON, no-JSON, reasoning-preamble text, missing keys,
  and API exceptions — all fail-open. Thorough; do not duplicate.
- **Concurrency — the real bug**: `waku/db.py` opens SQLite with
  `check_same_thread=True` and no app-level lock by default; only gateways that
  need multi-threading (`whatsapp.py`, `browser_agent.py`) pass
  `check_same_thread=False` and wrap calls in their own `threading.Lock()`.
  Worse: **`waku/tools/calendar.py`'s `_write_ics` (~line 32–52) does an
  unlocked read-modify-write of `calendar.ics`** — read the file, append the new
  `VEVENT`, write the whole file back. Two concurrent `create_event` calls can
  race: both read the same "before" body, both append, the second write wins
  and **silently drops the first event** from the `.ics` file (the SQLite row
  still exists — only the file view of the calendar loses data). Verified by
  reading the code directly; this is not hypothetical.

  This is realistic today, not a contrived edge case: `waku dashboard` already
  starts the Telegram poller on a background *thread* within the same process
  (see `waku/gateway/telegram.py: start_in_background`), and a separate
  `python -m waku` CLI invocation is a second *process* against the same
  `.waku/` home. Both can call `create_event` concurrently.

## Candidate Approaches

**Quality drift detection** (without hosted traffic):
1. New standalone script (`scripts/drift_check.py`) — isolated, but duplicates
   what `release_gate.py` already does (run suite → append JSONL history).
2. **Extend `release_gate.py`'s existing `eval_runs.jsonl` history** with a
   small `scripts/drift_report.py` that reads it and prints a pass-rate trend,
   flagging a drop vs. the last N runs. Reuses existing plumbing, no new
   always-on process.
3. Scheduled cron/launchd job — over-engineered for a laptop that isn't always
   on.

**Chaos scenarios** (deterministic tier, `ScriptedClient` pattern):
- Tool raises an unhandled exception mid-loop.
- LLM client times out mid-call (distinct from the retrieval-gate's own
  timeout coverage — this is the main loop call).
- Concurrent `respond()` calls racing on `state.db` / `calendar.ics` — the bug
  above.
- Malformed tool-call *arguments* from the model (lower priority — likely
  already handled by provider SDK parsing; skip unless a real report surfaces
  it).

**Local load test**: a small `threading`-based script reusing the same
`ScriptedClient` helper (`evals/helpers.py`), run with a higher N and reported
as a summary — not a separate harness from the concurrency chaos test above,
just that test run wider.

## Decision

**Ship now:**
1. A new deterministic test that reproduces the `calendar.ics` / `state.db`
   race (concurrent `create_event` calls via `threading`, asserting no event is
   lost in either the DB table or the `.ics` file). This test is **expected to
   fail on current code** — that failure is the point; it documents and locks
   in the bug until fixed.
2. Once the test proves the race, fix it by making **SQLite the single source
   of truth and regenerating `calendar.ics` from the DB inside the same
   transaction that inserts the event row**, serialized through a `.lock`
   sidecar file (acquire before the DB insert, release after the `.ics`
   rewrite). This is preferred over a bare `threading.Lock()` around the
   existing read-modify-write because **concurrency here is cross-process**
   (the dashboard's background Telegram thread and a separate `python -m waku`
   CLI invocation both touch the same `.waku/` home), and a bare
   `threading.Lock()` only protects threads within one process. Regenerating
   from the DB also removes the read-modify-write's other latent risk: a torn
   read of a partially-written `.ics` file.
3. LLM client timeout mid-call test, alongside the existing runaway-loop test
   in `evals/deterministic/test_tool_trigger.py`.

**Nice-to-have, not this round:**
- Drift report over `eval_runs.jsonl` — genuinely useful after a model swap or
  a `SOUL.md` edit, but `evals/dataset.jsonl` has only 11 cases today; a trend
  over that few cases is noise. Grow the dataset first.
- Tool-raises-mid-loop test — worth adding once the loop's actual exception
  handling around tool execution is confirmed during implementation (unverified
  today — see Open Question resolved below).

**Explicitly out of scope, not applicable to this app:**
- Hosted-traffic drift monitoring, canary/shadow deployment, k8s-style
  sustained-load testing, a distributed tracing backend beyond the existing
  Phoenix/Langfuse OTel export. There is no multi-tenant traffic and no
  rollout target — forcing these in would be curriculum-compliance theater,
  not a real improvement.

## Files to Touch

- `evals/deterministic/test_concurrency.py` (new) — the race-condition
  regression test; doubles as the local load test with a parametrized N.
- `evals/deterministic/test_tool_trigger.py` — add the LLM-timeout-mid-call
  case near the existing runaway-loop test.
- `waku/tools/calendar.py` — the actual fix (cross-process lock around
  `_write_ics` + the DB insert), landed only after the new test proves the bug.
- `evals/helpers.py` — extend if the `ScriptedClient` needs a variant that
  raises on a specific call index for the timeout test.

## Test Strategy

All new tests belong in `evals/deterministic/` — every one of them is a binary
"did it crash / did it lose data / did it stop at the limit" check, no
subjective quality judgment involved, consistent with CLAUDE.md's "never mix"
deterministic and judge tiers. The drift script (if and when built later) is
not a pytest at all — it is a manually-invoked report over `release_gate.py`
output, not part of CI.

## Resolved Decisions (were open questions)

1. **Lock type**: cross-process file lock, not `threading.Lock()` alone — both
   a same-process background thread (dashboard's Telegram poller) and a
   separate CLI process can touch the same `.waku/` home concurrently.
2. **Does the loop already catch per-tool exceptions?** Unverified — to be
   confirmed by reading `waku/loop/agent.py`'s tool-execution path at
   implementation time, not assumed here. If it doesn't, the fix (a bounding
   try/except around `tools.execute()`) ships alongside the new test, not as a
   separate follow-up.
3. **Periodic drift-replay**: deferred. Revisit once `evals/dataset.jsonl`
   has meaningfully more than 11 cases.
4. **`make drift` Makefile target**: skip. Keep any future drift script under
   `scripts/`, invoked manually, to avoid Makefile clutter for a feature that
   isn't built yet.

## Sequencing Note

This spec ships **before** [Reliability Patterns](2026-09-16-reliability-patterns-design.md).
Both specs touch `waku/tools/calendar.py`, on different axes: this one adds a
write-lock (data-integrity correctness bug, affects all writes regardless of
content), the other adds duplicate-event detection (business logic, assumes a
safe write path underneath it). Landing the lock first means the dedupe logic
is built on a foundation that isn't already losing writes.
