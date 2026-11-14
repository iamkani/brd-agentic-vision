# T05: Schema validators and CLI

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M10 | P0 | Haiku | 60 min | T00 |

- **Owns:** `cattle/validate/schemas.py`, `cattle/validate/__main__.py`.
- **Read:** `02_contracts.md` §4, §5, §6, §7, §8 (all tables), §10 (validate block).
- **Fixtures:** none.

## Goal
One validator per table, so every producer can prove its output is correct, and so the submission can be checked with one command. Validators return a list of human-readable error strings. An empty list means valid.

## Specification
- **`validate(table, df, **kw) -> list[str]`.**
  - Tables: `manifest`, `keyframes`, `detections`, `tracks`, `gtracks`, `identity`, `cow_samples`, `activity_minutes`, `activity`, `time_budgets`, `risk`, `id_check_index`, `check_set`, `features`.
  - An unknown table name raises `KeyError`.
  - Keyword arguments: `cows` (default `COWS`), `days` (default `[1, 2]`), `window` (a `(start, end)` pair of Timestamps: check coverage only inside it, for slices), `require_clips` (default `True`).
- **Generic checks for every table:**
  - Required columns, reported as `missing column <name>`.
  - Enums: `cam`, `cow_id` (including `UNK` where the contract allows it), `behavior` (output labels), `label` (codes), `basis`, `picture`, `cls`, `zone`.
  - `confidence` and `coverage` in [0, 1].
  - Parse datetime columns from either datetime dtype or ISO strings.
  - Treat NaN as empty string in text columns.
- **Specific checks:**
  - **`activity`:**
    - `start < end`, both on whole minutes;
    - `minutes == (end - start) / 60`;
    - `start` falls on the bout's day;
    - per (cow, day), sorted bouts tile 00:00 → next 00:00 (or the `window`), reporting `gap`, `overlap` or `coverage` errors;
    - every (cow, day) in `cows × days` present, else `coverage`.
  - **`activity_minutes`:** `(cow_id, minute)` unique (`duplicate`); exactly 1,440 minutes per cow-day, or the window length (`coverage`).
  - **`time_budgets`:**
    - all 10 behaviours per cow-day;
    - `minutes` sum to 1,440 (or the window length);
    - `fraction == minutes / total` within 1e-6;
    - `minutes_observed + minutes_estimated + minutes_assumed == minutes`.
  - **`risk`:**
    - one row per cow;
    - `risk_score` in [0, 1];
    - `rank` is a permutation of 1..n;
    - `1 <= rank_low <= rank <= rank_high <= n`;
    - `top_reasons` non-empty (`reason`);
    - with `require_clips`, ranks 1–3 have 2–3 `evidence_clips` (`clip`).
  - **`detections`:** `x1 < x2`, `y1 < y2`, all inside [0, 4000] × [0, 3000]; `cls` in {cow, person, horse, vehicle}.
  - **`keyframes`:** `img_path` like `keyframes/<cam>/<file_id>/<5 digits>.jpg`; `scale` in (0, 1]; `day` in {1, 2}.
  - **`id_check_index`:** without a window, 288 rows = 2 days × 3 cams × 48 slots `HH:MM` at :00/:30. Placeholder rows for slots with no footage are valid: empty `file_id`, `frame_idx = -1`, NaT `frame_ts`, `n_boxes = 0`.
  - **Extra columns** beyond the required ones are allowed in every table.
- **`validate_submission(outputs_dir, slice_window=None)`:**
  - Checks `activity.csv`, `activity_minutes.csv`, `time_budgets.csv`, `brd_risk.csv` and `id_check/index.csv`.
  - In full mode it also checks for `traces/selected/{success,failure,night}.md`, `eval/report.md`, `risk_cards/` (3 HTML files) and `viz/index.html`.
  - In slice mode it uses the window and `require_clips=False`.
- **CLI `python -m cattle.validate`:**
  - `<table> <path> [--cows C01,C02] [--days 1,2] [--no-clips]`, or `submission <dir> [--slice START END]`.
  - Reads CSV or Parquet by extension.
  - Prints `OK` or up to 30 errors. Exit code 0 or 1.

## Locked test
```python
# LOCKED TEST: tests/contract/test_schemas.py
import subprocess
import sys

import pandas as pd
import pytest

from cattle.constants import CLASSES
from cattle.validate.schemas import validate

COLS = ["cow_id", "day", "start", "end", "behavior", "minutes", "confidence", "basis", "cameras", "trace_ids"]


def bouts_for(cow):
    return [[cow, 1, "2000-01-10T00:00", "2000-01-10T06:00", "lying", 360, 0.8, "observed", "cam01", ""],
            [cow, 1, "2000-01-10T06:00", "2000-01-10T06:30", "feeding", 30, 0.9, "observed", "cam02;cam03", ""],
            [cow, 1, "2000-01-10T06:30", "2000-01-11T00:00", "other / not visible", 1050, 1.0, "estimated", "", ""]]


def frame(rows):
    return pd.DataFrame(rows, columns=COLS)


def errs(rows, **kw):
    kw.setdefault("cows", ["C01"])
    kw.setdefault("days", [1])
    return " | ".join(validate("activity", frame(rows), **kw)).lower()


def test_valid_bouts():
    assert validate("activity", frame(bouts_for("C01") + bouts_for("C02")), cows=["C01", "C02"], days=[1]) == []


def test_unknown_behavior():
    rows = bouts_for("C01"); rows[1][4] = "eating"
    assert "behavior" in errs(rows)


def test_unknown_basis():
    rows = bouts_for("C01"); rows[0][7] = "guessed"
    assert "basis" in errs(rows)


def test_gap_detected():
    rows = bouts_for("C01"); rows[1][2] = "2000-01-10T06:10"; rows[1][5] = 20
    assert "gap" in errs(rows)


def test_overlap_detected():
    rows = bouts_for("C01"); rows[1][2] = "2000-01-10T05:50"; rows[1][5] = 40
    assert "overlap" in errs(rows)


def test_minutes_must_match_span():
    rows = bouts_for("C01"); rows[0][5] = 300
    assert "minutes" in errs(rows)


def test_missing_cow_day_is_coverage():
    assert "coverage" in errs(bouts_for("C01"), cows=["C01", "C02"])


def test_unknown_cow():
    assert "cow" in errs(bouts_for("C11"))


def test_missing_column():
    df = frame(bouts_for("C01")).drop(columns=["basis"])
    assert "missing column" in " ".join(validate("activity", df, cows=["C01"], days=[1])).lower()


def test_confidence_range():
    rows = bouts_for("C01"); rows[0][6] = 1.5
    assert "confidence" in errs(rows)


def test_window_mode():
    rows = [["C01", 1, "2000-01-10T10:00", "2000-01-10T10:30", "lying", 30, 0.7, "observed", "cam01", ""],
            ["C01", 1, "2000-01-10T10:30", "2000-01-10T11:00", "standing", 30, 0.7, "observed", "cam01", ""]]
    win = (pd.Timestamp("2000-01-10 10:00"), pd.Timestamp("2000-01-10 11:00"))
    assert validate("activity", frame(rows), cows=["C01"], days=[1], window=win) == []
    assert validate("activity", frame(rows), cows=["C01"], days=[1]) != []


def minutes_frame(cow):
    mins = pd.date_range("2000-01-10", periods=1440, freq="min")
    return pd.DataFrame([dict(cow_id=cow, day=1, minute=m, behavior="lying", confidence=0.8, basis="observed",
                              cameras="cam01", n_samples=30) for m in mins])


def test_minutes_valid_duplicate_and_coverage():
    m = minutes_frame("C01")
    assert validate("activity_minutes", m, cows=["C01"], days=[1]) == []
    assert "duplicate" in " ".join(validate("activity_minutes", pd.concat([m, m.iloc[[5]]]), cows=["C01"], days=[1])).lower()
    assert "coverage" in " ".join(validate("activity_minutes", m.iloc[:-1], cows=["C01"], days=[1])).lower()


def budgets():
    rows = []
    for code, label in CLASSES:
        mins = 1440 if code == "lying" else 0
        rows.append(dict(cow_id="C01", day=1, behavior=label, minutes=mins, fraction=mins / 1440,
                         minutes_observed=mins, minutes_estimated=0, minutes_assumed=0))
    return pd.DataFrame(rows)


def test_time_budgets():
    assert validate("time_budgets", budgets(), cows=["C01"], days=[1]) == []
    bad = budgets(); bad.loc[0, "minutes"] = 5
    assert validate("time_budgets", bad, cows=["C01"], days=[1]) != []


def risk_frame():
    cows = [f"C{i:02d}" for i in range(1, 11)]
    return pd.DataFrame(dict(cow_id=cows, risk_score=[round(1 - i / 10, 2) for i in range(10)], rank=list(range(1, 11)),
                             rank_low=[max(1, r - 1) for r in range(1, 11)], rank_high=[min(10, r + 1) for r in range(1, 11)],
                             top_reasons=["feeding time -30% vs herd"] * 10, evidence_clips=[""] * 10,
                             coverage=[0.9] * 10, flags=[""] * 10))


def test_risk_rules():
    assert validate("risk", risk_frame(), require_clips=False) == []
    r = risk_frame(); r.loc[1, "rank"] = 1
    assert "rank" in " ".join(validate("risk", r, require_clips=False)).lower()
    r = risk_frame(); r.loc[0, "risk_score"] = 1.2
    assert "risk_score" in " ".join(validate("risk", r, require_clips=False)).lower()
    r = risk_frame(); r.loc[3, "top_reasons"] = ""
    assert "reason" in " ".join(validate("risk", r, require_clips=False)).lower()
    r = risk_frame(); r.loc[4, "rank_low"] = 7
    assert "rank_low" in " ".join(validate("risk", r, require_clips=False)).lower()
    assert "clip" in " ".join(validate("risk", risk_frame(), require_clips=True)).lower()


def test_detections_rules():
    good = pd.DataFrame([dict(file_id="f", cam="cam01", frame_idx=0, ts=pd.Timestamp("2000-01-10 10:00"), det_idx=0,
                              cls="cow", conf=0.9, x1=10.0, y1=20.0, x2=200.0, y2=150.0, model="m")])
    assert validate("detections", good) == []
    bad = good.copy(); bad.loc[0, "cls"] = "dog"
    assert validate("detections", bad) != []
    bad = good.copy(); bad.loc[0, "x2"] = 5.0
    assert validate("detections", bad) != []


def test_unknown_table_raises():
    with pytest.raises(KeyError):
        validate("nope", pd.DataFrame())


def test_cli_exit_codes(tmp_path):
    p = tmp_path / "risk.csv"
    risk_frame().to_csv(p, index=False)
    ok = subprocess.run([sys.executable, "-m", "cattle.validate", "risk", str(p), "--no-clips"], capture_output=True, text=True)
    assert ok.returncode == 0, ok.stdout + ok.stderr
    bad = risk_frame(); bad.loc[1, "rank"] = 1
    bad.to_csv(p, index=False)
    assert subprocess.run([sys.executable, "-m", "cattle.validate", "risk", str(p), "--no-clips"],
                          capture_output=True).returncode == 1
```

## Done when
`python scripts/verify_task.py T05` passes.

## Do not
- Stop at the first error. Collect all errors (cap at 200) so producers can fix them in one go.
