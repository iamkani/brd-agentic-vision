# T26: Live clip runner and alerts (L3)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M90 | P0 | Sonnet (orchestrator reviews the design) | 90 min | T03, T07, T08, T09, T14, T17, T18, T23 (T20 optional) |

- **Owns:** `cattle/live/runner.py`, `cattle/live/alert.py`.
- **Read:**
  - `02_contracts.md` §8.15, §9, §10;
  - `03_pitfalls.md` rows 41–43;
  - `01_project_definition.md` D10.
- **Fixtures:** `night_head.avi`. Use a fake detector backend and no client.

## Goal
L3 is judged on three things:
- a working live alert on a clip **the judges choose**;
- use of visible clinical signs;
- a design a producer could run on a ranch.

The runner takes any clip (a GHEC AVI or an MP4, from any camera, day or night, short or long). It runs the pipeline on it and updates each cow's risk. It then decides an alert tier and writes an alert a pen rider can act on: a text summary, a clip, and crops. **"No alert" is a valid, explained result.**

## Specification
`run_clip(path, out_dir, state_dir=None, cfg=None, detector_backend=None, client=None) -> dict`:
- **Input check.** Accept `.avi` (read with `cattle.io.avi`) or `.mp4`/`.mov` (read with ffmpeg directly). Anything else raises `ValueError`.
- **Frame rate.** Clips ≤ 2 min are decoded at 2 fps with `decode_window`. Longer clips use keyframes.
- **Mode.**
  - **Warm:** `state_dir` holds stored identity and risk for the clip's camera and time range (from the batch run). Reuse that identity.
  - **Cold:** detect and track; cows are named `unconfirmed-<n>` unless a tag read succeeds.
- **Cues and acute events.** Use `cfg["acute"]`; defaults below. Each event names the cow.
  - `down_flat`: box aspect ≥ 3.0 with no motion (≤ 0.01 pen units/s, or ≤ 2% of the image width per second when uncalibrated) for ≥ `down_min_s` (default 600).
  - `open_mouth_breathing` and `nasal_discharge`: only via `client` with a head crop. Whenever `client` is None, skip them and **always** record the warning `offline: head-sign check skipped`, even if no cow was detected.
  - `breathing_rate` (T30, if available): only on still cows in colour footage; report it with its quality.
- **Risk update.**
  - **Warm:** risk after = the stored risk adjusted by the clip's behaviour against the herd at the same hour.
  - **Cold:** `risk_before = None`, and the cues alone determine the tier.
- **Tiers.**
  - `urgent`: any acute event.
  - `check_today`: a visible clinical sign, or risk after ≥ 0.75.
  - `watch`: risk after ≥ 0.6, or a rise of ≥ 0.15.
  - `none` otherwise.
  - **Cool-down:** do not repeat the same tier for the same cow within 6 h. Record cool-downs in `state_dir/alerts_log.csv`.
- **Outputs** go to `out_dir/<run_id>/`, where `run_id = <YYYYmmddHHMMSS>-<clip stem>`:
  - `alert.json` in contracts §8.15 form, always written, including `checks_done` (a list of what was checked) and `warnings`;
  - `alert.html` when tier ≠ none: a one-screen alert with the cow, tier, reasons in plain language (English, with optional Spanish lines), an evidence clip (T17) and crops;
  - `no_alert.html` when tier = none: what was checked and that nothing reached a tier.
- **Return value:** the `alert.json` dict.

`cattle/live/alert.py`:
- `render_alert(result, out_dir)` writes the HTML files.
- `notify(result, channel="file")` supports the channels `file` (default), `mac` (`osascript` notification) and `print`. A failed notify is logged and never raises.

## Locked test
```python
# LOCKED TEST: tests/unit/test_live.py
import json

import pytest

from cattle.live.runner import run_clip

NIGHT = "tests/fixtures_private/night_head.avi"


class FakeBackend:
    def __init__(self, boxes):
        self.boxes = boxes

    def __call__(self, images):
        return [list(self.boxes) for _ in images]


def test_quiet_clip_gives_no_alert(tmp_path):
    res = run_clip(NIGHT, tmp_path, detector_backend=FakeBackend([("cow", 0.9, 100, 100, 400, 300)]), client=None)
    run = tmp_path / res["run_id"]
    assert res["tier"] == "none" and (run / "no_alert.html").exists()
    saved = json.loads((run / "alert.json").read_text())
    assert saved["mode"] == "cold" and saved["checks_done"] and saved["cows"][0]["cow_id"].startswith("unconfirmed")


def test_cow_down_flat_is_urgent(tmp_path):
    res = run_clip(NIGHT, tmp_path, detector_backend=FakeBackend([("cow", 0.9, 1000, 2000, 2000, 2250)]), client=None,
                   cfg={"acute": {"down_min_s": 10}})
    assert res["tier"] == "urgent"
    assert any("down" in e for c in res["cows"] for e in c["acute_events"])
    assert (tmp_path / res["run_id"] / "alert.html").exists()


def test_offline_records_warning(tmp_path):
    res = run_clip(NIGHT, tmp_path, detector_backend=FakeBackend([]), client=None)
    assert any("offline" in w for w in res["warnings"])


def test_rejects_unknown_file(tmp_path):
    p = tmp_path / "x.txt"
    p.write_text("hi")
    with pytest.raises(ValueError):
        run_clip(p, tmp_path)
```

## Done when
- `python scripts/verify_task.py T26` passes.
- The report lists the measured runtime on the fixture, which must be < 60 s on the M1.

## Do not
- Require the network.
- Block on a notification.
