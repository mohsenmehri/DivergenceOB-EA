# C2/M2 Implementation Repair V1 — Implementation Report

**Artifact:** `C2_M2_IMPLEMENTATION_REPAIR_V1.md`
**Date:** 2026-09-07
**Production:** **M0** (unchanged)
**Stage performed:** Coding → Static/Code Review. Real Backtest, Smoke Run, OOS, TRUE_FORWARD, Sweep/Tuning, Freeze, Production change are **NOT** performed.
**Owner-approved decisions applied:** D1–D8, D10–D12 approved; **δ_DI = 0** (D9).

---

## 1. Baseline SHA

| Item | Value |
|---|---|
| Source file | `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` |
| Baseline commit | `29dfb1529293c10b364014464977a150074da25f` |
| Baseline file SHA-256 | `567b94a2bb15fbd5f41c4d9fd4b601b0f8a5370e99b17e00e95b492965e91e3b` |
| Baseline line count | 51,944 |
| Baseline extracted from | `git show 29dfb15:...` (byte-verified before edit) |

## 2. Modified Files

Only one source file and this report are added/changed:

| File | Type | Status |
|---|---|---|
| `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` | source | **MODIFIED (repair)** |
| `C2_M2_IMPLEMENTATION_REPAIR_V1.md` | documentation | **NEW** |

No `.replay` data, no other source/`.mq5`, no production file, no threshold file was modified.

## 3. Exact Functions / Locations Changed

### 3.1 `CSetupTracker` (latch fields)
- **Lines 12622–12623** — added:
  - `int c2m2_step34_latch;` (`0=NONE,1=ALLOW,2=VETO,3=INVALID`)
  - `datetime c2m2_step34_latch_time;`
- **Lines 12661–12662** — `Reset()` initializes both to `0`.

### 3.2 New isolated C2/M2 module (inserted before `bool CreateSetupWithPrediction`)
- **Line 19337** — block comment `C2/M2 STEP3->4 REPAIR V1`.
- **Lines 19342–19349** — constants:
  - `C2M2_FEATURE_SHIFT = 1`
  - `C2M2_DI_ADVERSE_DELTA = 0.0`
  - `C2M2_STEP_FROM = 3`, `C2M2_STEP_TO = 4`
  - provenance strings (`C2M2_PROVENANCE_SOURCE_COMMIT`, `_SPEC_VERSION`, `_BUILD`).
- **Lines 19351–19356** — `ENUM_C2M2_LATCH` (`NONE`, `ALLOW`, `VETO`, `INVALID`).
- **Line 19358** — `bool C2M2_IsValidNumber(double v)` helper.
- **Line 19363** — `void C2M2_LogStep34Decision(...)` — evidence export writer.
- **Line 19412** — `bool C2M2_Step34ShouldVeto(...)` — isolated Shift-1 latched C2/M2 decision.

### 3.3 `ManagePositions` call site
- **Lines 42580–42597** — the shared CF-gate block now branches:
  ```
  is_c2m2 = (inp_cf_mode == 2 && cur_step == 3 && next == 4)
  if(is_c2m2) -> C2M2_Step34ShouldVeto(...)
  else        -> legacy CF_ShouldVeto + CF_LogEvent (unchanged)
  ```
- Legacy `CF_ShouldVeto` / `CF_LogEvent` bodies are **byte-identical** to baseline.

## 4. Shift=1 Implementation

- Feature read index changed from `0` to `1` **only** in the new C2/M2 module:
  - `ATR_Shift1  = g_tf[mi].GetBufferValue(g_tf[mi].buffer_atr, 1)`
  - `ADX_Shift1  = g_tf[mi].GetBufferValue(g_tf[mi].buffer_adx, 1)`
  - `DIPlus_Shift1  = g_tf[mi].GetBufferValue(g_tf[mi].buffer_plus_di, 1)`
  - `DIMinus_Shift1 = g_tf[mi].GetBufferValue(g_tf[mi].buffer_minus_di, 1)`
  - `BarOpenTime_Shift1 = iTime(_Symbol, g_tf[mi].timeframe, 1)`
- Symbol: `_Symbol`; timeframe: `g_tf[TF_IDX_MAIN].timeframe` = M1 (chart locked to `PERIOD_M1`).
- Indicator periods unchanged: `ATR(4)`, `ADX(14)` (with DI+ / DI− from the same ADX handle).
- `FeatureShift = 1` is hard-coded and exported in every evidence row.

## 5. PriceRef

- `PriceRef_Shift1 = iClose(_Symbol, g_tf[mi].timeframe, 1)`.
- `ATRPct_Shift1 = ATR_Shift1 * 100 / PriceRef_Shift1`.
- No Bid/Ask fallback in the C2/M2 path (Owner-approved D1). If `PriceRef_Shift1 <= 0` or any required feature is invalid, the row is exported as `C2M2_Decision = INVALID` and the gate is **fail-open** (no veto).

## 6. T_decision

- `T_decision = tk.time` (Owner-approved D2) — the tick that triggered the Step3→4 decision.
- `DecisionServerTime = TimeCurrent()` is exported alongside `T_decision` for reconciliation with existing artifacts.
- The decision is reached only after Market Gate → Spread Gate → Margin Gate, before `PlaceEntry(Step4)`.

## 7. Temporal Invariant

- `BarCloseTime_Shift1 = iTime(_Symbol, PERIOD_M1, 1) + PeriodSeconds(PERIOD_M1)`.
- `strict_before_check = (BarCloseTime_Shift1 < T_decision)` — **strict `<`** (Owner-approved D4), per row.
- If `strict_before_check == false`, row is exported as `INVALID` and gate is fail-open.

## 8. LATCH

- Owner-approved **D5 Option B**: C2/M2 decision latch is independent of `PlaceEntry` retry.
- Per `(SetupID, StepFrom=3, StepTo=4)`:
  - `NONE` → first valid decision sets latch to `ALLOW`, `VETO`, or `INVALID`.
  - `VETO` → subsequent calls return veto (no Step4 placement).
  - `ALLOW` → subsequent calls return no-veto and the existing `PlaceEntry` retry loop may run (retry preserved).
  - `INVALID` → subsequent calls return no-veto (fail-open, permanent for that setup/step).
- Duplicate veto prevention: latch checked **before** any C2/M2 re-evaluation.
- Latch is stored on the `CSetupTracker` instance and cleared by `Reset()` when the setup slot is reused.

## 9. C2/M2 Isolation

- The new `C2M2_*` functions are invoked **only** when:
  `inp_cf_mode == 2 && current_step == 3 && next == 4`.
- M0 (`inp_cf_mode == 0`) never reaches the new code.
- M1 and M3 continue to use the original `CF_ShouldVeto` / `CF_LogEvent` path unchanged.
- `C2M2_Step34ShouldVeto` also contains an internal `inp_cf_mode != 2` guard.

## 10. Evidence Export

New dedicated export (Owner-approved D7 Option B): `C2M2_Step34_Decisions_m{inp_cf_mode}.csv`.

Columns written (Owner-approved D8 + provenance):

```
Event;SetupID;StepFrom;StepTo;T_decision;DecisionServerTime;FeatureShift;
BarOpenTime_Shift1;BarCloseTime_Shift1;ATR_Shift1;ATRPct_Shift1;
ADX_Shift1;DIPlus_Shift1;DIMinus_Shift1;PriceRef_Shift1;
strict_before_check;C2M2_Decision;LatchStatus;Direction;Mode;Rule;
SourceCommit;SpecVersion;Build
```

- `Event = C2M2_DECISION`
- `C2M2_Decision = ALLOW | VETO | INVALID`
- `LatchStatus = FIRST` on first decision; later re-evaluations are suppressed by the latch.
- `Rule = M2`, `Mode = inp_cf_mode`.

The existing `CF_events_m0.csv` / `CF_events_m2.csv` are **not** relabeled; if the M2 3→4 A/V rows are no longer written to the shared CF log, the shared log is not treated as C2/M2 evidence.

## 11. M0 Preservation

- M0 branch (`inp_cf_mode != 2`) is byte-identical to the baseline shared gate path.
- M0 never calls `C2M2_Step34ShouldVeto`.
- New latch fields on `CSetupTracker` are initialized to `0`/`0` and do not affect M0 order logic.
- New evidence file is only opened when the C2/M2 path is reached (mode 2, Step3→4).
- No change to thresholds, inputs, `CF_ShouldVeto`, `CF_LogEvent`, `PositionOpenDataset`, RX code, or `PlaceEntry`.

## 12. No-Lookahead Safeguards

1. C2/M2 features are read from **buffer index 1** (last completed M1 bar) only.
2. Index 0 is used only by `CopyIndicatorBuffers(..., 0, count, ...)` to populate the buffer (allowed, necessary).
3. `PriceRef = Close[1]`, not `Close[0]`, not `Bid/Ask`.
4. `strict_before_check` is exported per row and asserted before a valid decision is accepted.
5. The forming-bar column is never used as a C2/M2 input.
6. `FeatureShift = 1` is hard-coded and exported, so the evidence is self-verifying.

## 13. Compile Result

**Compile result: NOT EXECUTED — BLOCKED.**

- No MetaEditor / MQL5 compiler (`metaeditor`, `runmeta`, `mql5`) is available in this sandbox.
- Therefore the required `Compile` stage cannot be verified here.
- Static review was performed instead (see §14).

## 14. Static / Code Review

Performed without executing anything:

1. **Diff review** — exactly 4 hunks:
   - `CSetupTracker` latch fields + reset (2 hunks)
   - new C2/M2 module (+146/-0 lines)
   - `ManagePositions` branch (+11/-3 lines)
2. **Balance review** — new region braces balanced (`{`==`}` == 7); whole-file raw brace count balanced (5,427 == 5,427).
3. **Identifier review** — all new identifiers defined before use; no new identifier collides with existing names.
4. **Isolation review** — M0/M1/M3 path calls `CF_ShouldVeto` + `CF_LogEvent` exactly as within 1-indent-of-baseline; `C2M2_Step34ShouldVeto` has an internal `inp_cf_mode != 2` guard.
5. **No-lookahead review** — C2/M2 reads are index `1` / `Close[1]`; `strict_before_check` exported.
6. **M0 regression review** — no change to M0 path, no change to thresholds, no new trade placement logic reached in M0.

## 15. Remaining Risks

1. **Compile not verified** — actual MQL5 compilation must be run in a MetaEditor/MT5 environment before any evidence validation or regression run.
2. **MQL5 reference syntax** — the new code uses `const MqlTick &tk` (already used in the baseline file) and direct array-element member access; no local `&` variable is used in the new code. Still requires compiler confirmation.
3. **`INVALID` is latched fail-open** — if `CopyIndicatorBuffers` or the strict-before check fails on the first decision, the setup/step is permanently treated as no-veto and only one `INVALID` row is emitted. This is a design choice; if a later valid decision is required, a future refinement would need a separate `INVALID_LOGGED` flag.
4. **Threshold values are still 0.0** — `inp_cf_q75_atr_pct_34` and `inp_cf_q75_adx_34` are placeholders and **were not changed**. With these values, `high34`/`high_adx` are false, so the C2/M2 gate will currently record `ALLOW` rows (no veto) until thresholds are explicitly authorized.
5. **δ_DI = 0** is hard-coded as `C2M2_DI_ADVERSE_DELTA = 0.0` (strict adverse-DI sign comparison), matching the Owner-approved value.
6. **M2 shared CF log change** — the M2 Step3→4 `A/V` shared `CF_LogEvent` rows are no longer emitted (the isolated C2/M2 export is the authority). The M2 `P/F` placement rows remain. This should be documented for any M2 regression comparison.
7. **Historical Shift-0 data** remains Shift-0 and is **not** relabeled.

---

## Final

| Item | Value |
|---|---|
| **Commit** | `final commit reported externally in the task response` |
| **Modified source SHA-256** | `842eec36547538b719bd8798a766dd535423196bb1a519185dac904226c61f07` |
| **Artifact SHA-256** | `reported externally in the task response (self-referential value is not embedded)` |
| **Modified source blob SHA-1** | `fb4245046f896d2236bb9f32ec75a261bc531f59` |
| **Compile result** | `NOT EXECUTED — BLOCKED` (no MQL5 compiler in sandbox) |
| **Verification status** | Static/Code review completed |

**Verdict:**

```
REPAIR BLOCKED
```

**Reason:** the code repair is implemented and static review passes, but the required **Compile** step could not be executed in this environment (no MetaEditor / MQL5 compiler). Evidence validation and any regression run must not begin until the repaired `.mq5` compiles successfully in an MT5/MetaEditor environment.

Production remains **M0**. No Backtest, Smoke Run, OOS, TRUE_FORWARD, Sweep/Tuning, Freeze, or Production change was performed.
