# 07: Team split (3 people)

The split follows the pipeline, so each person owns a **lane** end-to-end: their cards, human tasks, runs and the gate for their milestones. Hand-offs between lanes go only through the files in `02_contracts.md`. The lanes' peak loads are staggered (A early, B overnight, C on Sunday), which also staggers sleep.

**Fill in names:** A = ________ (the team member with the GPU workstation and the external SSD), B = ________, C = ________. Give lane C to whoever has the most animal-science background, because it owns the veterinary wording, the probability assumptions and the brief.

## 1. Lanes

| | A: Integrator, data and vision | B: Identity, behaviour and agent | C: Analytics, product and story |
|---|---|---|---|
| **Owns milestones** | M10, M20, M30 | M40, M50, M60 | M70, M80, M90, M100 |
| **Cards to dispatch and review** | T00, T01, T02, T03, T04, T06, T07, T08, T09, T11, T12, T28 | T10, T13, T14, T15, T16, T17, T18, T19, T20, T21, T29, T30 | T05, T22, T23, T24, T25, T26, T27, T31, T32, T33 |
| **Human tasks** | H01 cameras, H02 calibration, H03 detection check, H03b auto-label picking | H04 seeds (lead), H05 identity labels, H06 check set (coordinator; all three label 100 each) | H07 vet review, H08 brief (lead), H09 organizer questions, H10 literature priors |
| **Runs** | R01 data, R02 extraction, R03/R03b detection (RTX), R04 tracking and identity (with B) | R04 identity (with A), R05 L1 and database, R06 agent and traces | R07 L2, L3, dashboard and package; finalist presentation |
| **Shared-file owner** | `02_contracts.md`, `configs/`, `Makefile`, `requirements*` (others request changes) | — | `README.md`, the brief |
| **Load** | 12 cards (mostly Haiku, front-loaded), ~4.5 h human, ~6 h runs | 12 cards, ~4.5 h human, ~4 h runs | 10 cards, ~5 h human, ~2 h runs + presentation |

## 2. Timeline per person (CST)

| Time | A | B | C |
|---|---|---|---|
| Sat 10:30–13:00 | T00 → T01, T02 → T03 → T04; H01 cameras | T10 bouts, T15 sheets; read `06_prior_work.md`; H09 draft with C | T05 validators; H09 send; H10 literature priors |
| Sat 13:00–17:00 | T06, T07, T08, T09 → T11 skeleton (**M20 gate**); R01/R02 Day 1 | T17 clips, T13 linking; H04 Day 1 seeds (from the starter set) | T22 features, T23 risk (synthetic data) |
| Sat 17:00–23:00 | H02 calibration; R03 Day 1 detection; T12; download completes → R01 full, R02 Day 2 | T14 identity, T16 ID-check, T18 rules, T19 eval | T32 probability, T31 database, T24 cards; **sleep 23:00–06:00** |
| Sat 23:00–Sun 03:00 | Start R03 Day 2 overnight; H03 detection check; **sleep 02:00–07:00** | T20 agent, T21 gate; start R04 when detections land; **sleep 03:00–08:00** | (asleep) |
| Sun 06:00–10:00 | Check R03/R04 logs; T28 + H03b only if the recall gate failed | H05 identity labels (once awake); R05 first pass | T33 dashboard, T26 live; H06 check set (100) |
| Sun 08:00–11:00 | H06 (100) | H06 (100, coordinator); R05 evaluation (**M50 gate**) | H06 (100); T25 static fallback |
| Sun 11:00–15:00 | Fix cards from gate failures; help R06 | R06 agent run, traces, ablation (**M60 gate**); T29/T30 only if ahead | R07a risk and probability on real data (**M70 gate**); state DB full and lite |
| Sun 15:00–19:00 | L3 rehearsal support; packaging dry run | Trace polish; check `activity.csv` | R07b cards, dashboard, H07 review (**M80**); R07c live tests (**M90**) |
| Sun 19:00–22:00 | Fresh-clone check (H08) | Brief sections L1 and agent | Brief lead, T27, package, submit by 22:00 (**M100**) |

## 3. Hand-offs (who waits for what)

| From → to | What | When |
|---|---|---|
| A → all | Scaffold, `verify_task.py`, contracts v1.x | Sat 11:30 |
| A → B | Keyframes and detections, Day 1 | Sat ~18:00 |
| A → B | Calibration (`configs/calib`, `zones.json`) | Sat ~19:00 |
| B → C | `cow_samples`, `activity_minutes` (rules version) | Sun ~09:00 |
| B → C | Agent-revised L1 files and traces | Sun ~13:00 |
| C → all | H09 answers (API allowed? class definitions?) | As soon as they arrive; may change B's agent run |
| C → A | Final README, brief, package for the fresh-clone check | Sun ~20:00 |

## 4. How three people run this without stepping on each other

1. **One private implementation repository.** A creates it and adds B and C as collaborators. The public planning repository is separate. Keep event data out of Git. `.gitignore` covers videos, `work/` and the private fixtures.
2. **Private fixtures and data are shared outside git.** Pass the fixtures (36 MB) by AirDrop or a private drive folder. The external SSD physically moves between machines following `tasks/RUNS.md`.
3. **Branches:** `task/Txx` for workers, merged by the lane owner after `verify_task.py` passes. Run `make slice` after every merge and pull `main` before dispatching a new card.
4. **Shared files change only through A.** Post requests in `reports/STATUS.md` or the team chat.
5. **Token windows.** Each person dispatches workers from **their own** Claude account, giving three independent 5-hour windows. Haiku-class models take most cards. Sonnet-class models take the ones marked Sonnet. Frontier-model time is spent only on lane reviews (A: T03/T11; B: T14/T20; C: T23/T32/T33/T26) and the brief.
6. **Sync points (10 min each):** Sat 13:00, Sat 17:00 (M20), Sat 23:00, Sun 08:00, Sun 13:00, Sun 17:00, Sun 20:00 (freeze). At each one, every lane owner reports one line: gate status, next gate, top risk.
