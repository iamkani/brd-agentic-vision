# 01: Project definition

## 1. What we are building

We are building a reproducible pipeline that turns about 140 hours of pen video into three deliverables. The video is 3 cameras × 2 days, 10 Angus cows in Study Pen, roughly 1,750 recordings.

- **L1, 20 points.** Every cow gets one stable ID across all cameras and both days, and a behaviour label for every minute. Labels come from a cheap rule layer plus a genuine tool-using, self-checking agent for the hard cases.
- **L2, 20 points, and the first tie-breaker.** A BRD risk screen ranks the 10 cows with uncertainty. Its explanations use veterinary terms and are backed by video clips.
- **L3, 10 points.** A runner takes a clip in and puts an alert out. It updates risk, applies acute-event rules, and sends an alert a pen rider can act on, including a valid "no alert".

**Judges:** a veterinarian, an animal scientist, computer-vision and AI specialists, and a beef technologist. Ties go to the L2 score, then to a vote on the brief. The brief explicitly says a polished L1 beats a broken attempt at all three, so the milestone order follows L1 → L2 → L3.

## 2. Definition of done (project completion = 100%)

The project is complete when every item below passes. Each has a check anyone can run.

| # | Done when | Check |
|---|---|---|
| D1 | Raw data complete and verified: All expected files delivered, checksum match, `._*` stubs excluded from the manifest | The approved checksum verifier matches each camera’s delivered files and R01 confirms no stub rows in the manifest |
| D2 | `outputs/activity.csv` and `outputs/activity_minutes.csv` valid: 10 cows × 2 days × 1,440 minutes tiled, 10 classes with the exact names from the brief, a basis on every row | `python -m cattle.validate submission outputs/` |
| D3 | `outputs/time_budgets.csv` valid: each cow-day sums to 1,440 minutes, split into observed, estimated and assumed | same |
| D4 | `outputs/id_check/`: 288 images (2 days × 3 cameras × every 30 min) plus `index.csv` | same |
| D5 | `outputs/traces/selected/{success,failure,night}.md`: each step and claim cites a tool result | same, plus a human glance |
| D6 | `outputs/eval/report.md` and `metrics.json`: behaviour accuracy by class × day/night with n and 95% CI, identity accuracy, rules-vs-agent ablation | file exists; the brief quotes these numbers |
| D7 | `outputs/brd_risk.csv` valid; injection and null tests pass; rank ranges reported | `python scripts/verify_task.py T23` and the validator |
| D8 | 3 risk cards (ranks 1–3), each with 2–3 playable clips, reasons in veterinary terms with numbers, and a "veterinary review required" note | validator, plus sign-off on H07 |
| D9 | Visualization tool `outputs/viz/index.html` opens with the network off and shows all 10 cows | manual check, plus the T25 test |
| D10 | `make live CLIP=<file>` handles day, night, single-camera and never-seen clips; under 5 min per 5-min clip; works offline | log from R07 |
| D11 | README lists exact commands, models with versions and weight hashes, external code and data with licences, AI-tool disclosure, hardware and runtime, and data handling | `python scripts/verify_task.py T27` |
| D12 | Technical brief: ≤ 2 pages; every number matches `outputs/eval/metrics.json` | page count plus spot check |
| D13 | No event data or secrets anywhere public. Keep operational artefacts in approved private channels and send submission files through the official channel. | `git ls-files` review |
| D14 | The package builds and validates | `make package && make validate-submission` |
| D15 | **State database:** `outputs/pen_states_lite.sqlite` (one frame per ~30 s, < 50 MB) ships; the full `pen_states.sqlite` (every keyframe) is built by `make db`. Every annotated frame lists each animal's ID, box, pen position, zone, behaviour state, confidence and basis. | `python scripts/verify_task.py T31`; `PRAGMA integrity_check` and `PRAGMA foreign_key_check` are clean on both |
| D16 | **Prediction dashboard:** `make dashboard` (Streamlit) shows P(BRD) per cow with an interval and adjustable assumptions, a cow view, an annotated frame browser and data quality. It runs offline and is labelled "model-estimated, not a diagnosis". | `python scripts/verify_task.py T32 T33`; manual check on the real lite database |

## 3. Milestone ladder (10% increments)

Rules for the ladder:

- **Progress is the highest gate passed,** never the amount of code written.
- **Later milestones' code is built early.** It runs on fixtures and synthetic data while earlier full-data runs proceed (see `05_dispatch.md`). A milestone's *gate* needs its full-data run.

| % | Milestone | Outcome | Gate (all must pass) | Tasks / human / runs | Effort | Target |
|---|---|---|---|---|---|---|
| 10 | **M10 Data and IO** | Can read any file: clock, keyframes, light. Validators exist. Starter set (179 files) on disk. | `python scripts/verify_task.py T00 T01 T02 T03 T04 T05` all pass; manifest of downloaded files has 0 `decode_failed` | T00–T05, H01 | 6 model tasks, 0.5 h human | Sat 13:00 |
| 20 | **M20 Walking skeleton** | 1 hour × 3 cameras runs end-to-end; every submission file exists and validates, crude content allowed | `make slice` succeeds; `python -m cattle.validate submission work/slice/outputs --slice` passes; runtime < 30 min | T06–T11 | 6 model tasks, 1 h orchestrator | Sat 17:00 |
| 30 | **M30 Full-scale perception** | All files verified; ~264k keyframes; detections; calibration and zones per camera; QA report | keyframes ≥ 98% of expected (151/file); detection recall ≥ 0.90 day and ≥ 0.75 night on H03 sample; calibration RMSE ≤ 0.03 pen units on held-out points; `qa_report.md` exists | T12, H02, H03, R01–R03 | 1 h human, ~5 h compute | Sun 02:00 |
| 40 | **M40 Identity** | Tracks, cross-camera links and closed-set IDs with basis for both days; 288 ID-check frames | track-level ID accuracy reported on H05 labels (target ≥ 0.85 day, ≥ 0.60 night); duplicate-ID-per-frame ≤ 1%; every cow anchored on both days; 288 images | T13–T16, H04, H05, R04 | 2.5 h human, 1–2 h compute | Sun 08:00 |
| 50 | **M50 L1 baseline** | Rule labels for every cow-minute; L1 files; check set (≥ 300 labels) and evaluation; **state database built (full + lite)** | validator passes on L1 files; `eval/report.md` with per-class accuracy and CI; database integrity and FK checks clean; numbers logged on the board | T17–T19, T31, H06, R05 | 3 h human (split), 0.5 h compute | Sun 11:00 |
| 60 | **M60 Agent loop** | Gate selects bouts within budget; agent re-labels them with evidence; 3 traces; ablation | T20 and T21 locked tests pass; run log shows cases ≤ budget and cache hits; 3 traces rendered; ablation in eval report (agent ≥ rules on the flagged subset, or an honest note) | T20, T21, R06 | 0.5 h human, 2–4 h API | Sun 13:30 |
| 70 | **M70 Risk engine** | Hourly features, risk index 0–1, bootstrap rank ranges, reasons, flags; **assumption-based P(BRD) per cow with interval** | T22, T23 and T32 locked tests pass (incl. injection and null); `brd_risk.csv` valid on real data | T22, T23, T32, H10, R07a | 0.5 h orchestrator | Sun 15:00 |
| 80 | **M80 L2 deliverables** | Clips, 3 risk cards, **interactive prediction dashboard (Streamlit)**, static fallback, vet-language review | T24 and T33 tests pass; `make dashboard` renders every tab on the real lite database; H07 signed off | T24, T33 (T25 fallback), H07, R07b | 0.5 h human, 0.5 h compute | Sun 17:00 |
| 90 | **M90 L3 live** | Clip runner, alert tiers, acute rules, offline fallback | `make live` on 4 clips (day, night, single camera, unseen) within limits; valid alert or no-alert output each time | T26, R07c | 0.5 h rehearsal | Sun 19:00 |
| 100 | **M100 Shipped** | README, brief, package, submission | D1–D16 all ticked; submission confirmation saved | T27, H08 | 2.5 h human | Sun 22:00 |

## 4. Effort and capacity

| Resource | Available | Planned use |
|---|---|---|
| People | 3 students, ~20 effective hours each before the deadline | ~13 h for human tasks (labelling, calibration, review, brief), ~6 h for orchestration and runs. The rest is buffer and sleep. |
| Model workers | Unlimited sessions, bounded by token windows | 28 cards: ~13 Haiku-class (30–60 min each), ~15 Sonnet-class (45–90 min each). Up to 4 in parallel. |
| Orchestrator | One frontier-model session | Locks tests, reviews 3 designs (T20, T23, T26), runs gates, merges. Keep it cheap: run commands, don't re-read diffs. |
| M1 Mac | 8 cores, 8 GB RAM, limited internal disk; holds the external SSD during the download | Download, keyframe extraction (4 workers: ~1 h for everything; measured 7.7 s/file single-thread), small jobs, labelling UI |
| GPU workstation | GPU | Detection (estimate 3–4 h for ~264k frames; benchmark first in R03), embeddings |
| T7 SSD | exFAT, size and free space verified before runs | Raw video and all intermediate files (`work_root`) |
| Network | Confirm sustained download throughput | Start processing each verified partition as it becomes available. |
| Approved model API | Confirm allowance and renewal timing with the organizers | Reserve capacity for bulk review, L3 and final traces. |

## 5. Schedule and critical path (Sat 14 → Sun 15 Nov, CST)

| When | Models (build) | People | Runs |
|---|---|---|---|
| Sat 10:30–11:30 | Wave 0: T00 | H01 camera roles (starter set) | Download phase 1 |
| Sat 11:30–13:30 | Wave 1: T01, T02, T05, T09, T10 → T03 → T04 | H09 questions to organizers (optional) | Download phase 2 (Day 1) |
| Sat 13:30–17:00 | Wave 2: T06, T07, T08, T15, T17 → T11 | H04 seeds start (starter set) | R01 manifest of Day 1; R02 Day 1 extraction after Day 1 is ready |
| Sat 17:00–22:00 | Wave 3: T12, T13, T14, T16, T18, T19 | H02 calibration; H03 detection check | R03 Day 1 detection; download completes; R01 full verify; R02 Day 2 |
| Sat 22:00–Sun 03:00 | Wave 4: T20, T21, T22, T23, T24, T25, T26 (on synthetic data) | Sleep in shifts; one person watches runs | R03 Day 2 detection; R04 tracking, linking, identity |
| Sun 07:00–11:00 | Fixes from gate failures | H05 identity labels; H06 check set (3 people) | R05 L1, ID-check frames, evaluation |
| Sun 10:00–14:00 | Fixes | Trace review | R06 agent bulk run; ablation |
| Sun 12:00–17:00 | T27 | H07 vet-language review | R07a risk; R07b clips, cards, dashboard |
| Sun 14:00–19:00 | Fixes | L3 rehearsal | R07c live tests |
| Sun 18:00–20:00 | — | H08 brief | Final package |
| **Sun 20:00** | **Freeze** | | |
| Sun 20:00–22:00 | — | H08 final QA, submit | `make package && make validate-submission` |

**Critical path:** download → Day 2 extraction → detection → tracking and identity → L1 labels → features and risk → cards → brief. A second critical path is human labelling: seeds on Saturday afternoon and the check set on Sunday morning. Neither can be delegated to a model.

## 6. Checkpoints and cut lines

| Checkpoint | Must be true | If not, cut (in this order) |
|---|---|---|
| Sat 17:00 | M20 gate passed | Drop T12 and T16 polish. Use the skeleton's simple tracker for everything. |
| Sat 23:00 | Day 1 detections exist; seeds for ≥ 8 cows | Identity from continuity plus seeds only, marked `assumed`; skip appearance embeddings |
| Sun 08:00 | M40 gate passed | Day 2 night identity as `assumed`; skip cross-camera linking for Cam 0X with the worst calibration |
| Sun 12:00 | M50 gate passed | Agent only on Day 1 flagged bouts; rumination, grooming and social limited to agent-reviewed bouts |
| Sun 16:00 | M70 gate passed | Dashboard becomes a static HTML table; cards without charts |
| Sun 19:00 | M90 gate passed | L3: warm-start clips only, text alert only |
| Sun 20:00 | Freeze | No new features. Only README, brief, packaging and validator fixes. |

**Never cut:** validators, the check-set evaluation, README, brief, packaging.

**Prior-work additions (`06_prior_work.md`):**
- **T28 (detector fine-tune, P1)** runs only if the M30 recall gate fails.
- **T29 (crop classifier) and T30 (breathing rate)** are P2. They are the **first** things to drop at any checkpoint.

## 7. Risk register

| Risk | Signal | Mitigation | Owner |
|---|---|---|---|
| Download stalls or quota errors | rclone log shows errors or no progress for 10 min | Script is resume-safe; re-run `all`; ask organizers | Run owner |
| AVI decode surprises in other cameras | T01 facts differ from Cam 01 | T03 parses RIFF and NAL types itself; fixtures per camera | Orchestrator |
| Detector weak at night or under fisheye | Recall gate fails | Crop to camera ROI; larger `imgsz`; fine-tune on ~200 boxes (stretch) | CV person |
| Identity fails at night | Low accuracy on night H05 labels | Continuity across dusk, cross-camera geometry, basis `assumed`, honest brief | Orchestrator |
| Human labelling takes too long | H06 behind at Sun 10:00 | Cut to 200 samples; keep the stratification | Team lead |
| API allowance or outage | Agent errors or budget exhausted | Gate caps cases; reply cache; local fallback model; commit `estimated` | Orchestrator |
| M1 out of memory or disk | Crashes; disk < 5 GB | Heavy jobs on the GPU workstation; write only to `work_root` on the external SSD | Run owner |
| SSD needed on two machines at once | Download still running when detection should start | Day 1 extraction on the Mac while downloading; move the SSD to the GPU host after the download | Run owner |
| Scope creep | Tasks outside the board | Only the orchestrator adds tasks; cut lines | Team lead |
| Data-handling rule | Unclear whether API use is allowed | Ask organizers (H09); document; offline fallback | Team lead |

## 8. Operating principles

1. **Always shippable.** From M20 on, `make package` produces a valid ZIP. Keep the last good one.
2. **Build on fixtures, run on full data.** Code tasks never need the full dataset.
3. **Tests are the contract.** The orchestrator writes and locks them; workers implement. A skipped locked test counts as a failure at a gate.
4. **One source of truth.** `02_contracts.md` changes only through the orchestrator, who bumps its version line and notes the change on the board.
5. **Honest labels.** Every label and identity carries `observed`, `estimated` or `assumed`, and the brief reports how much of each there is.
6. **Measure before claiming.** Use the held-out check set for behaviour accuracy claims. T23 reports a relative behavioural risk index. T32 reports a separate, assumption-based P(BRD) scenario with its prior and limitations.
