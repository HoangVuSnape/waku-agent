# Monitoring & Chaos Testing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix a real, reproduced data-loss bug (concurrent writers silently
losing a calendar event from `calendar.ics`), and add a graceful-degradation
path so a failed LLM call reaches the user as a clear reply instead of an
unhandled exception killing the turn.

**Architecture:** Two independent, self-contained fixes, each landed as its
own commit. (1) Replace `calendar.ics`'s unlocked read-modify-write with a
full regeneration from `calendar_events` (the DB stays the single source of
truth) serialized by a dependency-free cross-process lockfile. (2) Wrap the
loop's LLM reasoning call in a try/except that turns a failure into a
`LoopResult` reply instead of propagating.

**Tech Stack:** Python stdlib only (`os`, `time`, `contextlib`, `threading`) —
no new dependency, per CLAUDE.md's "no new dependencies without discussion."
Tests use `pytest` and the existing `ScriptedClient`/`make_waku` helpers in
`evals/helpers.py`.

**Spec:** `docs/superpowers/specs/2026-09-16-monitoring-chaos-testing-design.md`

**Note on scope:** This plan does NOT add retry-with-backoff for LLM call
failures — that is a separate, larger design in
`docs/superpowers/specs/2026-09-16-reliability-patterns-design.md`, which
lands after this plan and builds on top of the graceful-catch added here
(retry attempts happen inside that catch, before finally falling through to
the same graceful reply). This plan also does not touch the
same-slot-different-title duplicate-event bug — also reliability-patterns'
scope, sequenced to land after this plan's lock is in place.

---

## File Structure

- **Create:** `evals/deterministic/test_concurrency.py` — the concurrent-write
  regression test. Its own file (not folded into `test_tool_trigger.py`)
  because it exercises the calendar tool directly with real threads and two
  real SQLite connections, a different test shape from the
  `ScriptedClient`-through-the-loop tests living in `test_tool_trigger.py`.
- **Modify:** `waku/tools/calendar.py` — replace `_write_ics` (unlocked
  read-modify-write) with `_file_lock` (a stdlib cross-process mutex) and
  `_regenerate_ics` (rebuilds the whole file from `calendar_events`); update
  `create_event` to run its existing duplicate-check + insert + regenerate
  under that lock.
- **Modify:** `evals/helpers.py` — `ScriptedClient._create` gains the ability
  to raise an exception instead of only returning canned responses, needed by
  the new LLM-failure test.
- **Modify:** `waku/loop/agent.py` — wrap the non-streaming reasoning call in
  a try/except.
- **Modify:** `evals/deterministic/test_tool_trigger.py` — add the new
  graceful-degradation regression test, next to the existing
  `test_iteration_guardrail_stops_runaway_loop`.

---

### Task 1: Fix the `calendar.ics` concurrent-write data loss

**Files:**
- Create: `evals/deterministic/test_concurrency.py`
- Modify: `waku/tools/calendar.py:32-52` (replace `_write_ics`), `waku/tools/calendar.py:280-295` (update `create_event`'s body)
- Test: `evals/deterministic/test_concurrency.py`

- [ ] **Step 1: Write the failing test**

Create `evals/deterministic/test_concurrency.py`:

```python
"""DETERMINISTIC EVAL — concurrent writers must not lose a calendar event.

waku's `calendar.ics` file is shared by every gateway attached to one
`.waku/` home: the dashboard's background Telegram poller thread and a
separate `python -m waku` CLI invocation both call `create_event` against
the SAME home directory, from different threads/processes. Before the fix
this test locks in, `_write_ics` did an unlocked read-modify-write of that
file — two concurrent writers could both read the same "before" state, and
whichever wrote second silently discarded the other's event.

The interleaving is forced deterministically with a `threading.Barrier`
rather than relying on scheduling luck, so this test is not flaky: both
concurrent create_event calls are guaranteed to have read calendar.ics
before either is allowed to write it back — exactly the lost-update window.
"""

from __future__ import annotations

import threading
from pathlib import Path

import waku.db as db
from waku.tools.calendar import make_tool


def test_concurrent_create_event_does_not_lose_ics_writes(tmp_path, monkeypatch):
    home = tmp_path / "home"
    home.mkdir()

    # Two independent connections simulate two separate gateways/processes
    # sharing one .waku home (e.g. the dashboard's background Telegram
    # thread and a separate CLI invocation) — check_same_thread=False because
    # each connection is created here but exercised on its own worker thread.
    conn_a = db.connect(home, check_same_thread=False)
    conn_b = db.connect(home, check_same_thread=False)
    create_event_a = make_tool(conn_a, home).fn
    create_event_b = make_tool(conn_b, home).fn

    # Seed one event synchronously first, so both threads below hit the
    # "file already exists" branch and actually race on reading it.
    seed_result = create_event_a(title="Seed Event", start="2026-07-11T09:00")
    assert "Event created" in seed_result

    barrier = threading.Barrier(2, timeout=5)
    original_read_text = Path.read_text

    def synced_read_text(self, *args, **kwargs):
        text = original_read_text(self, *args, **kwargs)
        if self.name == "calendar.ics":
            barrier.wait()  # force both threads to read before either writes
        return text

    monkeypatch.setattr(Path, "read_text", synced_read_text)

    results: dict[str, str] = {}
    errors: dict[str, BaseException] = {}

    def run_a() -> None:
        try:
            results["a"] = create_event_a(title="Meeting A", start="2026-07-11T10:00")
        except BaseException as exc:  # surface thread failures instead of a silent hang
            errors["a"] = exc

    def run_b() -> None:
        try:
            results["b"] = create_event_b(title="Meeting B", start="2026-07-11T11:00")
        except BaseException as exc:
            errors["b"] = exc

    t1 = threading.Thread(target=run_a)
    t2 = threading.Thread(target=run_b)
    t1.start()
    t2.start()
    t1.join(timeout=5)
    t2.join(timeout=5)

    assert not errors, f"thread(s) raised: {errors}"
    assert "Event created" in results["a"]
    assert "Event created" in results["b"]

    # The DB is never allowed to lose a row — SQLite's own locking handles
    # this even before any fix.
    rows = conn_a.execute("SELECT title FROM calendar_events ORDER BY start").fetchall()
    titles = {r["title"] for r in rows}
    assert titles == {"Seed Event", "Meeting A", "Meeting B"}, (
        f"expected all 3 events in the DB, got {titles}"
    )

    # The .ics file is the one that used to lose data under this exact race.
    ics_text = (home / "calendar.ics").read_text()
    for expected in ("Seed Event", "Meeting A", "Meeting B"):
        assert f"SUMMARY:{expected}" in ics_text, (
            f"'{expected}' missing from calendar.ics — a concurrent write was lost"
        )
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `.venv\Scripts\python.exe -m pytest evals/deterministic/test_concurrency.py -v`

Expected: **FAIL** on the last assertion block, with output containing
`AssertionError: 'Meeting A' missing from calendar.ics` (or `'Meeting B'` —
whichever thread's write lost the race; either is the correct failure mode).
This confirms the pre-existing bug: both DB rows exist, but the `.ics` file
is missing one event.

- [ ] **Step 3: Implement the fix**

In `waku/tools/calendar.py`, add `contextlib`, `os`, and `time` to the
imports at the top of the file:

```python
from __future__ import annotations

import contextlib
import os
import sqlite3
import subprocess
import sys
import time
from datetime import datetime
from email.utils import parseaddr
from pathlib import Path

from waku.tools.registry import Tool
```

Replace the `_write_ics` function (currently `waku/tools/calendar.py:32-52`)
with these two functions:

```python
@contextlib.contextmanager
def _file_lock(path: Path, timeout: float = 5.0, poll: float = 0.02):
    """A minimal cross-process mutex using atomic exclusive file creation —
    portable (no fcntl/msvcrt needed) and dependency-free. Two writers
    sharing one `.waku/` home (e.g. the dashboard's background Telegram
    thread and a separate `python -m waku` CLI invocation) both acquire this
    before touching calendar_events + calendar.ics together, so neither can
    observe the other's half-finished write."""
    deadline = time.monotonic() + timeout
    while True:
        try:
            fd = os.open(str(path), os.O_CREAT | os.O_EXCL | os.O_WRONLY)
            os.close(fd)
            break
        except FileExistsError:
            if time.monotonic() > deadline:
                raise TimeoutError(f"timed out waiting for lock: {path}") from None
            time.sleep(poll)
    try:
        yield
    finally:
        with contextlib.suppress(FileNotFoundError):
            os.remove(str(path))


def _regenerate_ics(home: Path, conn: sqlite3.Connection) -> None:
    """Rebuild calendar.ics from scratch from calendar_events — the DB is the
    single source of truth. Always a full rewrite, never a read-modify-write,
    so two writers can no longer silently lose an event by racing on a
    partial read of the file (see
    docs/superpowers/specs/2026-09-16-monitoring-chaos-testing-design.md)."""

    def dt(s: str) -> str:
        return s.replace("-", "").replace(":", "") + ("00" if len(s) == 16 else "")

    rows = conn.execute(
        'SELECT title, start, "end", attendees FROM calendar_events ORDER BY start'
    ).fetchall()
    body = "BEGIN:VCALENDAR\nVERSION:2.0\nPRODID:-//waku-agent//EN\n"
    for r in rows:
        body += (
            "BEGIN:VEVENT\n"
            f"SUMMARY:{r['title']}\n"
            f"DTSTART:{dt(r['start'])}\n"
            f"DTEND:{dt(r['end'])}\n"
            f"DESCRIPTION:attendees: {r['attendees'] or ''}\n"
            "END:VEVENT\n"
        )
    body += "END:VCALENDAR\n"
    (home / "calendar.ics").write_text(body, encoding="utf-8")
```

Then update `create_event`'s body in `make_tool` (currently
`waku/tools/calendar.py:280-295`) — replace the idempotence-guard-through-
`_write_ics` block:

```python
        # idempotence guard: same title+start = same event. A confused model
        # (or an impatient user) must not be able to triple-book a meeting.
        start = start[:16]  # normalize 2026-07-11T17:00:00 → 2026-07-11T17:00
        end = end[:16]
        existing = conn.execute(
            "SELECT id FROM calendar_events WHERE title = ? AND start = ?", (title, start)
        ).fetchone()
        if existing:
            return f"Event '{title}' at {start} already exists (not duplicated)."

        conn.execute(
            'INSERT INTO calendar_events (title, start, "end", attendees, notes) VALUES (?,?,?,?,?)',
            (title, start, end, attendees, notes),
        )
        conn.commit()
        _write_ics(home, title, start, end, attendees)
```

with:

```python
        # idempotence guard: same title+start = same event. A confused model
        # (or an impatient user) must not be able to triple-book a meeting.
        # The whole check-then-insert-then-regenerate sequence runs under one
        # lock: this also closes a smaller pre-existing race where two writers
        # could both pass the "not existing" check before either inserted.
        start = start[:16]  # normalize 2026-07-11T17:00:00 → 2026-07-11T17:00
        end = end[:16]
        with _file_lock(home / "calendar.ics.lock"):
            existing = conn.execute(
                "SELECT id FROM calendar_events WHERE title = ? AND start = ?", (title, start)
            ).fetchone()
            if existing:
                return f"Event '{title}' at {start} already exists (not duplicated)."

            conn.execute(
                'INSERT INTO calendar_events (title, start, "end", attendees, notes) VALUES (?,?,?,?,?)',
                (title, start, end, attendees, notes),
            )
            conn.commit()
            _regenerate_ics(home, conn)
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `.venv\Scripts\python.exe -m pytest evals/deterministic/test_concurrency.py -v`

Expected: **PASS**.

- [ ] **Step 5: Run the full deterministic suite to check for regressions**

Run: `.venv\Scripts\python.exe -m pytest evals/deterministic -q`

Expected: all tests **PASS**, including `test_create_event_writes_db_and_ics`
and `test_create_event_is_idempotent` in `evals/deterministic/test_tool_trigger.py`
(both exercise `create_event` and must still behave identically for a single
caller with no contention).

- [ ] **Step 6: Commit**

```bash
git add evals/deterministic/test_concurrency.py waku/tools/calendar.py
git commit -m "$(cat <<'EOF'
Fix calendar.ics losing events under concurrent writes

_write_ics did an unlocked read-modify-write: two callers sharing one
.waku/ home (e.g. the dashboard's background Telegram thread and a
separate `python -m waku` CLI invocation) could both read the same
file state and the second write would silently discard the first
event. The DB row survived either way; only the .ics file lost data.

Fix: calendar_events stays the single source of truth. create_event
now regenerates calendar.ics from the DB inside the same critical
section as the insert, serialized by a dependency-free cross-process
lockfile (_file_lock, stdlib only — no fcntl/msvcrt needed). This also
closes a smaller pre-existing TOCTOU race in the exact-duplicate check,
which now runs under the same lock.

evals/deterministic/test_concurrency.py reproduces the race
deterministically with a threading.Barrier forcing both writers to
read before either writes, so the test doesn't depend on scheduling
luck. It failed against the old _write_ics and passes against the fix.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Graceful degradation when the LLM call fails

**Files:**
- Modify: `evals/helpers.py:50-51` (`ScriptedClient._create`)
- Modify: `waku/loop/agent.py:79-86` (wrap the reasoning call)
- Modify: `evals/deterministic/test_tool_trigger.py` (new test, add near `test_iteration_guardrail_stops_runaway_loop`)
- Test: `evals/deterministic/test_tool_trigger.py`

- [ ] **Step 1: Extend `ScriptedClient` to support raising exceptions**

In `evals/helpers.py`, replace the `_create` method (currently lines 50-51):

```python
    def _create(self, **kwargs):
        return self._script.pop(0)
```

with:

```python
    def _create(self, **kwargs):
        item = self._script.pop(0)
        if isinstance(item, BaseException):
            raise item
        return item
```

This is additive and backward compatible: every existing test's script list
contains only `response(...)` objects, never exceptions, so their behavior is
unchanged.

- [ ] **Step 2: Write the failing test**

In `evals/deterministic/test_tool_trigger.py`, add this test directly after
`test_iteration_guardrail_stops_runaway_loop` (after line 99, before the
`# ---------- live tier` comment):

```python
def test_llm_call_failure_degrades_gracefully(tmp_path):
    """A transient provider failure (timeout, 5xx, rate limit) must not crash
    the turn — the caller gets a clear reply instead of an unhandled
    exception propagating out of respond()."""
    gate = response([text_block('{"retrieve": false, "query": "", "reason": "test"}')])
    script = [gate, TimeoutError("upstream timed out")]
    app = make_waku(tmp_path / "home", client=ScriptedClient(script))

    result = app.respond("what's on my calendar today?")

    assert "didn't respond" in result.reply
    assert result.tool_calls == []
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `.venv\Scripts\python.exe -m pytest evals/deterministic/test_tool_trigger.py::test_llm_call_failure_degrades_gracefully -v`

Expected: **FAIL/ERROR** with an unhandled `TimeoutError: upstream timed out`
raised out of `app.respond(...)` — this confirms `run_loop` today has no
protection around the reasoning call.

- [ ] **Step 4: Implement the fix**

In `waku/loop/agent.py`, replace the non-streaming call block (currently
lines 79-86):

```python
        if response is None:
            response = client.messages.create(
                model=model,
                system=system,
                messages=messages,
                tools=tools.schemas(),
                max_tokens=max_tokens,
            )
```

with:

```python
        if response is None:
            try:
                response = client.messages.create(
                    model=model,
                    system=system,
                    messages=messages,
                    tools=tools.schemas(),
                    max_tokens=max_tokens,
                )
            except Exception as exc:
                # Graceful degradation: a transient provider failure (timeout,
                # 5xx, rate limit) must reach the caller as a clear reply, not
                # an unhandled exception killing the turn. Deliberately NOT a
                # retry — see
                # docs/superpowers/specs/2026-09-16-reliability-patterns-design.md
                # for the retry-with-backoff design that wraps this same call.
                result.reply = f"(the model didn't respond: {exc} — please try again)"
                return result
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `.venv\Scripts\python.exe -m pytest evals/deterministic/test_tool_trigger.py::test_llm_call_failure_degrades_gracefully -v`

Expected: **PASS**.

- [ ] **Step 6: Run the full deterministic suite to check for regressions**

Run: `.venv\Scripts\python.exe -m pytest evals/deterministic -q`

Expected: all tests **PASS**, including `test_no_tool_turn_ends_loop_in_one_iteration`
and `test_iteration_guardrail_stops_runaway_loop` (both exercise `run_loop`'s
normal and guardrail paths and must be unaffected by the new try/except,
which only triggers on an actual exception from `client.messages.create`).

- [ ] **Step 7: Commit**

```bash
git add evals/helpers.py waku/loop/agent.py evals/deterministic/test_tool_trigger.py
git commit -m "$(cat <<'EOF'
Degrade gracefully instead of crashing on an LLM call failure

run_loop's non-streaming client.messages.create call had no exception
handling: a transient provider failure (timeout, 5xx, rate limit)
propagated all the way out of respond() uncaught. For the Telegram
gateway specifically, this means the user simply gets no reply at all
— python-telegram-bot swallows the handler exception and logs it.

Fix: catch the exception at the call site and return a LoopResult
explaining the failure instead of raising. This is deliberately NOT a
retry (see the separate reliability-patterns spec for that) — it's the
last-resort graceful-failure boundary that retry logic will eventually
sit inside of.

ScriptedClient (evals/helpers.py) now supports raising an exception
from its script list, not just returning canned responses, needed to
test this deterministically without a real flaky provider.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review Notes

- **Spec coverage:** the design's two "ship now" items (concurrent-write test
  + fix, LLM-timeout-mid-call test) are both covered by Task 1 and Task 2.
  The design's third "ship now" item — verifying whether the loop already
  catches per-tool exceptions — turned out to already be true:
  `ToolRegistry.execute` (`waku/tools/registry.py:47-58`) already wraps every
  `tool.fn(**args)` call in a try/except and returns an error string instead
  of crashing. No task needed for this; it was a verification, not a gap.
  The "nice-to-have" items (drift report, `make drift` target) are
  deliberately excluded per the spec's own decision to defer them.
- **Placeholder scan:** no TBD/TODO markers; every step has complete code.
- **Type consistency:** `_file_lock` and `_regenerate_ics` are used with the
  same signatures everywhere they appear across Task 1's steps.
- **Cross-spec dependency:** both new code comments (in `calendar.py` and
  `agent.py`) point forward to `2026-09-16-reliability-patterns-design.md`,
  so whoever implements that spec next sees exactly where their work attaches
  to this one.
