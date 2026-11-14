# 04: Testing and verification

## 1. Philosophy

- **The orchestrator states goals as tests; workers implement until the tests pass.** Checking a test result costs a few hundred tokens. Re-reading code costs thousands.
- There are three layers:
  1. **Locked tests.** Written by the orchestrator inside the task cards and installed verbatim.
  2. **Task tests.** Extra tests a worker adds, reviewed by the orchestrator.
  3. **Gates.** Checks on full data, run by people or the orchestrator (`tasks/RUNS.md`).
- **A skipped test is a failed test at a gate.**

## 2. Locked tests

- In a task card, a locked test is a fenced `python` block whose **first line** is `# LOCKED TEST: <repo path>`.
- `python scripts/verify_task.py Txx --install` writes every locked block of card Txx to its path.
- `python scripts/verify_task.py Txx` then:
  - checks every locked file is identical to its card block (line endings normalised);
  - runs `pytest -q -rs` on those files;
  - **fails on any skip**, unless `--allow-skip` is given;
  - exits 0 only if all of that passes.
- Several task IDs can be given at once (`verify_task.py T02 T03`).
- Workers never edit locked tests. If a locked test looks wrong, the worker stops and reports `BLOCKED: test`. Only the orchestrator edits the card.

## 3. Test types

### 3.1 Golden unit tests on real fixtures
These test exact facts measured from the real files, so a worker cannot satisfy them with a plausible-looking but wrong implementation.

```python
def test_keyframes_from_nal_types():
    idx = avi.index(NIGHT)
    assert list(idx.frame_idx[idx.is_key]) == [0, 19, 79, 139, 199, 259, 319, 379, 439, 499]
```

### 3.2 Invariant (property) tests
These test rules that must always hold, using generated inputs: bouts tile the day; minutes → bouts → minutes round-trips; budgets sum to 1,440; ranks form a permutation.

### 3.3 Contract tests
Every validator has one valid example and one failing example per rule. Every producer task asserts `validate(table, its_output) == []`.

### 3.4 Behaviour tests with fakes
Heavy or external parts are injected so tests stay fast and offline:
- the detector has a `backend=`;
- the agent has a `client=` (`FakeClient` with scripted JSON replies);
- the live runner takes both.

Tests check behaviour: coordinate mapping, track cuts at merges, evidence rejection, cache hits.

### 3.5 Integration (walking skeleton)
`make slice` runs every stage on the fixtures with fake backends, then validates all outputs in slice mode (coverage checked only inside the slice window).

### 3.6 Statistical acceptance tests on synthetic data
- **Injection test (T23):** plant a "sick" profile in one cow. She must rank first and stay in the top 2 across bootstrap runs.
- **Null test:** with no injection, the engine must set `no_cow_stands_out`.
- **Identity regression (T14):** two fragments of one cow get the same ID.

### 3.7 Robustness tests
Truncated files; `._` stubs; a missing clock text; an empty detection frame; night images; single-camera input; files crossing midnight.

### 3.8 Performance budgets
These are asserted loosely, only where they matter:
- `avi.index` on a 6 MB fixture: < 0.5 s.
- `iter_keyframes` on the night fixture (10 frames): < 10 s on the M1.

### 3.9 Gates on full data
See §6 and `tasks/RUNS.md`. These run on the real dataset and use human labels.

## 4. Fixtures

**Private fixtures** (event data, git-ignored) sit in `tests/fixtures_private/` and are created by T01 with `scripts/make_fixtures.py`. This edition uses example clocks and fixture expectations. The real-data gates use the independently measured private fixture reference:

| Fixture | Source | Facts |
|---|---|---|
| `night_head.avi` | Private Cam 01 night excerpt | Confirm expected keyframes, clocks, offsets and light statistics against the independently recorded private fixture manifest. |
| `day_head.avi` | Private Cam 01 day excerpt | Confirm expected keyframes, clocks, offsets and light statistics against the independently recorded private fixture manifest. |
| `cam02_day_head.avi`, `cam02_night_head.avi`, `cam03_day_head.avi`, `cam03_night_head.avi` | First 6 MB of one 10:00 and one 01:00 file per camera (T01) | Facts written to `FIXTURES.json` by T01. These are used only by task tests, not locked ones. |

- **Synthetic fixtures** are generated inside tests (pandas frames, small PIL or numpy images, ffmpeg `testsrc` clips). Shared small files go in `tests/fixtures/synthetic/`.
- **Rule:** tests never read `raw_root` or `work_root`, and never use the network.

## 5. What a good test case looks like

```python
def test_bout_split_on_basis_change():               # one behaviour per test, named after the rule
    m = minutes(["lying"] * 3, basis=["observed", "observed", "estimated"])   # Given: tiny explicit input
    b = minutes_to_bouts(m)                                                   # When: one call
    assert list(b.minutes) == [2, 1]                                          # Then: exact expected values
    assert list(b.basis) == ["observed", "estimated"]
```

| Good | Bad |
|---|---|
| Exact values from real fixtures or tiny hand-built inputs | `assert result is not None` |
| One rule per test; the name says which | One giant test that checks everything |
| Deterministic (seeded), < 5 s, no network | Depends on wall-clock time, internet, or GPU |
| Fails for the right reason (try the wrong implementation in your head) | Passes with a stub that returns an empty DataFrame |
| Tests the contract (schema and API) | Tests private helper names |

## 6. Milestone gates

| Gate | Command / evidence | Pass when |
|---|---|---|
| M10 | `python scripts/verify_task.py T00 T01 T02 T03 T04 T05` | exit 0, 0 skipped |
| M20 | `make slice` then `python -m cattle.validate submission work/slice/outputs --slice` | no errors; runtime < 30 min |
| M30 | `python -m scripts.qa_report` + `python -m scripts.evaluate --detections labels/det_check.csv` + calib RMSE in `configs/calib/*.json` | keyframes ≥ 98% of 151 × files; recall ≥ 0.90 day, ≥ 0.75 night; held-out RMSE ≤ 0.03 |
| M40 | `python -m scripts.evaluate --identity labels/id_eval.csv` + `python -m cattle.validate id_check_index outputs/id_check/index.csv` | accuracy reported (targets 0.85 day / 0.60 night); duplicate-ID-per-frame ≤ 1%; both-day anchors; 288 rows |
| M50 | `python -m cattle.validate submission outputs/` (L1 part) + `python -m scripts.evaluate --behaviour labels/check_set.csv --split test` | valid; report with per-class accuracy and 95% CI |
| M60 | `python scripts/verify_task.py T20 T21` + agent run log + `outputs/traces/selected/*.md` | tests pass; cases ≤ budget; 3 traces; ablation row in report |
| M70 | `python scripts/verify_task.py T22 T23` + validate `outputs/brd_risk.csv` | pass |
| M80 | `python scripts/verify_task.py T24 T25` + H07 sign-off | pass |
| M90 | `make live CLIP=…` on 4 clips (R07c) | each ≤ 5 min, valid output |
| M100 | D1–D16 in `01_project_definition.md` | all ticked |

## 7. How the orchestrator verifies a task (about 5 minutes, few tokens)

1. Read `reports/Txx.md` (≤ 300 words).
2. In the task worktree, run `python scripts/verify_task.py Txx`. It must exit 0 with 0 skips.
3. Run `git diff --stat main...task/Txx`. Only owned files and locked tests may change.
4. Run `python -m pytest -q tests/unit tests/contract`. Everything already merged must stay green.
5. Scan for forbidden patterns:
   ```bash
   grep -rnE 'cv2\.VideoCapture|\.track\(source|api_key *= *["'\'']' cattle scripts
   ```
   This should print nothing.
6. Merge, update the task board, and run `make slice` after any merge from M20 on.

## 8. Behaviour and identity evaluation protocol (M40, M50)

- **Check set (H06):** ≥ 300 samples, each a 10-second window for one cow.
  - 180 are random, stratified by day/night × camera.
  - 120 are targeted at rare classes (water zone, bunk, fast movement, close pairs).
  - Split `tune`/`test` by **whole hours** (no hour in both), about 1:2.
  - 10% of samples are double-labelled to measure agreement (Cohen's kappa).
- **Metrics** (on `test` only):
  - per-class precision, recall and F1;
  - overall accuracy and macro-F1, with 95% bootstrap CIs (1,000 resamples, seed 0);
  - all of the above split by day/night and by camera.
- **Identity (H05):** ≥ 100 tracklets from hours not used for seeds.
  - Report track-level accuracy, the `UNK` rate, and the duplicate-ID-per-frame rate, split by day/night.
- **Ablation (M60):** on test samples that the gate flagged, compare rules alone against rules plus agent: accuracy, n, and CI.
