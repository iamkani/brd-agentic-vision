# Human tasks (H01–H10)

These need eyes, judgement or the organizers, so no model can do them. Every output goes to the exact path and columns in `02_contracts.md` §8.16, so code can read it. Where possible, two people split a labelling task, and 10% of items are double-labelled so agreement can be measured.

| Task | When | Time | Who | Unblocks |
|---|---|---|---|---|
| H01 Camera roles, ROI, masks | Now (starter set is on disk) | 20–30 min | 1 | T06 `--use-roi`, T07 masks, H02 |
| H02 Calibration points and zones | Sat 17:00 (needs T09) | 60 min | 1 | T13 linking, T18 zones, M30 gate |
| H03 Detection check (100 frames) | After R03 Day 1 | 30 min | 1 | M30 gate |
| H03b Pick good auto-labels (optional, T28) | After R03b auto-label | 30–45 min | 1–2 | Fine-tuned detector |
| H04 Identity seeds (Day 1, Day 2 anchor, dusk) | Sat 16:00 (Day 1); Sun 07:00 (Day 2) | 90 min | 2 | T14 identity, M40 |
| H05 Identity evaluation labels | After R04 | 45 min | 1 | M40 gate |
| H06 Check set (300 samples) | Sun 07:00–10:00 | 3 × 60 min | 3 | M50 gate, agent ablation |
| H07 Veterinary-language review | Sun 16:00 | 30 min | 1 | M80 gate |
| H08 Brief, final QA, submission | Sun 18:00–22:00 | 2–2.5 h | 2 | M100 |
| H09 Questions to organizers | Today | 15 min | 1 | Data-handling decisions |
| H10 Literature priors for P(BRD) | Sat afternoon | 45 min | 1 (lane C) | `configs/prob.yaml`, the brief |

---

## H01 Camera roles, ROI and masks
1. **Dump one daytime frame per camera** (Day 1, 10:00). This works before any code exists. "Broken pipe" messages are expected.
   ```bash
   cd "${IMPLEMENTATION_ROOT:?Set IMPLEMENTATION_ROOT}"
   mkdir -p work
   for c in cam01 cam02 cam03; do
     F=$(find "${CATTLE_RAW_ROOT:?Set CATTLE_RAW_ROOT}/day1/$c" -type f -name 'Event*.Avi' | LC_ALL=C sort | head -1)
     # Confirm the selected file is a daytime clip before using it.
     ffmpeg -v error -i "$F" -map 0:v:0 -c copy -f data - 2>/dev/null \
       | ffmpeg -v error -f hevc -r 30 -i - -frames:v 1 -vf scale=1600:-2 "work/${c}_day.jpg"
   done
   ```
2. **Assign roles.** Compare the three images with `$CATTLE_RAW_ROOT/pen_map.png`:
   - **Back cam** sits in the back-left corner and looks down the pen toward the feed bunk.
   - **Front cams 1 and 2** sit at the bunk and look back.
   - Hint: Cam 01's night view looks end-on along the pen, with the chute's steel frame in the foreground.
3. **Read pixel coordinates** in each 1600-px image. Multiply by 2.5 to get full resolution.
   - **ROI:** one rectangle covering everywhere a cow can stand.
   - **Masks:** polygons over permanent glare or foreground steel.
4. **Fill `configs/cameras.yaml`** (`role`, `roi`, `masks`). Copy the cam02 and cam03 `channel` values from `tests/fixtures_private/FIXTURES.json` (T01). Send the file to the orchestrator.

**Done when:** three roles are assigned and three ROIs are set.

## H02 Calibration points and zones
1. **Pick landmarks.** For each camera, choose **≥ 6 ground-level points** whose pen position you can state: pen corners, fence-post bases, bunk ends, gate posts.
   - Pen coordinates are normalised (`02_contracts.md` §9): x runs from 0 (chute-side fence) to 1; y runs from 0 (back fence) to 1 (bunk edge).
   - From the pen map, the front cameras sit at about x = 0.38 and 0.62 on y = 1.
   - Click where objects **meet the ground**.
2. **Record them:**
   ```bash
   python -m scripts.click_points --image work/cam01_day.jpg --scale 0.4 --out configs/calib/cam01_points.json
   ```
   Name 2 points per camera `heldout_*`; they measure the error.
3. **Find the waterer** (it is not on the pen map). Scan daytime keyframes for cows with heads down at a trough or tank away from the bunk. Draw its polygon in pen coordinates in `configs/zones.json`.
4. **Set the bunk strip** to about y ∈ [0.92, 1.0], adjusted to the images. Add a `rest` polygon only if the cows clearly lie in one area at night.
5. **Check the fit.** The orchestrator fits each homography (T09). If held-out RMSE is > 0.03, redo the worst points.

## H03 Detection check (100 frames)
1. Run `python -m scripts.make_sheet --det-check --n 100`. It produces an HTML page plus an answers CSV, stratified by camera × day/night.
2. For each image, count the cows that are fully or partly visible. Do not count people or horses.
3. Save the result as `labels/det_check.csv`.

## H03b Pick good auto-labels (T28, optional)
Follow the prior work’s recipe (`06_prior_work.md`): auto-label, then pick by hand.
1. Open the auto-label sheet.
2. Set `keep = y` only where **every** cow is boxed and no box is wrong. Be strict: 300 clean images beat 500 noisy ones.

## H04 Identity seeds
1. Run `python -m scripts.make_sheet --seeds --hours 10` (Day 1, 10:00–10:59, all cameras). It produces one thumbnail per tracklet.
2. **Agree on marks for each cow:** ear-tag colour or number if legible, white patches, size and body shape, tail switch, scars. Write them in `labels/cows.csv` (`cow_id, ear_tag, tag_color, marks`).
3. **Name each tracklet** with a cow ID, consistent across cameras: use moments when two cameras see the same cow. Use `?` if unsure. The orchestrator converts the answers to `labels/seeds.csv`.
4. **Day 2 anchor.** Repeat for Day 2, 08:00–08:59, with ≥ 3 tracklets per cow. Without this, the "she ate less than yesterday" features may compare different animals.
5. **Dusk.** Follow 2–3 cows through 17:15–18:15 (colour turning to infrared) to get night crops of known cows.

**Done when:** each of the 10 cows has ≥ 3 seed tracklets per day on ≥ 2 cameras.

## H05 Identity evaluation labels
1. Run `python -m scripts.make_sheet --id-eval --exclude-hours 8,10 --n 120`. Include about 40 night tracklets.
2. Label each tracklet's cow, or `?`.
3. Save the result as `labels/id_eval.csv`.

Never use hours that were used for seeds.

## H06 Check set (behaviour ground truth)
1. Run `python -m scripts.make_sheet --check-set --n-random 180 --n-targeted 120`. It produces 10-second clips.
   - The `split` column (tune/test, assigned by whole hour) is pre-filled. Do not change it.
2. Label the behaviour **code** (`02_contracts.md` §6) and `sure` (1 = guess, 3 = certain).
   - **Rumination** needs visible, regular chewing.
   - **Feeding** needs the head in the bunk.
   - When in doubt, label `other` with sure = 1.
3. Three people take 100 each. The 30 shared samples are double-labelled.
4. Merge the answers into `labels/check_set.csv`.

## H07 Veterinary-language review
- **Cards and dashboard use the right terms:**
  - reduced appetite or bunk attendance;
  - increased recumbency;
  - reduced activity;
  - social isolation;
  - increased respiratory rate (only if T30 ran).
- **No diagnosis words:** "infected", "sick", "probability of BRD".
- **Confounders are mentioned where relevant:** estrus or mounting, lameness, staff handling.
- **"Not measured" is stated:** rectal temperature, clinical exam, nasal discharge at this resolution.
- **The dashboard opens offline.** Test it with Wi-Fi off.
- **P(BRD) wording.** Every probability shows its prior and the phrase "model-estimated under stated assumptions; not a diagnosis". The assumptions panel cites the H10 sources.
- Sign off with one line in `reports/STATUS.md`.

## H08 Brief, final QA, submission
**Brief, ≤ 2 pages.** Take every number from `outputs/eval/metrics.json`. Sections:
1. Approach: a pipeline diagram and the agent loop.
2. Data: what 10 cows × 2 days can and cannot support.
3. L1 results: detection recall; ID accuracy for day and night; behaviour accuracy per class with CIs; the minutes observed, estimated and assumed.
4. Agent ablation.
5. L2: ranking, uncertainty, the injection test, and the herd-level statement.
6. L3: the design (tiers, offline mode, cost, who receives the alert).
7. Limitations.
8. What we would do with more data.

**Final QA:**
- Tick D1–D16 in `01_project_definition.md`.
- **Fresh-clone check:** `git clone . /tmp/fresh && cd /tmp/fresh && make setup`, copy `tests/fixtures_private`, then `make test` and `make slice`.
- Run `make package && make validate-submission`.
- Submit by **22:00** and save the confirmation.

## H09 Questions to organizers
Send these questions to the organizers using the contact listed in the event brief. Keep replies in `docs/organizer_answers.md` in the private implementation workspace.
1. May we send frames or crops of the event video to the organizer-approved model API, and to other AI tools we use for development? Or must all processing stay local?
2. For `activity.csv`, is one row per behaviour bout what you expect, or one row per cow-minute? We plan to provide both.
3. Are there official definitions for the 10 behaviour classes? Specifically: does eating from the ground count as feeding, and is rumination expected to be detectable?
4. For L3, how will the judges' clip be delivered (format, length, single camera?), and will it run on our machine or yours?
5. Is there a list of the 10 cows' ear tags or IDs? And may we use unpublished models or data from participating labs, or only public resources?

## H10 Literature priors for the P(BRD) model
Goal: replace the placeholder values in `configs/prob.yaml` with sourced, defensible numbers. Keep it short.
1. **Prior prevalence.** Find a published BRD incidence range for **adult cows or this kind of research pen**, not newly received feedlot calves, whose rates are much higher. Pick a central value and note the range. If nothing fits, keep 10% and say "placeholder" in the brief.
2. **Effect sizes.** Find 2–3 studies reporting how much feeding time, bunk visits or lying time change in cattle before BRD diagnosis (for example, the feedlot feeding-behaviour studies). Turn them into the `effects` weights: roughly, how many log-odds a domain score of +5 should add.
3. **Write it up.** Record the sources (full citations, verified) in `docs/prob_assumptions.md`, and give the values to A, who edits `configs/prob.yaml` and bumps `version`.

Do not invent citations. If a number cannot be sourced in 45 minutes, keep it marked as an assumption.
