# C2/M2 — Implementation Repair Specification V1
### Name: `C2_M2_IMPLEMENTATION_REPAIR_SPEC_V1.md`

**Purpose:** design-level specification to repair the *actual* C2/M2 Step3→4 implementation so it matches the approved C2/M2 specification, **without executing or committing any code change**.

**Status / verdict (this document):** `REPAIR SPEC READY FOR REVIEW`

**Production:** **M0** (unchanged).

---

## 1. Scope

This document is an **Implementation Repair Specification**. It is produced after the final Shift-1 audit, which established:

```
SHIFT=1 STEP3→4 = NOT PROVABLE
```

Current source (`29dfb15`, `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5`) actually uses **Shift 0** for the Step3→4 feature path:

- ATR / ADX / DI+ / DI− → buffer index **0**
- PriceRef → `iClose(..., 0)`
- `PositionOpenDataset` Step4 rows → `IndicatorShiftUsed = 0`

This spec defines the required repair **design** only. It does **not**:
- modify any `.mq5`,
- create any code commit,
- derive/change any threshold,
- touch the current threshold values,
- change the C2/M2 rule,
- decide δ_DI,
- freeze anything,
- run OOS / TRUE_FORWARD / backtest / sweep / tuning,
- change Production.

The approved C2/M2 specification is **not** changed by this document.

---

## 2. Current Implementation

All locators refer to `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` at commit `29dfb15` (SHA-256 of source: `567b94a2bb15fbd5f41c4d9fd4b601b0f8a5370e99b17e00e95b492965e91e3b`).

### 2.1 Step3→4 decision entry

| # | Item | Locator |
|---|---|---|
| 1 | Ladder manager | `void ManagePositions(int idx)` — line 42331 |
| 2 | Current step read | `int cur_step = g_setups[idx].current_step;` — line 42334 |
| 3 | Next step computed | `int  next = cur_step + 1;` — line 42336 |
| 4 | For Step3→4 | `cur_step = 3`, `next = 4` |
| 5 | Entry condition | `tick_price <= next_entry` (BUY) / `tick_price >= next_entry` (SELL) — lines 42340–42342 |
| 6 | Market gate | `if(!IsMarketOpen())` — line 42374 |
| 7 | Spread gate | `if(!IsSpreadOK())` — line 42382 |
| 8 | Margin gate | `OrderCalcMargin(...)` / free-margin check — lines 42395–42408 |
| 9 | CF (C2/M2) gate | `bool cf_veto = CF_ShouldVeto(cur_step, next, is_bull);` — line 42432 |
| 10 | CF event log | `CF_LogEvent(idx, cur_step, next, ...)` — line 42433 |
| 11 | Place order | `PlaceEntry(...)` — line 42449 |
| 12 | Post-fill log/records | `CF_LogEvent(...,"P"/"F",...)`, `RecordPositionOpenBySetup(...)`, `RX_RecordStepOpen(...)` — lines 42457, 42470, 42473, 42477 |

### 2.2 Shift=0 locations (the broken path)

| # | Locator | What is wrong |
|---|---|---|
| 1 | `CF_ShouldVeto` line 19219 | `double price_tf = iClose(_Symbol, g_tf[mi].timeframe, 0);` → **PriceRef = Close[0]** |
| 2 | `CF_ShouldVeto` line 19221 | `double atr = GetBufferValue(buffer_atr, 0);` → **ATR = ATR[0]** |
| 3 | `CF_ShouldVeto` line 19222 | `double adx = GetBufferValue(buffer_adx, 0);` → **ADX = ADX[0]** |
| 4 | `CF_ShouldVeto` lines 19223–19224 | `pdi/mdi ... 0` → **DI+[0] / DI−[0]** |
| 5 | `CF_LogEvent` lines 19300–19305 | Same index-0 reads (ATR/ADX/DI+, DI−, RSI, EMA200) |
| 6 | `CopyIndicatorBuffers` lines 7523–7527 | `CopyBuffer(..., 0, count, buffer)` — buffer index 0 = forming bar |
| 7 | `GetBufferValue` lines 7617–7623 | returns `buffer[index]`; index 0 = first (forming) element |
| 8 | `RecordPositionOpenBySetup` line 18364 | `rec.indicator_shift_used = 0;` |
| 9 | `RecordPositionOpenBySetup` lines 18380–18422 | all indicator reads use index 0 |
| 10 | `PositionOpenDataset` export | `IndicatorShiftUsed` column is `0` for **all** rows (Step4 = 431 × 0) |
| 11 | Narrative note | `PositionOpenSummary_XAUUSD.txt` line in source (`ExportPositionOpenSummaryReport`, line 19062) says `shift=1`, **contradicted** by source/data |

### 2.3 Current latch state

- There is **no latch**. Source comment at line 42423 explicitly states: `No latch: every tick re-evaluated.`
- `CF_ShouldVeto` and `CF_LogEvent` are repeatable per tick.
- Duplicate Step3→4 events exist (A, then P on success, then F on fill). There is no per-setup/per-step one-shot decision latch.

### 2.4 Current thresholds / mode

- `inp_cf_mode = 0` (default; M0 baseline, log only) — line 968.
- `inp_cf_q75_atr_pct_34 = 0.0`, `inp_cf_q75_adx_34 = 0.0` — lines 969–970.
- `inp_cf_q75_atr_pct_23 = 0.0` — line 971.
- δ_DI is **undetermined** (Owner Decision Needed) and is not present as a runtime gate.
- In the real M0 export, all Step3→4 CF events are `Mode=0`, `Rule=NONE`, `Event=A/P/F` (no veto).

---

## 3. Specification Requirement

The approved C2/M2 specification requires the Step3→4 actual decision path to be exactly:

```
Step3→4
→ Shift=1
→ ATR[1]
→ ADX[1]
→ DI+[1] / DI−[1]
→ PriceRef = Close[1]
→ ATRPct_Shift1
→ C2/M2 decision
→ LATCH
→ PlaceEntry(Step4)
```

**Hard invariants from the approved spec (not to be changed):**
1. The C2/M2 rule is **Step3→4 only**.
2. Feature timeframe is **M1**; symbols: `_Symbol` (XAUUSD in the regression set).
3. Feature shift is **1 = last completed bar**.
4. `PriceRef = Close[1]` (M1).
5. `ATRPct_Shift1 = ATR_Shift1 * 100 / PriceRef_Shift1`.
6. The C2/M2 decision is **LATCHED** for a given `(SetupID, StepFrom=3, StepTo=4)`.
7. No forming-bar (`shift=0`) and no future-bar data may enter the C2/M2 decision.

---

## 4. Required Repair

The repair is **design-level**; no code is written here. Every item below is a proposal for the implementation team.

### R-1 — Feature read shift (core repair)
- Change the M1 feature reads used by the C2/M2 Step3→4 gate from index `0` to index `1`:
  - `ATR_Shift1 = GetBufferValue(buffer_atr, 1)`
  - `ADX_Shift1 = GetBufferValue(buffer_adx, 1)`
  - `DIPlus_Shift1 = GetBufferValue(buffer_plus_di, 1)`
  - `DIMinus_Shift1 = GetBufferValue(buffer_minus_di, 1)`
  - `PriceRef_Shift1 = iClose(_Symbol, g_tf[mi].timeframe, 1)`
- This applies to the **decision function** used for Step3→4.
- For M0 (`inp_cf_mode == 0`), the function must still return `false` **before** reading features, so trading behavior is unchanged (but see §11 for the M0 export conflict).

### R-2 — Separate or renamed feature computation
- To avoid touching any non-C2/M2 path, introduce an isolated function, e.g. `CF_GetM1Shift1Features(..., SFeatureSnapshot &out)`, callable from `CF_ShouldVeto` and from the evidence exporter.
- The old index-0 reads remain untouched outside this C2/M2 path (e.g. general snapshot code, `RecordPositionOpenBySetup`, etc.).

### R-3 — Latch
- Add per-setup step-transition latch state (see §8).
- The latch is scoped to `(SetupID, StepFrom=3, StepTo=4)` only.
- It must be `true` **only** when the C2/M2 gate has made its first valid decision for that scope.

### R-4 — Evidence export
- Add the Shift-1 evidence fields listed in §9 to the Step3→4 event row (either by extending `CF_events_*` or, to preserve M0 export byte-parity, by adding a separate export — see §11 open decision).

### R-5 — `IndicatorShiftUsed` alignment
- `RecordPositionOpenBySetup` currently sets `indicator_shift_used = 0`. The repair proposal must either:
  - add a new field `c2m2_feature_shift = 1` when the record corresponds to a C2/M2 Step3→4 decision, **or**
  - keep `indicator_shift_used` as the position-open live-capture field (0) and add a separate C2/M2 evidence export.
- Do **not** silently change the meaning of the existing `IndicatorShiftUsed` column without versioning (it currently means "0 = live state captured at actual fill time", source struct comment line 2838).

### R-6 — Provenance
- Add a provenance header to every C2/M2 evidence export: repo, branch/commit, source file SHA-256, ATR period, ADX period, timeframe, `FeatureShift=1`, input-mode snapshot, threshold hashes, rule version, date.

### R-7 — No other changes
- No threshold, no δ_DI, no rule change, no Production change.

---

## 5. Shift=1 Semantics

### 5.1 Definition
- `Shift=1` = **last completed M1 bar** relative to the decision moment.
- Timeframe: `g_tf[TF_IDX_MAIN].timeframe`; the EA is chart-locked to `PERIOD_M1` (source line 15648).
- Symbol: `_Symbol` (XAUUSD in the regression run).
- Indicator identities:
  - `ATR_Shift1` = value at buffer index 1 of `iATR(_Symbol, PERIOD_M1, inp_atr_period)` with `inp_atr_period = 4`.
  - `ADX_Shift1` = main-axis value at buffer index 1 of `iADX(_Symbol, PERIOD_M1, inp_adx_period)` with `inp_adx_period = 14`.
  - `DIPlus_Shift1` / `DIMinus_Shift1` = PLUSDI/MINUSDI lines at buffer index 1 of the same ADX handle.
  - `PriceRef_Shift1` = `iClose(_Symbol, PERIOD_M1, 1)`.

### 5.2 How to read the buffer
- Keep `CopyIndicatorBuffers(MIN_BUFFER_DEPTH)` reading from start position `0` (this is required so the buffer array is populated from the current bar backward), then read **index 1** via `GetBufferValue(buffer, 1)`.
- Buffer arrays are series arrays in MQL5 `CopyBuffer` output: index `0` = current (forming) bar, index `1` = last completed bar.
- No feature may be read from index `0` in the C2/M2 gate.

### 5.3 Validity / fallback
- If `PriceRef_Shift1 <= 0` or any required indicator is invalid, the decision must be **explicitly marked invalid** (recorded as `INVALID`/`UNKNOWN`) rather than silently falling back to Bid/Ask.
- **Conflict to record:** current code falls back to `Bid/Ask` when `iClose(...,0) <= 0` (lines 19219–19222). This is incompatible with `PriceRef = Close[1]`. The repair must remove the fallback from the C2/M2 feature path (or treat it as an evidence error). What the gate should do on invalid data (allow / veto / no-decision) is an **open decision**; this spec does not resolve it.

---

## 6. PriceRef Semantics

- Required exact definition:
  - `PriceRef_Shift1 = Close[1]` on the **same symbol and timeframe** as the decision features: `iClose(_Symbol, PERIOD_M1, 1)`.
  - `ATRPct_Shift1 = (ATR_Shift1 * 100.0) / PriceRef_Shift1`.
- Current implementation uses:
  - `double price_tf = iClose(_Symbol, g_tf[mi].timeframe, 0);` → `Close[0]`.
  - `atr_pct = (price_tf > 0.0 && atr > 0.0) ? (atr * 100.0 / price_tf) : 0.0;`
- Export must record:
  - `PriceRef_Shift1`
  - `ATR_Shift1`
  - `ATRPct_Shift1`
- The `Bid`/`Ask` columns may remain for diagnostics, but they are **not** the PriceRef.

---

## 7. `T_decision`

### 7.1 Definition
- `T_decision` = the server timestamp at which the **first valid Step3→4 decision** is made.
- Required ordering:

```
Market Gate  →  Spread Gate  →  Margin Gate  →
FIRST VALID DECISION  →  C2/M2 LATCHED VETO  →  PlaceEntry(Step4)
```

- "First valid decision" = the first tick where:
  1. `cur_step == 3`,
  2. `next == 4`,
  3. market / spread / margin gates all pass,
  4. the C2/M2 gate is actually invoked.

### 7.2 Recommended timestamp source
- Recommended: `T_decision = TimeCurrent()` at the CF gate call, on the same tick as the feature read.
- Also record `tick.time` (`MqlTick.time`, obtained from `SymbolInfoTick(_Symbol, tk)`) for reconciliation.
- **Conflict to record:** current export uses `TimeCurrent()` for `DecisionTime` (line 19312) but the entry-level condition uses `tick_price` from `SymbolInfoTick`. `TimeCurrent()` and `tick.time` can be shifted by a tick/second and may straddle a bar boundary; the spec must state which value is authoritative for the temporal invariant. This spec leaves it as an open decision, with the recommendation that both be exported and the invariant be checked against both.

### 7.3 Placement order requirement
- The existing source already has the required ordering internally: market → spread → margin → CF gate → PlaceEntry (lines 42374–42449).
- The repair must **not** reorder; it must only add the latch at the CF-gate point.

---

## 8. LATCH Semantics

### 8.1 State
- Add a per-setup, per-transition latch:
  - key: `(SetupID, StepFrom = 3, StepTo = 4)`
  - state: `NONE` → `VETO` or `ALLOW` (or `INVALID` if evidence invalid, depending on the open decision in §5.3).
- Reset behavior: cleared when the setup's final outcome is finalized / setup is removed (not retained across setups).

### 8.2 Set time
- Only at the **first valid decision** described in §7.1.
- If state is already `SET`, the C2/M2 gate must not re-evaluate; it must return the latched outcome.

### 8.3 Outcomes
| First valid decision | Latch value | Next action |
|---|---|---|
| VETO → | `VETO` | No `PlaceEntry(Step4)` for this setup; no further CF re-evaluation for this scope. |
| ALLOW → | `ALLOW` | Continue to `PlaceEntry(Step4)` for this tick. |

### 8.4 Duplicate prevention
- Prevent duplicate veto by checking the latch **before** calling `CF_ShouldVeto`.
- Prevent duplicate ALLOW events by making the evidence exporter emit the C2/M2 decision row once per latch set (one row of `Event=DECISION` with `Decision=ALLOW/VETO`), while keeping the existing `A/P/F` ladder events separate.
- **Conflict to record:** current code emits `A`, `P`, `F` on every attempt (and can emit repeated A on every tick while the step is still open). The repair must not accidentally remove the existing `P/F` event semantics; it should add a distinct latched decision row rather than overloading `A`.

### 8.5 Open design point: ALLOW latch and failed PlaceEntry
- If `ALLOW` is latched but `PlaceEntry` fails (e.g. transient order failure), the spec needs an explicit decision on retry:
  - Option A: `ALLOW` latch is final and no retry (strict).
  - Option B: `ALLOW` latch is the *C2/M2* decision only; the existing `PlaceEntry` retry loop remains separate and may retry order placement without re-running the C2/M2 veto.
- This is recorded as an **open decision**; the design preference in this spec is **Option B** (does not change current ladder resilience), but it is not resolved here.

---

## 9. Evidence Instrumentation

### 9.1 Required fields per Step3→4 C2/M2 decision row

| Field | Type | Source / computation |
|---|---|---|
| `SetupID` | int | `g_setups[setup_idx].setup_id` |
| `StepFrom` | int | `3` |
| `StepTo` | int | `4` |
| `T_decision` | datetime | `TimeCurrent()` at first valid decision (see §7) |
| `FeatureShift` | int | literal `1` |
| `BarOpenTime_Shift1` | datetime | `iTime(_Symbol, PERIOD_M1, 1)` |
| `BarCloseTime_Shift1` | datetime | `iTime(_Symbol, PERIOD_M1, 1) + 60` (= start of current M1 bar, `iTime(...,0)`) |
| `ATR_Shift1` | double | `GetBufferValue(buffer_atr, 1)` |
| `ATRPct_Shift1` | double | `ATR_Shift1 * 100 / PriceRef_Shift1` |
| `ADX_Shift1` | double | `GetBufferValue(buffer_adx, 1)` |
| `DIPlus_Shift1` | double | `GetBufferValue(buffer_plus_di, 1)` |
| `DIMinus_Shift1` | double | `GetBufferValue(buffer_minus_di, 1)` |
| `PriceRef_Shift1` | double | `iClose(_Symbol, PERIOD_M1, 1)` |
| `strict_before_check` | bool | `(BarCloseTime_Shift1 < T_decision)` |
| `tick_time` | datetime | `tk.time` (reconciliation) |
| `C2M2_Decision` | string | `ALLOW` / `VETO` / `INVALID` |
| `LatchStatus` | string | `FIRST` / `LATCHED_SKIP` / `NONE` |
| `provenance/source hash` | string | file SHA-256 + commit + version |

### 9.2 Export mechanism (design proposal)
- Either extend the existing `CF_events_m0.csv`/`CF_events_m2.csv` headers **with a clearly versioned header**, or add a dedicated export, e.g.:
  - `C2M2_Step34_Decisions_m{inp_cf_mode}.csv`
- The dedicated export is preferred because it avoids changing existing `CF_events_*` semantics and makes the M0 regression gate cleaner (see §11).

### 9.3 PositionOpenDataset alignment
- Add a C2/M2-oriented field (e.g. `C2M2_Step34_FeatureShift`) with value `1` on the Step4 row when the row is the first valid C2/M2 decision for that setup.
- Do not modify the existing meaning of `IndicatorShiftUsed`.

---

## 10. No-Lookahead Proof

The repair must make the following proof **testable**;

### 10.1 Static proof
- Grep / lint the C2/M2 gate to confirm:
  - no `GetBufferValue(... , 0)` call exists inside the C2/M2 feature function,
  - no `iClose(..., 0)`, `iHigh(...,0)`, `iLow(...,0)`, `iOpen(...,0)` is used for C2/M2 features,
  - `PriceRef_Shift1` is read from index `1`,
  - the only index-0 use in the C2/M2 path is the `CopyBuffer(..., 0, count, ...)` population (which is allowed and necessary).

### 10.2 Runtime proof
- Compute `BarOpenTime_Shift1`, `BarCloseTime_Shift1`, and `T_decision` on the same tick.
- Assert `strict_before_check == true`, i.e. `BarCloseTime_Shift1 < T_decision`.
- Export `strict_before_check` per decision row.
- If `strict_before_check == false`, the row must be marked `INVALID` and must not be used as a decision record.

### 10.3 Future-leak proof
- All indicators needed at `T_decision` are historical-series values completed before `T_decision`.
- `M1_Forming` equivalent must be `0` in the C2/M2 evidence row (the closed bar is used).
- Feature export must not contain any value that depends on the bar(s) after `T_decision`.
- Re-run reconciliation against the previous Step1 RX Shift=1 snapshot technique (`RX_FEATURE_SHIFT=1`, line 12790) as a cross-check that the same bar-time alignment is being used.

---

## 11. M0 Preservation

### 11.1 What must be preserved
- **Trading/order behavior** in M0 must be byte-identical at the order layer:
  - same entry triggers, same market/spread/margin behavior,
  - same `PlaceEntry(Step4)` decisions (M0 never vetoes),
  - same `current_step` progression, same lifecycle events,
  - same position/fill records (except where a new evidence field is added).
- Production stays **M0**.
- No threshold/rule/δ_DI change.

### 11.2 Known conflict: M0 research feature export
- If `CF_LogEvent` is changed to read Shift-1 values, `CF_events_m0.csv` feature columns will change from Shift-0 to Shift-1 **for all historical/new M0 rows**. This changes a research artifact, not trading behavior.
- Two design options:
  - **Option A (transparent versioning):** keep `CF_events_m0.csv` exactly as-is (Shift-0 columns, not used by C2/M2) and add a **new** `C2M2_*` Shift-1 evidence export. M0 export byte-parity is preserved.
  - **Option B (unify semantics):** change `CF_LogEvent` to Shift-1 across all modes so the CF export represents the actual C2/M2 feature path; requires M0 regression gate to accept a *documented, intentional* research-export semantic change.
- This spec recommends **Option A** for strict M0 preservation, but records it as an **open decision** because it affects whether the existing `CF_events_*` files remain comparable.

### 11.3 Guarantee pattern (design)
- `inp_cf_mode == 0` returns before feature computation, so `CF_ShouldVeto` cannot affect trading.
- The C2/M2 latch applies only to the Step3→4 decision when `inp_cf_mode > 0` (or when the C2/M2 rule is explicitly enabled).
- All index-1 reads are isolated inside the C2/M2 computation function; general snapshot code and `RecordPositionOpenBySetup` remain untouched.

---

## 12. Validation Plan

This is a **specification only**; none of the following are executed in this task.

### 12.1 Compile
- Compile the repaired `.mq5` in Strategy Tester / MetaEditor.
- Gate: no error/warning introduced in the CF/C2/M2 feature path; no changes to unrelated code paths.

### 12.2 Smoke Test
- Run M0 with the same inputs on a short window.
- Gate: same number of setups, same Step4 fills, same lifecycle event counts as expected M0 baseline; `inp_cf_mode=0` produces no veto.

### 12.3 Shift=1 Evidence Audit
- Run the repaired C2/M2 evidence export for Step3→4.
- Gate:
  - `FeatureShift = 1` for all rows,
  - `strict_before_check = true` for all rows,
  - `PriceRef_Shift1` values are non-zero and equal `Close[1]` at `T_decision`,
  - all 431 (or current population) Step3→4 rows carry complete Shift-1 features,
  - no index-0 feature is used in the decision path (static proof).

### 12.4 M0 Regression Gate
- Compare M0 trading outputs (placements, fills, lifecycle, position-open step counts) against the pre-repair M0 baseline.
- If Option A is used, also assert `CF_events_m0.csv` is byte-identical (except for the new evidence file).
- Gate: zero trading-output delta.

### 12.5 C2/M2 Evidence Reconciliation
- Reconcile the new C2/M2 Shift-1 evidence against:
  - the approved C2/M2 specification,
  - the M0 population (the same 431 Step3→4 decisions),
  - RX Step1 Shift=1 snapshot methodology,
  - the Shift-0 `PositionOpenDataset`/`CF_events_m0` files (to document that they are legacy forming-bar records and are no longer the C2/M2 authority).
- Gate: the C2/M2 evidence is self-consistent and temporally valid; any remaining discrepancy is documented, not silently resolved.

---

## 13. Risks / Open Decisions

1. **M0 export byte-parity vs Shift-1 unification** (§11.2). Recommendation: **Option A** (new evidence export). Owner must decide.
2. **PriceRef fallback conflict** (§5.3). Current code falls back to Bid/Ask; target requires `Close[1]`. No-veto vs invalid-row behavior is undecided.
3. **M1/M3 side effect** (§4 R-1). `CF_ShouldVeto` is shared by M1 (3→4), M2 (3→4), and M3 (2→3). Changing the shared read to Shift-1 changes M1/M3 semantics too, which the C2/M2 spec does not cover. Recommendation: keep M1/M3 unchanged at Shift-0 unless separately approved.
4. **Latch semantics for ALLOW + PlaceEntry retry** (§8.5). Strict-latch vs retry-preserving behavior is undecided. Recommendation: **Option B** (C2/M2 decision latched; order retry loop unchanged).
5. **`T_decision` authority** (§7). `TimeCurrent()` vs `tick.time` may straddle a bar boundary. Both should be exported.
6. **δ_DI undetermined.** The M2 rule includes an adverse-DI condition implicitly through source (`di_against`), but the δ_DI owner decision is still pending. C2/M2 cannot be considered activated or frozen until resolved.
7. **Threshold placeholders.** `inp_cf_q75_*` are `0.0`; thresholds are not derived here and must not be touched. No freeze/DEVELOPMENT Q75 derivation is performed.
8. **Historical Shift-0 rows cannot be retro-fixed.** All past `CF_events_m0.csv`/`PositionOpenDataset` Step4 rows are Shift-0 and must be treated as legacy forming-bar records, not C2/M2 Shift-1 evidence.
9. **Terminal log / ReportTester absent.** Not needed to produce this spec, but remains a general evidence gap for any future full freeze clearance.
10. **Provenance storage location.** Whether provenance lives in a CSV header, a sidecar manifest, or a separate `Manifest.json` is undecided. Recommendation: sidecar manifest keyed by export file name + SHA-256.

---

## 14. Explicit Non-Changes

- No `.mq5` file is changed in this task.
- No code commit is created in this task.
- No threshold is derived or changed.
- No existing threshold value is manipulated.
- The C2/M2 rule is not changed.
- δ_DI is not decided.
- No freeze is issued.
- No OOS / TRUE_FORWARD / backtest / sweep / tuning executed.
- Production is **not** changed and remains **M0**.
- No pre-existing artifact is modified.

---

## 15. Final Repair Recommendation

The implementation repair should consist of:

1. **Isolated C2/M2 Shift-1 feature function** reading `ATR[1]`, `ADX[1]`, `DI+[1]`, `DI−[1]`, `PriceRef=Close[1]`, computed with `ATRPct_Shift1`, with no index-0 fallback.
2. **Per-`(SetupID,3,4)` latch** evaluated at the first valid post-market/spread/margin decision, with a distinct latched-decision evidence row.
3. **Dedicated `C2M2_Step34_Decisions_*` evidence export** (recommended, Option A) recording all fields in §9, plus provenance.
4. **Temporal invariant** `BarCloseTime_Shift1 < T_decision` asserted and exported per row.
5. **M0 unchanged at the trading/order layer**, M1/M3 paths left unchanged unless separately approved.
6. **Validation plan** (§12) executed only after explicit Owner approval.

The **approved C2/M2 specification is unchanged**. This document is a design proposal for aligning the implementation with that specification and explicitly records all conflicts listed in §13 for Owner decision.

---

**FINAL**

```
REPAIR SPEC READY FOR REVIEW
```

Production remains **M0**. No code was changed. No test, backtest, OOS, TRUE_FORWARD, sweep, or tuning was executed. Commit of this documentation-only artifact follows.
