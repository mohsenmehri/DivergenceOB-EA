# C2 / M2 — FINAL FREEZE EVIDENCE AUDIT ARTIFACT

| Field | Value |
|---|---|
| Candidate | C2/M2 Step3→4 High-ATR + Adverse Trend/Momentum VETO |
| Semantics | **LATCHED** |
| Shift | **1 / last completed bar** (C2/M2 specification) |
| ATR feature | **ATRPct** |
| Threshold method | **DEV Q75 — method only. No numeric threshold was derived in this artifact.** |
| Population | M0 → Event=A → Step3→4 → first valid decision per Setup → DEV |
| OOS | **Quarantined.** No OOS outcome analysis/reporting was performed. |
| Production | **M0** (unchanged). |
| Data basis | M0 real exports from branch `agent-regression-data-real` (FETCH_HEAD `f349093b5137f63ed7e9cf8b4833d86f76b9421c`). |
| This artifact | Final integration of Freeze Evidence. Documentation only. **No threshold derivation, no rule freeze, no backtest/OOS/TRUE_FORWARD, no code/production change.** |

---

## 0. Evidence authority / provenance of findings

- **Internal Agent-track audits (used as prior internal context, re-verified here):**
  `C2_M2_PRE_FREEZE_AUDIT.md`, `C2_M2_FREEZE_READINESS_AUDIT.md`, `C2_M2_INDEPENDENT_FREEZE_CLEARANCE_AUDIT.md`, `C2_M2_FINAL_FREEZE_EVIDENCE_AUDIT.md`, `C2_M2_MINIMUM_FREEZE_EVIDENCE_EXECUTION_PLAN.md`. These are prior internal audit artifacts. (A prior G-P validation artifact was produced in a transient workspace; its reproducible findings are re-stated and re-checked below from raw data/source.)
- **M1 results:** belong to the **Research Challenge / M1 research track**. **Not** used as C2/M2 freeze evidence.
- **External AI Challenge:** **NOT an evidence authority.** Any external-AI proposals are explicitly **not** treated as fact for C2/M2 Freeze. They were not used in this artifact.
- **Current artifact author:** independent audit integration (this run), based only on the raw M0 artifacts and the ratified C2/M2 specification.

---

## 1. Coverage Audit

**Cutoff:** `2026-08-18 23:59:58`

### CF event log (M0) — temporal coverage
- Rows total: **19,087** (A=9,605, P=4,741, F=4,741; no V).
- First decision time: `2014-01-03 04:17:09`.
- Last decision/order event time: `2026-08-18 19:57:18`.
- Time after cutoff: **0 rows**.
- Interval `2026-08-18 19:57:18 → 2026-08-18 23:59:58` contains **no CF decision/order rows**.
- **Interpretation:** cannot distinguish “no further signal/order needed” from “missing tail” solely from CF_events. The lifecycle and Label layers reach the cutoff (below), which supports that the last open setup continued to `23:59:58`; but a terminal/run log is not available to prove the absence of an omitted decision in this interval.

### Lifecycle layer
- `STEP*_OPEN` event types present; maximum EventTime = **`2026-08-18 23:59:58`** (reaches cutoff).
- For the 431 A34 setups: **431/431 have `STEP4_OPEN`**; missing = 0.
- Lifecycle event counts (overall): STEP1_OPEN 4864, STEP2_OPEN 2970, STEP3_OPEN 1149, STEP4_OPEN 431, STEP5_OPEN 191, plus TP/BE/EMA/EXIT events.

### PositionOpen / outcome layer
- Unique setups in `PositionOpenDataset_XAUUSD.csv` (M0): **4,864**; A34 setups present: **431/431**.
- Max `OpenTime` = **`2026-08-18 19:57:18`** (last opened step).
- StepNo distribution: 1=4864, 2=2970, 3=1149, 4=431, 5=191.
- For each A34 setup, the StepNo=1 row is complete for `StrategyOutcomeComplete`, `SetupFinalEvent`, `SetupMaxStep`, `SetupFinalProfit`, `SetupReachedStep5`, `SetupFinalStop`, `SetupFinalTarget` (0 missing/empty).

### Labels layer
- 4,864 label rows; A34 setups all present (431/431).
- `IsFinalized` = 1 for all 431 A34 rows; `IsCensored` = 0 for all 431 A34 rows.
- All-label `IsCensored` distribution: 0 × 4,863, 1 × 1. **The single censored row is NOT among A34.**
- Max of Step4Time/Step5Time/CloseTime = **`2026-08-18 23:59:58`** (reaches cutoff).
- `EventPath` is empty for **80/431** A34 label rows even though `IsFinalized=1`; `FinalEvent` is populated for all 431. This is a minor field-completeness note, not an outcome-coverage failure.

### Censored rows
- Among the C2/M2 population (A34): **0 censored**.
- Overall dataset: **1 censored row** (`CENSORED_EOT_OPEN`), not in A34.

### Gap / missing window summary
- CF event layer last observed decision `19:57:18`; no CF rows in the following `3h2m40s` before cutoff. Not classified as definitive truncation, but **not fully provable** as “no missing tail” without the real terminal/run log.
- Lifecycle + Labels cover the cutoff exactly. PositionOpen open-time stops at `19:57:18` (consistent with last opened Step4), while outcome/close layer covers to `23:59:58`.

**COVERAGE AUDIT = PARTIAL / NOT FULLY PROVEN**
- Event-layer and PositionOpen-layer reach `19:57:18`; lifecycle + labels reach `23:59:58`.
- Because the real Terminal Log is unavailable (Git-LFS pointer only), full "no truncation / no missing window" evidence is **not fully proven**.

---

## 2. Missingness Audit (Step3→4 A population)

Audited fields on M0 `Event=A`, StepFrom=3, StepTo=4 rows (all 431; DEV subset 366):

| Field | n=431 missing | n=366 (DEV) missing | invalid/non-positive (431 / DEV) | Notes |
|---|---|---|---|---|
| `ATR` | 0 | 0 | 0 / 0 | present |
| `ATRPct100` | 0 | 0 | — | present |
| `ADX` | 0 | 0 | 0 / 0 | present |
| `DIPlus` | 0 | 0 | 0 / 0 | present |
| `DIMinus` | 0 | 0 | 0 / 0 | present |
| `DecisionTime` | 0 | 0 | — | present |
| `Bid` / `Ask` | 0 | 0 | 0 / 0 | present |

- **Shift=1 compliance / `PriceRef(Shift1)` evidence: NOT PROVEN.** The existing CF event export records indicator values read at buffer index **0** (forming bar) in the producing source; it does **not** contain the required last-completed-bar / Shift-1 close reference at the Step3→4 decision. The RX export uses shift=1 but captures the **Step1** snapshot, not the Step3→4 decision-time population. Therefore Shift-1 ATR/ADX/DI/PriceRef at Step3→4 is **UNPROVEN** from current artifacts.
- **Cause/effect note:** No missing `ATRPct100`, `ADX`, `DI`, `DecisionTime`, or `Bid/Ask` was found in the A34 rows, so there is no population-affecting missingness in these fields. The missing/unproven item is **not a NaN row** but a **feature-convention provenance gap** (Shift=1 not implemented/proven in the available decision-time export).
- `EventPath` empty (80/431) is a label-field completeness note; `FinalEvent` populated, `IsFinalized=1`, `IsCensored=0`.

**MISSINGNESS AUDIT = PARTIAL / NOT FULLY PROVEN**
- NaN/empty missingness = none in required numeric fields.
- Shift=1 feature-convention evidence = missing/unproven.

---

## 3. Population Provenance

| Item | Value | Evidence |
|---|---|---|
| Arm | M0 / CF baseline | `Mode` column in CF_events_m0 = 0 for all rows; no `V` events. |
| Event filter | `Event == 'A'` | CF_events_m0 |
| Step filter | `StepFrom == 3` and `StepTo == 4` | CF_events_m0 |
| Unit | first valid decision per Setup | For M0 (no V), this is one `A` row per Setup; dedup confirmed 431 unique Setups / 0 duplicates. |
| Deduplication | SetupID unique across A34 | 431 unique / 431 rows; 0 duplicate rows; A→P→F order never violated. |
| DEV/OOS boundary | DEV = 2014-01-01 00:00:00 → 2023-12-31 23:59:59; OOS = 2024-01-01 00:00:00 → 2026-08-18 23:59:58 | From ratified specification §12. |
| DEV population **n** (Step3→4 A, first-valid) | **366** | Reproducible from CF_events_m0; matches the prior audited DEV n. |
| OOS population **n** (Step3→4 A, first-valid) | **65** | enumerated for coverage only; **not analyzed for outcomes (quarantined)**. |
| All A34 | 431 | |
| A34 → P → F mapping | 431/431 have P and F at 3→4 | CF log; no missing F. |

**POPULATION PROVENANCE = PASS (for M0 event-log structure; Shift-1 feature evidence is separately UNPROVEN).**

---

## 4. Feature Provenance

| Feature | Definition per spec | Evidence available | Determined value/convention |
|---|---|---|---|
| ATR period | M1 timeframe ATR | `/tmp/cf_src.mq5` at `29dfb15` (`inp_atr_period = 4`; indicator handle `iATR(..., atr_p)`) | **ATR period = 4** (source default; actual run parameter not independently proven). |
| ADX period | M1 timeframe ADX | `/tmp/cf_src.mq5` (`inp_adx_period = 14`; `iADX(..., adx_p)`) | **ADX period = 14** (source default; actual run parameter not independently proven). |
| DI source | ADX buffers (`PLUSDI_LINE`, `MINUSDI_LINE`) | `/tmp/cf_src.mq5`, `CF_LogEvent` reads `buffer_plus_di[0]`, `buffer_minus_di[0]` | M1 ADX DI buffers; **read at shift 0 in the available event export**. |
| PriceRef definition | Spec: `PriceRef(Shift1)` = close of same last fully-closed M1 bar before `T_decision` | **Not available at Step3→4.** CF export uses `iClose(...,0)` (forming), or bid/ask fallback. RX export uses shift=1 but Step1 snapshot. | **UNKNOWN / NOT PROVEN for Step3→4.** |
| Shift | Spec: **1** | Existing M0 Step3→4 export uses **0** in CF_LogEvent; RX manifest says shift=1 but is Step1-observation layer. | **UNPROVEN for the C2/M2 Step3→4 population.** |
| Timestamp alignment | Feature timestamp = last closed bar strictly before `T_decision` | CF export logs `DecisionTime` but no Shift-1 bar-time/close for A34. | **UNPROVEN.** |
| ATRPct formula | `ATR * 100 / PriceRef(Shift1)` | Formula defined in spec; CF export provides `ATRPct100` computed with forming price (`iClose(...,0)`/bid/ask). | Spec formula; **current export is Shift-0 form, not Shift-1.** |

**FEATURE PROVENANCE = PARTIAL / NOT FULLY PROVEN**
- Period definitions recoverable: ATR=4, ADX=14, M1 ADX DI source.
- **Shift=1 / PriceRef(Shift1) / timestamp alignment at Step3→4 = GAP / UNPROVEN.**

---

## 5. Input / Configuration Provenance

| Input | Observable value | Source | Provenance status |
|---|---|---|---|
| CF mode | Mode=0 (M0, no veto) | CF_events_m0 `Mode` column for every row | **PROVEN (observed)** |
| Veto enabled | no (M0) | no V events; Mode=0 | **PROVEN (observed)** |
| ATR period | 4 | source default `inp_atr_period=4` | **Partially proven** (source default; actual parameter input file not available). |
| ADX period | 14 | source default `inp_adx_period=14` | **Partially proven** (source default; actual parameter input file not available). |
| Spread filter | source default `inp_use_spread_filter=false`, `inp_max_spread_points=30` | `/tmp/cf_src.mq5` | **Actual run value UNKNOWN** (no input config file; M0 A34 contains spreads >30, consistent with filter off or permissive). |
| Margin/leverage/account | not in event export | n/a | **UNKNOWN** |
| Spread filter & margin config as Freeze condition | not explicit in §14.3/§15.2 | ratified spec | treated as **NON-BLOCKER / evidence gap** (see §9). |
| Terminal/run log | Git-LFS pointer stub (`20260905.log`) | `.replay/.../20260905.log` | **NOT AVAILABLE** (real log content not recoverable). |

**INPUT / CONFIGURATION PROVENANCE = PARTIAL / NOT FULLY PROVEN.**
- Do not guess unknown values. Where not observable, recorded as **UNKNOWN**.

---

## 6. Artifact Provenance

| Path (in repo/workspace) | Branch/Commit | SHA-256 | Cutoff | Role |
|---|---|---|---|---|
| `.replay/v2_research_export_v2_cf/CF_events_m0.csv` | `agent-regression-data-real` @ f349093 | `642753751f89ceed346a6c069b4878f92bb87a5fb8b352e65e377ea1bc4ff74d` | 2026-08-18 23:59:58 | M0 decision/order events; population, coverage, missingness, DEV n (366). |
| `.replay/v2_research_export_v2_cf/PositionOpenDataset_XAUUSD.csv` | same | `4d013f581b3fa69e5d54a7957af218572c80e22ddbbd45f831f5bf02c77fc19d` | same | PositionOpen/outcome layer; Step1/Step4 coverage. |
| `.replay/v2_research_export_v2_cf/RX_XAUUSD_RX_1388707200_XAUUSD_SetupLifecycleEvents.csv` | same | `e2c73bfa632a28a03aea9f584c19a4cfee5d5dbf1e562cf38483dd0d598dc9f5` | same | Lifecycle layer; STEP4_OPEN and cutoff reach. |
| `.replay/v2_research_export_v2_cf/RX_XAUUSD_RX_1388707200_XAUUSD_Labels.csv` | same | `c607b2045b2250b0f5a49ded32c23ef1625ed4ddd07289ade21ad1f620695f43` | same | Label layer; finalized/censored status for A34. |
| `.replay/v2_research_export_v2_cf/RX_XAUUSD_RX_1388707200_XAUUSD_Manifest.json` | same | `aa01463639bb4c03114064eec69e098e1d8923f128d002c3904a703f0d68de67` | same | Run identity, capture policy. |
| `.replay/v2_research_export_v2_cf/RX_XAUUSD_RX_1388707200_XAUUSD_DatasetQualityReport.html` | same | `2a779b71de8f9144b3581b25b774876248324abf22f817216052d34f8e4d62d0` | same | Dataset quality / Step4Plus count. |
| `.replay/v2_research_export_v2_c-M1/CF_events_m1.csv` | same | `227d3298585ede5cba54261d5a9473d4607b4ad0e95d93b369e89caa0e762c3d` | same | **Reference only (M1)** — not used as C2/M2 freeze authority. |
| `/tmp/cf_src.mq5` (source) | commit `29dfb1529293c10b364014464977a150074da25f` | (not in workspace) | — | Gate order / feature-read semantics / input defaults for artifact provenance. |
| `.replay/.../20260905.log` | same data branch | Git-LFS pointer oid `1da3f759...` | — | Terminal log pointer — **real content unavailable**. |
| `.replay/.../20260906.log` (M1) | same | Git-LFS pointer oid `5d6132cf...` | — | M1 terminal log pointer — not used. |

> Files under `.replay/` are workspace-local restored artifacts (not committed to the deliverable). The raw source branch/commit is `agent-regression-data-real` @ `f349093`. Source code commit `29dfb15` provides the gate/feature-generation semantics.

---

## 7. δ_DI

- **Status: OWNER DECISION REQUIRED.** **Not frozen. No value selected by this audit.**
- The spec requires direction-aware DI; the exact `δ_DI` convention (strict vs non-strict inequality, tie handling, or any DI offset) must be explicitly decided by the Owner **before Freeze**.
- This is a rule-semantics decision, **not** a threshold derivation.
- **No numeric `δ_DI` value was chosen.**

---

## 8. G-P

### 8.1 G-P Core
- Gate order in source: Market Gate → Spread Gate → Margin Gate → Event=A/V → PlaceEntry. Verified from `/tmp/cf_src.mq5` (29dfb15).
- `Event=A` is logged **after** gate evaluation and **before** `PlaceEntry`; gate-failing paths return before the log.
- M0 A34: 431 rows / 431 unique Setups; 0 duplicates; 431/431 have P and F at 3→4; A→P→F order OK.
- **G-P CORE = PASS.**

### 8.2 G-P Freeze Clearance
- Remains constrained by the not-yet-proven **Shift=1 feature convention / PriceRef(Shift1)** and by the incomplete **coverage/config/log** evidence stack.
- **G-P FREEZE CLEARANCE = NOT YET CLEARED.**

---

## 9. Final BLOCKER / NON-BLOCKER / SATISFIED table

Criterion: a **BLOCKER** is an item that the ratified Formal Registration Specification or its Freeze requirements explicitly require to be satisfied before Freeze. Items not explicitly required for Freeze are **NON-BLOCKER / EVIDENCE GAP**.

| Evidence Item | Exact Spec Requirement | Classification | Minimum Required Action |
|---|---|---|---|
| G-P Core (gate order / Event=A placement / M0 first-valid structural) | §11, §14.1, §14.3 | **SATISFIED** | None. |
| G-P Freeze Clearance | §14.3 / §17.13 | **BLOCKER** | Complete remaining evidence and obtain explicit `G-P = PASS` / Freeze Clearance. |
| Coverage / data-quality gate result | §14.2 / §15.2.7 | **BLOCKER** | Produce + record three-layer coverage result to cutoff (`PASS/FAIL` + artifact). |
| Shift=1 compliance / PriceRef(Shift1) at Step3→4 | §3.2, §4, §5, §13, §15.2.3 | **BLOCKER** | Produce a Shift-1 Step3→4 feature/provenance artifact (last completed bar before `T_decision`) — **not currently available**. |
| Missingness audit result | §14.4 / §15.2.8 | **BLOCKER (partially satisfied on NaN)** | Record documented cause → effect → decision; NaN fields are 0, but Shift-1 convention gap remains. |
| Input / feature / population provenance | §15.1 | **BLOCKER** | Record ATR/ADX/DI periods, PriceRef, shift, buffer source, filter, dedup, DEV/OOS, n, input hashes. |
| θ_ATR / θ_ADX + derivation log | §15.1 / §15.2.2 | **BLOCKER (procedural, gated)** | Only after G-P Clearance + explicit authorization: derive per §6/§7 DEV Q75. **Not done here.** |
| Pre-registration hash / Freeze document | §15.1 | **BLOCKER** | Create hash of registration spec + Freeze document with hash/timestamp/author. |
| Artifact hashes | §15.2.9 | **BLOCKER** | Record hashes of every input/derived artifact. |
| Spread config provenance | not explicit §14.3/§15.2 condition | **NON-BLOCKER / EVIDENCE GAP** | Optional; record actual run setting or state UNKNOWN. |
| Margin config provenance | not explicit §14.3/§15.2 condition | **NON-BLOCKER / EVIDENCE GAP** | Optional; record account/leverage/margin evidence or state UNKNOWN. |
| Terminal log evidence | not explicit §14.3/§15.2 condition | **NON-BLOCKER / EVIDENCE GAP** | Optional; supply real log or state unavailable. |
| `EventPath` empty (80/431) | no explicit freeze condition | **NON-BLOCKER / EVIDENCE GAP** | Optional field; `FinalEvent`/`IsFinalized` complete. |
| M1 results / external AI proposals | not authority for C2/M2 | **NON-BLOCKER / NON-AUTHORITY** | Exclude; do not use as freeze fact. |
| δ_DI | Owner decision before Freeze | **OWNER DECISION NEEDED** | Owner decides δ_DI convention; record in §8. No value chosen. |

---

## 10. Verdict

```
SPECIFICATION READINESS = FREEZE NOT READY
G-P CORE = PASS
G-P FREEZE CLEARANCE = NOT YET CLEARED
FREEZE READINESS = NOT READY
C2/M2 STATUS = NOT YET FROZEN
PRODUCTION STATUS = M0
```

### Explicit declarations
- **No threshold was derived** in this artifact.
- **No rule was frozen.**
- **No Backtest / OOS / TRUE_FORWARD was executed.**
- **Production remains M0.**

### Key remaining blockers to Freeze Clearance
1. **G-P Freeze Clearance** — not cleared.
2. **Shift=1 / PriceRef(Shift1) feature evidence at Step3→4** — not proven (existing event export uses Shift 0).
3. **Coverage / data-quality result** — partially covered; real run-log unavailable.
4. **Provenance artifacts + hashes + pre-reg/Freeze document** — not fully recorded.
5. **δ_DI** — Owner decision required before Freeze.
6. **θ derivation** — gated on G-P Clearance + explicit authorization (not performed).

### Where a required evidence item cannot be proven from existing data
- **UNKNOWN / NOT CLEARED** is used for: actual run-input configuration (spread/margin), terminal log, and Shift-1 Step3→4 feature/PriceRef provenance. **No assumption or guess was converted to PASS.**
