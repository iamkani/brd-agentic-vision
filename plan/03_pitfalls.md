# 03: Pitfalls and their guards

Every item lists the mistake, why it bites on *this* dataset, and the guard that stops it: a rule, a
locked test, or a gate. Several of these were hit for real during the first analysis.

## Data and input/output

| # | Mistake | Why it bites here | Guard |
|---|---|---|---|
| 1 | Opening the AVIs with OpenCV, Ultralytics or plain `ffmpeg -i` | The private fourcc `GHEC` means "no decoder found" or 0 frames, sometimes silently | All reads go through `cattle.io.avi`. Orchestrator grep for `cv2.VideoCapture(` and `model.track(` on raw paths (§7 of `04_testing.md`). |
| 2 | Globbing `*.Avi` | It picks up macOS `._*` AppleDouble stubs (4 KB, not video) | `manifest.scan` skips names starting with `._` (T04 test). R01 reports the stub count and leaves the source volume intact. |
| 3 | Trusting AVI index flags for keyframes | Without `idx1`, as in truncated copies or fixtures, every packet looks like a keyframe | Keyframes come from HEVC NAL types 16–21. T03 test: the truncated fixture has exactly 10 keyframes, not 515. |
| 4 | Time = file-name time + frame/30 | Short files, clock jitter, midnight crossings | Per-frame clock text (T02, T03). Validators require monotonic `ts`. |
| 5 | A clock regex with `\d\d` | Stamps are not zero-padded (`9:4:52.017`, `0:0:50.307`); the first analysis lost 30% of stamps this way | T02 locked tests use real unpadded stamps |
| 6 | Day taken from the folder | A recording can cross the day boundary | `day = ts.date()` (contracts §5). T10 test crosses midnight. |
| 7 | Decoding every frame | 94 s per file ≈ 44 h for the dataset | Keyframes only (7.7 s per file). `decode_window` only for bursts and clips. |
| 8 | Writing a full-resolution JPEG per keyframe | ~264k × ~0.5 MB = 100–170 GB | `max_width ≤ 1600` or ROI crops (T06 test checks image sizes) |
| 9 | Heavy jobs on a small internal disk | Out of memory, disk full | Work output under `work_root` on the external SSD. Detection on the GPU workstation. T06 checks free space before it starts. |
| 10 | Hard-coded or unquoted paths | Mount names can contain spaces and differ across hosts | `configs/paths.yaml` plus env overrides; tests run in `tmp_path` |
| 11 | Treating ffmpeg "Broken pipe" as an error | Expected when the decoder stops after N frames and the demuxer is still writing | Judge success by the decoder's exit code and frame count, not demuxer stderr |
| 12 | Copying Drive folder names to exFAT | Organizer folder names can contain punctuation that exFAT forbids | Own layout `day<d>/cam0<n>` |

## Vision

| # | Mistake | Why it bites here | Guard |
|---|---|---|---|
| 13 | Downscaling 4000×3000 to 640 px | Cows shrink to 25–55 px | Camera ROI crop first, then `imgsz=1280`. Recall gate at M30. |
| 14 | One threshold set for day and IR night | Night images are grey, low contrast, and over half of all footage | The `picture` column drives thresholds; every metric is split day/night |
| 15 | Positions from box centres under fisheye | Zones are wrong at the image edges | Foot point (bottom-centre) plus a homography fitted around the cow area. RMSE gate on held-out points. |
| 16 | Ignoring glare and foreground steel | False boxes, bad counts | `masks` in `cameras.yaml` applied before detection. QA flags frames with > 10 cows. |
| 17 | Counting people, horses or vehicles as cows | Staff visits distort behaviour | Class map, `staff_present` flag, and those minutes excluded from risk features |
| 18 | Carrying a track through a pile-up | Identity swaps that look confident | Tracker cuts tracks at merges (T08 test) |

## Identity

| # | Mistake | Why it bites here | Guard |
|---|---|---|---|
| 19 | IDs per camera | The brief demands one ID across all cameras and both days | Cross-camera link (T13) plus closed-set assignment (T14) |
| 20 | Forcing every track onto a cow | Confident wrong IDs poison L2 | `UNK` with `min_score` and `min_margin`; basis `estimated` |
| 21 | Forcing non-overlapping fragments of one cow onto different cows | Reproduced in `cow_reid.py` with 30-s windows | T14 regression test |
| 22 | One mean prototype per cow | Gallery collapse after adding video crops (also in `cow_reid.py`) | Prototypes per (cow, camera, lighting). T14 test. |
| 23 | DINOv2 default centre-crop | Cuts the head (ear tag) and rump of wide crops | `pad_to_square` before embedding. T14 test. |
| 24 | No Day 2 anchor | Day-over-day features would compare different animals | H04 seeds include Day 2 morning. M40 gate: every cow anchored on both days. |
| 25 | Leave-one-out on near-duplicate crops reported as accuracy | Inflated numbers | Accuracy only from H05 tracklets in held-out hours |

## Behaviour labels

| # | Mistake | Why it bites here | Guard |
|---|---|---|---|
| 26 | Claiming rumination from 0.5 fps keyframes | Chewing is about 1 per second, so it can't be seen at 0.5 fps | Rules never output `ruminating_*`. Only the agent or bursts do. The brief says so. |
| 27 | "At the bunk" taken to mean feeding | Cows stand at the bunk without eating | Feeding needs head-down or an agent check. The gate flags bunk-standing bouts. |
| 28 | Mixing minute and bout logic, or no tie-break | Inconsistent CSVs | T10 locked tests (majority, priority, tiling, round trip) |
| 29 | Filling unseen minutes with lying or standing | Hides missing data | `other / not visible` plus basis. The budgets split observed, estimated and assumed. |
| 30 | Tuning thresholds on the evaluation labels | Optimistic accuracy that judges will not reproduce | `check_set.csv` split into `tune`/`test` by hour block. Evaluate on `test` only. |

## Agent

| # | Mistake | Why it bites here | Guard |
|---|---|---|---|
| 31 | Agent per minute | ~9,600 rumination-like minutes alone would exhaust the daily allowance | Gate on bouts with a hard `max_cases` (T21 tests) |
| 32 | Hallucinated evidence | Judges check that the agent "checks itself" | `commit` must cite evidence IDs produced in this case (T20 test) |
| 33 | No offline path | Live demo with no internet | `offline_fallback` commits the rule label as `estimated` (T20, T26 tests) |
| 34 | Keys or event data in traces or the repo | Compliance | Key only via environment variable; trace redaction test; `.gitignore` |
| 35 | Non-reproducible agent output | Re-runs cost tokens and change results | Reply cache keyed by model, prompt and image hashes, shipped with outputs |

## L2 risk

| # | Mistake | Why it bites here | Guard |
|---|---|---|---|
| 36 | Presenting a probability as validated, or the risk index as a probability | There are no health labels; some or all cows may be healthy | The T23 output stays an index. P(BRD) comes only from T32: labelled "model-estimated under stated assumptions", prior shown and adjustable, `basis = assumed`, interval shown. Disclaimer on cards, dashboard and brief. |
| 37 | Raw totals instead of herd-relative values | Weather and feeding times shift every cow | Robust z against pen-mates in the same hour (T23) |
| 38 | Ignoring coverage | A cow hidden more often looks inactive | Coverage weighting plus the `low_coverage` flag |
| 39 | Ignoring confounders | Estrus (mounting), lameness and staff handling mimic illness signs | Flags; reasons mention alternatives; vet review (H07) |
| 40 | Ranks without uncertainty | 10 animals over 2 days is a small sample | Bootstrap rank ranges. Injection and null tests (T23). |

## L3 live

| # | Mistake | Why it bites here | Guard |
|---|---|---|---|
| 41 | Demo depends on the API or internet | Room Wi-Fi fails | Offline mode tested (T26) |
| 42 | Clip-format surprises | Judges may hand over MP4, one camera, or under a minute | Runner accepts GHEC AVI and MP4; fixture tests |
| 43 | Alert spam | Producers ignore noisy systems | Tiers, cool-down, and an explicit "no alert" card |

## Process and packaging

| # | Mistake | Why it bites here | Guard |
|---|---|---|---|
| 44 | Workers editing tests or shared files | Silent weakening of goals; merge conflicts | Locked tests verified verbatim; ownership table in `05_dispatch.md` |
| 45 | Big-bang integration at the end | Interfaces break at 21:00 on Sunday | M20 walking skeleton; `make slice` after every merge |
| 46 | README commands never run on a clean machine | Judges can't reproduce the results | T27 test plus a fresh-clone check in H08 |
| 47 | Brief numbers that don't match outputs | Credibility | The brief takes numbers from `outputs/eval/metrics.json` (H08 checklist) |
| 48 | Submitting at 22:59 | Upload or portal failure | Freeze at 20:00, submit at 22:00 |
| 49 | Public repo with event data | Breaks the event rule | Public repository for planning documents. Use approved private channels for operational artefacts (D13). |
| 50 | Silent skips | A missing fixture makes a test skip, which looks green | `verify_task.py` fails on any skip unless `--allow-skip` |

## From prior work (`06_prior_work.md`)

| # | Mistake | Why it bites here | Guard |
|---|---|---|---|
| 51 | Treating grey night footage as thermal | It is near-infrared illumination; no camera here measures temperature | Video-DART is reported as D + A (+ R); T is "not measured" (cards, brief) |
| 52 | Clustering crops to find cow identities | Appearance clusters can merge cows with similar coats or split one cow across views | Supervised scorers from seeds only (T14) |
| 53 | Transferring accuracy from the prior deck | Different cows and cameras; likely frame-random splits | Our numbers come only from held-out hours (T19, H05, H06) |
| 54 | Using unpublished lab weights or data without permission | The brief allows free, public resources listed in the repo | Ask the organizers (H09); disclose in the README |
| 55 | Forcing "eating from ground" into standing | The deck has it as a class; the brief does not | Ask (H09); map to `feeding` only if confirmed |

## Team deliverables (state database and dashboard)

| # | Mistake | Why it bites here | Guard |
|---|---|---|---|
| 56 | A dashboard that needs internet, a paid service or a server the judges can't start | Judges score the repo, and live demos fail | Streamlit run locally (`make dashboard`), with the T25 static page as fallback |
| 57 | Shipping the full state database in the ZIP | Every keyframe gives ~250 MB | The lite database (every 15th keyframe, < 50 MB) ships; `make db` rebuilds the full one |
| 58 | The probability's prior pulled from nowhere | The number looks precise but is arbitrary | H10 sources the prior and effects; the dashboard shows them as editable assumptions |
