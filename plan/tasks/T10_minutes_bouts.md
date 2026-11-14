# T10: Minutes, bouts, time budgets (L1 writer)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M20 | P0 | Haiku | 60 min | T05 |

- **Owns:** `cattle/behavior/bouts.py`, `scripts/write_l1.py`.
- **Read:**
  - `02_contracts.md` §5, §6, §7, §8.7–§8.10, and the bouts block of §10;
  - `03_pitfalls.md` rows 6, 28 and 29.
- **Fixtures:** none.

## Goal
Turn per-cow 2-second samples into the three L1 submission files: `activity_minutes.csv`, `activity.csv` (bouts) and `time_budgets.csv`. Every cow must get every minute of both days, with the exact class names and an honest basis.

## Specification
Input samples follow contracts §8.7. Only these columns are needed: `cow_id`, `ts`, `label` (code), `confidence`, `basis`, `cams`, and `weight_s` (default 2.0 when absent).

- **`samples_to_minutes(samples)`** groups by (`cow_id`, `minute = ts.floor("min")`):
  - **behavior:** the code with the most seconds (sum of `weight_s`); ties go to higher priority (`PRIORITY`); output the **label** (`CODE_TO_LABEL`).
  - **confidence:** winner seconds ÷ total seconds × mean confidence of the winner's samples.
  - **basis:** the winner samples' basis with the most seconds. A tie resolves conservatively: assumed > estimated > observed.
  - **cameras:** the sorted union of all `cams` entries (split on `;`), joined with `;`.
  - **n_samples:** the number of samples.
  - **day:** 1 or 2 from the minute's date. Minutes outside 2000-01-10..08 are dropped.
  - **Columns:** `cow_id, day, minute, behavior, confidence, basis, cameras, n_samples`.
- **`fill_minutes(minutes, cows, days)`** adds every missing minute as `other / not visible`, confidence 1.0, basis `observed`, cameras `""`, n_samples 0.
- **`minutes_to_bouts(minutes)`** merges consecutive minutes per (cow, day) with the same behavior **and** basis.
  - Columns: `cow_id, day, start, end, behavior, minutes, confidence (mean), basis, cameras (union), trace_ids`.
  - `trace_ids` is the union of the minutes' `trace_id` values if that column exists, else `""`.
- **`bouts_to_minutes(bouts)`** is the exact inverse expansion. Each minute copies its bout's confidence, basis and cameras; `n_samples` is 0.
- **`time_budgets(minutes)`** follows contracts §8.10: all 10 labels per cow-day, zeros included, with the observed, estimated and assumed split.
- **`scripts/write_l1.py --samples PATH --out DIR [--cows ...] [--days 1,2]`**:
  - writes the three CSVs, with ISO minute strings;
  - validates each with `cattle.validate`;
  - exits 1 on any error.

## Locked test
```python
# LOCKED TEST: tests/unit/test_bouts.py
import pandas as pd
import pytest

from cattle.behavior.bouts import bouts_to_minutes, fill_minutes, minutes_to_bouts, samples_to_minutes, time_budgets
from cattle.validate.schemas import validate


def run(cow, start, n, code, basis="observed", conf=0.9, cams="cam01", step=2):
    t0 = pd.Timestamp(start)
    return [dict(cow_id=cow, ts=t0 + pd.Timedelta(seconds=i * step), label=code, confidence=conf, basis=basis,
                 cams=cams, weight_s=2.0) for i in range(n)]


def test_majority_by_seconds_and_confidence():
    s = pd.DataFrame(run("C01", "2000-01-10 10:00:00", 20, "lying") + run("C01", "2000-01-10 10:00:40", 10, "standing"))
    m = samples_to_minutes(s)
    assert list(m.behavior) == ["lying"] and list(m.day) == [1]
    assert m.minute.iloc[0] == pd.Timestamp("2000-01-10 10:00")
    assert m.confidence.iloc[0] == pytest.approx(0.9 * 40 / 60)
    assert m.n_samples.iloc[0] == 30


def test_priority_breaks_ties():
    s = pd.DataFrame(run("C01", "2000-01-10 10:00:00", 15, "standing") + run("C01", "2000-01-10 10:00:30", 15, "feeding"))
    assert list(samples_to_minutes(s).behavior) == ["feeding"]


def test_basis_tie_is_conservative():
    s = pd.DataFrame(run("C01", "2000-01-10 10:00:00", 15, "lying", "observed") +
                     run("C01", "2000-01-10 10:00:30", 15, "lying", "estimated"))
    assert list(samples_to_minutes(s).basis) == ["estimated"]


def test_cameras_union():
    s = pd.DataFrame(run("C01", "2000-01-10 10:00:00", 15, "lying", cams="cam03") +
                     run("C01", "2000-01-10 10:00:30", 15, "lying", cams="cam01;cam02"))
    assert samples_to_minutes(s).cameras.iloc[0] == "cam01;cam02;cam03"


def test_midnight_and_out_of_range_days():
    s = pd.DataFrame(run("C01", "2000-01-10 23:59:30", 30, "lying") + run("C01", "2000-01-12 00:00:10", 5, "lying"))
    m = samples_to_minutes(s)
    assert sorted(set(m.day)) == [1, 2]
    assert m[m.day == 2].minute.min() == pd.Timestamp("2000-01-11 00:00")
    assert m.minute.max() < pd.Timestamp("2000-01-12")


@pytest.fixture
def filled():
    s = pd.DataFrame(run("C01", "2000-01-10 10:00:00", 30, "feeding"))
    return fill_minutes(samples_to_minutes(s), cows=["C01", "C02"], days=[1])


def test_fill_minutes(filled):
    assert len(filled) == 2 * 1440
    assert validate("activity_minutes", filled, cows=["C01", "C02"], days=[1]) == []
    gap = filled[filled.cow_id == "C02"].iloc[0]
    assert gap.behavior == "other / not visible" and gap.basis == "observed" and gap.n_samples == 0


def test_bouts_tile_the_day(filled):
    b = minutes_to_bouts(filled)
    assert validate("activity", b, cows=["C01", "C02"], days=[1]) == []
    c1 = b[b.cow_id == "C01"].sort_values("start")
    assert list(c1.behavior) == ["other / not visible", "feeding", "other / not visible"]
    assert list(c1.minutes) == [600, 1, 839]
    assert len(b[b.cow_id == "C02"]) == 1


def test_bouts_split_on_basis_change():
    s = pd.DataFrame(run("C01", "2000-01-10 10:00:00", 30, "lying", "observed") +
                     run("C01", "2000-01-10 10:01:00", 30, "lying", "estimated"))
    b = minutes_to_bouts(fill_minutes(samples_to_minutes(s), cows=["C01"], days=[1]))
    lying = b[b.behavior == "lying"].sort_values("start")
    assert list(lying.basis) == ["observed", "estimated"] and list(lying.minutes) == [1, 1]


def test_round_trip(filled):
    back = bouts_to_minutes(minutes_to_bouts(filled))
    a = filled.sort_values(["cow_id", "minute"]).reset_index(drop=True)
    b = back.sort_values(["cow_id", "minute"]).reset_index(drop=True)
    assert len(a) == len(b) and (a.behavior == b.behavior).all() and (a.basis == b.basis).all()


def test_time_budgets(filled):
    tb = time_budgets(filled)
    assert validate("time_budgets", tb, cows=["C01", "C02"], days=[1]) == []
    one = tb[(tb.cow_id == "C01") & (tb.behavior == "feeding")].iloc[0]
    assert one.minutes == 1 and one.minutes_observed == 1
    assert tb[tb.cow_id == "C01"].minutes.sum() == 1440
```

## Done when
`python scripts/verify_task.py T10` passes.

## Do not
- Interpolate labels across gaps here. Gap handling belongs in T18. This module only aggregates.
