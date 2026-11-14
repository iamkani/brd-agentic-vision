# T29: Crop behaviour classifier (second opinion)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M50/M60 | P2 (stretch) | Sonnet | 60 min | T18, T19 |

- **Owns:** `cattle/behavior/crop_classifier.py`.
- **Read:** `06_prior_work.md` §2 item 4; `02_contracts.md` §6, §7, §8.7; the public API of T18.
- **Fixtures:** none (synthetic embeddings).

## Goal
The prior work motivates testing a crop classifier for common activity classes. Train a small classifier on embeddings of labelled cow crops. It gives a learned **second opinion** on posture, feeding and drinking. The gate (T21) flags bouts where rules and classifier disagree. The classifier only replaces a rule where it beats that rule on the check set's **tune** split (never test).

## Specification
- **`class CropClassifier(classes, min_conf=0.6, C=1.0)`.** A scikit-learn `LogisticRegression` (multinomial) on L2-normalised embedding vectors, wrapped in `CalibratedClassifierCV(method="sigmoid", cv=3)` when ≥ 30 samples exist per class.
  - **`fit(X, y)`** where y holds class codes. Raises `ValueError` if a class in `classes` has fewer than 5 samples.
  - **`predict(X) -> list[str]`** returns `"unknown"` when the max probability is < `min_conf`.
  - **`predict_proba(X) -> ndarray`.**
  - **`save(path)` / `load(path)`** use joblib.
- **`add_second_opinion(samples, clf, embeddings) -> DataFrame`.**
  - Adds `clf_label` and `clf_conf` per sample, where an embedding exists.
  - Adds `disagree` = (`clf_label` ∉ {`label`, `"unknown"`}) and `clf_conf` ≥ 0.8.
- **Training data:** crops from `labels/check_set.csv` rows with `split == "tune"` (frame at the sample midpoint), plus optional extra labels. Embeddings come from T14's `embed_crops` on the GPU workstation. Never use `test` rows.

## Locked test
```python
# LOCKED TEST: tests/unit/test_crop_classifier.py
import numpy as np
import pandas as pd
import pytest

from cattle.behavior.crop_classifier import CropClassifier, add_second_opinion

CLASSES = ["lying", "standing", "feeding"]


def blobs(n_per=60, seed=0, spread=0.15):
    rng = np.random.default_rng(seed)
    centers = np.eye(3, 16) * 3
    X = np.vstack([c + rng.normal(0, spread, (n_per, 16)) for c in centers])
    y = sum(([c] * n_per for c in CLASSES), [])
    return X, y


def test_fit_predict_accuracy():
    X, y = blobs()
    clf = CropClassifier(CLASSES).fit(X, y)
    Xt, yt = blobs(seed=1)
    assert np.mean(np.array(clf.predict(Xt)) == np.array(yt)) >= 0.95


def test_abstains_when_unsure():
    X, y = blobs()
    clf = CropClassifier(CLASSES, min_conf=0.9).fit(X, y)
    middle = np.full((1, 16), 1.0)
    assert clf.predict(middle) == ["unknown"]


def test_too_few_samples_raises():
    X, y = blobs(n_per=3)
    with pytest.raises(ValueError):
        CropClassifier(CLASSES).fit(X, y)


def test_save_load_round_trip(tmp_path):
    X, y = blobs()
    clf = CropClassifier(CLASSES).fit(X, y)
    clf.save(tmp_path / "m.joblib")
    again = CropClassifier.load(tmp_path / "m.joblib")
    assert again.predict(X[:5]) == clf.predict(X[:5])


def test_second_opinion_flags_confident_disagreement():
    X, y = blobs()
    clf = CropClassifier(CLASSES).fit(X, y)
    samples = pd.DataFrame(dict(sample_id=[0, 1], label=["standing", "lying"]))
    emb = {0: X[0], 1: X[1]}
    out = add_second_opinion(samples, clf, emb)
    assert list(out.clf_label) == ["lying", "lying"]
    assert list(out.disagree) == [True, False]
```

## Done when
`python scripts/verify_task.py T29` passes.

## Do not
- Train on `split == "test"`.
- Report classifier accuracy from training data.
