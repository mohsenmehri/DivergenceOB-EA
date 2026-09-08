# C2/M2 — M2 FULL BACKTEST RUN ATTEMPT (V1)

Status: **M2 FULL BACKTEST = INCOMPLETE**
Performance verdict: **NOT ISSUED**

This record documents that the requested M2 backtest was **attempted to be
prepared but could NOT be executed** in the current environment. No performance
metrics are reported and no verdict is issued, in accordance with the integrity
requirement.

---

## 1. What was done in this turn

- Re-fetched the correct source branch (`arena/01a06d3c-divergenceob-ea`,
  HEAD `c995cdf4bf525199ec5bea88163d8907f85c9698`).
- Confirmed the C2/M2 repaired code path is the current tracked source:
  `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5`
  - SHA-256: `327952595e1b1b04feedc8fb373874e7ee5624de6fbe007e97d75a3929734f9b`
  - No diff vs. the previous instrumentation commit `32d8b9b`.
- Confirmed the rule inputs in the EA are **not hard-coded**; they are tester
  inputs and must be supplied as specified:
  - `inp_cf_mode = 2`
  - `inp_cf_q75_atr_pct_34 = 0.04668375`
  - `inp_cf_q75_adx_34 = 47.50305`
  - `inp_cf_q75_atr_pct_23 = 0.0`
- Confirmed the M2 code path uses the required repaired behavior for
  `inp_cf_mode == 2`, `Step3→4`, latched decision, `FeatureShift=1`,
  `PriceRef=Close[1]`, `strict_before_check=true`, `BarCloseTime_Shift1 <
  T_decision`, and `DI_ADVERSE` (`DIMinus_Shift1 > DIPlus_Shift1` for BUY /
  `DIPlus_Shift1 > DIMinus_Shift1` for SELL).
- Confirmed main `2f04d01` population provenance used by the freeze:
  `C2M2_M0_Shift1_Population_DEV.csv` (SHA-256 `09d0a302...`).

## 2. Why the run could not be executed

This sandbox does **not** contain a MetaTrader 5 / MetaEditor installation,
an MQL5 compiler, a Strategy Tester execution engine, the broker's XAUUSD tick
history, or a compiled `.ex5`.

Identified absent artifacts:

| Required for execution | Present? |
|---|---|
| `metaeditor64.exe` / MQL5 compiler | No |
| `terminal64.exe` / Strategy Tester | No |
| Compiled `.ex5` | No |
| Broker XAUUSD real-tick history (2014-01-01 → 2026-08-18) | No |
| `C2M2_Step34_Decisions_m2.csv` from a completed run | No (not in repo) |

Therefore the requested full run **did not start**, and none of the required
outputs (Setups, Step1–5, Attempted/Vetoed/Placed/Filled/Missed, Net Profit,
PF, Win Rate, Max DD, SL/DeepStop/BE/TP-family, BUY/SELL, PnL vs M0, affected
setups, loss-tail reduction, profitable continuations lost, yearly, DEV/OOS,
discrepancies) can be reported truthfully.

## 3. Static integrity checks that ARE confirmed (not a run)

| Check | Static result |
|---|---|
| M2 path uses `inp_cf_mode == 2` | Confirmed |
| `Step3 → Step4` only | Confirmed (`cur_step == 3 && next == 4`) |
| `FeatureShift = 1` | Confirmed (`C2M2_FEATURE_SHIFT = 1`) |
| `PriceRef = Close[1]` | Confirmed (`iClose(..., 1)`) |
| `strict_before_check = (BarCloseTime_Shift1 < T_decision)` | Confirmed |
| Latched decision | Confirmed (`C2M2_Step34ShouldVeto` latch logic) |
| `DI_ADVERSE`, BUY/SELL as specified | Confirmed (`C2M2_DI_ADVERSE_DELTA = 0`, `mdi - pdi > 0` / `pdi - mdi > 0`) |
| Threshold inputs are placeholders `0.0` in source | Confirmed – they must be set in the tester |
| No frozen-rule re-calculation | Confirmed – values from freeze record are reused verbatim |
| No OOS tuning | Confirmed – nothing was run or tuned |

## 4. What is needed before an actual run can be performed

An MT5 terminal with the broker's XAUUSD M1 real-tick history, a MetaEditor
compile of the current `.mq5`, then the Strategy Tester run with:

| Setting | Value |
|---|---|
| Symbol | same XAUUSD as prior M0 run |
| Timeframe | M1 |
| Model | Every tick based on real ticks |
| Start | `2014.01.01` |
| End | `2026.08.18 23:59:58` |
| Optimization | No |
| Visual Mode | OFF |
| `inp_cf_mode` | `2` |
| `inp_cf_q75_atr_pct_34` | `0.04668375` |
| `inp_cf_q75_adx_34` | `47.50305` |
| `inp_cf_q75_atr_pct_23` | `0.0` |
| `inp_c2m2_collect_shift1_m0` | `false` |
| Deposit / Leverage / other settings | same as prior M0 run |
| Production | M0 |

## 5. Final status

```
M2 FULL BACKTEST = INCOMPLETE
PERFORMANCE VERDICT = NOT ISSUED
```

No freeze, code change, production change, or rule change was made.
