# Design: Reliability Patterns

**Date:** 2026-09-16
**Status:** Approved design — ready for implementation planning
**Sub-project:** B of 3 (Cost Control / Reliability Patterns / Monitoring & Chaos Testing)
**Ships after:** [Monitoring & Chaos Testing](2026-09-16-monitoring-chaos-testing-design.md) — the
`calendar.ics` write-lock fix lands first; this spec's dedupe logic assumes a
safe write path underneath it.

## Motivation

Adapting "circuit breaker / retry-with-backoff / idempotent operations /
graceful degradation" from an AI Evals & Reliability Engineer curriculum,
scoped to what makes sense for a single-process local agent (not a distributed
system). One of the three problems below is not hypothetical: it's a
reproduced live bug from this project's own manual testing.

## Current State

- **Loop**: `waku/loop/agent.py` — a plain iteration loop calling
  `client.messages.create`/`stream` with no retry. Any exception from the
  non-streaming call path propagates and kills the turn. The only existing
  resilience is streaming falling back to non-streaming on error, and
  `WAKU_LLM_TIMEOUT` bounding one call.
- **Provider layer**: `OpenAICompatClient._call` in `waku/loop/models.py`
  retries exactly once, only for a `max_tokens`/`max_completion_tokens`
  param-name mismatch — deliberately *not* a general retry ("retrying on any
  error masked the real failure," per the existing code comment). No backoff,
  no rate-limit/5xx handling anywhere else. `_create` already special-cases
  OpenRouter's "200 with no choices" rate-limit shape by raising a
  `RuntimeError` — a usable hook for classifying "retryable" errors.
- **Existing fail-open pattern** (for reference, not being changed): both
  `retrieval_gate.should_retrieve` and the graph's `triage.classify_message`
  catch exceptions and default to a safe fallback. Both are small-model
  *decisions* with no external side effect — safe to default blindly. This
  does **not** generalize to tool calls, which do have side effects (see
  idempotency below).
- **Idempotency guard**: `waku/tools/calendar.py`'s `create_event` — exact
  `(title, start)` match after normalizing to `YYYY-MM-DDTHH:MM`.
  `test_create_event_is_idempotent` proves this catches same-title retries
  within one turn. It does **not** catch different titles at the same slot.
- **Reproduced bug**: in a real live run against a free OpenRouter model
  (nemotron-3-super via the local OmniRoute proxy), the model created two
  calendar events for the same time slot with *different* titles ("Meeting
  with Alex" and "Music with Alex") for what was clearly meant to be one
  event. The exact-match dedupe did not catch it, and a `SOUL.md` rule telling
  the model not to double-book did not stop it either — confirming that
  prompt-only guardrails are not sufficient here.
- **No circuit breaker anywhere**: a dead/rate-limited provider gets called
  again on every loop iteration up to `max_iterations`, burning the entire
  iteration budget on guaranteed failures instead of stopping fast.

## Decisions

### 1. Duplicate detection (same slot, different title)

**Chosen approach**: extend the idempotency check inside `create_event` itself
— query existing events within a **±15 minute** window of the requested start,
then compare candidates using stdlib `difflib.SequenceMatcher` on title
similarity, weighted toward shared significant tokens (attendee names carry
the strongest signal — both events in the reproduced bug shared "Alex"). If a
likely duplicate is found, refuse and return a message instructing the model
to reconcile (e.g. call `list_events` first) rather than silently creating a
second event.

Rejected alternatives:
- *Requiring `list_events` before `create_event` near an existing slot* —
  would need the tool registry to thread recent tool-call history into
  `execute()`, a bigger structural change, and still only helps if the
  duplicate is created in a later turn, not within the same batch of parallel
  tool calls (the actual reproduced bug was two tool_use blocks in one
  response).
- *`SOUL.md` rule alone* — already tried; did not stop the model in the live
  test. Keep the rule as documentation, not as the enforcement mechanism.

**Window and threshold**: ±15 minutes (narrower than the ±30 minutes
originally considered, to reduce false positives between legitimately close
but distinct meetings), with token-overlap on names weighted above raw string
similarity on the full title.

**Explicit v1 limitation, accepted rather than solved**: if a user genuinely
wants two events in the same slot, there is no override mechanism in this
version. The tool will refuse and the model can relay that to the user. Do not
build a `force=true` bypass now — YAGNI. Revisit only if this proves annoying
in practice.

### 2. Retry-with-backoff

**Chosen approach**: a small stdlib exponential-backoff wrapper (2–3 attempts,
e.g. 1s/2s/4s with jitter via `time.sleep` + `random.uniform` — no new
dependency) around the LLM reasoning call only — `client.messages.create` in
`agent.py` and the equivalent path in `OpenAICompatClient._call`. Classify
exceptions into retryable (timeout, the existing "no choices"/rate-limit
`RuntimeError`, HTTP 429/5xx) vs non-retryable (4xx auth/schema errors).

**Retries never wrap tool execution.** A tool call that already ran (e.g.
`create_event`) must never be blindly re-sent — the idempotency guard from
decision 1 is the safety net for that path, not retry suppression. Retry logic
touches only the reasoning call, before any tool has executed.

**Retry attempts do not count against `max_iterations`.** The guardrail bounds
how many times the model reasons/acts; a transport-layer retry of the same
reasoning attempt is not a new iteration.

### 3. Circuit breaker

**Chosen approach**: an in-memory consecutive-failure counter for the active
`(provider, model)` pair, scoped **per running process, not per turn**. After
N consecutive failures within a short window (e.g. 3 in 30 seconds), stop
calling the LLM and return a clear, iteration-budget-preserving error instead
of exhausting `max_iterations` on calls that will provably fail. The counter
resets on any success, and resets when the provider/model changes (a switch is
a different circuit).

This overrides an earlier per-turn framing: scoping the breaker to a single
turn would mean a long-running gateway process (Telegram, voice) re-hammers a
known-dead provider on every new incoming message. Scoping it to the process
means a dead provider trips once and stays tripped until it recovers, which is
the actual failure mode worth protecting against for an always-on gateway.

No persistence to disk — this state resets on restart, which is acceptable;
persisting it would touch `.waku/state.db` and trigger the "never wipe/modify
runtime data without asking" caution for no real benefit at this scale.

## Files to Touch

- `waku/tools/calendar.py` — extend the idempotency guard with the slot-window
  + fuzzy-title check. (Lands after the write-lock fix from the Monitoring &
  Chaos Testing spec.)
- `waku/loop/agent.py` — wrap the reasoning call in retry-with-backoff; add
  the per-process consecutive-failure circuit-breaker counter around the same
  call.
- `waku/loop/models.py` — a small `_is_retryable(exc)` classifier shared by
  the Anthropic path (via `agent.py`) and `OpenAICompatClient._call`.
- `evals/deterministic/test_tool_trigger.py` — new regression tests (below).
- `evals/helpers.py` — extend `ScriptedClient` to support raising on the Nth
  call, needed for both the retry and circuit-breaker tests.

## Test Strategy

New regression test, sibling to `test_create_event_is_idempotent`:

```python
def test_create_event_rejects_same_slot_different_title(tmp_path):
    """Reproduces the live nemotron-3-super bug: same slot, different title
    ('Meeting with Alex' / 'Music with Alex') must not double-book."""
    gate = response([text_block('{"retrieve": false, "query": "", "reason": "test"}')])
    script = [gate] + [
        response([tool_block("create_event", {"title": "Meeting with Alex", "start": "2026-07-11T17:00"}, "tu_1"),
                  tool_block("create_event", {"title": "Music with Alex", "start": "2026-07-11T17:00"}, "tu_2")], "tool_use"),
        response([text_block("Booked once.")]),
    ]
    app = make_waku(tmp_path / "home", client=ScriptedClient(script))
    result = app.respond("meeting with alex 5pm saturday")
    rows = app.conn.execute("SELECT COUNT(*) FROM calendar_events").fetchone()[0]
    assert rows == 1, f"expected 1 event, got {rows}"
```

This fails on current code (exact-match dedupe lets both through) and must
pass once the slot-window + fuzzy-title check lands. Also add a negative case
— two different people, same time slot, must NOT be flagged as duplicates —
to guard against the fuzzy check being too aggressive.

Retry and circuit-breaker each get a `ScriptedClient` variant that raises
`TimeoutError`/`RuntimeError` on the first N `.create()` calls then succeeds
(retry: assert the loop still completes and produces a reply; breaker: assert
a clean early `LoopResult.reply` after N consecutive failures without
exhausting `max_iterations`) — both fully offline, deterministic, no network
call, no API key needed.

## Sequencing Note

Depends on [Monitoring & Chaos Testing](2026-09-16-monitoring-chaos-testing-design.md)
landing its `calendar.ics` write-lock first. Both specs touch
`waku/tools/calendar.py` on independent axes (write-safety vs.
duplicate-detection); landing the lock first avoids building dedupe logic on a
write path that can still silently lose data.
