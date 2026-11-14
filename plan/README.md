# BRD Agentic Vision: execution plan

This folder is the plan for Challenge A of an animal-science hackathon,
"Agentic Vision for Angus Behavior and BRD Prediction". The work is split into small, sandboxed
tasks that cheaper models (Haiku-class) or teammates can execute. A frontier model (the
**orchestrator**) sets the goals, owns the acceptance tests and verifies results by running them,
not by re-reading code.

The [Codex review](CODEX_REVIEW.md) records the launch corrections applied to this copy and the decisions still needed before pipeline dispatch.

- **Schedule:** The milestone board uses an example two-day hackathon schedule. Use the organizer’s actual submission deadline for a real run.

## Read in this order

| You are | Read |
|---|---|
| Orchestrator (frontier model or team lead) | `01` → `02` → `04` → `05`, then each task card as you dispatch it |
| Worker model | Only `templates/worker_prompt.md`. It tells you which card and which contract sections to read. |
| Teammate doing human tasks | `tasks/HUMAN.md`, then `03_pitfalls.md` |
| Whoever runs full-data jobs | `tasks/RUNS.md` |

## Files

| File | What it holds |
|---|---|
| `01_project_definition.md` | Goal, definition of done, the 10 milestones (10% → 100%), effort, schedule, checkpoints, cut lines, risks |
| `02_contracts.md` | **Single source of truth:** verified data facts, folder layout, IDs, time rules, class list, every table schema, configs, module APIs, Makefile targets |
| `03_pitfalls.md` | The obvious mistakes for this dataset and challenge, each with the guard that prevents it |
| `04_testing.md` | Test types, locked acceptance tests, fixtures and their golden values, milestone gates, how verification works |
| `05_dispatch.md` | Roles, model tiers, token budgets, the task graph in waves, file ownership, how to launch, verify and merge a worker |
| `06_prior_work.md` | Prior computer-vision work and its influence on the plan (T28–T30) |
| `07_team_split.md` | Three lanes (A data and vision, B identity, behaviour and agent, C analytics, product and story), per-person timeline, hand-offs, collaboration rules |
| `tasks/T00…T33_*.md` | 34 task cards for models. T28–T30 are P1/P2 additions from prior work. T31–T33 are the team deliverables: state database, P(BRD) model, Streamlit dashboard. |
| `tasks/HUMAN.md` | Tasks only people can do: labelling, calibration, review, brief |
| `tasks/RUNS.md` | Full-data runs on the right machine, with expected outputs and checks |
| `templates/worker_prompt.md` | The exact prompt to start a worker |
| `templates/task_report.md` | The report every task ends with |

## Milestone board

Tick a box only when its gate (see `01_project_definition.md` §3) has been run and passed.

| % | Milestone | Gate passed | Time | Notes |
|---|---|---|---|---|
| 10 | M10 Data and IO | [ ] | | |
| 20 | M20 Walking skeleton (1-hour slice, every output valid) | [ ] | | |
| 30 | M30 Full-scale perception | [ ] | | |
| 40 | M40 Identity | [ ] | | |
| 50 | M50 L1 baseline (labels, budgets, check-set evaluation) | [ ] | | |
| 60 | M60 Agent loop and traces | [ ] | | |
| 70 | M70 Risk engine | [ ] | | |
| 80 | M80 L2 deliverables (clips, cards, dashboard) | [ ] | | |
| 90 | M90 L3 live alert | [ ] | | |
| 100 | M100 Shipped | [ ] | | |

## Task board

Status values: `todo`, `doing (<who/model>)`, `review`, `done`, `blocked`. The orchestrator updates this table.

| Task | Title | M | Tier | Status |
|---|---|---|---|---|
| T00 | Repo scaffold, configs, verify harness | 10 | Sonnet | todo |
| T01 | Private fixtures from real files | 10 | Haiku (data machine) | todo |
| T02 | Clock-text parser | 10 | Haiku | todo |
| T03 | AVI reader (RIFF index, NAL keyframes, clock, decode) | 10 | Sonnet | todo |
| T04 | Manifest scanner and light probe | 10 | Haiku | todo |
| T05 | Schema validators and CLI | 10 | Haiku | todo |
| T06 | Keyframe extraction (parallel, resumable) | 20 | Sonnet | todo |
| T07 | Detector wrapper | 20 | Sonnet | todo |
| T08 | Tracker for 0.5 fps keyframes | 20 | Sonnet | todo |
| T09 | Calibration, pen coordinates, zones | 20 | Haiku | todo |
| T10 | Minutes, bouts, time budgets | 20 | Haiku | todo |
| T11 | Walking skeleton integration | 20 | Sonnet | todo |
| T12 | Data-quality report | 30 | Haiku | todo |
| T13 | Cross-camera linking | 40 | Sonnet | todo |
| T14 | Closed-set identity assignment | 40 | Sonnet | todo |
| T15 | Labelling sheets | 40 | Haiku | todo |
| T16 | ID-check frames | 40 | Haiku | todo |
| T17 | Evidence clip cutter | 50 | Haiku | todo |
| T18 | Behaviour cues and rule labels | 50 | Sonnet | todo |
| T19 | Evaluation and report | 50 | Haiku | todo |
| T20 | Agent core | 60 | Sonnet (+ orchestrator review) | todo |
| T21 | Gate and budget planner | 60 | Haiku | todo |
| T22 | Behaviour features | 70 | Sonnet | todo |
| T23 | Risk engine | 70 | Sonnet (+ orchestrator review) | todo |
| T24 | Risk cards | 80 | Haiku | todo |
| T25 | Offline dashboard | 80 | Sonnet | todo |
| T26 | Live clip runner and alerts | 90 | Sonnet | todo |
| T27 | README, disclosures, submission validator, package | 100 | Haiku | todo |
| T28 | Detector fine-tune data and trainer (P1) | 30 | Sonnet | todo |
| T29 | Crop behaviour classifier, second opinion (P2) | 50/60 | Sonnet | todo |
| T30 | Breathing rate from bursts (P2) | 70/90 | Haiku | todo |
| T31 | State database: every animal's state per annotated frame | 50 | Sonnet | todo |
| T32 | Assumption-based P(BRD) per animal | 70 | Sonnet (+ review) | todo |
| T33 | Interactive prediction dashboard (Streamlit) | 80 | Sonnet | todo |

## Ground rules (short version)

1. **Tests are the contract.** Locked tests live inside the task cards. Workers install them with `scripts/verify_task.py` and must not edit them.
2. **Build on fixtures, run on full data.** Workers never touch the raw recordings. Full runs follow `tasks/RUNS.md`.
3. **Always shippable from M20 on.** `make package && make validate-submission` must work at every milestone. Keep the last good ZIP.
4. **Observed, estimated or assumed** is recorded on every label and every identity.
5. **No accuracy claim without the check set.**
6. **Event data stays private.** This public repository shares the planning documents. Keep event data, event-derived media, private fixtures, and secrets out of public repositories and unapproved services.
