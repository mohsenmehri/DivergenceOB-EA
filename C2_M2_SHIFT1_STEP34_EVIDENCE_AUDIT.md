# C2/M2 — Shift=1 / PriceRef at the Real Step3→4 Decision

**Audit type:** Blocker 1 evidence audit (audit-only)
**Date:** 2026-09-07
**Repository:** `mohsenmehri/DivergenceOB-EA`
**Branch (data):** `agent-regression-data-real` → `FETCH_HEAD f349093b5137f63ed7e9cf8b4833d86f76b9421c`
**Source ground-truth commit:** `29dfb1529293c10b364014464977a150074da25f`
**Source file:** `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5`
**Source SHA-256:** `567b94a2bb15fbd5f41c4d9fd4b601b0f8a5370e99b17e00e95b492965e91e3b`

---

## 1. Scope

This audit resolves **Blocker 1** only: can we prove, **without changing any code**, that the C2/M2 Step3→4 decision features (`ATR`, `ADX`, `DI+`, `DI−`, `PriceRef`) were computed with **Shift=1 / `Close[1]`** at the **actual Step3→4 decision time**, using source code and existing exported artifacts?

Evidence authority is limited to:
1. `.mq5` source code
2. exported CF event data
3. `PositionOpenDataset`
4. Lifecycle / RX data
5. other artifacts

M1 is **not** used as an evidence authority. External AI is not an evidence authority. `PositionOpenDataset` having a `Shift`-related field is **not** sufficient by itself; its connection to the Step3→4 decision must be traced through source.

**Hard prohibitions (unchanged):**
- No new threshold derivation.
- No threshold change.
- No C2/M2 freeze.
- No new rule registration.
- No `.mq5` change.
- No production change (Production = **M0**).
- No OOS / TRUE_FORWARD / backtest / sweep / tuning.
- No M1 or External AI authority.

---

## 2. Source Ground Truth

| Item | Value |
|---|---|
| Repo | `mohsenmehri/DivergenceOB-EA` |
| Source commit | `29dfb1529293c10b364014464977a150074da25f` |
| File | `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` |
| File SHA-256 | `567b94a2bb15fbd5f41c4d9fd4b601b0f8a5370e99b17e00e95b492965e91e3b` |
| File size / lines | 51,944 lines |
| CF mode input (default) | `inp_cf_mode = 0` → **M0 baseline, log only** (line 968) |
| CF thresholds (default) | `inp_cf_q75_atr_pct_34 = 0.0`, `inp_cf_q75_adx_34 = 0.0` (lines 969–970) |
| ATR period | `inp_atr_period = 4` (line 965) |
| ADX period | `inp_adx_period = 14` (line 974) |
| Chart TF guard | `_Period != PERIOD_M1` → refuse to run (line 15648) |
| Main TF | `g_tf[TF_IDX_MAIN] = g_tf[0].Initialize(PERIOD_CURRENT, …)` (line 44602) → **M1** |

The actual exported data used in this audit is the `v2_research_export_v2_cf` replay under `.replay/`.

---

## 3. Actual Step3→4 Code Path

The Step3→4 ladder decision is made inside `ManagePositions(int idx)` (**line 42331**).

```cpp
void ManagePositions(int idx)
{
   if(!g_setups[idx].is_active) return;
   int cur_step = g_setups[idx].current_step;      // line 42334
   if(cur_step >= 5) return;
   int  next = cur_step + 1;                        // line 42336
```

For a Step3→4 decision: `cur_step = 3`, `next = 4`.

The decision pipeline inside `ManagePositions` is:

1. **Level touch / entry condition** — `tick_price <= next_entry` (BUY) or `tick_price >= next_entry` (SELL) (lines 42340–42342).
2. **Optional bar-reclaim confirmation** — `STEP_CONFIRM_BAR_RECLAIM` uses `iClose(...,1)` / `iHigh(...,1)` / `iLow(...,1)` plus failsafe ATR from `GetBufferValue(buffer_atr, 1)` (lines 42350–42361). Note: this uses **Shift=1**, but it is the *bar-reclaim confirmation gate*, not the C2/M2 feature set. In the data export the default input is `inp_step_confirm_mode = STEP_CONFIRM_TOUCH` (line 913), so this branch is **not the active configuration evidence** here.
3. **Optional H1 trend guard** — `StepTrendGuardOK` reads H1 `ADX/DI+/DI−/EMA20/50/100/200` at **Shift=1** (lines 42310–42327). Default `inp_use_step_trend_guard = false` (line 918), so this is also **not** shown active in the M0 export.
4. **Market/spread/margin gates** (lines 42374–42419).
5. **CF (C2/M2) veto gate and event log** — **line 42432**:

```cpp
bool cf_veto = CF_ShouldVeto(cur_step, next, is_bull);                 // line 42432
CF_LogEvent(idx, cur_step, next, is_bull, tk, cf_veto ? "V" : "A", ...); // line 42433
if(cf_veto) return;
```

6. On successful fill, the code emits `P` and `F` events and calls `RecordPositionOpenBySetup(idx, next, …)` (**lines 42457, 42470, 42473**).

Therefore **the C2/M2 veto feature computation for Step3→4 is `CF_ShouldVeto(3, 4, is_bull)`**, called immediately before the Step4 order send.

---

## 4. Feature Calculation Path

### 4.1 `CF_ShouldVeto` — lines 19214–19240

```cpp
bool CF_ShouldVeto(int cur_step, int next, bool is_bull)
{
   if(inp_cf_mode <= 0) return false;                 // M0: no veto ever
   if(next != 3 && next != 4) return false;           // only 2->3 and 3->4
   int mi = TF_IDX_MAIN;                              // M1
   g_tf[mi].CopyIndicatorBuffers(MIN_BUFFER_DEPTH);
   double price_tf = iClose(_Symbol, g_tf[mi].timeframe, 0);   // <-- Shift 0
   if(price_tf <= 0.0)
      price_tf = is_bull ? SymbolInfoDouble(_Symbol, SYMBOL_BID)
                         : SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double atr     = g_tf[mi].GetBufferValue(g_tf[mi].buffer_atr, 0);       // <-- Shift 0
   double adx     = g_tf[mi].GetBufferValue(g_tf[mi].buffer_adx, 0);       // <-- Shift 0
   double pdi     = g_tf[mi].GetBufferValue(g_tf[mi].buffer_plus_di, 0);   // <-- Shift 0
   double mdi     = g_tf[mi].GetBufferValue(g_tf[mi].buffer_minus_di, 0);  // <-- Shift 0
   double atr_pct = (price_tf > 0.0 && atr > 0.0) ? (atr * 100.0 / price_tf) : 0.0;
   bool high34 = (inp_cf_q75_atr_pct_34 > 0.0) && (atr_pct >= inp_cf_q75_atr_pct_34);
   bool high23 = (inp_cf_q75_atr_pct_23 > 0.0) && (atr_pct >= inp_cf_q75_atr_pct_23);
   if(inp_cf_mode == 1 && next == 4) return high34;                 // M1
   if(inp_cf_mode == 3 && next == 3) return high23;                 // M3
   if(inp_cf_mode == 2 && next == 4)                                 // M2 (this rule)
   {
      bool high_adx    = (inp_cf_q75_adx_34 > 0.0) && (adx >= inp_cf_q75_adx_34);
      bool di_against  = is_bull ? (mdi > pdi) : (pdi > mdi);
      return high34 && high_adx && di_against;
   }
   return false;
}
```

### 4.2 `CF_LogEvent` — lines 19242 onwards

The exported decision rows use the **same** Shift-0 reads:

```cpp
int mi = TF_IDX_MAIN;                                  // M1
g_tf[mi].CopyIndicatorBuffers(MIN_BUFFER_DEPTH);
double price_tf = iClose(_Symbol, g_tf[mi].timeframe, 0);   // Shift 0
double atr      = g_tf[mi].GetBufferValue(g_tf[mi].buffer_atr, 0);  // Shift 0
double atr6     = g_tf[mi].GetBufferValue(g_tf[mi].buffer_atr, 5);
double adx      = g_tf[mi].GetBufferValue(g_tf[mi].buffer_adx, 0);  // Shift 0
double pdi      = g_tf[mi].GetBufferValue(g_tf[mi].buffer_plus_di, 0);  // Shift 0
double mdi      = g_tf[mi].GetBufferValue(g_tf[mi].buffer_minus_di, 0); // Shift 0
...
double atr_pct  = (price_tf > 0.0 && atr > 0.0) ? (atr * 100.0 / price_tf) : 0.0;
```

The exported header is:

```
Event;SetupID;DecisionTime;StepFrom;StepTo;Direction;Bid;Ask;SpreadPoints;
ATR;ATRPct100;ATRRegime;ADX;DIPlus;DIMinus;...
```

- `DecisionTime` = `TimeCurrent()` at the decision.
- `FillTime` = step fill time.
- There is **no** `BarOpenTime_Shift1`, `BarCloseTime_Shift1`, `PriceRef_Shift1`, `FeatureShift`, or `ATRPct_Shift1` column.

### 4.3 Buffer semantics — `CopyIndicatorBuffers` and `GetBufferValue`

`CopyIndicatorBuffers(count)` calls (lines 7523–7527):

```cpp
CopyBuffer(handle_atr, 0, 0, count, buffer_atr) > 0
CopyBuffer(handle_adx, MAIN_LINE,   0, count, buffer_adx) > 0
CopyBuffer(handle_adx, PLUSDI_LINE, 0, count, buffer_plus_di) > 0
CopyBuffer(handle_adx, MINUSDI_LINE,0, count, buffer_minus_di) > 0
```

- Start position `0` and `CopyBuffer` semantics mean buffer index `0` is the **current, forming bar** (as-series index).
- `GetBufferValue(buffer, 0)` returns the first element, i.e. the **forming-bar** value (lines 7617–7623).

### 4.4 Indicators used

Handles are created in `CreateIndicatorHandles` (lines 7472–7474):

```cpp
handle_atr = iATR(symbol, timeframe, atr_p);   // ATR(4) on M1
handle_adx = iADX(symbol, timeframe, adx_p);   // ADX(14) on M1
```

So the C2/M2 feature family is **M1 ATR(4), M1 ADX(14), M1 DI+(14), M1 DI−(14)**.

---

## 5. Shift Evidence

| Requirement | Source finding |
|---|---|
| 1. ATR with Shift=1 at Step3→4 | **No.** `CF_ShouldVeto` reads `GetBufferValue(buffer_atr, 0)` → Shift **0** (forming). |
| 2. ADX with Shift=1 at Step3→4 | **No.** `GetBufferValue(buffer_adx, 0)` → Shift **0**. |
| 3. DI+/DI− with Shift=1 at Step3→4 | **No.** `GetBufferValue(buffer_plus_di/minus_di, 0)` → Shift **0**. |
| 4. FeatureShift == 1 | **No.** No `FeatureShift` column/field exists in the Step3→4 path; the hard-coded index is **0**. |
| 5. `PriceRef == Close[1]` | **No.** Source uses `iClose(..., 0)` → **forming close**. |
| 6. Bar_CloseTime_Shift1 < T_decision | **Not exportable.** No bar-time fields on the Step3→4 decision records. |
| 7. No forming-bar feature in Step3→4 | **No.** The only recorded Step3→4 feature set comes from index **0** (forming). |

**Important parallel path (not C2/M2):** the source does contain Shift-1 reads near the Step3→4 decision, but in **different gates**:

- Bar-reclaim confirmation: `iClose(...,1)`, `iHigh(...,1)`, `iLow(...,1)` and failsafe `g_tf[0].GetBufferValue(buffer_atr, 1)` (lines 42352, 42358).
- H1 trend guard: `g_tf[h1].GetBufferValue(..., 1)` (lines 42314–42322).

These do **not** satisfy the requirement because:
1. They are entry-confirmation/deferral gates, not the C2/M2 feature values.
2. Their defaults (`STEP_CONFIRM_TOUCH`, `inp_use_step_trend_guard = false`) mean they are **not provably active** in the exported M0 data.
3. The C2/M2 rule specifically requires the **M1 ATR/ADX/DI/PriceRef feature set** at Shift=1; the source proves this set is read at Shift=0.

---

## 6. PriceRef Evidence

- The C2/M2 veto computes `price_tf = iClose(_Symbol, g_tf[mi].timeframe, 0)` and `atr_pct = atr*100/price_tf` (lines 19219–19228, 19302–19307).
- `PriceRef_Shift1 = Close[1]` is **not implemented** anywhere in the Step3→4 CF path.
- The exported CF row stores `Bid`, `Ask`, `ATR`, `ATRPct100`; it does **not** store `price_tf`, `Close[0]`, `Close[1]`, `BarOpenTime`, or `BarCloseTime`.
- The `PositionOpenDataset` Step4 row stores `FillPrice`, `PlannedEntryLevel`, and `M1_ATR_PCT_PRICE`, but **no** `PriceRef_Shift1`, `M1_Close`, or bar-time columns.
- RX Step1 features capture `CandleClose` from `g_current_snapshot` (Step1 signal state) and per-TF values from `iClose(..., RX_FEATURE_SHIFT)`; they do **not** export a `M1_Close` / `PriceRef_Shift1` column, and they are **Step1-only**.

Therefore **`PriceRef = Close[1]` at Step3→4 is not proven and is contradicted by source (which uses `Close[0]`).**

---

## 7. Timestamp Alignment

### 7.1 What exists

| Source | Timing fields | Bar-open/close fields | Shift field | Can prove `BarClose_Shift1 < T_decision`? |
|---|---|---|---|---|
| `CF_events_m0.csv` | `DecisionTime`, `FillTime` | none | none | **No** |
| `PositionOpenDataset_XAUUSD.csv` | `OpenTime` | none | `IndicatorShiftUsed` | **No** |
| RX Features (Step1) | `SignalTime`, `Step1FillTime`, `FeatureAsOfTime` | `M1_BarTime`, `M1_FormingBarTime` | `M1_Shift`, `M1_Forming` | **Only for Step1** |
| RX SetupLifecycleEvents | `EventTime`, `Step` | none | none | **No** |
| Step1_OutcomeComparison | `OpenTime` | none | none | **No** |

### 7.2 Verified RX Step1 alignment (for contrast)

From `RX_XAUUSD_RX_1388707200_XAUUSD_Features.csv`:

- 4,864 unique setup rows (one per setup).
- `M1_Shift = 1` for **4,864 / 4,864**.
- `M1_Forming = 0` for **4,864 / 4,864**.
- `M1_BarTime < FeatureAsOfTime` for **4,864 / 4,864** (typically 1 minute before).
- Closed-bar vs forming-bar ATR differ in **4,810 / 4,864** rows → Shift=1 vs Shift=0 are materially different.

This proves the RX **Step1** snapshot uses the last completed bar. It does **not** apply to Step3→4 because RX captures exactly one row per setup at `RX_CaptureStep1Snapshot` (line 12956), and `RX_RecordStepOpen` for steps 2–5 only records `step_time`, not feature snapshots (line 13123).

### 7.3 Step3→4 timing

- CF Step3→4 decision rows carry only `DecisionTime` and `FillTime`.
- `PositionOpenDataset` StepNo=4 rows carry only `OpenTime`.
- `RX` STEP4_OPEN lifecycle rows carry only `EventTime`.
- No bar-open/bar-close time for the Shift=1 bar is recorded at the Step3→4 decision.

**Conclusion:** strict `Bar_CloseTime_Shift1 < T_decision` is **not provable** for Step3→4 from existing artifacts.

---

## 8. Existing Artifact Evidence

All paths below are under `.replay/v2_research_export_v2_cf/`.

### 8.1 `CF_events_m0.csv`

- SHA-256: `642753751f89ceed346a6c069b4878f92bb87a5fb8b352e65e377ea1bc4ff74d`
- Total events: **19,087**
- Step3→4 events: `A=431`, `P=431`, `F=431`
- All Step3→4 rows: `Mode=0`, `Rule=NONE` → **M0, no veto**
- Max Step3→4 `DecisionTime`: `2026.07.28 01:00:06`
- No Shift/bar-time/PriceRef column.

### 8.2 `PositionOpenDataset_XAUUSD.csv`

- SHA-256: `4d013f581b3fa69e5d54a7957af218572c80e22ddbbd45f831f5bf02c77fc19d`
- Total rows: **9,605**
- Step counts: `1=4864`, `2=2970`, `3=1149`, `4=431`, `5=191`
- `IndicatorShiftUsed`:
  - Step1 = 4,864 × `0`
  - Step2 = 2,970 × `0`
  - Step3 = 1,149 × `0`
  - **Step4 = 431 × `0`**
  - Step5 = 191 × `0`
- Max StepNo=4 `OpenTime`: `2026.07.28 01:00:06`
- StepNo=4 setup set == CF Step3→4 decision setup set: **431/431 intersection**.
- For all 431 Step3→4 CF `A` rows, the values exactly equal the corresponding StepNo=4 row:
  - `ATR` == `M1_ATR` → 431/431
  - `ADX` == `M1_ADX` → 431/431
  - `DIPlus` == `M1_DI_PLUS` → 431/431
  - `DIMinus` == `M1_DI_MINUS` → 431/431

This shows the exported Step3→4 decision values are the same Shift-0 forming values as recorded at Step4 open.

### 8.3 `PositionOpenSummary_XAUUSD.txt`

- SHA-256: `437bc5b9a05033d60cf7fb14a7d281bceaae39d2941ad81a15cbed52abb5badd`
- Contains narrative note:
  `Note : Indicator snapshot uses latest closed bar at position open (shift=1)`
- **This note is contradicted by source and data**:
  - `RecordPositionOpenBySetup` sets `rec.indicator_shift_used = 0` (line 18364).
  - All `GetBufferValue(..., 0)` reads are used (lines 18380–18422).
  - The exported `IndicatorShiftUsed` column is `0` for all 9,605 rows.
- Because the evidence hierarchy puts `.mq5` source and exported data above narrative notes, this line is treated as an **unreliable/incorrect note**, not proof of Shift=1.

### 8.4 RX Features / Labels / Lifecycle

- `RX_..._Features.csv` SHA-256: `08f8d6bc4c7b89d95b5e569d48bd4b9e1542c7cda24ff5a28f742d827b403f96`
- `RX_..._SetupLifecycleEvents.csv` SHA-256: `e2c73bfa632a28a03aea9f584c19a4cfee5d5dbf1e562cf38483dd0d598dc9f5`
- RX Features: **4,864 rows / 4,864 unique setups**, all `M1_Shift = 1`, all `M1_Forming = 0`.
- RX Labels: **4,864 rows**.
- RX Lifecycle: **28,447 events**, `STEP4_OPEN = 431`, but only `EventTime`/`EventPrice` per event — **no Shift=1 feature capture for Step3→4**.
- The source confirms `RX_CaptureStep1Snapshot` is the only feature snapshot (line 12956), and is invoked once at Step1 (line 19423).

### 8.5 `Step1_OutcomeComparison_XAUUSD.csv`

- Contains Step1 features only (`RecordID`, `OpenTime`, M1 features).
- Not evidence for Step3→4 features.

### 8.6 Other artifacts

- `PositionOpenIndicatorStats_*`: summaries of the same StepNo rows (Shift 0 per data).
- `ReportTester-91306235.xlsx` (binary), real terminal log: not used in this audit; neither is needed to resolve the Shift=1 question because source already proves the Step3→4 feature read index is 0.

---

## 9. Proven

1. The actual Step3→4 decision is in `ManagePositions` → `CF_ShouldVeto(3,4,is_bull)` → `CF_LogEvent(...,3,4,...)` (lines 42331, 42432–42433).
2. The exported Step3→4 decision features are identifiable:
   - `CF_events_m0.csv`: 431 `Event=A, StepFrom=3, StepTo=4` rows.
   - `PositionOpenDataset`: 431 `StepNo=4` rows.
   - The two sets match 1:1 and the ATR/ADX/DI+/DI− values are exactly equal for all 431 rows.
3. The M1 feature source is `CopyBuffer(..., 0, count, buffer)` with `GetBufferValue(..., 0)`, i.e. **Shift 0 (forming bar)**.
4. The M1 ATR and ADX periods used are ATR(4) / ADX(14).
5. RX Step1 feature snapshots **do** use `RX_FEATURE_SHIFT = 1`, have `M1_BarTime < FeatureAsOfTime`, and set `M1_Forming=0`; this is real evidence — but only for **Step1**, not Step3→4.
6. Source and data are internally consistent: CF Step3→4 values are identical to `PositionOpenDataset` Step4 values, and `IndicatorShiftUsed=0` on all of them.

---

## 10. Not Proven

1. **ATR_Shift1 at Step3→4** — not proven; source uses index 0 (forming).
2. **ADX_Shift1 at Step3→4** — not proven; source uses index 0.
3. **DI+/DI− at Shift1 at Step3→4** — not proven; source uses index 0.
4. **`PriceRef_Shift1 == Close[1]` at Step3→4** — not proven and contradicted by `iClose(...,0)`; no PriceRef/close column is exported.
5. **`Bar_CloseTime_Shift1 < T_decision` at Step3→4** — not exported / not provable.
6. **No forming-bar feature leaked into Step3→4** — not provable; in fact the Step3→4 feature set **is** the forming-bar (Shift 0) set.
7. **`PositionOpenDataset` Shift=1 connection to Step3→4** — the only shift field (`IndicatorShiftUsed`) is `0`, and it is set by source to `0`; the summary's `shift=1` note is contradicted by source/data.
8. **CF veto latch** — the source comment on line 42423 explicitly says `No latch: every tick re-evaluated`, which is also inconsistent with a "LATCHED" C2/M2 rule (secondary, but relevant to the claimed rule).

---

## 11. Minimum Required Instrumentation

To prove `SHIFT=1 STEP3→4 = PROVEN` in a future run, the Step3→4 decision export must include, per decision attempt:

| Field | Purpose |
|---|---|
| `SetupID` | join key |
| `StepFrom` / `StepTo` | must be 3 / 4 |
| `T_decision` | decision timestamp |
| `Bar_OpenTime_Shift1` | M1 shift-1 bar open time |
| `Bar_CloseTime_Shift1` | M1 shift-1 bar close time |
| `ATR_Shift1` | ATR(4) read from buffer index 1 |
| `ADX_Shift1` | ADX(14) main line at index 1 |
| `DIPlus_Shift1` | plus DI at index 1 |
| `DIMinus_Shift1` | minus DI at index 1 |
| `PriceRef_Shift1` | `iClose(...,1)` |
| `ATRPct_Shift1` | `ATR_Shift1 * 100 / PriceRef_Shift1` |
| `FeatureShift` | literal `1` |
| `strict Bar_CloseTime_Shift1 < T_decision` | presence / value `true` |

Also required for provenance:
- source commit + file SHA-256
- `FeatureShift = 1` asserted in code
- M1 timeframe; ATR period=4; ADX period=14
- input/hash snapshot of `inp_cf_mode`, thresholds, and step-confirm settings
- no forming-bar column used by the C2/M2 gate
- per-event linkage to the Step4 fill / lifecycle event

Until such export exists, `SHIFT=1 STEP3→4` is **NOT PROVABLE** from existing evidence.

---

## 12. Final Verdict

```
SHIFT=1 STEP3→4 = NOT PROVABLE
```

**Reason:** the actual `.mq5` source at commit `29dfb15` and the real exported artifacts prove the opposite for the C2/M2 Step3→4 feature path:
- `CF_ShouldVeto` and `CF_LogEvent` read `ATR/ADX/DIPlus/DIMinus` from buffer index **0** and `price_tf` from `iClose(..., 0)` → **Shift 0 (forming bar)**.
- `PositionOpenDataset` StepNo=4 rows all have `IndicatorShiftUsed = 0` and exactly equal the CF Step3→4 decision values.
- The only real Shift=1 feature evidence is the **Step1** RX snapshot, which is not a Step3→4 decision-time capture.

No threshold was derived or changed. No rule was registered. No C2/M2 freeze was issued. No `.mq5` was modified. No production change was made. **Production = M0.** No M1-instrument or External-AI authority was used.
