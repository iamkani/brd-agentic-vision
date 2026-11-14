# T20: Agent core (loop, tools, client, cache, traces)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M60 | P0 | Sonnet (orchestrator reviews the design) | 90 min | T03, T05, T17 |

- **Owns:** `cattle/agent/client.py`, `cattle/agent/tools.py`, `cattle/agent/loop.py`, `cattle/agent/cache.py`, `cattle/agent/trace.py`, `scripts/run_agent.py`.
- **Read:**
  - `02_contracts.md` §6, §7, §8.12, §9 (agent.yaml), §13;
  - `03_pitfalls.md` rows 31–35.
- **Fixtures:** none. Use `FakeClient` and fake tools. **No network in tests.**

## Goal
The brief scores "a genuine agent loop that uses tools and checks itself". Build a small, auditable loop. A vision-language model plans, calls tools (look, measure, read), observes the results, and commits a label only when the claim cites real tool results from this case. Everything is traced, cached and bounded. The loop must degrade safely: no internet, a spent budget or a bad reply all produce an `estimated` fallback, never a crash.

## Specification

### `client.py`
- **`LLMClient` protocol:** `complete(messages: list[dict]) -> str`. Each message is `{"role": "system"|"user"|"assistant", "content": str | list}`. A content list may contain `{"type": "text", "text": ...}` and `{"type": "image", "path": ...}` items.
- **`OpenAICompatClient(base_url, model, api_key_env, timeout=60)`:**
  - POST `{base_url}/chat/completions`;
  - images become `data:image/jpeg;base64,...` URL parts;
  - the key comes **only** from the environment variable;
  - 3 retries with exponential backoff on 429 and 5xx.
- **`FakeClient(replies: list[str])`:** returns the replies in order and appends each `messages` argument to `.calls`. When exhausted, it raises `RuntimeError("FakeClient exhausted")`.

### `tools.py`
- **`ToolResult(summary: str, data: dict = {}, cam: str | None = None, ts: str | None = None, image: str | None = None)`.**
- **`ToolRegistry`:**
  - `register(name, fn, description, params)`;
  - `call(name, args) -> ToolResult`, which **raises** on an unknown tool or a tool error so the loop can record it;
  - `describe() -> str`, a compact tool list for the system prompt.
- **`default_registry(ctx)`** builds the real tools over project data:
  - `zoom`, `burst`, `switch_camera`, `look_back`, `look_ahead`;
  - `zone_lookup`, `neighbors`, `herd_state`, `measure_motion`;
  - `read_tag`, `query_log`, `cut_clip`.

  It is used by `scripts/run_agent.py` and is not unit-tested here.

### `loop.py`
- **`Case(case_id, cow_id, cam, t_start, t_end, rule_label, rule_confidence, context: dict)`.**
- **`Limits(max_steps=6)`.**
- **`Verdict(status, label, confidence, basis, evidence_ids, steps)`**, where status is `committed`, `limit` or `error`.
- **`run_case(case, registry, client, limits, trace) -> Verdict`.**

**Reply protocol.** The model must answer with one JSON object per turn:
- `{"thought": str, "action": "<tool name>" | "commit", "args": {...}}`;
- a commit's `args` are `{"label": <class code>, "confidence": 0–1, "evidence": ["e1", ...]}`.

**Rules:**
1. **Steps.** Each parsed action (a tool call or a commit attempt) is one step. Stop when `steps == max_steps`, without making another model call.
2. **Evidence IDs.** Every tool call, successful or failed, gets the next evidence ID `e1`, `e2`, … A failed or unknown tool produces an `error` trace record, and its summary is sent back to the model.
3. **Commit checks.** A commit is accepted only if the label is a valid class code **and** every cited ID refers to a **successful** result of this case. Otherwise, send back `commit rejected: <reason>` and continue.
4. **Basis.** An accepted commit gets `basis = "observed"` if the cited results come from ≥ 2 distinct (`cam`, `ts`) pairs, else `"estimated"`.
5. **Malformed JSON.** Send one repair message. A second consecutive malformed reply means `status = "error"`.
6. **Fallbacks.** On `limit`, on `error`, and when the client raises (offline), the verdict is `rule_label`, `rule_confidence` and basis `estimated`.
7. **Determinism.** Prompts must be deterministic. Never put the current time in them, so the cache works.

### `cache.py`
- **`ReplyCache(path)`:** a SQLite table `(key TEXT PRIMARY KEY, reply TEXT, tokens_in INT, tokens_out INT, created TEXT)`.
- **`CachedClient(inner, cache, model)`:**
  - the key is `sha256(model + json.dumps(messages, sort_keys=True))`, with image paths replaced by the SHA-256 of their file bytes;
  - on a hit, return the cached reply without calling `inner`.

### `trace.py`
- **`TraceWriter(path, redact=None)`** writes JSONL records per contracts §8.12 and replaces every string listed in `redact` with `[REDACTED]`.
- **`render_markdown(path) -> str`** renders a readable trace: the case header, a numbered list of steps with tool, arguments, evidence ID and summary, and a `## Verdict` section.

### `scripts/run_agent.py`
- **Flags:** `--cases P --budget-cases N --resume --select-traces`.
- **Run:**
  - runs the cases from T21 with `CachedClient(OpenAICompatClient(...))`;
  - writes `outputs/traces/<case_id>.jsonl` and `<work>/agent_labels.parquet`;
  - stops at `daily_token_budget`.
- **`--select-traces`** picks 3 cases and renders them to `outputs/traces/selected/{success,failure,night}.md`:
  - **success:** committed, observed, and agrees with the check set if available;
  - **failure:** a limit, an error, or a disagreement with the check set;
  - **night:** a committed `grey_IR` case.

## Locked test
```python
# LOCKED TEST: tests/unit/test_agent_loop.py
import json

import pandas as pd
import pytest

from cattle.agent.cache import CachedClient, ReplyCache
from cattle.agent.client import FakeClient
from cattle.agent.loop import Case, Limits, run_case
from cattle.agent.tools import ToolRegistry, ToolResult
from cattle.agent.trace import TraceWriter, render_markdown

T0 = pd.Timestamp("2000-01-10 10:00:00")


def registry():
    reg = ToolRegistry()
    reg.register("zoom", lambda cam, t, **k: ToolResult(summary=f"head down at bunk on {cam}", data={"head_down": True},
                                                       cam=cam, ts=t), "Enlarged crop of the cow", {"cam": "str", "t": "str"})
    reg.register("switch_camera", lambda cam, t, **k: ToolResult(summary=f"muzzle in bunk on {cam}", cam=cam, ts=t),
                 "Same moment from another camera", {"cam": "str", "t": "str"})

    def broken(**k):
        raise RuntimeError("camera offline")

    reg.register("broken", broken, "always fails", {})
    return reg


def case(rule_label="standing"):
    return Case(case_id="case-1", cow_id="C03", cam="cam01", t_start=T0, t_end=T0 + pd.Timedelta(minutes=2),
                rule_label=rule_label, rule_confidence=0.4, context={"zone": "bunk", "picture": "colour"})


def act(action, **args):
    return json.dumps({"thought": "x", "action": action, "args": args})


def run(replies, tmp_path, name="t.jsonl", limits=None, client=None):
    client = client or FakeClient(replies)
    v = run_case(case(), registry(), client, limits or Limits(), TraceWriter(tmp_path / name))
    return v, client


def test_success_two_independent_views_is_observed(tmp_path):
    v, _ = run([act("zoom", cam="cam01", t="10:00:30"), act("switch_camera", cam="cam02", t="10:00:30"),
                act("commit", label="feeding", confidence=0.85, evidence=["e1", "e2"])], tmp_path)
    assert (v.status, v.label, v.basis) == ("committed", "feeding", "observed")
    assert v.evidence_ids == ["e1", "e2"] and v.confidence == pytest.approx(0.85)


def test_single_view_is_estimated(tmp_path):
    v, _ = run([act("zoom", cam="cam01", t="10:00:30"), act("commit", label="feeding", confidence=0.7, evidence=["e1"])],
               tmp_path)
    assert v.status == "committed" and v.basis == "estimated"


def test_unknown_evidence_rejected_then_limit_falls_back(tmp_path):
    v, _ = run([act("commit", label="feeding", confidence=0.9, evidence=["e9"])] * 3, tmp_path, limits=Limits(max_steps=2))
    assert (v.status, v.label, v.basis) == ("limit", "standing", "estimated")


def test_invalid_label_rejected(tmp_path):
    v, _ = run([act("zoom", cam="cam01", t="1"), act("commit", label="eating", confidence=0.9, evidence=["e1"]),
                act("commit", label="feeding", confidence=0.9, evidence=["e1"])], tmp_path)
    assert v.status == "committed" and v.label == "feeding"


def test_malformed_json_gets_one_repair(tmp_path):
    v, _ = run(["not json", act("zoom", cam="cam01", t="1"), act("commit", label="lying", confidence=0.6, evidence=["e1"])],
               tmp_path)
    assert v.status == "committed" and v.label == "lying"


def test_two_malformed_in_a_row_is_error_fallback(tmp_path):
    v, client = run(["nope", "still nope", act("commit", label="lying", confidence=0.6, evidence=[])], tmp_path)
    assert (v.status, v.label, v.basis) == ("error", "standing", "estimated") and len(client.calls) == 2


def test_step_limit(tmp_path):
    v, client = run([act("zoom", cam="cam01", t=str(i)) for i in range(20)], tmp_path, limits=Limits(max_steps=6))
    assert v.status == "limit" and v.steps == 6 and len(client.calls) == 6


def test_tool_errors_become_observations(tmp_path):
    v, _ = run([act("broken"), act("fly"), act("zoom", cam="cam01", t="1"),
                act("commit", label="feeding", confidence=0.7, evidence=["e3"])], tmp_path)
    assert v.status == "committed"
    lines = [json.loads(line) for line in (tmp_path / "t.jsonl").read_text().splitlines()]
    assert any(r["type"] == "error" and "camera offline" in (r.get("summary") or "") for r in lines)


def test_offline_client_falls_back(tmp_path):
    class Down:
        calls = []

        def complete(self, messages):
            raise ConnectionError("no internet")

    v = run_case(case(), registry(), Down(), Limits(), TraceWriter(tmp_path / "t.jsonl"))
    assert (v.status, v.label, v.basis) == ("error", "standing", "estimated")


def test_cache_avoids_second_call(tmp_path):
    replies = [act("zoom", cam="cam01", t="1"), act("commit", label="feeding", confidence=0.7, evidence=["e1"])]
    inner = FakeClient(replies * 2)
    cached = CachedClient(inner, ReplyCache(tmp_path / "cache.sqlite"), model="fake")
    run_case(case(), registry(), cached, Limits(), TraceWriter(tmp_path / "a.jsonl"))
    n = len(inner.calls)
    run_case(case(), registry(), cached, Limits(), TraceWriter(tmp_path / "b.jsonl"))
    assert n == 2 and len(inner.calls) == n


def test_trace_redaction_and_render(tmp_path):
    client = FakeClient([act("zoom", cam="cam01", t="sk-secret-123"), act("commit", label="feeding", confidence=0.7,
                                                                          evidence=["e1"])])
    run_case(case(), registry(), client, Limits(), TraceWriter(tmp_path / "t.jsonl", redact=["sk-secret-123"]))
    text = (tmp_path / "t.jsonl").read_text()
    assert "sk-secret-123" not in text and "[REDACTED]" in text
    md = render_markdown(tmp_path / "t.jsonl")
    assert "## Verdict" in md and "feeding" in md and "e1" in md
```

## Done when
- `python scripts/verify_task.py T20` passes.
- The report includes the exact system prompt you wrote, in ≤ 40 lines.
- The orchestrator reviews it for clarity, the commit rules, and how the evidence format is taught to the model.

## Do not
- Call any real API in tests.
- Log API keys.
- Send images except through `OpenAICompatClient` with the configured endpoint (contracts §13; H09 decides whether that endpoint is allowed).
