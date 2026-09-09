# STOP PATTERN ANALYSIS — REAL BACKTEST DATA v1

**Date:** 2026-09-06 · **Run analyzed:** `RX_1388707200_XAUUSD` (XAUUSD; MTF M1|M5|M15|M30|H1|H4; backtest 2014→2026-08) · baseline M0 profile (`PROFILE_D_FULL`)
**Sources (REAL, branch `agent-regression-data-real` @ `662a7ec`):**
`v2_research_export_v2_cf/RX_..._Features.csv` (4,864 rows) · `RX_..._Labels.csv` (4,864) · `RX_..._SetupLifecycleEvents.csv` (28,447 events) · `PositionOpenDataset_XAUUSD.csv` (9,605 legs) · `RX_..._Manifest.json` (schema `RX_STEP1_DATASET_V1`: capture `bar=last_completed, shift=1, forming=diagnostic_only`).
**Grant:** descriptive association analysis ONLY — no code change, no new backtest, no threshold extraction, no strategy/production decision, no causal claims.

---

## 1. Data Availability Audit (requested files)

| Requested source | Status |
|---|---|
| PositionOpenDataset_<SYM>.csv | ✅ REAL, 9,605 legs (shift-0 fill capture; `IndicatorShiftUsed=0` on all) |
| PositionOpenIndicatorStats_<SYM>.csv | ✅ REAL (summary stats file; not needed as input) |
| CF_events_m0.csv / m1.csv | ✅ REAL on data branch (`v2_research_export_v2_cf/`, `_c-M1/`) — **kept OUT of the main chain** (M1-era shift-0 decision-log family; not merged, per rule 5) |
| RX Features / Labels / Lifecycle | ✅ REAL (see header) |
| Step1_OutcomeComparison*.csv | ✅ REAL (auxiliary) |
| **RiskLab_Raw_<SYM>.csv** | ❌ absent in repo → `NOT PROVABLE FROM AVAILABLE DATA` for anything requiring it |
| **SignalRegistry_<SYM>.csv** | ❌ absent → `NOT PROVABLE FROM AVAILABLE DATA` |
| MarketEdge_..._XAUUSD.csv (v2/v2-new full snapshot) | ⚠️ only LFS pointers on arena branch; REAL under `v2_research_export_v2_cf` on data branch — not needed as primary input here |
| C2/M2 Decision Log (`C2M2_M0_Shift1_Population_DEV.csv`) | not present in analyzed data → not merged (rule 5), noted only |

## 2. Analysis construction (QC-locked)

- **Main lens = snapshot Shift=1 canonical** (per manifest: closed bar, pre-outcome, `FeatureAsOfTime` at signal→step1-fill moment): RX Features ⟕ RX Labels on `SetupID` — 1:1, zero duplicates either side, join coverage 100%.
- **Separate lens = PositionOpen Shift=0** (fill-time, per-leg): reported in §7 only; never mixed into §5.
- Population: 4,864 setups → 1 censored (`EOT_OPEN`) excluded from outcome denominators → **N = 4,863** (BUY 3,612 / SELL 1,251).
- Outcomes kept independent by construction: **stop_flag** = FinalEvent **final token == `SL$`** (dollar stop close); **pnl_negative** = `FinalProfit < 0`. Labels are money-based per QC (`MFEMoney/MAEMoney/MaxDDMoney` used; *_points* ignored).
- QC constraints honored mechanically: divergence block filtered with `DivSpan > 0` (see §6 finding); canonical pivot RSI = `P1RSI/P2RSI/P3RSI`; confirmed/live separation via `SignalSource` (see §8 limitation); Shift separation via canonical vs `*Forming` columns (72 forming columns EXCLUDED per manifest `diagnostic_only`).
- Ratios/lifts reported only where the positive cell has n ≥ 30; everything else flagged *UNSTABLE*. No p-values produced; **associations only**.

## 3. Outcome structure (exact)

| Metric | Value |
|---|---|
| N (uncensored, FinalEvent≠∅) | **4,863** |
| Stop count / rate | **88 / 1.81 %** |
| pnl_negative count / rate | **88 / 1.81 %** |
| Cross-tab stop×loss | **perfect overlap: every stop is a loss, every loss is a stop** (0 off-diagonal) |

**Close-type mix (final token):** TP1 1,894 (39.0%) · TP2 1,818 (37.4%) · TP3 702 (14.4%) · NET_BASKET_BE 361 (7.4%) · SL$ 88 (1.8%).

**Money cohorts (medians [Q1,Q3]):** STOP: profit −301.26 [−305.29, −300.31] · MFE 2.52 · MAE 299.00 · MaxDD 300.97. Others WIN_OK (n=4,775): profit 2.68 [2.08, 5.19] · MFE 2.72 · MAE 4.48 · DD 5.80.
**Structural fact:** `DollarStopUSD = 300` on 100% of step-1 legs → the hard stop is a fixed $300 basket-level stop; engineered exits (TP*/NET_BASKET_BE) land ≥ 0 in this run ⇒ **loss ≡ stop in this dataset**.

## 4. Stop/Loss patterns — snapshot Shift=1 canonical (pre-outcome)

Base rate 1.8%, stop n = 88. Stable cells (q4y ≥ 30) in bold; full instability flagged `*U*`.

| Feature | miss | stop med [Q1,Q3] | non-stop med [Q1,Q3] | Q4 lift |Q4 (y/n)|
|---|---|---|---|---|---|
| **ATRDistance (grid width)** | 0.0% | **2.315 [0.92, 5.71]** | **1.150 [0.70, 2.06]** | **2.17** (48/1225) |
| **M1_ATR** | 0.0% | **0.930 [0.37, 2.29]** | **0.460 [0.28, 0.82]** | **2.14** (48/1237) |
| **TickVolume (signal bar)** | 0.0% | **186.5 [82, 338]** | **116.0 [64, 194]** | **1.95** (43/1221) |
| **SpreadPoints** | 0.0% | **36.15 [31.3, 66.25]** | **33.4 [27.7, 42.0]** | **1.71** (38/1226); q1-lift 0.59 |
| **PriceDivStrength** | 0.0% | 0.000 | 0.000 | **1.59** (35/1216) |
| M1_ATRRegime | 0.0% | 1.087 | 1.049 | 1.23 *U* (27/1216) |
| RSIDivStrength | 0.0% | 0.435 vs 0.339 |  | 1.32 *U* (29/1216) |
| SpreadATRNorm | 0.0% | 0.445 vs 0.711 (lower at stops) |  | q1-lift 2.00 *U* (18/1216) |
| M1_ADX | 0.0% | 25.13 vs 27.67 |  | 0.86 *U* — **NO positive tail association** |
| M1_RSI | 0.0% | 48.76 vs 49.41 |  | 0.86 *U* — none |
| M1_DIPlus/DIMinus/DIGap | 0.0% | ≈21.5/19.7/5.5 vs ≈20.4/20.1/5.7 |  | ~1.0 *U* — none material |
| M1_CCI / slopes (RSI/CCI/ADX) | 0.0% | small diffs |  | <1.05 *U* — none |
| P1RSI / P2RSI / P3RSI | 0.0% | 27.5/34.0/42.7 vs 28.9/34.9/42.6 |  | q4 ~0.6 *U* — none (if anything, high pivot-RSI tail *anti*-stops, unstable) |
| DivergenceAngle | 0.0% | 0.829 vs 0.714 |  | 1.14 *U* |
| DivGapRatio | 0.0% | 0.879 vs 1.000 |  | 0.93 *U* |
| OBDistPct | 0.0% | 0.009 vs 0.008 |  | 0.91 *U* |
| BOS structure block | 0.0% | all-zero/degenerate in pop |  | 1.00 (no signal) |
| EntryHour / BarsSinceSignal / SignalCLV / BodyRatio / CandleATRNorm | 0.0% | ≈equal medians |  | ~1.0 *U* |

(LOSS vs non-loss table is numerically identical row-for-row because loss ≡ stop; listed separately in the analysis script, not duplicated here.)

## 5. Temporal / structural splits (same N=4,863)

**Yearly stop(=loss) rate:** 2014 1.5% (5/341) · 2015 1.2% · 2016 1.4% · 2017 1.3% · 2018 0.9% · 2019 0.8% · 2020 1.7% · 2021 1.5% · 2022 1.6% · 2023 1.9% · 2024 1.0% · 2025 2.2% (10/462) · **2026 7.2% (23/318)** ← single-year cell; regime drift descriptive only, unstable cell.
**Session:** Asia 1.8% (n=1,843) · London 1.8% (n=1,049) · NewYork 1.6% (n=1,053) · Overlap 2.1% (n=918) — flat.
**Direction:** BUY 2.1% (n=3,612) vs SELL 1.0% (n=1,251) — descriptive 2× ratio; different bases, no inference claimed.

## 6. Divergence block (QC filter applied)

`DivSpan > 0` holds for **4,863 / 4,863 rows (100%)** — the Fable filter is **vacuous in this dataset** (reported, not repaired). Within it: only `PriceDivStrength` top-quartile has a stable n (lift 1.59, 35/1216); `RSIDivStrength` (1.32 *U*) and `DivergenceAngle` (1.14 *U*) are unstable. Pivot RSI medians are near-identical across outcome groups. **No strong, independently usable divergence-association to stop/loss was found at n=88.**

## 7. PositionOpen Shift=0 lens (separate; fill-time)

9,605 legs (step counts: 1→4,864 / 2→2,970 / 3→1,149 / 4→431 / 5→191); all `IndicatorShiftUsed=0`. Step-1 legs (n=4,864, stop 88):

| Feature (fill-time, shift-0) | stop med | non-stop med | Q4 lift (y/n) |
|---|---|---|---|
| **M1_ATR_PCT_PRICE** | **0.0302 [0.015, 0.0587]** | **0.0205 [0.015, 0.0309]** | **1.91** (42/1216) |
| **M1_ATR** | **0.705** | **0.350** | **2.01** (45/1238) |
| M1_ADX | 24.52 | 26.30 | 0.82 (18/1216) |
| M1_DI_GAP | 4.84 | 5.21 | 1.05 (23/1216) |
| M1_RSI | 48.55 | 49.39 | 0.86 (19/1216) |

Consistent direction with the shift-1 lens, and consistent with volatility-dominant stop clustering. (fill-price quartile association is a price-level proxy/mixture of eras — noted, not used.)

## 8. The seven requested answers

1. **Main REAL stop pattern:** the stop cohort concentrates in **high-volatility / wide-grid-geometry conditions at signal time** — median `ATRDistance` 2.32 vs 1.15 (tail lift ×2.17) and `M1_ATR` 0.93 vs 0.46 (×2.14), with `TickVolume` (×1.95) and `SpreadPoints` (×1.71) tails in the same direction; spread-to-ATR ratio is *lower* at stops (stops occur in high-ATR moments). Structurally: stop = fixed $300 basket stop ⇒ the loss tail is the dollar-stop event itself.
2. **Main REAL loss pattern:** in this run **loss ≡ stop** (perfect overlap) — no non-stop negative closes exist; all engineered exits (TP1/TP2/TP3/NET_BASKET_BE) end ≥ 0. Any future "loss" research on this profile is de facto stop research.
3. **Features worth entering Feature Research (exploratory shortlist, association-grade):** `ATRDistance` (grid geometry), `M1_ATR`/ATR-family across TFs, `TickVolume`, `SpreadPoints`/`SpreadATRNorm`; second tier: `M1_ATRRegime`, and the divergence-quality tail (`PriceDivStrength`, `RSIDivStrength`) — all as **risk-tail associations, not decision rules**.
4. **Findings that are EXPLORATORY ONLY:** everything above — single-run, in-sample, univariate Q4-lift descriptions; features are mutually entangled (ATR↔ATRDistance↔TickVolume), so no independency is claimed; nothing is a validated predictor.
5. **Findings NOT statistically reliable:** every cell flagged *UNSTABLE* (y<30) — including ALL divergence-quality cells except PriceDivStrength-tail; 2026 cell (23/318); the direction 2× ratio; the session spread; price-level proxies; and ANY multi-feature interaction (never tested).
6. **Still missing data:** RiskLab_Raw_* and SignalRegistry_* (absent → NOT PROVABLE items); confirmed-vs-live signal-source variation (constant `PROFILE_D_FULL` at step-1; `LADDER` only on later legs); trade-cost/slippage fields; per-leg stop mechanics ≥ step 2 beyond labels; an independent second run for robustness.
7. **Is current data sufficient to continue research?** YES for exploratory stop/loss pattern work at setup level (4,863 setups, 88 stops, 0% miss on analyzed columns, clean shift separation) and at leg level (9,605 legs). NO for stable subgroup inference — the 88-stop ceiling caps Q4 cells; any confirmation needs at minimum a second independent run (e.g., different symbol/period) before any pattern graduates beyond exploratory.

## 9. Constraints honored

No guessed/generated numbers (every figure computed from the listed CSVs); `stop_flag` and `pnl_negative` kept independent then observed identical (a data fact, reported as such); Snapshot Shift=1 and PositionOpen Shift=0 strictly separated; C2/M2 decision-log family excluded from the main chain; only features with known timestamp moment used; associations only, zero causal claims; no code change; no new backtest; no threshold extracted; no strategy/production decision — **Production = M0**. Fable QC constraints applied mechanically and reported where vacuous/unsplittable; nothing became a strategy rule.
