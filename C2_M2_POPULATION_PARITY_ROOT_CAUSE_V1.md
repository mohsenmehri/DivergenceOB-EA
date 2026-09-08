# C2/M2 — Population Parity Root-Cause Analysis (M0 vs M2)

Status: Root-cause analysis only. **NO NEW BACKTEST RUN / NO CODE CHANGE / NO
THRESHOLD CHANGE / NO PERFORMANCE VERDICT.**

---

## 1. Executive Summary

- The M0 and M2 runs on GitHub are **not the same execution configuration**.
  Their population differs from **Step1**, i.e. before any C2/M2 Step3→4 veto.
- The 49 M2 vetoes explain **only part** of the Step4 difference and none of the
  Step1/Step2/Step3 differences.
- **The 82 (artifact-consistent: 83) missing setups cannot be attributed to
  `inp_cf_mode=2` itself.** The C2/M2 code path is invoked only inside
  `ManagePositions` for `cur_step==3 && next==4`; it cannot remove a setup at
  Step1/2/3.
- The exact per-`SetupID` provenance of the missing setups is **NOT PROVABLE**
  from the repo artifacts available in this environment because the M0
  detailed CSVs are Git-LFS pointers and the LFS media host could not be
  fetched here. No guessed SetupIDs are reported.
- The M0 reference figures supplied by the owner (4,864 setups / 9,605
  position rows) do **not** match the M0 artifact stored at
  `v2_research_export_v2-new` (manifest rows = 4,865; `SetupID_XAUUSD.txt`
  next = 4,866; `PositionOpenSummary` All Opened = 9,606). This is itself a
  discrepancy that must be resolved before any matched comparison.

Primary classification: **F — Multiple causes**, with the dominant documented
cause being **D — Configuration / provenance mismatch**, plus
**G — Insufficient evidence** for exact SetupID attribution. Cause **A
(only M2 veto)** is explicitly rejected. A matched M0 rerun is required.

---

## 2. Evidence identifiers

| Item | Value |
|---|---|
| M2 run commit | `0e91848010505f4d32d56ea874d8f17b6c067d4f` |
| M2 folder | `v2_research_export_v2_cfv2/` |
| M2 decision CSV | `C2M2_Step34_Decisions_m2.csv` SHA-256 `4ccf19c83722e8199936dd4bb9bd13252e1e23da31d36d09d9550279ff49a0b4` |
| M2 position dataset | `PositionOpenDataset_XAUUSD.csv` SHA-256 `45f494a2386044edc0e70dd4b806beaac2e7b76d905968c272cb892c647ec2a9` |
| M2 manifest | SHA-256 `1ddbfba2ab775f443faa96a8112ffe1696a9c4c69337cf0336ad75944d6f2a68` |
| M2 MT5 report | `ReportTester-91306235.xlsx` (in commit) |
| M0 folder | `v2_research_export_v2-new/` |
| M0 manifest | SHA-256 `67d408af4c5e674ee1a469ea312d988796ff52cb1a9cb8db6b5c851b2b411995` |
| M0 SetupID file | `SetupID_XAUUSD.txt` = `4866` (next ID) |
| M0 Stats file | `Stats_XAUUSD.txt` (NextSetupID=4866, Stat0=4865) |
| M0 Position summary | `PositionOpenSummary_XAUUSD.txt` (All Opened 9606, Time 2026.08.19 23:59) |
| M0 detailed CSVs | Git-LFS pointers in repo; content could NOT be fetched in this sandbox (LFS media host unreachable). |

## 3. M0 vs M2 Population Table

| Stage | M0 (owner ref) | M2 (actual) | Diff (M0−M2) | Explanation / evidence |
|---|---|---:|---:|---|
| Setups | 4,864 | 4,782 | **82** | Occurs before/at Step1. C2/M2 gate cannot affect this. M0 artifact conflicts with owner ref by 1 (manifest 4,865, next ID 4,866). |
| Step1 | 4,864 | 4,782 | **82** | Same as above; no C2/M2 code path in setup creation. |
| Step2 | 2,970 | 2,921 | **49** | Attrition at Step1→2. Occurs before C2/M2 gate; not caused by veto. |
| Step3 | 1,149 | 1,132 | **17** | Attrition at Step2→3. Occurs before C2/M2 gate; not caused by veto. |
| Step4 | 431 | 376 | **55** | 49 = confirmed M2 veto; remaining 6 = upstream population/coverage difference. |
| Step5 | 191 | 169 | **22** | Downstream of Step4; includes both veto effect and upstream difference. |

### Derived M2-only funnel (actual)

M2: 4,782 Step1 → 2,921 Step2 → 1,132 Step3 → 425 Step3→4 decision attempts
→ 367 ALLOW + 49 VETO + 9 INVALID → 376 Step4 positions → 169 Step5.

Because 425 attempts < 1,132 Step3 rows, many Step3 setups never reached the
Step3→4 gate (market/spread/margin/EOT timing, not captured in C2M2 CSV).

---

## 4. 82 Missing Setup Analysis (exact per-setup)

**NOT PROVABLE.** The M0 detailed `PositionOpenDataset`/`RX_*` data needed for
a SetupID-level diff are stored as Git-LFS objects in the repository and were
not retrievable in this sandbox (LFS batch endpoint returned a signed S3
`github-cloud.githubusercontent.com` URL, and that host is unreachable from this
environment). Only the M0 non-LFS metadata (manifest, Stats, Position summary,
SetupID file) and M0 source were accessible.

Available facts that bound the problem:

- M0 `SetupID_XAUUSD.txt` next ID = 4,866 → IDs 1..4,865.
- M2 `SetupID_XAUUSD.txt` next ID = 4,783 → IDs 1..4,782.
- M0 manifest rows = 4,865; M2 manifest rows = 4,782 → difference 83.
- Owner-provided M0 says 4,864 setups / 9,605 rows; stored M0 says 4,865 setup
  IDs and 9,606 opened positions. Therefore even the exact M0 baseline is
  ambiguous by one unit.

No per-setup stage classification (Missing before Step1, Missing at Step1,
Present Step1↔2, etc.) can be claimed without the M0 CSVs. Categories below are
stage-level only.

| Category | Can we classify? |
|---|---|
| Missing before Step1 | Yes at aggregate level (difference already exists at Step1) |
| Missing at Step1 | NOT PROVABLE per setup |
| Present Step1 but missing Step2 | Aggregate = 49 |
| Present Step2 but missing Step3 | Aggregate = 17 |
| Present Step3 but missing Step3→4 decision | Aggregate possible: 1,132 Step3 vs 425 attempts; file does not encode why the others never reached gate |
| Affected by M2 veto | Exactly 49 SetupIDs, all confirmed |
| Same setup but different lifecycle / SetupID assignment | NOT PROVABLE |
| Unknown / insufficient evidence | The exact difference is here |

---

## 5. Stage-by-Stage Attrition Analysis

1. **Setups/Step1 (−82):** This is upstream of trading steps. The C2/M2 block
   is only called in `ManagePositions`, at `cur_step==3 && next==4`, after all
   entry gates. It cannot reduce setup creation or Step1 placement.
2. **Step2 (−49):** A Step1→2 attrition difference. This occurs before any CF
   gate and before C2/M2. It cannot be caused by the C2/M2 veto.
3. **Step3 (−17):** A Step2→3 attrition difference. Same reasoning.
4. **Step4 (−55):** 49 of the 55 are the confirmed M2 vetoes. The remaining 6
   are consistent with the upstream population already being smaller (M0
   Step3 1,149 → Step4 431; M2 Step3 1,132 → Step4 376), i.e. not all are veto.
5. **Step5 (−22):** Downstream effect of both the veto and the smaller
   upstream base. Not a separate M2-only effect.

Conclusion for "did M2 change only Step3→4?":
The M2 code set only changes a Step3→4 decision path, but **the observed M0/M2
population differs before Step3→4**, so either the runs were not matched in
some upstream way (different source build, settings, tick/data, date cutoff),
or the M0 reference is not the correct matched artifact. The M2 code itself
does not contain an upstream population-changing condition.

---

## 6. Configuration / Provenance Comparison

| Setting | M0 | M2 | Same? | Could affect population? |
|---|---|---|---|---|
| Source file | `v2_research_export_v2-new/v2_research_export_v2.mq5` (SHA `d961dcb9…`) | `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` (SHA `32795259…`) | **NO** | **YES — different build** |
| Diffs vs M0 | — | adds CF/C2M2 module, latch fields, Step1 `CF_LogEvent`, Step≥4 `UpdateDrawdown` call, CF gate in `ManagePositions` | — | Logging/gate changes only; no detection-change found |
| Broker | NOT AVAILABLE (M0 report LFS) | LiteFinance Global LLC | ? | possible |
| Account currency | NOT AVAILABLE | USD | ? | possible |
| Deposit | NOT AVAILABLE | 500,000 | ? | possible |
| Leverage | NOT AVAILABLE | 1:100 | ? | possible |
| Symbol | XAUUSD | XAUUSD | YES | no |
| Timeframe | M1 | M1 | YES | no |
| Tester model | NOT AVAILABLE | 100% real ticks | ? | **possible** |
| Date range / cutoff | M0 summary: `2026.08.19 23:59` | M2 report: `2026.08.18 23:59` / position max `2026-08-18 19:57:18` | **NO** | **YES** |
| TestProfile | NOT AVAILABLE | `PROFILE_D_FULL` | ? | possible |
| `inp_reset_setup_id` | NOT AVAILABLE | `true` | ? | **SetupID numbering** |
| `inp_cf_mode` | NOT AVAILABLE (baseline defaults to 0) | `2` | ? | only Step3→4 |
| `inp_cf_q75_atr_pct_34` | NOT AVAILABLE | `0.04668375` | ? | only Step3→4 |
| `inp_cf_q75_adx_34` | NOT AVAILABLE | `47.50305` | ? | only Step3→4 |
| `inp_cf_q75_atr_pct_23` | NOT AVAILABLE | `0.0` | ? | only Step3→4 |
| `inp_c2m2_collect_shift1_m0` | NOT AVAILABLE | `false` | ? | no |
| `inp_enable_export` | NOT AVAILABLE | `false` | ? | no trade logic |
| `inp_fast_backtest_mode` | NOT AVAILABLE | `false` | ? | **possible export/timing difference** (M2 actual) |
| `inp_use_spread_filter` | NOT AVAILABLE | `false` | ? | possible entry timing |
| `inp_broker_protection_sl_atr` | NOT AVAILABLE | `0.0` | ? | possible exit/execution |
| `inp_fixed_lot_size` / dynamic lot | NOT AVAILABLE | `0.01` / dynamic off | ? | possible margin/data |
| Margin/spread gates | NOT AVAILABLE | default plus `inp_max_spread_points=30` | ? | possible |

**M0 config provenance is NOT AVAILABLE** because the M0 `ReportTester-*.xlsx`
is a Git-LFS pointer and could not be resolved in this environment. No value
has been inferred.

---

## 7. 9 INVALID Root-Cause Analysis

### Data

| SetupID | DecisionTime | BarOpen/Close Shift1 | FeatureShift | strict_before | ATRPct | ADX | DIPlus | DIMinus | Decision | Latch |
|---:|---|---|---:|---|---:|---:|---:|---:|---|---|
| 165 | 22:15:00 | …/22:15:00 | 1 | false | 0.003467 | 42.2029 | 4.9400 | 26.5163 | INVALID | FIRST |
| 466 | 03:20:00 | …/03:20:00 | 1 | false | 0.026725 | 51.5788 | 39.1510 | 3.9774 | INVALID | FIRST |
| 1055 | 11:42:00 | …/11:42:00 | 1 | false | 0.030417 | 39.8334 | 10.7301 | 45.3401 | INVALID | FIRST |
| 1414 | 02:03:00 | …/02:03:00 | 1 | false | 0.028377 | 33.1438 | 10.9343 | 33.6895 | INVALID | FIRST |
| 2592 | 12:30:00 | …/12:30:00 | 1 | false | 0.060436 | 23.3950 | 19.0052 | 20.9901 | INVALID | FIRST |
| 2713 | 07:47:00 | …/07:47:00 | 1 | false | 0.025112 | 54.5804 | 6.2666 | 36.5776 | INVALID | FIRST |
| 3044 | 06:59:00 | …/06:59:00 | 1 | false | 0.020942 | 21.7100 | 11.0611 | 22.0186 | INVALID | FIRST |
| 3292 | 01:00:00 | …/01:00:00 | 1 | false | 0.043335 | 53.9662 | 4.9102 | 31.2072 | INVALID | FIRST |
| 3533 | 01:02:00 | …/01:02:00 | 1 | false | 0.022919 | 19.2017 | 13.9319 | 21.6113 | INVALID | FIRST |

All 9 have `BarCloseTime_Shift1 == T_decision`, so strict-before
(`bar_close < t_decision`) is false.

### Root cause

- `C2M2_Step34ShouldVeto` computes:
  ```
  strict_before = (bar_close < t_decision)
  ```
  and `bar_close = bar_open + PeriodSeconds(M1)`.
- When the EA receives a tick whose `tick.time` equals the M1 bar close time
  (the bar boundary), `bar_close < t_decision` is false → INVALID.
- This is a **tester timestamp boundary condition**. The tick at exactly the
  bar-close / next-bar-open instant makes `T_decision` equal to the (closed)
  Shift-1 bar close. It is not a data-missing condition; the rows have real
  indicator values.
- The code then latches `C2M2_LATCH_INVALID` and returns `false`
  ("fail-open"): the ladder is **not** blocked and Step4 proceeds. The 9
  SetupIDs all appear in `PositionOpenDataset` and reach Step4 (7 to Step4,
  2 to Step5 in the M2 lifecycle).
- Compatibility with D4/D5: D4 requires strict `BarCloseTime_Shift1 <
  T_decision`; the boundary rows violate it and are marked INVALID. D5 latch is
  honored (LatchStatus=FIRST, latch=INVALID). **However, invariant "all rows
  strict_before=true" is intentionally not met for these 9; they are invalid
  boundary rows.** Whether they should be fail-open or fail-closed is a
  policy question, not yet decided.

### Does this affect the M2 population parity?

- It does not remove setups; it fails open. These 9 are all in `ALLOW + INVALID
  = 376` and thus are included in Step4.
- It does not explain the Step1/Step2/Step3 differences.

---

## 8. 49 Veto Validation

Verified from `C2M2_Step34_Decisions_m2.csv`:

| Check | Result |
|---|---|
| Veto count | 49 |
| ALLOW count | 367 |
| INVALID count | 9 |
| Duplicate SetupID in CSV | 0 |
| FeatureShift = 1 (all 425) | Pass |
| LatchStatus = FIRST (all 425) | Pass |
| `strict_before_check=true` for ALLOW/VETO | Pass (416/416) |
| Thresholds re-checked row-by-row | Pass (`>=0.04668375`, `>=47.50305`) |
| DI_ADVERSE direction re-checked | Pass |
| VETO matches rule exactly | 49/49 |
| Veto rate | `49/425 = 11.53%` |

Outcome classification (from M2 `PositionOpenDataset`, final state = Step3 for
all vetoed setups because they never reached Step4/5):

| Group | Count | Step3 PnL (price PnL only) |
|---|---:|---:|
| Vetoed sets that were already in Step3 `SL$` | 19 | -5,743.89 |
| Vetoed sets that were already positive at Step3 (TP-family) | 30 | +175.75 |
| Total vetoed | 49 | -5,568.14 |

**No causal benefit claim is made.** The 30 Step3-positive setups would have
entered Step4/5 in M0; their counterfactual M2 outcome is **not observed**
because the veto removed them. The 19 Step3-loss setups were stopped before
Step4 and their large negative Step3 PnL is observed.

---

## 9. Root-Cause Classification

| Option | Accept? |
|---|---|
| A. Expected difference caused only by M2 Step3→4 veto | **NO** — differences exist before Step3→4 |
| B. Upstream population changed unexpectedly | Not demonstrated; no upstream code-path change found, but cannot be excluded due to different build/config |
| C. Tester/timing/data difference | **Likely contributor** (M0 vs M2 different date cutoff, M0 LFS/data unavailable, boundary INVALID rows) |
| D. Configuration/provenance mismatch | **YES — dominant confirmed mismatch**: different `.mq5` build, M0 config/report unavailable, M0 reference inconsistent with M0 artifact |
| E. Code-path interaction | **No evidence found**; C2/M2 only touches Step3→4 |
| F. Multiple causes | **YES — final classification** |
| G. Insufficient evidence | **YES for exact 82-SetupID attribution** |

Synthesis: **F.** The dominant, evidence-supported cause is **D**. Exact
per-SetupID attribution is under **G** because M0 detailed artifacts are LFS
and could not be read. Causes A and E are not supported by code evidence.

---

## 10. Whether M0 Rerun Is Required

**YES.** A matched M0 rerun is required before any M2-vs-M0 performance
conclusion.

Required matched M0 configuration:

| Setting | Value |
|---|---|
| EA source | Same M2 build `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5`, SHA `32795259…` |
| Run mode | `inp_cf_mode=0` |
| Collector | `inp_c2m2_collect_shift1_m0=false` |
| Threshold inputs | leave M2 values present: `0.04668375` / `47.50305` / `0.0` (inactive in mode 0) |
| Symbol | XAUUSD |
| Timeframe | M1 |
| Model | Every tick based on real ticks |
| Date From | `2014.01.01 00:00:00` |
| Date To / cutoff | same as M2: `2026-08-18 23:59:58` (not `2026-08-19`) |
| Broker / account | same broker, USD account as M2 run (LiteFinance) if possible |
| Deposit / Leverage | same as M2: 500,000 / 1:100 |
| TestProfile | `PROFILE_D_FULL` |
| `inp_reset_setup_id` | `true` (same as M2) |
| Fast backtest mode | `false` (same as M2) |
| Export | `inp_enable_export=false`, dataset export `true` |
| Spread/margin | same as M2 (spread filter off, max 30, broker-protection SL 0.0) |
| Other inputs | **copy all inputs from the M2 `ReportTester-91306235.xlsx`**, replacing only `inp_cf_mode=0` and collector=false |

After the matched M0 run, compare:
- setup count / Step1–Step5 funnel,
- `SetupID` set and lifecycle by `SetupID`,
- same `C2M2` decision files (ensure M0 has no C2M2 decision log because
  `inp_cf_mode=0`; if using the same path, confirm it is inactive),
- only then perform M2-vs-M0 metrics on the matched population.

---

## 11. Exact Recommended Next Step

1. Confirm with Owner which M0 artifact is authoritative: `v2_research_export_v2-new`
   (manifest rows 4,865, next ID 4,866, PositionOpenSummary 9,606) or the
   owner-provided figures (4,864 / 9,605). These differ by one.
2. Re-run M0 using the exact M2 source build and the input set listed above,
   with `inp_cf_mode=0`, `inp_c2m2_collect_shift1_m0=false`, cutoff
   `2026-08-18 23:59:58`.
3. Perform SetupID-level parity using the newly generated M0
   `PositionOpenDataset`/`RX_*` and the M2 run's same files. If Step1/Step2/Step3
   counts still differ, compare the non-LFS run metadata and M0 `ReportTester`
   inputs to identify the mismatched setting.
4. Do not issue a performance verdict until the matched population parity is
   confirmed.

---

## Final required line

```
NO NEW BACKTEST RUN
NO CODE CHANGE
NO THRESHOLD CHANGE
NO PERFORMANCE VERDICT
```
