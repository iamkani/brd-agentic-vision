# T21: Gate and budget planner

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M60 | P0 | Haiku | 45 min | T05, T10 |

- **Owns:** `cattle/agent/gate.py`.
- **Read:** `02_contracts.md` §6, §7, §8.8; `03_pitfalls.md` rows 27 and 31.
- **Fixtures:** none.

## Goal
Decide which **bouts** the agent should review within a hard budget. The agent cannot afford per-minute review: rumination-like minutes alone are about 9,600. Selecting at the bout level, prioritised by doubt and by importance for L2, keeps the agent useful and affordable.

## Specification
`select_cases(minutes, max_cases, audit_frac=0.03, seed=0, priority_cows=None, conf_threshold=0.5) -> DataFrame`

- **Input.** `minutes` follows contracts §8.8, plus optional columns `picture` and `zone`, the per-minute majority from samples.
- **Bouts.** Build them as maximal runs of consecutive minutes per (cow, day) with the same behavior.
- **Flag a bout** with the first matching reason:
  - `low_confidence`: mean confidence < `conf_threshold`.
  - `bunk_not_feeding`: zone is mostly `bunk` and the behaviour is not feeding or drinking.
  - `night_uncertain`: picture is mostly `grey_IR` and mean confidence < 0.7.
  - `rumination_probe`: a lying or standing bout of ≥ 20 minutes outside the bunk. At most 2 per cow-day, the longest first.
- **Audit sample.** From the **unflagged** bouts, draw a seeded random sample of `ceil(audit_frac × n_unflagged)` with reason `audit`.
- **Priority order.**
  1. Bouts of `priority_cows`, the preliminary top-risk cows.
  2. Then by ascending mean confidence.
  3. Then by reason order: `low_confidence`, `bunk_not_feeding`, `night_uncertain`, `rumination_probe`, `audit`.
  4. Then by cow_id and start time.
- **Cap.** Truncate to `max_cases`.
- **Output columns:** `case_id` (`<cow>-d<day>-<HHMM>`), `cow_id, day, t_start, t_end, n_minutes, behavior, confidence, reason, priority` (rank 1..n). Deterministic for the same inputs and seed.

## Locked test
```python
# LOCKED TEST: tests/unit/test_gate.py
import pandas as pd

from cattle.agent.gate import select_cases


def minutes(cow, start, labels, conf=0.9, picture="colour", zone="open"):
    t0 = pd.Timestamp(start)
    return [dict(cow_id=cow, day=1, minute=t0 + pd.Timedelta(minutes=i), behavior=b, confidence=conf, basis="observed",
                 cameras="cam01", n_samples=30, picture=picture, zone=zone) for i, b in enumerate(labels)]


def test_cases_are_bouts_not_minutes():
    cases = select_cases(pd.DataFrame(minutes("C01", "2000-01-10 10:00", ["standing"] * 30, conf=0.3)),
                         max_cases=100, audit_frac=0.0)
    assert len(cases) == 1 and cases.iloc[0].n_minutes == 30 and cases.iloc[0].reason == "low_confidence"


def test_flags_low_confidence_and_bunk_standing_not_confident_lying():
    m = pd.DataFrame(minutes("C01", "2000-01-10 10:00", ["lying"] * 10, conf=0.95) +
                     minutes("C02", "2000-01-10 10:00", ["standing"] * 5, conf=0.9, zone="bunk") +
                     minutes("C03", "2000-01-10 10:00", ["walking"] * 5, conf=0.3))
    cases = select_cases(m, max_cases=100, audit_frac=0.0)
    assert set(cases.cow_id) == {"C02", "C03"}
    assert set(cases.reason) == {"bunk_not_feeding", "low_confidence"}


def test_budget_cap_lowest_confidence_first():
    rows = []
    for i in range(10):
        rows += minutes(f"C{i + 1:02d}", "2000-01-10 10:00", ["standing"] * 3, conf=0.2 + 0.05 * i)
    cases = select_cases(pd.DataFrame(rows), max_cases=4, audit_frac=0.0)
    assert list(cases.cow_id) == ["C01", "C02", "C03", "C04"] and list(cases.priority) == [1, 2, 3, 4]


def test_audit_sample_deterministic():
    m = pd.DataFrame(sum((minutes(f"C{i:02d}", f"2000-01-10 {10 + i}:00", ["lying"] * 10, conf=0.95)
                          for i in range(1, 10)), []))
    a = select_cases(m, max_cases=50, audit_frac=0.3, seed=1)
    b = select_cases(m, max_cases=50, audit_frac=0.3, seed=1)
    assert len(a) == 3 and a.equals(b) and set(a.reason) == {"audit"}


def test_priority_cows_first():
    m = pd.DataFrame(minutes("C05", "2000-01-10 10:00", ["standing"] * 3, conf=0.4) +
                     minutes("C06", "2000-01-10 10:00", ["standing"] * 3, conf=0.45))
    cases = select_cases(m, max_cases=1, audit_frac=0.0, priority_cows=["C06"])
    assert list(cases.cow_id) == ["C06"]
```

## Done when
`python scripts/verify_task.py T21` passes.
