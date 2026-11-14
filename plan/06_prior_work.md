# 06: Prior work, the prior computer-vision research, and what it changes

- **Source:** Slides supplied to the team
  - Source details are retained in the team’s private reference notes.
- **Scope:** Prior research helped choose candidate methods for the hackathon pipeline.

## 1. What the deck reports (as stated on the slides)

- Curated auto-labels made fine-tuning a practical option for cattle detection.
- Supervised identification and crop-based activity classification informed the optional classifier work.
- The research setup explored thermal sensing and proposed frequency-domain respiration analysis. The hackathon pipeline has separate measurement limits.

## 2. What it changes in this plan

1. **Video container.** T03 handles a vendor-tagged HEVC video stream by reading the AVI layout and extracting the payload.
2. **Detector fine-tuning.** T28 (P1) samples keyframes across lighting conditions, then uses manual review to curate the auto-labels before training.
3. **Identity must be supervised, never clustered.** T14 already uses seeds and closed-set assignment. When ≥ 30 crops per cow exist, prefer a supervised scorer (logistic regression on embeddings, or YOLO11-cls) trained on seeds plus verified tracklets, **split by time block**.
   - **Caution:** Measure identity accuracy on held-out time blocks from the intended dataset.
4. **Activity: a learned crop classifier is the right "second opinion".** New card **T29** (P2) trains it on labelled crops. The gate (T21) uses disagreement between rules and classifier as a flag, and the classifier replaces box-aspect posture only where it wins on the `tune` split.
5. **Respiration** follows the deck's planned frequency-domain method. New card **T30** (P2) estimates breathing rate from 30 fps bursts of still cows (`decode_window`), daylight first. Where it works, it adds the R in video-DART for L2 and a visible clinical sign for L3.
6. **Temperature cannot be measured here.** The pen map shows three RGB cameras and no thermal camera. The grey night footage is near-infrared illumination, **not thermal**. Our video-DART is therefore D + A (+ R if T30 works), and T is reported as "not measured" (pitfall 51).
7. **"Eating from ground"** is a class in the deck but not among the brief's 10. Map it to `feeding` only if this pen has hay or feed on the ground; otherwise it is standing or other. Ask the organizers (H09).

## 3. Cautions

- The deck's numbers come from a different pen, cattle and camera set. Treat them as evidence of **feasibility**, not as expected accuracy.
- **Lab assets:** if the team can access this group's labelled data or trained weights, remember the brief allows "any free, public dataset, pretrained model or code" listed in the repository. Unpublished lab assets need explicit organizer permission (H09) and must be disclosed in the README.
- **Where Autodistill runs:** Autodistill (GroundedSAM, GroundingDINO) needs a GPU and model downloads. Run it on the GPU workstation in R03b, never inside worker sandboxes. Without it, T28 falls back to auto-labels from the COCO-pretrained detector.
