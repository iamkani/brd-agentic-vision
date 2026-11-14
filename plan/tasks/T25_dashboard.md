# T25: Offline dashboard (the L2 visualization tool)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M80 | P2 (static fallback for T33) | Haiku | 45 min | T23 |

> **v1.2 note:** the main visualization tool is now the interactive Streamlit dashboard (T33). This static page is the double-click fallback for anyone who cannot start Streamlit.

- **Owns:** `viz/` (templates and assets), `scripts/build_viz.py`.
- **Read:** `02_contracts.md` §8.9, §8.10, §8.14; `01_project_definition.md` §6 (cut lines: a static table is the fallback).
- **Fixtures:** none.

## Goal
The brief asks for a "visualization tool associated with a brd_risk.csv". Build a single HTML file that opens by double-click, works **offline**, and needs no server. It shows:
- the risk ranking with uncertainty bars;
- per-cow time budgets for day 1 against day 2;
- a per-cow day timeline (ethogram strip) built from the bouts;
- links to the risk cards and clips.

## Specification
`scripts/build_viz.py --outputs DIR`:
- **Reads:** `DIR/brd_risk.csv`, `DIR/time_budgets.csv` and `DIR/activity.csv`. If present, also reads `DIR/eval/metrics.json`.
- **Writes `DIR/viz/data.json`** with keys:
  - `cows` (list of IDs);
  - `risk` (rows);
  - `budgets` (rows);
  - `bouts` (rows: cow, day, start minute index, end minute index, behaviour, basis);
  - `meta` (generated time, number of observed, estimated and assumed minutes).
- **Writes `DIR/viz/index.html`:**
  - The data is **embedded inline** (`<script>const DATA = …</script>`), because browsers block `fetch` of local files over `file://`.
  - Plain JS and inline SVG; no external libraries or URLs.
  - Sections:
    1. Ranking table: rank with range, index, reasons, flags, card link.
    2. Uncertainty plot: a horizontal bar from rank_low to rank_high per cow.
    3. Per-cow time-budget bars, day 1 against day 2.
    4. Ethogram strips, 1,440 px per day, coloured by behaviour, with estimated and assumed parts hatched.
    5. A legend, plus a footer with the "not a diagnosis, veterinary review" sentence.
- The output must stay under 5 MB. Downsample the bouts if needed.

## Locked test
```python
# LOCKED TEST: tests/unit/test_viz.py
import json
import subprocess
import sys

import pandas as pd


def test_build_viz_offline(tmp_path):
    out = tmp_path / "outputs"
    out.mkdir()
    cows = [f"C{i:02d}" for i in range(1, 11)]
    pd.DataFrame(dict(cow_id=cows, risk_score=[1 - i / 10 for i in range(10)], rank=range(1, 11), rank_low=range(1, 11),
                      rank_high=range(1, 11), top_reasons="x", evidence_clips="", coverage=0.9, flags="")
                 ).to_csv(out / "brd_risk.csv", index=False)
    pd.DataFrame([dict(cow_id=c, day=d, behavior="lying", minutes=1440, fraction=1.0, minutes_observed=1440,
                       minutes_estimated=0, minutes_assumed=0) for c in cows for d in (1, 2)]
                 ).to_csv(out / "time_budgets.csv", index=False)
    pd.DataFrame([dict(cow_id=c, day=1, start="2000-01-10T00:00", end="2000-01-11T00:00", behavior="lying", minutes=1440,
                       confidence=0.9, basis="observed", cameras="cam01", trace_ids="") for c in cows]
                 ).to_csv(out / "activity.csv", index=False)
    r = subprocess.run([sys.executable, "-m", "scripts.build_viz", "--outputs", str(out)], capture_output=True, text=True)
    assert r.returncode == 0, r.stderr[-1500:]
    html = (out / "viz" / "index.html").read_text()
    data = json.loads((out / "viz" / "data.json").read_text())
    assert len(data["cows"]) == 10 and {"risk", "budgets", "bouts", "meta"} <= set(data)
    assert "http://" not in html and "https://" not in html
    assert "const DATA" in html and "C10" in html
    assert (out / "viz" / "index.html").stat().st_size < 5_000_000
```

## Done when
- `python scripts/verify_task.py T25` passes.
- A human opens the page once with Wi-Fi off (part of H07).
