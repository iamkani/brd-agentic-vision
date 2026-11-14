# Full-data runs (R01–R07)

Runs execute merged code on the real data, on the right machine. They are slow, so start each in a terminal you can watch, under `caffeinate -i` on the Mac. **Every run ends with a check.** If the check fails, open a fix card (`05_dispatch.md` §9). Do not patch outputs by hand.

**Environment** (Mac; adjust the paths on the PC):
```bash
cd "${IMPLEMENTATION_ROOT:?Set IMPLEMENTATION_ROOT}" && source .venv/bin/activate
: "${CATTLE_RAW_ROOT:?Set CATTLE_RAW_ROOT}" "${CATTLE_WORK_ROOT:?Set CATTLE_WORK_ROOT}"
: "${CATTLE_MANIFEST:?Set CATTLE_MANIFEST to the approved checksum manifest}"
export CATTLE_RAW_ROOT CATTLE_WORK_ROOT
```

| Run | Machine | Needs | Time (estimate) | Feeds |
|---|---|---|---|---|
| R01 Data completion and manifest | Mac | Download complete, T04 | 15 min | everything |
| R02 Keyframe extraction | Mac (4 workers) or PC | T06, H01 ROIs | ~1 h for all | R03 |
| R03 Detection (+ R03b fine-tune) | GPU workstation (fallback: Mac, `mps`) | T07, R02 | Benchmark first; est. 3–4 h | R04, H03 |
| R04 Tracking, linking, identity | PC or Mac | T08, T09, T13, T14, H02, H04 | 1–2 h | R05, H05 |
| R05 L1 outputs and evaluation | Mac | T10, T16, T18, T19, H06 | 30 min | M50 |
| R06 Agent run, traces, ablation | Any (needs API) | T20, T21, `configs/agent.yaml` | 2–4 h of API time | M60 |
| R07 L2, L3, package | Mac | T22–T27 | 1 h + L3 rehearsal | M70–M100 |

**Shared storage.** Do Day 1 extraction once its files are verified. Confirm the download and extraction are complete before transferring the drive to the GPU workstation. Bring it back for R05–R07.

---

## R01 Data completion and manifest
```bash
bash "${CATTLE_VERIFY_SCRIPT:?Set CATTLE_VERIFY_SCRIPT to the approved checksum verifier}" verify
find "$CATTLE_RAW_ROOT" -name '._*' -type f | wc -l       # count macOS AppleDouble stubs
python -m scripts.build_manifest --drive-manifest "$CATTLE_MANIFEST" --workers 4
python -m cattle.validate manifest "$CATTLE_WORK_ROOT/manifest.parquet"
```
**Check:**
- The row counts match the approved delivery manifest for the processed days and cameras;
- 0 `decode_failed`;
- picture counts are about 50% grey_IR;
- every `md5` is filled and the manifest contains no `._*` entries.

## R02 Keyframe extraction
```bash
python -m scripts.extract_keyframes --day 1 --workers 4 --max-width 1600 --use-roi
# After R01 has produced the complete manifest, reuse the Day 1 cache and publish both days.
python -m scripts.extract_keyframes --workers 4 --max-width 1600 --use-roi
python -m cattle.validate keyframes "$CATTLE_WORK_ROOT/keyframes.parquet"
```
**Check:**
- rows ≥ 98% of 151 × files, with both days represented. Check reconciliation against the source manifest by folder day and camera, accounting for shortened files and the analysis window. Require unique `(file_id, frame_idx)` keys and hand the published combined index to R03;
- `keyframes_errors.csv` is empty or explained;
- free space on the external SSD is still > 100 GB.

## R03 Detection
```bash
python -m scripts.detect --benchmark 500 --weights models/yolo11s.pt --imgsz 1280 --device 0     # note frames/s
python -m scripts.detect --weights models/yolo11s.pt --imgsz 1280 --batch 16 --device 0 --resume
python -m scripts.qa_report --work "$CATTLE_WORK_ROOT" --out reports/qa_day1.md
```
Then H03 labels `labels/det_check.csv`, and `python -m scripts.evaluate --detections labels/det_check.csv --pred "$CATTLE_WORK_ROOT/detections.parquet"`.

**Check (M30 gate):** count recall ≥ 0.90 by day and ≥ 0.75 at night. If not, run R03b.

**R03b fine-tune (T28, P1):**
1. `python -m scripts.sample_frames --n 400`
2. `python -m scripts.autolabel`, then H03b picks the good labels
3. `python -m scripts.train_detector --epochs 60 --imgsz 1280`
4. Re-run detection with `best.pt` and re-check recall. Keep whichever model scores better on `det_check`.

## R04 Tracking, linking, identity
```bash
make track && make link && make identify        # wired by the orchestrator to T08, T09+T13, T14
python -m scripts.id_check_frames --work "$CATTLE_WORK_ROOT" --out outputs/id_check
```
Then H05 labels identity, and `python -m scripts.evaluate --identity labels/id_eval.csv`.

**Check (M40 gate):**
- accuracy reported for day and night;
- duplicate-ID-per-frame ≤ 1%;
- every cow anchored on both days;
- 288 ID-check images.

## R05 L1 outputs and evaluation
```bash
make label && make l1 && make idcheck
python -m scripts.evaluate --behaviour labels/check_set.csv --pred outputs/activity_minutes.csv --split test --out outputs/eval
python -m cattle.validate submission outputs/
```
Then build the state database:
```bash
python -m scripts.build_state_db --work "$CATTLE_WORK_ROOT" --outputs outputs --every-n 1 --out "$CATTLE_WORK_ROOT/pen_states.sqlite"
python -m scripts.build_state_db --work "$CATTLE_WORK_ROOT" --outputs outputs --every-n 15 --out outputs/pen_states_lite.sqlite
```

**Check (M50 gate):**
- the L1 files validate;
- `outputs/eval/report.md` holds per-class accuracy with CIs;
- both databases pass the integrity and FK checks, and the lite one is < 50 MB.

**Record the numbers on the board.** Re-run the database build after R06, so it includes the agent labels.

## R06 Agent run, traces, ablation
1. Fill `base_url` and `model` in `configs/agent.yaml` (after the H09 answer). Export `AGENT_API_KEY` in your shell; never write it to a file.
2. Run:
   ```bash
   python -m scripts.run_agent --cases "$CATTLE_WORK_ROOT/agent_cases.parquet" --budget-cases 1500 --resume
   python -m scripts.run_agent --select-traces
   make l1 && python -m scripts.evaluate --ablation --rules-pred outputs/eval/rules_minutes.csv --agent-pred outputs/activity_minutes.csv
   ```

**Check (M60 gate):**
- cases ≤ budget;
- cache hit rate is reported;
- 3 traces are rendered;
- an ablation row is in the report.

**Timing.** Confirm the approved provider’s allowance before starting. Reserve capacity for L3 and reruns.

## R07 L2, L3 and package
```bash
make features && make risk && make prob && make db && make clips && make cards && make viz   # M70, M80
CATTLE_DB=outputs/pen_states_lite.sqlite make dashboard                     # check every tab, then Ctrl-C
make live CLIP="$DAY_CLIP"     # approved daytime test clip
make live CLIP="$NIGHT_CLIP"   # approved night test clip
make live CLIP="$MP4_CLIP"     # approved single-camera MP4
make live CLIP="$UNSEEN_CLIP"  # approved clip held out of development
make validate-submission && make package
```
**Check:**
- **M70:** `brd_risk.csv` is valid, and the injection and null tests still pass.
- **M80:** H07 has signed off.
- **M90:** each live run finishes in ≤ 5 min with a valid alert or no-alert output, including with Wi-Fi off.
- **M100:** see H08.
