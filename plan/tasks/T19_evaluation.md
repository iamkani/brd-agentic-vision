# T19: Evaluation and report

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M50 | P0 | Haiku | 60 min | T05, T10 |

- **Owns:** `cattle/eval/metrics.py`, `cattle/eval/report.py`, `scripts/evaluate.py`.
- **Read:** `02_contracts.md` §8.6, §8.8, §8.16; `04_testing.md` §8.
- **Fixtures:** none.

## Goal
Every accuracy number in the brief comes from here, computed only on the held-out `test` split, with sample sizes and 95% confidence intervals. The judges score "behavior accuracy" and "correct cow identification". This module makes our claims checkable.

## Specification
`cattle/eval/metrics.py`:
- **`evaluate_behavior(pred_minutes, labels) -> dict`.**
  - Inputs: `pred_minutes` has columns `cow_id, minute, behavior`; `labels` has `cow_id, minute, behavior, split`, plus optionally `picture` and `cam`. Behaviors are either output labels or codes; normalise both to codes.
  - Keep only `split == "test"`, and join on (`cow_id`, `minute`).
  - Return keys:
    - `n`, `accuracy`, `macro_f1`;
    - `per_class` (code → `precision`, `recall`, `f1`, `n`);
    - `confusion` (true code → pred code → count);
    - `by_picture` (same metrics per `picture` when present);
    - `ci` (`accuracy_lo`, `accuracy_hi` from `bootstrap_ci`).
- **`evaluate_identity(identity, labels) -> dict`.** Joins on `gtrack_id` and returns `n`, `accuracy` (UNK counts as wrong), `unk_rate`, `ci`, and `by_picture` when present.
- **`bootstrap_ci(correct, n=1000, seed=0) -> (lo, hi)`.** Percentile 2.5 and 97.5 of resampled means, using `np.random.default_rng(seed)`.
- **`agreement(labels_a, labels_b) -> dict`.** Cohen's kappa and percent agreement on double-labelled samples.

`cattle/eval/report.py`:
- **`write_report(metrics, out_dir)`** writes `metrics.json` (machine-readable, read by the brief) and `report.md`.
- `report.md` has the sections `## Behaviour`, `## Identity`, `## Agent ablation` (when present) and `## Notes`.

`scripts/evaluate.py`:
- **Flags:** `--behaviour labels/check_set.csv --pred outputs/activity_minutes.csv --split test`; `--identity labels/id_eval.csv --pred <work>/identity.parquet`; `--detections labels/det_check.csv --pred <work>/detections.parquet`; `--ablation --rules-pred A --agent-pred B`; `--out outputs/eval`.
- **Detection check:** count recall is the mean of `min(pred, true) / true` per frame, split by picture.

## Locked test
```python
# LOCKED TEST: tests/unit/test_eval.py
import numpy as np
import pandas as pd
import pytest

from cattle.eval.metrics import bootstrap_ci, evaluate_behavior, evaluate_identity


def frame(beh, split="test"):
    return pd.DataFrame(dict(cow_id="C01", minute=pd.date_range("2000-01-10 10:00", periods=len(beh), freq="min"),
                             behavior=beh, split=split, picture="colour"))


def test_behaviour_metrics_exact():
    labels = frame(["lying", "lying", "feeding", "feeding", "walking", "standing"])
    pred = labels[["cow_id", "minute"]].assign(behavior=["lying", "standing", "feeding", "feeding", "walking", "standing"])
    m = evaluate_behavior(pred, labels)
    assert m["n"] == 6 and m["accuracy"] == pytest.approx(5 / 6)
    assert m["per_class"]["lying"]["recall"] == pytest.approx(0.5)
    assert m["per_class"]["standing"]["precision"] == pytest.approx(0.5)
    assert m["confusion"]["lying"]["standing"] == 1


def test_output_labels_and_codes_both_accepted():
    labels = frame(["other / not visible", "ruminating (lying)"])
    pred = labels[["cow_id", "minute"]].assign(behavior=["other", "ruminating_lying"])
    assert evaluate_behavior(pred, labels)["accuracy"] == 1.0


def test_only_test_split_scored():
    labels = frame(["lying", "lying"])
    labels["split"] = ["tune", "test"]
    pred = labels[["cow_id", "minute"]].assign(behavior=["walking", "lying"])
    m = evaluate_behavior(pred, labels)
    assert m["n"] == 1 and m["accuracy"] == 1.0


def test_bootstrap_ci_deterministic():
    x = np.array([1] * 80 + [0] * 20)
    lo, hi = bootstrap_ci(x, n=1000, seed=0)
    assert (lo, hi) == bootstrap_ci(x, n=1000, seed=0)
    assert 0.70 < lo < 0.76 and 0.84 < hi < 0.90


def test_identity_metrics():
    ident = pd.DataFrame(dict(gtrack_id=["g1", "g2", "g3", "g4"], cow_id=["C01", "C02", "UNK", "C04"]))
    labels = pd.DataFrame(dict(gtrack_id=["g1", "g2", "g3", "g4"], cow_id=["C01", "C03", "C05", "C04"]))
    m = evaluate_identity(ident, labels)
    assert m["n"] == 4 and m["accuracy"] == pytest.approx(0.5) and m["unk_rate"] == pytest.approx(0.25)
```

## Done when
`python scripts/verify_task.py T19` passes.
