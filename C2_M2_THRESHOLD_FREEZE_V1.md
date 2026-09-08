# C2/M2 — Threshold Freeze V1

Status: **THRESHOLD FREEZE = PASS**

This is the formal freeze record for the C2/M2 thresholds derived from the
approved M0 Shift-1 DEV population. No backtest, OOS run, code change,
production change, or threshold re-calculation was performed.

## Frozen thresholds

```
τ_ATR = 0.04668375
τ_ADX = 47.50305
```

## Method

```
Method = DEV Q75 — Quantile Type-7
```

Quantile Type-7 uses `h = (N - 1) * p` and linear interpolation between the
adjacent order statistics at index `floor(h)` and `ceil(h)`.

## Population

```
Event = A
Step  = 3 -> 4
FeatureShift = 1
Mode = 0
Rule = M0_COLLECT
DEV  = 2014-01-01 00:00:00 .. 2023-12-31 23:59:59
strict_before_check = true
BarCloseTime_Shift1 < T_decision
```

Validation on the population file:

| Check | Result |
|---|---|
| Total rows read | 358 |
| Population rows after all filters | 358 |
| Unique SetupID | 358 |
| Duplicate SetupID | 0 |
| Missing / NaN / Invalid ATRPct_Shift1 | 0 |
| Missing / NaN / Invalid ADX_Shift1 | 0 |
| FeatureShift = 1 | 358 / 358 |
| strict_before_check = true | 358 / 358 |
| BarCloseTime_Shift1 < T_decision | 358 / 358 |

No OOS or outcome data is used.

## Source

| Field | Value |
|---|---|
| Repository | `mohsenmehri/DivergenceOB-EA` |
| Branch | `main` |
| Commit | `2f04d019fca4b355a692d4471e1be08cb76f81f9` |
| Source file | `v2_research_export_v2_cfv2/C2M2_M0_Shift1_Population_DEV.csv` |
| File SHA-256 | `09d0a3028520a499624a82f02ab45a9f23ee6f6bff51ffb71c4637fcc06b13f5` |
| Git blob SHA | `35c92727711ef27114893fe4e4e809bb064b6092` |

## Rule definition

```
VETO =
    (ATRPct_Shift1 >= τ_ATR)
AND (ADX_Shift1 >= τ_ADX)
AND (DI_ADVERSE)

DI_ADVERSE:
    BUY  : DIMinus_Shift1 > DIPlus_Shift1
    SELL : DIPlus_Shift1  > DIMinus_Shift1
```

## δ_DI

```
δ_DI = 0
```

## Notes

- The M1/Shift-0 value `0.06754475` was NOT used.
- The placeholder input value `0.0` was NOT treated as a frozen threshold.
- These frozen values are recorded in this artifact only. No code/input file
  was changed.

## Status block

```
THRESHOLD FREEZE = PASS
τ_ATR = 0.04668375
τ_ADX = 47.50305
METHOD = DEV Q75 Type-7
δ_DI = 0
OOS USED = NO
BACKTEST = NO
CODE CHANGE = NO
PRODUCTION = M0
```
