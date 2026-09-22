# Raw NIRS Dataset Documentation & Cleaning Guide
### Practice12 (Artinis Oxysoft export) — for Skeletal Muscle Mitochondrial Oxidative Capacity Analysis

This document explains **what the raw exported file actually contains**, **how it maps onto
the protocol** (baseline → occlusion → exercise → repeated recovery occlusions), and **exactly
what needs to be cleaned/reshaped** before the data can be fed into
`mitochondrial_capacity_analysis.Rmd` (see that script's own `README.md` for the analysis
method itself — this document is the missing link between the *raw export* and the *inputs that
script expects*).

It is based on:
- `Practice12 (version 1).xlsb` (provided as `.xlsx`) — the raw Artinis Oxysoft export
- The data-owner's annotation table (`Docoumentation_of_the_data.docx`)
- The analysis `README.md` for `mitochondrial_capacity_analysis.Rmd`

---

## 1. What the raw file is

The file is a direct **Artinis Oxysoft export**, not a clean tabular dataset. It contains:

| Rows | Content |
|---|---|
| 1–33 | Device/session metadata (export date, sample rate, optode geometry, device IDs, wavelengths) |
| 35–65 | **Column legend** — maps each data column number to what it actually measures |
| 67 | Data table header row |
| 68 onward | The actual time-series data, one row per sample |

Key metadata (rows 1–29):
- **Sample rate:** 10 Hz (one row = 0.1 s)
- **Total samples in file:** 6,416 (0–6,415), i.e. 641.6 s of recording
- **Devices:** 2 receivers (Rx1, Rx2 — "laser machine 1" and "laser machine 2" per the
  annotation), 6 light sources (Tx1a, Tx2a, Tx3a on Rx1; Tx1b, Tx2b, Tx3b on Rx2)
- **Optode distance:** 45 mm, with a 4 mm gradient between the 3 transmitters per receiver
  (i.e. Tx1/Tx2/Tx3 sit at ~45/49/53 mm from the receiver) — this is a **multi-distance
  probe**, used to calculate Tissue Saturation Index (TSI) via spatially-resolved spectroscopy
- **DPF (differential pathlength factor):** 4 (used internally by Oxysoft to convert optical
  density to haemoglobin concentration — already applied to O2Hb/HHb/tHb values in this export)

## 2. Column legend (rows 35–65)

Each data column is one of these — the annotation doc's abbreviations are expanded here:

| Col # | Excel col | Signal | Meaning |
|---|---|---|---|
| 1 | A | Sample number | Row index, starts at 0. **Time (s) = sample number ÷ 10** |
| 2 | B | Rx1 TSI% (Tx1a,2a,3a) | Combined tissue saturation index, Rx1 |
| 3 | C | Rx1 TSI Fit Factor | Quality-of-fit for the TSI regression, Rx1 (closer to 0 = better fit) |
| 4–7 | D–G | Rx1‑Tx1a: O2Hb, HHb, tHb, HbDiff | Oxygenated/deoxygenated/total Hb, and O2Hb−HHb, at distance 1 |
| 8–11 | H–K | Rx1‑Tx2a: O2Hb, HHb, tHb, HbDiff | Same, at distance 2 |
| 12–15 | L–O | Rx1‑Tx3a: O2Hb, HHb, tHb, HbDiff | Same, at distance 3 |
| 16 | P | Rx2 TSI% (Tx1b,2b,3b) | Combined tissue saturation index, Rx2 |
| 17 | Q | Rx2 TSI Fit Factor | Quality-of-fit, Rx2 |
| 18–21 | R–U | Rx2‑Tx1b: O2Hb, HHb, tHb, HbDiff | Distance 1, Rx2 |
| 22–25 | V–Y | Rx2‑Tx2b: O2Hb, HHb, tHb, HbDiff | Distance 2, Rx2 |
| 26–29 | Z–AC | Rx2‑Tx3b: O2Hb, HHb, tHb, HbDiff | Distance 3, Rx2 |
| 30 | **AD** | Event | Protocol phase marker (see §3) — blank for almost every row, populated only on the sample where a phase boundary occurs |

**tHb = O2Hb + HHb** (total haemoglobin/myoglobin — used for visually confirming occlusion
windows, see §5).

## 3. Protocol timeline (decoded from the Event column, col. AD)

The annotation doc defines letter codes a–h; the Event column (col. AD) marks exactly one
sample per code, formatted `"A1"`, `"B1"`, … `"H1"`. Decoding the actual timestamps in this file
(sample ÷ 10 Hz):

| Event | Sample # | Time (s) | Protocol meaning (per annotation) | Segment duration |
|---|---|---|---|---|
| **A1** | 286 | 28.6 | **a = baseline start** | — |
| **B1** | 1118 | 111.8 | **b = occlusion starts** | A1→B1 = **83.2 s** (baseline) |
| **C1** | 1420 | 142.0 | **c = reperfusion** (occlusion released) | B1→C1 = **30.2 s** (occlusion) |
| **D1** | 2140 | 214.0 | **d = exercise starts** | C1→D1 = **72.0 s** (reperfusion, pre-exercise) |
| **E1** | 2742 | 274.2 | **e = exercise ends** | D1→E1 = **60.2 s** (exercise) |
| **F1** | 3796 | 379.6 | **f = intermittent occlusion series starts** (8 occlusions × 8 s over 3 min) | E1→F1 = **105.4 s** (post-exercise rest, before occlusion series) |
| **G1** | 5632 | 563.2 | **g = end of the 3-minute occlusion series** | F1→G1 = **183.6 s** (recovery occlusion series, ≈3.06 min) |
| **H1** | 6280 | 628.0 | **h = end of study** | G1→H1 = **64.8 s** (final tail segment) |

**⚠ Flag for verification:** the annotation describes the baseline (A1→B1) as "two minutes," but
the actual marked interval in this file is 83.2 s (~1.4 min). The occlusion (30.2 s), exercise
(60.2 s), and recovery-occlusion-series (183.6 s ≈ 3 min) durations all match the written
protocol closely, so this is very likely just a labeling approximation in the annotation
doc rather than a data problem — but worth a quick sanity check against the session notes
before finalizing results.

**Also note:** the file contains 286 samples (28.6 s) of data *before* A1, and ~135 samples
(13.5 s) *after* H1. Per the annotation ("delete all dataset above the baseline dataset
(A1)"), the pre-A1 samples are setup/positioning data and should be dropped. The small tail
after H1 can also be dropped as it falls outside the defined protocol window.

## 4. Decisions applied to this dataset (standard NIRS/mitochondrial-capacity protocol)

Three structural questions had to be resolved before the data can be reshaped. Standard
practice for this type of multi-distance, dual-probe NIRS setup was applied:

**a) Two probes (Rx1 vs Rx2). — SUPERSEDED, see `README.md` §0.** This section originally
assumed Rx1/Rx2 were two separate sites and should never be combined. The lab has since
confirmed both probes were placed on the **same muscle**, so the reengineered pipeline
**averages the (channel-selected) Rx1 and Rx2 signals together** into one muscle signal
instead of analyzing them separately. See `README.md` §0 and §4 for the current, authoritative
version of this decision and how it's sequenced relative to Tx-channel selection.

**b) Three transmitter distances per probe (Tx1/Tx2/Tx3).** These sample slightly different
tissue depths (increasing distance = deeper/more muscle, less skin/fat contribution), so
averaging their O2Hb/HHb directly would mix signals from different tissue layers — not
standard practice. The standard approach for multi-distance probes is to use the **TSI Fit
Factor** as a data-quality indicator (values closer to 0 indicate a better linear fit of optical
density vs. distance, i.e. a more reliable signal) and **select the single channel (Tx1, Tx2, or
Tx3) with the best/most consistent Fit Factor** for the O2Hb/HHb concentration data used in
the mV̇O2 slope calculations. In this file that means comparing the Fit Factor trend for Rx1
(col. C) and Rx2 (col. Q) over the analysis window and picking the corresponding Tx channel.

**c) Individual occlusions within the 3-minute recovery series (F1→G1).** The file only marks
the *start and end of the whole series* — not each of the 8 individual 8‑second occlusions.
Per the analysis script's own guidance (see its README, Section 4, step 3), these must be
**identified visually from the tHb trace**: tHb (= O2Hb + HHb) rises sharply and plateaus at
the start of each occlusion, then falls back at release. Between F1 (379.6 s) and G1 (563.2 s),
expect ~8 such plateaus, roughly evenly spaced across the 183.6 s window (~8 s occlusion +
~15 s release per cycle, consistent with the "8 s × 8" description). Plot tHb over this window
first and mark each occlusion's start/end before building the occlusion timing file.

## 5. Cleaning steps (raw export → analysis-ready CSVs)

For **each probe separately** (Rx1, then repeat for Rx2):

1. **Strip the metadata block.** Discard rows 1–66 (session info + legend); keep only the data
   table (header at row 67, data from row 68).
2. **Trim to the protocol window.** Delete all rows with sample number < 286 (before A1) and,
   optionally, rows after sample 6280 (after H1) if you don't need the tail.
3. **Build `time_s`.** Add a column: `time_s = sample_number / 10`.
4. **Select the O2Hb/HHb channel.** Using the Fit Factor column for this probe, pick the
   best-fit Tx channel (Tx1a/2a/3a for Rx1, or Tx1b/2b/3b for Rx2) and pull its O2Hb and HHb
   columns. Rename them to `O2Hb` and `HHb` to match the analysis script's expected format.
5. **Assemble `your_nirs_data.csv`** with exactly three columns: `time_s`, `O2Hb`, `HHb`,
   covering the full trimmed window (baseline through end of recovery occlusions).
6. **Build `your_occlusion_times.csv`** with columns `occlusion_id`, `phase`, `start_time`,
   `end_time`:
   - One row, `phase = "rest"`, for the baseline occlusion: `start_time = 111.8`,
     `end_time` = start + ~3–4 s (per the analysis script's own recommendation to use only
     the first few seconds of a longer occlusion — not the full 30.2 s window).
   - One row per individual occlusion identified visually in the F1→G1 window (step in §4c),
     each with `phase = "recovery"`.
7. **Set `exercise_end_time = 274.2`** (the E1 timestamp) in the analysis script's
   configuration chunk — this is what the script uses to convert each recovery occlusion's
   start time into "time since exercise."
8. **Set `apply_bv_correction_flag`** per the analysis script's guidance — check whether O2Hb
   and HHb move in opposite directions during occlusions in this data (they should); if not,
   blood-volume correction should be turned on.
9. Repeat steps 1–8 independently for Rx2 to get a second, separate pair of CSVs.

## 6. Quick QC checklist before running the analysis script

- [ ] Confirm which Tx channel was selected for each probe, and note its typical Fit Factor
      value (for the methods write-up).
- [ ] Plot tHb across the F1→G1 window and confirm 8 distinct occlusion plateaus before
      finalizing `your_occlusion_times.csv`.
- [ ] Sanity-check the baseline duration discrepancy noted in §3 against original session notes.
- [ ] Confirm O2Hb and HHb move in opposite directions during the resting occlusion (evidence
      the signal reflects oxygen consumption, not just blood-volume shift).
- [ ] Verify Rx1 and Rx2 results are reported/interpreted as two separate sites, not averaged.

## 7. Open items still needing a decision

**Resolved — see `README.md` §0 and §9 for the current status of all of these:**
- Rx1 vs Rx2 handling: confirmed same muscle, now averaged (not treated separately).
- Tx channel combination: use best (lowest) Fit Factor channel per probe.
- Recovery occlusion timestamps: detected automatically from the tHb trace in code
  (not marked manually), with a mandatory diagnostic plot for visual verification.

**Still open** (discovered while inspecting the actual sample files — see `README.md` §3.3, §9):
- `practice3.xlsx` has an unexplained extra `E2` event and no `H1` event.
- `practice_4.xlsx` records at 1 Hz, 10× coarser than the other two sample files — confirm
  this is intentional.
