# 05: Dispatch: running the plan with distributed, cheaper workers

## 1. Roles

| Role | Who | Does | Does not |
|---|---|---|---|
| Orchestrator | Frontier model session (e.g. Opus-class) or the team lead | Owns `02_contracts.md` and the locked tests. Dispatches cards, verifies (§4), merges, runs gates, updates the boards. Reviews designs of T20, T23 and T26. | Write feature code, re-read whole diffs, or run long jobs in its own context |
| Worker | Haiku-class or Sonnet-class model in an isolated worktree | Exactly one card: install locked tests, implement owned files, pass `verify_task.py`, commit, report | Touch files it doesn't own, use the network or raw data, edit tests, add dependencies |
| Human | The 3 team members | `tasks/HUMAN.md` (labelling, calibration, review, brief); `tasks/RUNS.md` on the right machine | Hand-edit outputs instead of fixing code |

With three people, each person owns a lane (cards, human tasks, runs, gates) and acts as orchestrator for that lane. Shared files stay with A. See `07_team_split.md`.

## 2. Model tiers and budgets

| Tier | Example model IDs | Use for | Time box | Input budget |
|---|---|---|---|---|
| Haiku-class | `claude-haiku-4-5`, `claude-haiku-5-5` | Pure functions with golden tests, CLIs over existing APIs, HTML/CSV writers, validators, reports | 30–60 min | ≤ 150k tokens |
| Sonnet-class | `claude-sonnet-5-5` | Algorithms with fakes, ffmpeg or Ultralytics integration, trackers, identity, agent loop, live runner | 45–90 min | ≤ 300k tokens |
| Frontier | `claude-opus-5-5` | Contracts, locked tests, design review of T20/T23/T26, gate decisions, brief help | — | Keep each review ≤ 10k tokens |

If a Haiku worker reports `BLOCKED` twice on the same card, re-dispatch it to a Sonnet worker with the report attached. Never loop the same tier more than twice.

## 3. Worker protocol (copied into `templates/worker_prompt.md`)

Lane A follows the bootstrap steps in T00 before the first worker dispatch. T01 uses the data-machine procedure in its card. Run feature CLIs as modules from the current worktree root. The standalone verifier keeps its documented file invocation.

1. Read the card, then the contract sections the card lists, then only the files under **Read**.
2. Run `python scripts/verify_task.py Txx --install`, then `git add` the locked tests and commit them with the message `Txx: locked tests`.
3. Implement only the files under **Owns**.
4. Loop: run `python scripts/verify_task.py Txx`, fix, and repeat until exit 0. Also keep `python -m pytest -q tests/unit tests/contract` green.
5. Stop conditions:
   - **3 failed attempts on the same error** → `BLOCKED`.
   - **Time box reached** → `PARTIAL`.
   - **A locked test looks wrong** → `BLOCKED: test`.
6. Commit only owned files, with the message `Txx: <summary>`.
7. Write `reports/Txx.md` from `templates/task_report.md` (≤ 300 words). Put any change needed outside **Owns** under **Requests**.

## 4. Orchestrator protocol per task

1. **Dispatch.** Create the worktree (§7), copy private fixtures if the card needs them, start the worker with the prompt, and set the board to `doing`.
2. **Verify.** Run the 6 steps in `04_testing.md` §7. Never read the full diff unless a step fails.
3. **Merge.** Run `git merge --no-ff task/Txx`, apply any **Requests** yourself (Makefile targets, configs), set the board to `done`, and from M20 on run `make slice`.
4. **On failure.** Re-dispatch with the report and the failing output, at the same tier once, then at a higher tier.
5. **Gate.** When all tasks of a milestone are `done` and its runs are complete, run the gate and tick the milestone board.

## 5. Task graph (waves)

Arrows show hard dependencies. Within a wave, tasks run in parallel; keep at most 4–5 workers alive.

```
Wave 0  T00 scaffold
Wave 1  T01 fixtures · T02 clock · T05 validators · T09 calibration · T10 bouts
        T03 avi        (needs T01, T02)
        T04 manifest   (needs T03)
Wave 2  T06 extract    (T03, T04, T05) · T07 detect (T05) · T08 track (T05)
        T15 sheets     (T05) · T17 clips (T03)
        T11 skeleton   (T06, T07, T08, T09, T10)           → M20 gate
Wave 3  T12 qa (T05) · T13 link (T08, T09) · T14 identity (T13) · T16 id-check (T03, T05)
        T18 rules (T09, T10) · T19 eval (T05, T10) · T28 detector fine-tune data (T06, T07)  [P1]
Wave 4  T20 agent (T03, T05, T17) · T21 gate (T05, T10) · T22 features (T10)
        T23 risk (T22) · T24 cards (T17, T23) · T25 dashboard (T23)
        T26 live (T03, T07, T08, T09, T14, T17, T18, T23; T20 optional)
        T29 crop classifier (T18, T19) [P2] · T30 breathing rate (T03) [P2]
        T31 state database (T10, T13, T14, T18) · T32 P(BRD) (T23) · T33 dashboard (T31, T32)
Wave 5  T27 ship (everything)
```

**Building is not running.** Wave 3 and Wave 4 code is built and tested on fixtures while full-data runs (R01–R04) proceed. Their gates are passed later, when `tasks/RUNS.md` produces real outputs.

## 6. File ownership

Shared files are owned by the orchestrator: `Makefile`, `configs/*`, `requirements*`, `pyproject.toml`, `cattle/constants.py`, `cattle/config.py`, `plan/*`. Workers request changes to them; they do not edit them.

| Task | Owns |
|---|---|
| T00 | everything in the scaffold, then hands the shared files to the orchestrator |
| T01 | `scripts/make_fixtures.py`, `tests/fixtures_private/*` (local only), `tests/unit/test_fixtures.py` |
| T02 | `cattle/io/clock.py` |
| T03 | `cattle/io/avi.py` |
| T04 | `cattle/io/manifest.py`, `cattle/quality/light.py`, `scripts/build_manifest.py` |
| T05 | `cattle/validate/schemas.py`, `cattle/validate/__main__.py` |
| T06 | `cattle/extract/keyframes.py`, `scripts/extract_keyframes.py` |
| T07 | `cattle/perception/detect.py`, `scripts/detect.py` |
| T08 | `cattle/perception/track.py` |
| T09 | `cattle/calib/homography.py`, `cattle/calib/zones.py`, `scripts/click_points.py` |
| T10 | `cattle/behavior/bouts.py`, `scripts/write_l1.py` |
| T11 | `scripts/run_slice.py`, `tests/integration/test_slice.py` |
| T12 | `scripts/qa_report.py` |
| T13 | `cattle/identity/link.py` |
| T14 | `cattle/identity/assign.py`, `cattle/identity/embed.py` |
| T15 | `cattle/tools/sheets.py`, `scripts/make_sheet.py` |
| T16 | `scripts/id_check_frames.py`, `cattle/tools/idcheck.py` |
| T17 | `cattle/io/clips.py` |
| T18 | `cattle/behavior/cues.py`, `cattle/behavior/rules.py` |
| T19 | `cattle/eval/metrics.py`, `cattle/eval/report.py`, `scripts/evaluate.py` |
| T20 | `cattle/agent/client.py`, `tools.py`, `loop.py`, `cache.py`, `trace.py`, `scripts/run_agent.py` |
| T21 | `cattle/agent/gate.py` |
| T22 | `cattle/analytics/features.py` |
| T23 | `cattle/analytics/risk.py` |
| T24 | `cattle/analytics/cards.py`, `cattle/analytics/templates/*` |
| T25 | `viz/*`, `scripts/build_viz.py` |
| T26 | `cattle/live/runner.py`, `cattle/live/alert.py` |
| T27 | `README.md`, `scripts/validate_submission.py`, `scripts/package.py`, `docs/*` |
| T28 | `cattle/perception/yolo_data.py`, `scripts/sample_frames.py`, `scripts/autolabel.py`, `scripts/train_detector.py` |
| T29 | `cattle/behavior/crop_classifier.py` |
| T30 | `cattle/signals/__init__.py`, `cattle/signals/breathing.py` |
| T31 | `cattle/db/__init__.py`, `cattle/db/state_db.py`, `scripts/build_state_db.py` |
| T32 | `cattle/analytics/brd_prob.py` |
| T33 | `app/__init__.py`, `app/logic.py`, `app/dashboard.py` |

Every task also owns its locked test files and any extra tests under `tests/` whose names start with its module name.

## 7. Launching a worker

T00 creates the repo, the shared dependency-only venv and pytest `pythonpath = ["."]`. From the current worktree root, use `python -m scripts.<name>` for feature CLIs so they import that worktree’s `cattle/` package in both interactive runs and subprocesses.

```bash
# Set IMPLEMENTATION_ROOT to the implementation checkout before dispatch.
cd "${IMPLEMENTATION_ROOT:?Set IMPLEMENTATION_ROOT}"
git worktree add ../wt/T02 -b task/T02 main
cp -R tests/fixtures_private ../wt/T02/tests/       # only if the card says it needs private fixtures
cd ../wt/T02 && source "$IMPLEMENTATION_ROOT/.venv/bin/activate"
export PYTHON="$(command -v python)"
claude --model claude-haiku-4-5 -p "$(sed 's/{{TASK}}/T02/g' plan/templates/worker_prompt.md)"
```

- **Alternative:** in the Claude desktop app, start a new session in `../wt/T02`, pick a Haiku or Sonnet model, and paste `templates/worker_prompt.md` with `{{TASK}}` replaced.
- **Sandbox expectations:** no network beyond the model API itself, no `raw_root`, no package installs. If a card needs a package that is not in `requirements.txt`, the worker requests it and the orchestrator decides.
- **Cleanup after merge:** `git worktree remove ../wt/T02 && git branch -d task/T02`.

## 8. Token-saving rules (for everyone)

- Workers read the card plus the listed contract sections plus the listed files. They never browse the repo.
- Never print more than 50 lines. Use `pytest -q`, `head`, and `tail`. Never print whole DataFrames or JSON files.
- Never open videos or images in a model context except in human tasks. Tests assert numbers.
- The orchestrator verifies by exit codes and `--stat` diffs, not by reading code.
- Reports stay ≤ 300 words. Failures quote only the last 20 lines of output.
- Re-dispatch with the failing test name and its error, not the whole history.

## 9. Escalation

| Situation | Action |
|---|---|
| Locked test seems wrong | The orchestrator checks the card against `04_testing.md` §4 facts. If the test is wrong, fix the card, re-install, and log it in the contracts change log. |
| Contract gap (a field or API is missing) | The orchestrator amends `02_contracts.md` (bump the version), updates affected cards, and notifies running workers |
| Two tasks need the same file | Split the file, or the orchestrator applies the change after both merge |
| Gate fails on full data | Find the stage with the QA report, open a fix card `Txx-fix-<n>` with a locked test reproducing the failure on a small extract, and dispatch it |
| Behind schedule at a checkpoint | Apply the cut lines in `01_project_definition.md` §6. Never cut validators, evaluation, README, brief or packaging. |

## 10. Status reporting

After each merge, the orchestrator updates the boards in `plan/README.md` in a single line per task. At every checkpoint it also writes 3 lines in `reports/STATUS.md`: milestone reached, next gate, top risk.
