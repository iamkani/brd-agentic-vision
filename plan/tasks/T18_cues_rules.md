# T18: Behaviour cues and rule labels

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M50 | P0 | Sonnet | 90 min | T09, T10 (and T13, T14 for real runs) |

- **Owns:** `cattle/behavior/cues.py`, `cattle/behavior/rules.py`.
- **Read:**
  - `02_contracts.md` §6, §7, §8.4–§8.7, §9 (rules.yaml), and the cues/rules block of §10;
  - `03_pitfalls.md` rows 26–30.
- **Fixtures:** none (synthetic samples).

## Goal
The cheap, transparent first layer of L1. Turn tracked, identified boxes into per-cow 2-second samples with cues, then label them with explicit rules. Rules handle the easy, visible classes: lying, standing, walking, feeding, drinking and not visible. They **never** claim rumination, grooming or social interaction; those come only from the agent (T20) or bursts. Rules also mark night or weak-cue labels as `estimated`.

## Specification
`cattle/behavior/cues.py`:

`build_cow_samples(tracks, gtracks, identity, bin_s=2.0, keyframes=None) -> DataFrame` follows contracts §8.7, plus 3 extra columns: `picture`, `box_aspect` (width ÷ height of the best box) and `posture_conf`.
1. **Join.** Tracks (with pen coordinates) join gtracks, then identity, giving `cow_id` per row. Rows with `UNK` are dropped.
2. **Pick the best observation.** Bin rows by `ts.floor("2s")`. Per (cow, bin), keep the largest box, preferring colour over grey and calibrated cameras.
3. **Fill fields.**
   - `cams` = the sorted, distinct cameras that saw her in the bin.
   - `posture` = `lying` if `box_aspect > lying_aspect`; `standing` if ≤ `lying_aspect`; else `unknown`.
   - `posture_conf` = 0.9 when the aspect is more than 15% away from the threshold, else 0.6.
   - `head_down` = NA unless a later model fills it.
   - `nn_dist` = distance to the nearest other cow in the same bin (pen units; NaN if alone or uncalibrated).
   - `staff_present` = whether any `person`, `horse` or `vehicle` detection exists in the bin on any camera (pass detections via `keyframes=None, detections=None` keyword arguments if needed).
4. **Not visible.** Every (cow, bin) where the cow is not seen but the camera **has footage** gets a row with `visible = False`.

`cattle/behavior/rules.py`:

`label_rules(samples, cfg=None) -> DataFrame` adds `label`, `confidence`, `basis` and `source="rule"`. `cfg` defaults to `configs/rules.yaml`. Rules are applied in order:
1. `visible == False` → `other`, confidence 1.0, basis `observed`.
2. Zone `water` and `head_down` True → `drinking`.
3. Zone `bunk` and `head_down` True → `feeding`. Zone `bunk` with `head_down` NA and posture standing → `feeding` with confidence 0.5 and basis `estimated`, so the agent gate flags it.
4. `speed > walk_speed` → `walking`.
5. Posture `lying` → `lying`; posture `standing` → `standing`; unknown → `other` with basis `estimated`.

Confidence starts at `posture_conf`, or 0.8 for zone-based labels.
- **At night** (`picture == "grey_IR"`): subtract `night_conf_penalty`, and set basis to `estimated` whenever the result is < 0.7.
- **By day:** basis is `observed` unless a rule above says otherwise.

## Locked test
```python
# LOCKED TEST: tests/unit/test_rules.py
import pandas as pd

from cattle.behavior.rules import label_rules


def s(**kw):
    base = dict(cow_id="C01", ts=pd.Timestamp("2000-01-10 10:00"), day=1, visible=True, cams="cam01", pen_x=0.5,
                pen_y=0.5, zone="open", posture="standing", speed=0.0, head_down=False, nn_dist=0.2,
                staff_present=False, picture="colour", box_aspect=1.4, posture_conf=0.9)
    base.update(kw)
    return base


CASES = [
    (s(zone="bunk", head_down=True), "feeding"),
    (s(zone="water", head_down=True), "drinking"),
    (s(speed=0.08), "walking"),
    (s(posture="lying", box_aspect=2.6), "lying"),
    (s(), "standing"),
    (s(visible=False, posture="unknown"), "other"),
    (s(zone="bunk", head_down=False), "standing"),
]


def test_rule_labels():
    out = label_rules(pd.DataFrame([c for c, _ in CASES]))
    assert list(out.label) == [label for _, label in CASES]
    assert set(out.source) == {"rule"}


def test_rules_never_claim_rumination_grooming_social():
    rows = [s(posture="lying", box_aspect=2.6, nn_dist=0.01), s(nn_dist=0.01)] * 50
    labels = set(label_rules(pd.DataFrame(rows)).label)
    assert not labels & {"ruminating_lying", "ruminating_standing", "grooming", "social"}


def test_night_lowers_confidence_and_marks_estimated():
    day = label_rules(pd.DataFrame([s()])).iloc[0]
    night = label_rules(pd.DataFrame([s(picture="grey_IR", posture_conf=0.6)])).iloc[0]
    assert night.confidence < day.confidence and night.basis == "estimated" and day.basis == "observed"


def test_not_visible_is_observed_absence():
    r = label_rules(pd.DataFrame([s(visible=False, posture="unknown")])).iloc[0]
    assert r.label == "other" and r.basis == "observed" and r.confidence == 1.0


def test_bunk_without_head_cue_is_estimated_feeding():
    r = label_rules(pd.DataFrame([s(zone="bunk", head_down=pd.NA)])).iloc[0]
    assert r.label == "feeding" and r.basis == "estimated" and r.confidence <= 0.5
```

## Done when
`python scripts/verify_task.py T18` passes. Ask the orchestrator to add `picture`, `box_aspect` and `posture_conf` to contracts §8.7 (this is already planned as version 1.1).

## Do not
- Tune thresholds on the check set's `test` split.
- Add rumination heuristics from keyframes.
