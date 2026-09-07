# C2/M2 — Approved Threshold Derivation (Shift-1, DEV)

**Status:** THRESHOLD_DERIVATION = **FAIL**
**Date:** 2026-09-07
**Production:** M0 (unchanged)
No data was derived, no threshold value was produced, and no OOS data was used.

---

## 1. Intent

Execute the pre-registered, approved threshold method:

```
M0 baseline
→ Event=A
→ StepFrom=3
→ StepTo=4
→ first valid decision per Setup
→ FeatureShift=1 / closed bar
→ DEV only (2014-01-01 → 2023-12-31 23:59:59)

ATRPct_Shift1 → Q75   (τ_ATR)
ADX_Shift1     → Q75   (τ_ADX)
```

Hard rule used in this document: *if any part of the population or Shift=1 cannot be proven, derivation is FAIL and no provisional value is produced.*

---

## 2. Scope / non-changes

- No production change.
- No rule change.
- No executable-code change.
- No Full Run, OOS evaluation, TRUE_FORWARD, Sweep/Tuning, Freeze.
- No performance verdict.

---

## 3. Data sources considered

| Source | Commit / branch | Notes |
|---|---|---|
| `v2_research_export_v2_cf/CF_events_m0.csv` | `662a7ec` / `agent-regression-data-real` | M0 baseline, **Shift 0** (forming-bar features, read via `CF_ShouldVeto`/`CF_LogEvent` with `GetBufferValue(...,0)`; `iClose(...,0)`) |
| `v2_research_export_v2_cf/PositionOpenDataset_XAUUSD.csv` | `662a7ec` | all `IndicatorShiftUsed=0` |
| `v2_research_export_v2_cf/RX_..._Features.csv` | `662a7ec` | `M1_Shift=1`, **but this is the Step1 snapshot only**, not Step3→4 |
| `v2_research_export_v2_cfv2/*` | `e716f7b` | smoke/test outputs |
| `C2M2_Step34_Decisions_m2.csv` | **not present** in `e716f7b` | required Shift-1 Step3→4 evidence **missing** |
| `rule_freeze.json` | **not present** in any reachable commit | referenced in source comments only |

---

## 4. Population analysis (documented, but NOT usable)

Using the **valid but Shift-0** M0 baseline, I still computed the *candidate* population to show why it must not be used:

- `CF_events_m0.csv` total rows: **19,087**
- `Event=A`, `StepFrom=3`, `StepTo=4`: **431 rows**
- unique `SetupID`: **431**
- first valid decision per Setup: **431**
- DEV subset (2014-01-01 → 2023-12-31 23:59:59): **366**
- OOS subset (2024-01-01 onward): **65**

These are **Shift-0** values and therefore do **not** satisfy the approved `FeatureShift=1 / closed-bar` requirement.

### Descriptive stats of the Shift-0 DEV subset (informational only — NOT derived thresholds)

Feature: `ATRPct100` (Shift-0), n=366

| metric | value |
|---|---:|
| Min | 0.005747 |
| Q25 | 0.02637275 |
| Median | 0.04187550 |
| Q75 | 0.06754475 |
| Q90 | 0.10072500 |
| Max | 0.33578100 |

Feature: `ADX` (Shift-0), n=366

| metric | value |
|---|---:|
| Min | 13.35000 |
| Q25 | 29.453075 |
| Median | 39.795300 |
| Q75 | 50.359850 |
| Q90 | 60.594700 |
| Max | 85.226500 |

> The Shift-0 ATRPct Q75 = `0.06754475`, which is exactly the **M1 forming-bar reference** already documented. This is **not** the C2/M2 Shift-1 threshold and must not be frozen as such.

---

## 5. Why derivation FAILS

### 5.1 Missing required Shift-1 Step3→4 population
- The approved population requires **closed-bar / Shift=1** feature values at the **Step3→4 first-valid decision**.
- The only Shift-1 artifacts in the repo are the **RX Step1 snapshot** (`RX_..._Features.csv`, `M1_Shift=1`), which is **Step1 only**.
- `CF_events_m0.csv` and `PositionOpenDataset` are **Shit-0** (`IndicatorShiftUsed=0` for all Step rows, source reads index `0` / `Close[0]`).
- Therefore **FeatureShift=1 at Step3→4 is NOT provable from the repository.**

### 5.2 Missing `C2M2_Step34_Decisions_m2.csv`
- The valid C2/M2 evidence export from the repaired Shift-1 module is not present in `e716f7b` (`git ls-tree` for `C2M2*` is empty).
- Without that export, there is no actual Shift-1 Step3→4 population to compute Q75 from.

### 5.3 Missing Freeze / provenance source
- No `rule_freeze.json`, no frozen values, no derivation log, no input-hash provenance exists in the repo.
- The source comments say "from rule_freeze.json", but that file is absent.

### 5.4 OOS contamination check
- OOS was **not** used in this check (I did not derive anything).
- However, because no Shift-1 derivation was possible, there is **nothing to accept or reject** on OOS.

---

## 6. What would be required to make it PASS

1. A committed `C2M2_Step34_Decisions_m2.csv` (or equivalent Shift-1 Step3→4 evidence) containing:
   - `FeatureShift=1`
   - `BarOpenTime_Shift1`, `BarCloseTime_Shift1`
   - `strict_before_check=true`
   - `ATR_Shift1`, `ATRPct_Shift1`, `ADX_Shift1`, `DIPlus_Shift1`, `DIMinus_Shift1`, `PriceRef_Shift1`
   - `T_decision`
   - `Event=A`/M0-style decision rows or an explicit report that they are derived from the M0 baseline.
2. `FeatureShift=1` and `strict_before_check=true` proven for **every** population row.
3. `Event=A` + `StepFrom=3` + `StepTo=4` + first-valid-per-Setup + DEV filter applied.
4. Only then can Q75 be computed using a reproducible percentile method and recorded with full input hash/provenance.

---

## 7. Provenance (as requested)

| Item | Present? |
|---|---|
| dataset | M0 baseline `CF_events_m0.csv` (present) |
| population | Shift-0 only — Shift-1 population **absent** |
| split | DEV/OOS definition documented in `C2_M2_FORMAL_REGISTRATION_SPECIFICATION.md` |
| FeatureShift | **NOT PROVEN for Step3→4** |
| StepFrom=3 / StepTo=4 | present in M0 CF events (Shift-0) |
| first-valid-per-Setup | implemented in this check (431/431), Shift-0 |
| source commit | `662a7ec` (data), repaired source on `60a643a` |
| rule_freeze.json | **MISSING** |
| derivation log | **NOT produced** (derivation not permitted due to failed precondition) |

---

## Final Status

```
THRESHOLD_DERIVATION = FAIL
τ_ATR = NOT DERIVED (Shift-0 Q75=0.06754475 is M1 only, NOT C2/M2)
τ_ADX = NOT DERIVED (Shift-0 Q75=50.35985 not valid for C2/M2)
OOS_USED = NO
FREEZE_READY = NO
PRODUCTION = M0
```
