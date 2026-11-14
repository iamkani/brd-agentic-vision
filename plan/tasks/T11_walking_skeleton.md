# T11: Walking skeleton (1-hour slice, every output valid)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M20 | P0 | Sonnet (orchestrator reviews) | 90 min | T06, T07, T08, T09, T10 (and T05) |

- **Owns:** `scripts/run_slice.py`, `tests/integration/test_slice.py`.
- **Read:**
  - `02_contracts.md` §8 (all submission files) and §11;
  - `01_project_definition.md` §3 (M20 row);
  - the public APIs of T03, T05–T10.
- **Fixtures:** `day_head.avi`, `night_head.avi`.

## Goal
Prove that every stage connects before any stage is good. The skeleton runs end-to-end on a small slice and writes **every** submission file in schema-valid form, even though the content is crude. Later tasks replace the stand-ins one by one; `make slice` must keep passing after every merge.

## Specification
`scripts/run_slice.py` has two modes.

**`--fixtures --fake-detector --out DIR`** (used by the locked test):
1. Treat `tests/fixtures_private/day_head.avi` as cam01 (09:04:51–09:05:06).
2. Run `extract_file` (`max_width=800`).
3. Run `Detector` with a built-in deterministic fake backend: two cows drifting slowly, one standing (aspect 1.4), one lying (aspect 2.5).
4. Run `track`, then `add_pen_coords` with an identity-like placeholder H for cam01 and the zones from `configs/zones.json`.
5. **Stand-in identity:** tracks in order of first appearance become C01, C02, … with basis `assumed`.
6. **Stand-in labels:**
   - box aspect > 1.9 → `lying`;
   - otherwise `standing`;
   - not visible → `other`;
   - basis `estimated`.
7. Build `cow_samples` on 2-second bins.
8. Run `write_l1` restricted to the slice window, for all 10 cows.
9. **Placeholder risk:** score 0.5 for every cow, ranks by cow ID, `rank_low = 1`, `rank_high = 10`, reason `"insufficient data (slice)"`, flag `no_cow_stands_out`.
10. **ID-check index:** an empty `id_check/index.csv` with the header only, because no :00 or :30 slot falls in the window.
11. Write `DIR/outputs/...` and `DIR/slice_window.json` (`{"start": ISO, "end": ISO}`, whole minutes covering the clip), then run `validate_submission(DIR/outputs, slice_window=...)` and exit 1 on errors.

**`--raw-root ... --day 1 --hour 10 --cams cam01,cam02,cam03 --out DIR [--weights yolo11s.pt]`** (used in R-runs):
- the same pipeline on one real hour with the real detector;
- the stand-ins are replaced automatically by real modules when they exist (T13, T14, T16, T18).

Each stage prints one line with its row count and seconds.

## Locked test
```python
# LOCKED TEST: tests/integration/test_slice.py
import json
import subprocess
import sys

import pandas as pd

from cattle.validate.schemas import validate_submission


def test_slice_end_to_end_on_fixtures(tmp_path):
    r = subprocess.run([sys.executable, "-m", "scripts.run_slice", "--fixtures", "--fake-detector", "--out", str(tmp_path)],
                       capture_output=True, text=True, timeout=900)
    assert r.returncode == 0, r.stdout[-2000:] + r.stderr[-2000:]
    out = tmp_path / "outputs"
    for name in ["activity.csv", "activity_minutes.csv", "time_budgets.csv", "brd_risk.csv", "id_check/index.csv"]:
        assert (out / name).exists(), name
    win = json.loads((tmp_path / "slice_window.json").read_text())
    window = (pd.Timestamp(win["start"]), pd.Timestamp(win["end"]))
    assert window[0] <= pd.Timestamp("2000-01-10 09:04:52") < window[1]
    errors = validate_submission(out, slice_window=window)
    assert errors == [], errors[:10]
    act = pd.read_csv(out / "activity.csv")
    assert set(act.cow_id) == {f"C{i:02d}" for i in range(1, 11)}
```

## Done when
- `python scripts/verify_task.py T11` passes.
- The report asks the orchestrator to wire `slice:` in the Makefile to run the fixture mode.

## Do not
- Edit other tasks' modules. If an API is missing or awkward, write it under Requests.
