# C2/M2 — M2 Full Backtest Result Analysis (actual run output on GitHub)

Status: **M2 FULL BACKTEST = COMPLETE (run was performed by Owner in MT5; file set exists on GitHub).**
Performance verdict (M2 vs M0 research evidence): **INCONCLUSIVE — matched-M0 comparison not possible from current artifacts.**
No code change, threshold change, freeze change, or production change was made.

---

## 1. Source / provenance

| Item | Value |
|---|---|
| Repository | `mohsenmehri/DivergenceOB-EA` |
| Commit | `0e91848010505f4d32d56ea874d8f17b6c067d4f` |
| Folder | `v2_research_export_v2_cfv2/` |
| Primary MT5 report | `ReportTester-91306235.xlsx` |
| M2 decision CSV | `C2M2_Step34_Decisions_m2.csv` |
| Position dataset | `PositionOpenDataset_XAUUSD.csv` |
| Position summary | `PositionOpenSummary_XAUUSD.txt` |
| Symbol / period in report | XAUUSD / M1 (2014.01.01 – 2026.08.19) |
| Position summary timestamp | `2026.08.18 23:59` |
| Company / currency | LiteFinance Global LLC / USD |
| Initial Deposit / Leverage | 500,000 / 1:100 |
| History Quality | 100% real ticks |
| Bars / Ticks | 4,128,799 / 555,972,017 |
| Inputs in report | `inp_cf_mode=2`, `inp_cf_q75_atr_pct_34=0.04668375`, `inp_cf_q75_adx_34=47.50305`, `inp_cf_q75_atr_pct_23=0.0`, `inp_c2m2_collect_shift1_m0=false`, `TestProfile=3`, `inp_reset_setup_id=true`, `inp_enable_export=false`, `inp_fast_backtest_mode=false` |

The requested threshold inputs are the values actually present in the MT5
report, so no re-calculation or re-freeze occurred.

---

## 2. Data integrity

### M2 decision log (`C2M2_Step34_Decisions_m2.csv`)

| Check | Result |
|---|---|
| Rows | 425 |
| Unique `SetupID` | 425 |
| Duplicate `SetupID` | 0 |
| `StepFrom` / `StepTo` | `3` / `4` for all rows |
| `Mode` | `2` for all rows |
| `Rule` | `M2` for all rows |
| `FeatureShift` | `1` for all rows |
| `LatchStatus` | `FIRST` for all rows (latched once per setup) |
| `C2M2_Decision` counts | ALLOW 367 / VETO 49 / INVALID 9 |
| `strict_before_check=true` | ALLOW + VETO (416 rows) all true; **9 INVALID rows are false** |
| `BarCloseTime_Shift1 < T_decision` | true for ALLOW + VETO (416); **false for the 9 INVALID rows** |
| Threshold applied in rows | `0.04668375` and `47.50305` re-checked row-by-row; no mismatch with logged VETO/ALLOW |
| DI_ADVERSE | BUY: `DIMinus > DIPlus`; SELL: `DIPlus > DIMinus`; no mismatch |
| Event column value | `C2M2_DECISION` for all rows (not `A`; this is the M2 decision log naming, not the population file) |

**Discrepancies in the decision log:**

1. `Event` is `C2M2_DECISION`, not `A`. The `A` convention belongs to the
   population collection file; the M2 run uses the decision-log label. This is a
   labeling difference, not a rule difference, but should be noted.
2. 9 INVALID rows exist. They occur when `BarCloseTime_Shift1 == T_decision`
   (strict-before fails). They are fail-open: not vetoed, and they proceed to
   Step4. Therefore strict-before is **not** true for 100% of logged rows; it is
   true for 100% of ALLOW/VETO rows and 0% of INVALID rows.
3. INVALID rows contain real feature values (not missing) but are excluded from
   the veto decision.

### Veto statistics

- Veto rate (all 425 decisions): `49 / 425 = 11.53%`.
- Veto rate among non-INVALID decisions: `49 / 416 = 11.78%`.
- ALLOW + INVALID = `367 + 9 = 376`, which exactly equals the number of Step4
  position rows in the run. So the 49 vetoed setups are the only veto-driven
  Step4 removals in this run.

---

## 3. M2 performance (actual run output)

### Headline from MT5 report (authoritative)

| Metric | M2 value |
|---|---|
| Total Net Profit | **-1,614.62** |
| Gross Profit | 36,426.14 |
| Gross Loss | -38,040.76 |
| Profit Factor | **0.957556** |
| Win Rate (total trades) | **57.91%** (5,432 / 9,380) |
| Expected Payoff | -0.172134 |
| Balance Drawdown Absolute | 3,270.48 |
| Equity Drawdown Absolute | 3,299.59 |
| Balance Drawdown Maximal | 3,473.90 (0.69%) |
| Equity Drawdown Maximal | 3,490.33 (0.70%) |
| Total Trades | 9,380 |
| Total Deals | 18,760 |
| Short Trades (won %) | 2,299 (61.46%) |
| Long Trades (won %) | 7,081 (56.76%) |
| Largest profit / loss trade | +115.76 / -172.09 |

### Setup / Step funnel (actual)

| Metric | M2 value | Source |
|---|---|---|
| Setup count (unique) | 4,782 | PositionOpenDataset |
| Position rows | 9,380 | PositionOpenDataset / MT5 report |
| Step1 | 4,782 | PositionOpenDataset |
| Step2 | 2,921 | PositionOpenDataset |
| Step3 | 1,132 | PositionOpenDataset |
| Step4 | 376 | PositionOpenDataset |
| Step5 | 169 | PositionOpenDataset |
| Attempted Step3→4 decisions | 425 | C2M2 file |
| Vetoed at Step3→4 | 49 | C2M2 file |
| ALLOW | 367 | C2M2 file |
| INVALID / fail-open | 9 | C2M2 file |
| Placed/Filled at Step4 | 376 | = ALLOW + INVALID |
| Missed at Step4 | 0 (within attempted set) | inferred from the above |
| BUY rows / SELL rows | 7,081 / 2,299 | MT5 report |

### Dataset-derived PnL decomposition (price PnL only, not MT5 net)

Using `PositionOpenDataset_XAUUSD.csv`; `PositionRealizedProfit` is the EA
price-move PnL. **It excludes commissions/swaps/etc. MT5 net is -1,614.62.**

| Category (unique setup final event) | Setups | Price PnL |
|---|---|---|
| SL$ family (`SL$`) | 90 | -27,395.40 |
| BE family (`NET_BASKET_BE`) | 323 | +1,497.37 |
| TP-family | 4,368 | +24,456.91 |
| EOT/censored | 1 | 0.00 |
| Total (unique setup price PnL) | 4,782 | -1,441.12 |

| Side | Position rows | Price PnL |
|---|---|---|
| BUY | 7,081 | -3,047.36 |
| SELL | 2,299 | +1,606.24 |

| Step | Position rows | Price PnL |
|---|---|---|
| Step1 | 4,782 | -1,526.34 |
| Step2 | 2,921 | -508.64 |
| Step3 | 1,132 | +242.55 |
| Step4 | 376 | +619.45 |
| Step5 | 169 | -268.14 |

`PositionOpenSummary_XAUUSD.txt` reports: All Opened Positions 9,380;
Setup STOP rows 365; Setup STEP5 rows 845; Setup SUCCESS rows 9,012;
Setup LOSS rows 368; Setup DEEPSTOP rows 257.

---

## 4. M2 vs M0

The Owner-provided M0 reference is:

| Metric | M0 |
|---|---|
| Setups | 4,864 |
| Position rows | 9,605 |
| Step1/2/3/4/5 | 4,864 / 2,970 / 1,149 / 431 / 191 |
| Net Profit | -626.69 |
| SL$ | -26,928.14 |
| BE | +1,661.37 |
| TP-family | +24,640.08 |
| BUY / SELL | -2,680.13 / +2,053.44 |
| Censored at EOT | 1 setup (#4864) |

**Critical parity discrepancy:** the M2 run has a different underlying
population signature (4,782 setups / 9,380 position rows, Step funnel
4,782/2,921/1,132/376/169). The M0 reference has 4,864 setups / 9,605 rows and
a larger funnel at every step (4,864/2,970/1,149/431/191). This is not a
matched pair.

### M2 vs M0 table (with explicit non-comparability warning)

| Metric | M0 reference | M2 actual | Delta (M2 − M0) | Interpretation |
|---|---|---|---|---|
| Setups | 4,864 | 4,782 | -82 | Base population differs; not a treatment effect |
| Position rows | 9,605 | 9,380 | -225 | Base population differs |
| Step1 | 4,864 | 4,782 | -82 | Base population differs |
| Step2 | 2,970 | 2,921 | -49 | Base population differs |
| Step3 | 1,149 | 1,132 | -17 | Base population differs |
| Step4 | 431 | 376 | -55 | 49 veto + 6 base-population difference |
| Step5 | 191 | 169 | -22 | Base population differs + veto downstream |
| Net Profit | -626.69 | -1,614.62 (MT5 net) | -987.93 | M2 worse in this run, but **not comparable** |
| SL$ | -26,928.14 | -27,395.40 (price PnL) | -467.26 | Not comparable |
| BE | +1,661.37 | +1,497.37 (price PnL) | -164.00 | Not comparable |
| TP-family | +24,640.08 | +24,456.91 (price PnL) | -183.17 | Not comparable |
| BUY | -2,680.13 | -3,047.36 (price PnL) | -367.23 | Not comparable |
| SELL | +2,053.44 | +1,606.24 (price PnL) | -447.20 | Not comparable |

Because the M0 reference and the M2 run do not share the same base population
signature, **none of the deltas above can be attributed to the M2 veto alone.**
A matched M0 re-run with the same EA build, broker, symbol, real-tick history,
start/end, deposit, leverage and inputs is required before any causal delta.

---

## 5. Veto analysis

- 49 setups vetoed at Step3→4, all with `LatchStatus=FIRST`, `MaxStep=3`.
- Of the 49:
  - 19 were already in an `SL$` final event at Step3; their Step3 price PnL
    sum is **-5,743.89**.
  - 30 were already in a TP/positive final state at Step3; their Step3 price
    PnL sum is **+175.75**.
  - Total Step3 price PnL of vetoed setups: **-5,568.14**.
- Therefore the veto set is **mixed**: it removes 19 setups that were already
  losers at Step3 and 30 setups that were already winners at Step3.
- The 30 positive-at-Step3 setups are the "profitable continuation lost"
  candidates; they were not allowed to enter Step4/5. Their actual Step4/5
  outcome is **unknown** because the veto prevented it, so **lost-continuation
  profit is not measurable from the provided files**.

---

## 6. Loss-tail

- Not provable as a causal reduction from the provided files.
- Reason: loss-tail reduction requires a matched M0 counterfactual (what the
  49 vetoed setups would have done at Step4/5) and a matched M0 run with the
  same base population. Neither is present.
- Observed: the vetoed set contains both Step3 losers and Step3 winners, so
  the rule is not a clean "remove only bad tails" filter. The 9 INVALID/fail-open
  rows also bypass the gate and reach Step4/5.

---

## 7. Temporal analysis

- Run reaches the requested cutoff: max position open time is
  `2026-08-18 19:57:18`; PositionOpenSummary timestamp is `2026.08.18 23:59`.
- C2/M2 decision rows: min `2014-01-08`, max `2026-07-28`.
- **Net annual / DEV / OOS profit from MT5 (including commissions) is NOT
  extractable from the provided files** because no per-trade net table is
  present in `ReportTester-91306235.xlsx` (it is a summary report without a
  trades sheet).

Auxiliary dataset-derived price-PnL (excludes commissions/swaps):

| Period | Position rows | Dataset price PnL |
|---|---|---|
| DEV 2014–2023 | 7,136 | -3,176.71 |
| OOS 2024–2026-08-18 | 2,244 | +1,735.59 |

Per-year dataset price-PnL (excludes commissions/swaps):

| Year | Rows | Price PnL |
|---|---|---|
| 2014 | 714 | -513.53 |
| 2015 | 703 | -278.17 |
| 2016 | 552 | -242.77 |
| 2017 | 646 | -308.63 |
| 2018 | 724 | +70.35 |
| 2019 | 719 | -260.43 |
| 2020 | 777 | -76.15 |
| 2021 | 770 | -557.40 |
| 2022 | 823 | -256.80 |
| 2023 | 708 | -753.18 |
| 2024 | 847 | +571.15 |
| 2025 | 811 | +1,798.29 |
| 2026 (to cutoff) | 586 | -633.85 |

These are **not** the MT5 net PnL and must not be used as a performance verdict.

---

## 8. Parity / coverage

- M2 run is internally consistent: Step position rows sum to 9,380 = MT5 Total
  Trades; ALLOW + INVALID = Step4 rows = 376; ProfileStats D_FULL matches the
  dataset funnel.
- **M0 reference is not parity-matched to the M2 run.** Setups, position rows
  and every step count differ.
- The 9 INVALID/fail-open rows are a coverage/integrity gap: they entered
  Step4/5 without a valid strict-before decision.
- No missing/NaN features observed on ALLOW/VETO rows; the only invalid state is
  the 9 strict-before failures.

---

## 9. Final verdict

```
DATA INTEGRITY = PARTIAL PASS (416/425 strict-before valid; 9 INVALID/fail-open rows; Event label is C2M2_DECISION not A)
M2 FULL BACKTEST = COMPLETE
M2 vs M0 PERFORMANCE COMPARISON = NOT COMPARABLE (base population mismatch)
RESEARCH EVIDENCE VERDICT = INCONCLUSIVE
```

**Reason:** The M2 run actually completed and its rule/thresholds are correctly
applied, but (a) the M0 reference does not match the M2 run's base population,
(b) MT5 net profit is negative (-1,614.62) and worse than the un-matched M0
reference, (c) the veto set removes both Step3 losers and Step3 winners, and
(d) 9 INVALID/fail-open rows bypass strict-before. Without a matched M0 re-run,
no causal improvement or harm can be asserted.

---

## 10. Owner summary (≤10 lines)

- M2 run completed to cutoff; thresholds and DI rule were applied correctly.
- M2 MT5 net = -1,614.62, PF 0.9576, 9,380 trades; dataset SL/BE/TP price PnL
  = -27,395.40 / +1,497.37 / +24,456.91.
- M2 removed 49 Step4 setups: 19 Step3 losers (-5,743.89) and 30 Step3 winners
  (+175.75) → not a clean "tail-only" filter.
- 9 INVALID/fail-open rows bypassed strict-before; `Event` is `C2M2_DECISION`
  not `A`.
- M0 reference is **not parity-matched** (4,864 vs 4,782 setups; funnel differs),
  so M2-vs-M0 deltas are **not causal**.
- Research verdict: INCONCLUSIVE; no advancement to next stage based on this
  comparison.
- Next step: run a matched M0 baseline with same EA build/inputs/broker/real
  ticks/date window/deposit/leverage, then compare on the same base population.
