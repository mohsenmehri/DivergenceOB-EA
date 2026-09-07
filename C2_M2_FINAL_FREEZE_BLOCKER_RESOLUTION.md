# C2 / M2 — FINAL FREEZE BLOCKER RESOLUTION

| Field | Value |
|---|---|
| Candidate | C2/M2 Step3→4 High-ATR + Adverse Trend/Momentum VETO |
| Semantics | **LATCHED** |
| Shift (C2/M2 spec) | **1 / last completed bar** |
| ATR feature | **ATRPct** |
| Threshold method | **DEV Q75 — method only. No threshold derived.** |
| Population | M0 → Event=A → Step3→4 → first valid decision per Setup → DEV |
| Cutoff | `2026-08-18 23:59:58` |
| Production | **M0** (unchanged) |
| Prior artifact | `C2_M2_FINAL_FREEZE_EVIDENCE_AUDIT_ARTIFACT.md`, commit `ec13c467` |
| Purpose | Resolve/assign status for **BLOCKER 1 (Shift=1 / PriceRef)** and **BLOCKER 2 (Coverage to cutoff)** |
| No-op | **No threshold derivation, no rule freeze, no backtest, no OOS, no TRUE_FORWARD, no sweep/tuning, no rule change, no threshold change, no production change, no .mq5 change.** |

---

## 1. Scope

- Audit only the two remaining freeze blockers using **actual project source and data artifacts**.
- **Evidence Authority = source code + real data/artifacts only.** Agent 1 / Agent 2 are supporting research/audit context; **External AI Challenge is NOT an evidence authority** and was not used to pass any gate.
- M1 is **not** used to prove C2/M2.
- No assumption was converted to PASS. Where not provable, recorded as **UNKNOWN / NOT PROVABLE**.

---

## 2. Evidence Sources

| Source | Location / identity | Used for |
|---|---|---|
| CF/M0 event export | `agent-regression-data-real` @ f349093 `v2_research_export_v2_cf/CF_events_m0.csv` | Population, coverage timeline, Step3→4 rows |
| Position Open dataset | same branch, `PositionOpenDataset_XAUUSD.csv` | Step->position layer, Step1 shift note |
| RX lifecycle | same branch, `RX_XAUUSD_RX_1388707200_XAUUSD_SetupLifecycleEvents.csv` | Lifecycle layer, STEP4_OPEN, EOT censoring |
| RX labels | same branch, `RX_XAUUSD_RX_1388707200_XAUUSD_Labels.csv` | Finalized/censored, CloseTime, Step4/5 times |
| RX features | same branch, `RX_XAUUSD_RX_1388707200_XAUUSD_Features.csv` | Step1 snapshot timestamps, capture policy |
| Dataset quality report / manifest | same branch | Run identity, capture policy |
| PositionOpen summary | `PositionOpenSummary_XAUUSD.txt` | Confirms **PositionOpen indicator snapshot uses last closed bar at position open (shift=1)** |
| **Source/EA** | commit **`29dfb1529293c10b364014464977a150074da25f`** file `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` (function `CF_ShouldVeto`, `CF_LogEvent`, next-step block) | **Ground truth for CF Step3→4 shift / PriceRef / gate placement** |
| Terminal log | `v2_research_export_v2_cf/20260905.log` (Git-LFS pointer only) | Real log **NOT AVAILABLE** |
| ReportTester | `ReportTester-91306235.xlsx` (binary) | **UNKNOWN (not parsed)** |

---

## 3. BLOCKER 1 — Shift=1 / PriceRef(Shift1)

### 3.1 Source code ground truth (CF/EA at `29dfb15`)

**Function `CF_ShouldVeto`** (decision-time gate, Step3→4 path):

```
int mi = TF_IDX_MAIN;                                  // M1
g_tf[mi].CopyIndicatorBuffers(MIN_BUFFER_DEPTH);
double price_tf = iClose(_Symbol, g_tf[mi].timeframe, 0);   // <-- shift 0 forming
if(price_tf <= 0.0)
   price_tf = is_bull ? SYMBOL_BID : SYMBOL_ASK;
double atr = g_tf[mi].GetBufferValue(g_tf[mi].buffer_atr, 0);      // <-- shift 0
double adx = g_tf[mi].GetBufferValue(g_tf[mi].buffer_adx, 0);      // <-- shift 0
double pdi = g_tf[mi].GetBufferValue(g_tf[mi].buffer_plus_di, 0);  // <-- shift 0
double mdi = g_tf[mi].GetBufferValue(g_tf[mi].buffer_minus_di, 0); // <-- shift 0
double atr_pct = (price_tf > 0.0 && atr > 0.0) ? (atr * 100.0 / price_tf) : 0.0;
```

**Function `CF_LogEvent`** (same feature snapshot written to `CF_events_m0.csv`):

```
double price_tf = iClose(_Symbol, g_tf[mi].timeframe, 0);            // <-- shift 0
double atr  = g_tf[mi].GetBufferValue(g_tf[mi].buffer_atr, 0);       // <-- shift 0
double adx  = g_tf[mi].GetBufferValue(g_tf[mi].buffer_adx, 0);       // <-- shift 0
double pdi  = g_tf[mi].GetBufferValue(g_tf[mi].buffer_plus_di, 0);   // <-- shift 0
double mdi  = g_tf[mi].GetBufferValue(g_tf[mi].buffer_minus_di, 0);  // <-- shift 0
double atr_pct = (price_tf > 0.0 && atr > 0.0) ? (atr * 100.0 / price_tf) : 0.0;
```

### 3.2 Answers to the six sub-questions

1. **Are ATR / ADX / DI_PLUS / DI_MINUS for the Step3→4 decision taken from last closed bar (Shift=1)?**
   **NO.** The actual CF source reads all four from **buffer index 0 (forming bar)**. Shift=1 is **not** used by the CF Step3→4 decision path in the producing source.
2. **Is `PriceRef(Shift1)` exactly/deterministically defined and used at the decision timestamp?**
   **NO.** The source uses `iClose(...,0)` (forming-bar close) or bid/ask fallback; it does **not** use the last fully-closed M1 bar close. The C2/M2 `PriceRef(Shift1)` definition is **not implemented/proven** in this source.
3. **Does `CF_events_m0.csv` only record event/decision and cannot prove Shift?**
   It records the event + feature columns, but those feature columns were produced at **Shift 0** (formation), so the CSV alone cannot prove Shift=1. It records the source's Shift-0 values.
4. **Does `PositionOpenDataset` record only Step1 with Shift=1 and is therefore insufficient for Step3→4?**
   **Yes.** `PositionOpenSummary_XAUUSD.txt` states: *“Indicator snapshot uses latest closed bar at position open (shift=1)”*. The PositionOpen dataset is a **position-open (Step1/ladder-open)** snapshot. It is **Step1/ladder-step open**, **not** the Step3→4 first-valid decision snapshot. It is **not sufficient** to prove Shift=1 at Step3→4.
5. **Can source code prove the real Shift at Step3→4?**
   **File:** `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` (commit `29dfb15`).
   **Functions:** `CF_ShouldVeto`, `CF_LogEvent`.
   **Relevant code path:** `CF_ShouldVeto` → `CopyIndicatorBuffers` → `GetBufferValue(..., 0)` / `iClose(...,0)`; `CF_LogEvent` identical.
   **Shift value:** **0 (forming)** for all ATR/ADX/DI/price in the Step3→4 CF path.
   **Timestamp alignment:** `DecisionTime` = `TimeCurrent()` at the evaluation; **no last-completed-bar timestamp / Shift-1 bar-close is captured** in the CF event export.
   → **This source proves the opposite of Shift=1.** It proves the existing CF/M0 experiment used **Shift 0**.
6. **If source cannot prove it:**
   ```
   SHIFT=1 STEP3→4 = NOT PROVABLE FROM EXISTING EVIDENCE
   ```
   More precisely, the existing evidence **disproves** Shift=1 for the CF Step3→4 population (it shows Shift=0), and the Shift-1 C2/M2 feature at Step3→4 has **no artifact** in the current exports.

### 3.3 Required future instrumentation (minimum to prove Shift=1 at Step3→4)

A future C2/M2-compliant rerun must export, **at each Step3→4 first-valid decision**:
1. `T_decision` (= evaluation timestamp).
2. `Bar_OpenTime_Shift1` (= open time of last fully-closed M1 bar strictly before `T_decision`).
3. `ATR_Shift1`, `ADX_Shift1`, `DIPlus_Shift1`, `DIMinus_Shift1` (all from buffer index 1 / last completed bar).
4. `PriceRef_Shift1` = `Close[1]` of that same Shift-1 bar.
5. `ATRPct_Shift1 = ATR_Shift1 * 100 / PriceRef_Shift1`.
6. A provenance header/row stating `shift=1`, timeframe, ATR period, ADX period, DI buffer source, and input hash.
7. Verification check that `Bar_OpenTime_Shift1` is strictly before `T_decision`.

Without such instrumentation, Shift=1 at Step3→4 remains **NOT PROVABLE** from existing evidence.

---

## 4. BLOCKER 2 — Coverage

Cutoff: `2026-08-18 23:59:58`

| Layer | STATUS | Last provable timestamp | Note |
|---|---|---|---|
| CF events | **PARTIAL (globally)** — but **COMPLETE for Step3→4 population** | last CF row `2026-08-18 19:57:18` | Last **Step3→4 A34** decision = `2026-07-28 01:00:06` (setup 4837). All 90 CF rows after it are Step0→1 / 1→2 / 2→3 / 4→5; **none are 3→4**. |
| PositionOpen | **PARTIAL (globally)** — **COMPLETE for A34** | max OpenTime `2026-08-18 19:57:18` | Last open step is Step3 (setup 4864). All 431 A34 setups have Step1+Step4 rows. |
| SetupLifecycleEvents | **COMPLETE** | `2026-08-18 23:59:58` | `CENSORED_EOT_OPEN` for setup 4864 at EOT; STEP4_OPEN present for 431/431 A34. |
| RX Labels | **COMPLETE** | `CloseTime` max `2026-08-18 23:59:58` | A34 IsFinalized=1/431, IsCensored=0/431. |
| RX Features | **COMPLETE for Step1 snapshot** | `FeatureAsOfTime` max `2026-08-18 13:04:00` | Step1 snapshot only (shift=1) — not Step3→4. |
| ReportTester | **UNKNOWN** | not parsed (binary `.xlsx`) | Not required by §14.2 three-layer gate; cannot be verified from this artifact. |
| Terminal log | **NOT AVAILABLE** | Git-LFS pointer only | Real log content not recoverable. |
| Step1 outcome data | **PARTIAL** | `OpenTime` max `2026-08-17 09:43:00`; 2,325 rows | Step1-only comparator; not required for A34/Step3→4 coverage. |

### 4.1 Resolution of the `19:57:18` vs `23:59:58` discrepancy

- **The discrepancy is event semantics + open-position end-of-test censoring, plus artifact boundary.** It is **not missing A34 coverage.**
- Last setup (4864) timeline: Step1 open `13:04:00` → Step2 open `14:53:24` → **Step3 open `19:57:18`** → remained open → **`CENSORED_EOT_OPEN` at `23:59:58`**.
- `19:57:18` is the **last position OPEN event** (Step3). `23:59:58` is the **end-of-test censoring/close event** for that same open setup.
- **The C2/M2 A34 population lies entirely before the cutoff:** last Step3→4 A = `2026-07-28 01:00:06`. No Step3→4 decision is expected in the `19:57:18 → 23:59:58` tail because the only open setup there had reached Step3 and was censored before Step3→4.
- **Therefore the A34 (Step3→4) population coverage is consistent and effectively complete**, even though the CF/PositionOpen event streams do not continue to the exact cutoff.
- **Caveat (never assumed):** without the real terminal log, we cannot *prove* that no Step3→4 decision attempt occurred after `19:57:18`; we can only show from lifecycle + labels that no later Step4 open / A34 row exists and the last open setup was censored at EOT. That is consistent, but the absence of the terminal log leaves a small `UNPROVEN` band for global coverage proof.

---

## 5. Evidence Table

| Evidence Item | Exact Spec Requirement | BLOCKER / NON-BLOCKER / SATISFIED | Minimum Required Action |
|---|---|---|---|
| G-P Core (gate order / Event=A placement / M0 first-valid structural) | §11, §14.1, §14.3 | **SATISFIED** | None. |
| Shift=1 / PriceRef(Shift1) at Step3→4 | §3.2, §4, §5, §13, §15.2.3 | **BLOCKER (not provable from existing evidence; existing source uses Shift=0)** | Add future C2/M2 instrumentation/export listed in §3.3. |
| Coverage to cutoff (A34 population) | §14.2 | **SATISFIED for the A34 population**; **PARTIAL globally / terminal log UNPROVEN** | Obtain real terminal log (or explicitly accept/censor-bound evidence) for full global proof. |
| Missingness (ATR/ADX/DI/DecisionTime) | §14.4 | **SATISFIED (NaN)** | None for NaN; still need Shift-1 feature artifact. |
| Population provenance (DEV n=366) | §15.1 | **SATISFIED / `n=366`** | None. |
| Feature provenance (period/source) | §15.1 | **SATISFIED (ATR=4, ADX=14, M1 DI source)** | None; Shift-1 evidence still missing. |
| Input/config provenance (spread/margin) | (not explicit §14.3/§15.2) | **NON-BLOCKER / Evidence Gap** | Optional: record actual run config or state UNKNOWN. |
| Terminal log evidence | (not explicit §14.3/§15.2) | **NON-BLOCKER / Evidence Gap** | Optional; supply real log. |
| ReportTester | not a §14.2 layer | **NON-BLOCKER / UNKNOWN** | Optional. |
| Current CF source shift | — | **NON-BLOCKER / but proves Shift=0 for existing export** | Must not be used as C2/M2 Shift-1 proof. |
| δ_DI owner decision | pre-Freeze | **OWNER DECISION NEEDED** | Owner decides δ_DI; no value chosen. |
| θ_ATR / θ_ADX derivation | §6/§7 | **BLOCKER (procedural, gated)** | Only after clearances + authorization. **Not done here.** |

---

## 6. What is Proven

- **G-P Core = PASS** (gate order + Event=A placement + M0 Step3→4 uniqueness/first-valid).
- **DEV Step3→4 A population n = 366**; OOS A34 n = 65; all A34 unique, all map to P/F, all have STEP4_OPEN lifecycle.
- **No NaN/invalid** in ATR/ATRPct/ADX/DIPlus/DIMinus/DecisionTime/Bid/Ask for A34.
- **Feature period/source:** ATR = 4, ADX = 14, DI from M1 ADX buffers (source `29dfb15`).
- **PositionOpen uses shift=1 at position open** (Step1/ladder open) — per `PositionOpenSummary_XAUUSD.txt`.
- **Lifecycle + Labels reach the cutoff** (23:59:58); the last open setup was **censored at end-of-test**.
- **A34 (Step3→4) population coverage is effectively complete** — last Step3→4 decision is well before cutoff and no later Step3→4 row exists.

## 7. What is Not Proven

- **Shift=1 / PriceRef(Shift1) at Step3→4 — NOT PROVEN.** The actual CF source at `29dfb15` reads buffer index **0** and `iClose(...,0)` (forming), so the existing M0 CF export is **Shift=0**, not Shift=1.
- **Global coverage to the exact cutoff — terminal-log proof UNPROVEN.** No real terminal/run log; CF/PositionOpen streams end at `19:57:18`, explained by event semantics/EOT censoring but not independently proven by a terminal log.
- **Actual run-input config (spread/margin) — UNKNOWN.** Only CF mode=0 and source-default ATR/ADX periods are proven; actual filter settings not recoverable.
- **ReportTester content — UNKNOWN** (binary not parsed).
- **δ_DI — OWNER DECISION PENDING.**
- **θ_ATR/θ_ADX — not derived (prohibited).**

---

## 8. Required Future Instrumentation

For the next **C2/M2-compliant run** (future, explicit authorization):
1. Export, at each Step3→4 first-valid decision: `T_decision`, `Bar_OpenTime_Shift1`, `ATR_Shift1`, `ADX_Shift1`, `DIPlus_Shift1`, `DIMinus_Shift1`, `PriceRef_Shift1` (= `Close[1]`), and `ATRPct_Shift1`.
2. State `shift=1`, timeframe M1, ATR period, ADX period, DI buffer source, input hash, and verify `Bar_OpenTime_Shift1 < T_decision`.
3. Keep the three-layer coverage gate (event + lifecycle + PositionOpen) plus real terminal log for global tail proof.

Until this exists, **Shift=1 Step3→4 cannot be proven from existing evidence.**

---

## 9. Final Freeze Decision

**Decision: B) G-P FREEZE CLEARANCE = NOT CLEARED**

Reason: **BLOCKER 1 is not resolved** — the existing source/export prove the CF Step3→4 feature path used **Shift 0**, so the C2/M2 Shift=1 / PriceRef(Shift1) at Step3→4 is **NOT PROVABLE** from existing evidence. Under the rule, if even one blocker is not provable, Freeze Clearance is **NOT CLEARED**.

**Final Status:**

```
SPECIFICATION READINESS = FREEZE NOT READY
G-P CORE VERIFICATION = PASS
G-P FREEZE CLEARANCE = NOT CLEARED
FREEZE READINESS = NOT READY
C2/M2 STATUS = NOT YET FROZEN
PRODUCTION STATUS = M0
THRESHOLD DERIVATION = PROHIBITED
FREEZE = PROHIBITED
BACKTEST / OOS / TRUE_FORWARD = PROHIBITED
SWEEP / TUNING = PROHIBITED
CODE CHANGE = PROHIBITED
PRODUCTION CHANGE = PROHIBITED
.MQ5 CHANGE = NONE
```

**Minimum evidence needed to eventually clear G-P Freeze:**
1. A future **C2/M2 Shift-1** Step3→4 export (see §3.3 / §8).
2. A **real terminal log** (or explicit, documented end-of-test-censor coverage note) to close the global tail proof.
3. Explicit **δ_DI** Owner decision.
4. Full **§15 provenance/artifacts/hashes** + **pre-registration/Freeze document**.
5. Only then, with explicit authorization, derive θ (DEV Q75).

**Explicit declarations:** No threshold was derived; no rule frozen; no backtest/OOS/TRUE_FORWARD performed; no code/production change; Production remains **M0**. M1 and External AI were not used as evidence authority; Agent 1/2 are supporting audit context only.
