# C2/M2 Implementation Repair — Open Decisions Review

**Artifact name:** `C2_M2_IMPLEMENTATION_REPAIR_DECISION_LIST_V1.md`
**Reference artifact:** `C2_M2_IMPLEMENTATION_REPAIR_SPEC_V1.md`
**Reference commit:** `2510c8259697bd1d3e5e6cac5cfeaf8925305bb3`
**Production:** M0 (unchanged)
**Status / verdict of this document:** `DECISION LIST READY FOR OWNER REVIEW`

---

## 1. Scope

This is a **decision-list** conversion of the open decisions in the referenced Repair Specification. It is intended to be reviewed and approved **before any coding**.

Hard constraints for this step:
- No `.mq5` change.
- No new code.
- No code commit.
- No Backtest / Smoke Test / OOS / TRUE_FORWARD / sweep / tuning.
- No threshold extraction or change.
- No δ_DI determination.
- No C2/M2 freeze.
- No new rule registration.
- No Production change (M0).

This document does **not** authorize coding. It records options, analysis, and recommendations as `RECOMMENDED — PENDING OWNER APPROVAL`. It does **not** silently change the approved specification.

---

## 2. Reference Specification

- **Repair Specification:** `C2_M2_IMPLEMENTATION_REPAIR_SPEC_V1.md`
- **Commit of reference:** `2510c8259697bd1d3e5e6cac5cfeaf8925305bb3`
- **Approved C2/M2 target path (unchanged):**
  ```
  Step3→4 → Shift=1 → ATR[1] → ADX[1] → DI+[1]/DI−[1]
  → PriceRef = Close[1] → ATRPct_Shift1
  → C2/M2 decision → LATCH → PlaceEntry(Step4)
  ```
- **Source ground truth (from prior audit):** commit `29dfb1529293c10b364014464977a150074da25f`, file `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5`, SHA-256 `567b94a2bb15fbd5f41c4d9fd4b601b0f8a5370e99b17e00e95b492965e91e3b`.
- Current implementation uses Shift=0 (index 0 / `Close[0]`) and has no latch.

---

## 3. Decision Matrix

| ID | Topic | Status | Decisions Requiring Owner? | Recommendation |
|---|---|---|---|---|
| D1 | PriceRef | OPEN | YES | `Close[1]`, no Bid/Ask fallback for C2/M2 |
| D2 | T_decision | OPEN | YES | `tick.time` as authoritative, with `TimeCurrent()` for reconciliation |
| D3 | Shift/Timeframe | CONFIRM | NO (facts) | `_Symbol`, M1, Shift=1, ATR=4, ADX=14 |
| D4 | Temporal Invariant | OPEN | YES | strict `<` (not `<=`) |
| D5 | LATCH | OPEN | YES | Option B (veto latched; allow retry via separate placement loop) |
| D6 | C2/M2 isolation | OPEN | YES | Isolated C2/M2 feature/decision module; shared gate left unchanged for M1/M3 |
| D7 | Export Architecture | OPEN | YES | Option B (independent `C2M2_Step34_Decisions_*` export) |
| D8 | Evidence Fields | CLOSED-ish (verify) | Review | Field list in §11 |
| D9 | δ_DI | OPEN | YES (Owner only) | No option selected |
| D10 | Threshold | LOCKED | -- | None |
| D11 | Historical Data | PRINCIPLE | Confirm | Never relabel Shift-0 as Shift-1 |
| D12 | M0 Preservation | OPEN | YES | Invariants in §15 |

---

## 4. D1 — PriceRef

**Decision ID:** D1

**Question:** Must C2/M2 use exactly `PriceRef_Shift1 = Close[1]`? Is any fallback to Bid/Ask compatible with the approved specification?

**Option A:** Mandatory `PriceRef_Shift1 = iClose(_Symbol, PERIOD_M1, 1)`; no fallback; invalid price → decision marked `INVALID`, no veto-derived evidence claim.

**Option B:** Mandatory `Close[1]`, but if invalid, fall back to `Bid`/`Ask` (current source behavior at lines 19219–19222), keeping the gate usable.

**Option C:** Mandatory `Close[1]`; if invalid, fall back to `Close[0]` (current forming close).

**Pros**
- **A:** exact specification compliance; verifiable PriceRef; no forming/future contamination; clean `strict_before_check`; strongest no-lookahead proof.
- **B:** keeps gate available under missing-price edge cases; resembles current design.
- **C:** simple, but uses a forming bar and is not specified.

**Cons**
- **A:** introduces an `INVALID` decision outcome; requires a documented policy for what the gate does on invalid input (allow / veto / no-decision) — that policy remains an owner decision.
- **B:** violates specification; PriceRef can be a broker tick price, not the M1 closed price; breaks temporal and no-lookahead proof.
- **C:** directly violates Shift=1 and re-introduces forming-bar dependence.

**Impact on Specification:** A = fully aligned; B/C = silently deviates from `PriceRef = Close[1]`.

**Impact on M0:** A = no order change in M0 (`inp_cf_mode==0` returns before feature reads); B/C = same trade layer currently, but evidence semantics misleading.

**Impact on Evidence:** A = evidence is self-consistent and provable; B/C = evidence cannot prove Shift=1/PriceRef=Close[1].

**Recommended Option:** **A** — `RECOMMENDED — PENDING OWNER APPROVAL`.

**Owner Approval Required:** YES.

---

## 5. D2 — T_decision

**Decision ID:** D2

**Question:** Which timestamp is `T_decision`: `tick.time`, `TimeCurrent()`, or another timestamp?

**Criteria:** must represent the real decision moment for Step3→4 and must support provable temporal ordering.

**Option A:** `T_decision = tick.time` from the same `SymbolInfoTick(_Symbol, tk)` used for the entry condition.

**Option B:** `T_decision = TimeCurrent()` (current source behavior for `DecisionTime`, line 19312).

**Option C:** Both, with `tick.time` authoritative and `TimeCurrent()` recorded as `DecisionServerTime`.

**Pros**
- **A:** directly tied to the actual price/tick that triggered the decision; reproducible against the tick trace; best for `Bar_CloseTime_Shift1 < T_decision` ordering.
- **B:** simple, stable, matches existing export; server clock semantics.
- **C:** preserves comparability with existing `DecisionTime` while making the authoritative time the tick time.

**Cons**
- **A:** `tick.time` may not be present or may differ from server `TimeCurrent()` by a tick/second; historical comparisons to current `DecisionTime` become indirect.
- **B:** `TimeCurrent()` may be updated on the next tick/event, so it is not guaranteed to be the exact decision trigger time; can be equal to the forming-bar start at the boundary, weakening strict `<`.
- **C:** requires defining which field is authoritative; slightly more schema/backfill work.

**Impact on Specification:** A/C provide the strongest "real decision moment"; B is acceptable for event logging but weak for exact temporal ordering.

**Impact on M0:** None (export/decision-time only; M0 never vetoes).

**Impact on Evidence:** A/C make `strict_before_check` auditable against tick time; B alone leaves a one-tick ambiguity at bar boundaries.

**Recommended Option:** **C** — `RECOMMENDED — PENDING OWNER APPROVAL`. (Record `T_decision = tick.time`; keep `DecisionServerTime = TimeCurrent()` for reconciliation with existing artifacts.)

**Owner Approval Required:** YES.

---

## 6. D3 — Shift/Timeframe

**Decision ID:** D3

**Question:** Confirm Symbol / Timeframe / Shift / periods.

| Parameter | Required by C2/M2 | Current source | Decision |
|---|---|---|---|
| Symbol | `_Symbol` | `_Symbol` in all feature reads | **CONFIRMED** |
| Timeframe | `PERIOD_M1` | `g_tf[TF_IDX_MAIN].timeframe`, chart locked to M1 (`_Period != PERIOD_M1` → refuse) | **CONFIRMED** |
| Shift | `1` (last completed) | currently `0` | **REQUIRED REPAIR FROM 0 → 1** |
| ATR period | `4` | `inp_atr_period = 4` | **CONFIRMED** |
| ADX period | `14` | `inp_adx_period = 14`, DI lines same ADX handle | **CONFIRMED** |

Independent sub-decisions:
- **No sub-decision required** on symbol/timeframe/periods; they are already fixed and match the approved specification.
- The **only** item requiring change is Shift from `0` to `1`.
- A separate decision is whether to add a **runtime input** (e.g. `inp_c2m2_feature_shift`) versus hard-coding `1`. Hard-coding the literal `1` in the C2/M2 module is recommended (specification is Shift=1; no tunable).

**Pros of hard-coding:** cannot be silently changed; simpler no-lookahead proof; exact spec alignment.
**Cons of hard-coding:** any future change requires code change; but spec changes are out of scope.
**Impact on Specification:** Aligns.
**Impact on M0:** None (isolated module; see D6).
**Impact on Evidence:** FeatureShift literal `1` makes evidence self-documenting.

**Recommended Option:** Confirm all five values; change only Shift to `1`; hard-code `FeatureShift=1` in the C2/M2 module. `RECOMMENDED — PENDING OWNER APPROVAL`.

**Owner Approval Required:** YES (for the Shift=1 assignment; the identity values themselves are confirmed facts).

---

## 7. D4 — Temporal Invariant

**Decision ID:** D4

**Question:** Define the final testable invariant. Must it be strict `<` or can it be `<=`?

**Option A:** Strict: `Bar_CloseTime_Shift1 < T_decision`.

**Option B:** Non-strict: `Bar_CloseTime_Shift1 <= T_decision`.

**Pros**
- **A:** guarantees the Shift-1 bar is already closed before the decision; no boundary ambiguity; strongest no-lookahead guarantee.
- **B:** more forgiving at exact bar-open boundary; avoids rejecting decisions taken exactly at the first tick of a new bar.

**Cons**
- **A:** a decision exactly at the new-bar start (`T_decision == Bar_CloseTime_Shift1`) would be marked invalid unless the tick timing is precisely before the closed-bar close; this may require using `tick.time` (D2).
- **B:** allows a decision at the exact instant a bar closes, which may include a partial/first tick of the forming bar and weakens the proof.

**Impact on Specification:** A matches the strict wording `Bar_CloseTime_Shift1 < T_decision`.
**Impact on M0:** None (evidence check only).
**Impact on Evidence:** A makes `strict_before_check` a hard proof; B makes it a soft/failure-tolerant check.

**Recommended Option:** **A — strict `<`** — `RECOMMENDED — PENDING OWNER APPROVAL`. Note dependency: A requires D2 authority (`tick.time`) so boundary ticks are captured before `Bar_CloseTime_Shift1 + 1 sec`.

**Owner Approval Required:** YES.

---

## 8. D5 — LATCH

**Decision ID:** D5

**Question:** Which latch semantics?

**Option A:** Latch on the **first valid decision**; the decision is final at that instant (no re-evaluation, no retry after ALLOW).
**Option B:** Latch the **C2/M2 decision** (ALLOW or VETO) at the first valid decision; the existing `PlaceEntry` retry loop may retry order placement after an ALLOW without re-running the C2/M2 gate. No re-evaluation of the C2/M2 decision itself.

**Effects**

| Property | Option A | Option B |
|---|---|---|
| duplicate decisions | none after first valid | none after first valid (C2/M2), but `A/P/F` placement events may repeat |
| delayed fills | any failed fill after ALLOW → no retry | placement retry possible after ALLOW |
| veto fidelity | maximum (one-shot, immutable) | maximum for C2/M2 decision itself; order retry is outside C2/M2 |
| Step4 placement | potentially blocked on transient failure | ladder retry resilience preserved |
| research evidence | clean one row per `(SetupID,3,4)` | one C2/M2 decision row + separate placement events |

**Pros**
- **A:** simplest, strongest "one decision per setup" semantics; no ambiguity in duplicate veto suppression.
- **B:** preserves current ladder retry robustness; keeps C2/M2 decision latched while separating order execution from veto.

**Cons**
- **A:** can stall a legitimate Step4 ladder add if `PlaceEntry` fails once (e.g. transient margin/tick failure); changes current ladder reliability.
- **B:** introduces two distinct states (C2/M2 decision latch vs placement attempt); slightly more complex design and requires explicit evidence definition to prevent re-classifying placement events as new C2/M2 decisions.

**Impact on Specification:** Both satisfy "LATCHED VETO" for the C2/M2 decision. A is simpler; B adds a documented separation of decision vs execution.
**Impact on M0:** Either way M0 has no veto; B is closer to current execution behavior.
**Impact on Evidence:** A = minimal; B = requires a clear `C2M2_Decision` row, leaving `A/P/F` as ladder-event semantics.

**Recommended Option:** **B** — `RECOMMENDED — PENDING OWNER APPROVAL` (C2/M2 decision is latched; `PlaceEntry` retry is not part of the C2/M2 decision scope). If Owner prefers maximum strictness, choose A.

**Owner Approval Required:** YES.

---

## 9. D6 — C2/M2 Isolation

**Decision ID:** D6

**Question:** How to repair C2/M2 without unintentionally changing M0, M1, M3, or the shared `CF_ShouldVeto`?

**Option A:** Introduce a **new dedicated C2/M2 module** (e.g. `C2M2_GetShift1Features(...)`, `C2M2_ShouldVeto(...)`, `C2M2_Latch(...)`), used only for `StepFrom=3, StepTo=4` and only when the C2/M2 rule is active. Leave `CF_ShouldVeto` as-is for M1/M3/M0 logging.

**Option B:** Modify the shared `CF_ShouldVeto` to read index 1; add mode-specific conditions so M0/M1/M3 behavior unchanged.

**Option C:** Replace `CF_ShouldVeto` entirely with C2/M2 logic for all modes.

**Pros**
- **A:** maximum isolation; no risk to M1/M3; M0 untouched by construction; clean evidence path; explicit owner-reviewable scope.
- **B:** minimal new functions; shared code stays; but requires careful mode-specific guards and still risks cross-mode behavior change.
- **C:** simplest future C2/M2 semantics; highest cross-mode risk.

**Cons**
- **A:** more code/duplication and requires new evidence naming.
- **B/C:** risk of silently changing M1/M3; more regression surface; M1/M3 are outside the C2/M2 approved scope.

**Impact on Specification:** A provides a clean isolated implementation; B/C may be acceptable only if M1/M3 are explicitly approved to also change.
**Impact on M0:** A = untouched; B/C = requires proof by input-mode guard.
**Impact on Evidence:** A = separate C2/M2 evidence records; B/C = reuses CF event namespace, harder to distinguish.

**Recommended Option:** **A** — `RECOMMENDED — PENDING OWNER APPROVAL`.

**Owner Approval Required:** YES.

---

## 10. D7 — Export Architecture

**Decision ID:** D7

**Question:** Change semantics of `CF_events_m0.csv`, or use a dedicated C2/M2 export?

**Option A:** Modify `CF_events_m0.csv` (and `CF_events_m2.csv`) to record Shift-1 values.

**Option B:** Add a dedicated export, e.g. `C2M2_Step34_Decisions_m{inp_cf_mode}.csv`, leaving existing `CF_events_*` untouched.

**Pros**
- **A:** one event stream; simpler correlation with existing A/P/F events; fewer files.
- **B:** M0 untouched; existing CF events remain comparable to historical baselines; C2/M2 evidence is self-contained and clearly versioned; isolated attribution.
- **A:** no new file management; shorter reporting chain.

**Cons**
- **A:** changes M0 research-artifact semantics; invalidates comparison to all existing `CF_events_m0.csv` rows; raises M0 regression gate complexity (must accept documented intentional change).
- **B:** additional file; needs join key design (SetupID, StepFrom, StepTo, T_decision).

**Impact on Specification:** B keeps the approved C2/M2 evidence separate and explicit; A conflates legacy forming-bar logs with new Shift-1 evidence.
**Impact on M0:** B leaves M0 untouched; A changes M0 research output.
**Impact on Evidence:** B is the only option where historical Shift-0 rows can be kept unrelabeled while new Shift-1 evidence is added.

**Recommended Option:** **B** — `RECOMMENDED — PENDING OWNER APPROVAL` (must keep M0 untouched).

**Owner Approval Required:** YES.

---

## 11. D8 — Evidence Fields

**Decision ID:** D8

**Question:** Final minimum evidence field list and rationale.

| Field | Reason |
|---|---|
| `SetupID` | joins the decision to the actual setup and to PositionOpen/RX data |
| `StepFrom` | must be `3` |
| `StepTo` | must be `4` |
| `T_decision` | exact decision time (authoritative timestamp, see D2) |
| `FeatureShift` | literal `1`; machine-verifiable statement of the shift used |
| `BarOpenTime_Shift1` | `iTime(_Symbol,PERIOD_M1,1)`; identifies the closed bar used |
| `BarCloseTime_Shift1` | `iTime(...,1)+60`; the instant the Shift-1 bar closed |
| `ATR_Shift1` | primary feature; must be `GetBufferValue(buffer_atr,1)` |
| `ATRPct_Shift1` | `ATR_Shift1 * 100 / PriceRef_Shift1`; the actual veto metric |
| `ADX_Shift1` | M2 momentum feature |
| `DIPlus_Shift1` | adverse-DI feature component |
| `DIMinus_Shift1` | adverse-DI feature component |
| `PriceRef_Shift1` | `Close[1]`; denominator of ATRPct; no-lookahead anchor |
| `strict_before_check` | `Bar_CloseTime_Shift1 < T_decision`; the temporal proof |
| `provenance/source hash` | commit + source SHA-256 + version; prevents ambiguous evidence |

Additional recommended (non-required but useful):
- `Direction`, `tick_price`, `DecisionServerTime=TimeCurrent()`, `C2M2_Decision=ALLOW/VETO/INVALID`, `LatchStatus=FIRST/LATCHED_SKIP/NONE`, `Mode`, `Rule`.

**Recommendation:** Use the exact list above as the minimum; add the optional fields as diagnostics. `RECOMMENDED — PENDING OWNER APPROVAL` for the optional additions.

**Owner Approval Required:** YES (approval of the optional fields; the required list is already in the Repair Specification).

---

## 12. D9 — δ_DI

**Question:** Must an Owner decision on δ_DI be obtained before coding? What options exist? What effect does it have?

- **YES, an Owner decision is required before coding the behavioral C2/M2 gate.** Without δ_DI the C2/M2 "adverse DI" component is not fully specified.
- Possible options (for Owner), not chosen here:
  1. Keep current source rule: `di_against = is_bull ? (mdi > pdi) : (pdi > mdi)` (no explicit δ_DI).
  2. Use a defined numeric δ_DI for DI separation (threshold; not derived here).
  3. Use a directional-status method (`DIStatus`) with an Owner-specified tolerance.
  4. Defer δ_DI and keep C2/M2 as evidence-only (no behavioral gate) until Owner resolves.
- Effect:
  - If δ_DI remains unresolved → the C2/M2 decision rule is not fully specified; coding may proceed only for Shift-1 feature path + evidence, not for a final behavioral veto.
  - If Owner picks a method → the C2/M2 gate can be reconciled against the Shift-1 evidence.
  - If Owner picks "no δ_DI / evidence-only" → Shift-1 evidence repair can proceed; no behavioral veto.

**No δ_DI is selected in this document.**

**Owner Approval Required:** YES (Owner only).

---

## 13. D10 — Threshold

**Decision ID:** D10

**Question:** Confirm threshold policy for this stage.

- **No threshold is derived, changed, or otherwise determined.**
- Current `inp_cf_q75_atr_pct_34 = 0.0`, `inp_cf_q75_adx_34 = 0.0` (source lines 969–970) are **placeholders**, not C2/M2 thresholds.
- The Repair Specification should leave all threshold inputs untouched.
- The Shift-1 feature/evidence repair is independent of threshold values; a future, explicitly authorized threshold derivation is still prohibited at this stage.

**Recommended Option:** Locked — no threshold work. `RECOMMENDED — PENDING OWNER APPROVAL` (as confirmation of locking).

**Owner Approval Required:** YES (confirm the lock).

---

## 14. D11 — Historical Data

**Decision ID:** D11

**Question:** What to do with existing Shift-0 historical data?

**Principle (fixed):** `Historical Shift-0 evidence must not be relabeled as Shift-1.`

- Existing `CF_events_m0.csv` / `PositionOpenDataset` Step4 rows (`IndicatorShiftUsed=0`) remain **Shift-0, forming-bar records**.
- They may be used as the **historical population list** (which setups reached Step3→4), but **not** as evidence that the Step3→4 decision used Shift-1.
- If a new dedicated C2/M2 export is added, new rows are Shift-1 evidence and are joined to the historical population only by `SetupID`/`StepFrom`/`StepTo`, never by values.
- Historical data cannot be retroactively repaired.

**Recommendation:** Adopt the principle verbatim; still needs Owner confirmation. `RECOMMENDED — PENDING OWNER APPROVAL`.

**Owner Approval Required:** YES.

---

## 15. D12 — M0 Preservation

**Decision ID:** D12

**Question:** Which invariants must hold before coding to prove the repair does not change M0?

Define the following pre-coding M0 invariants:

1. **No order-side change in M0.** M0 executes the same `PlaceEntry` calls, same step progression, same stop/target geometry, same market/spread/margin gates.
2. **M0 never vetoes.** `inp_cf_mode == 0` must continue to bypass the C2/M2 veto (return `false` before any C2/M2 decision).
3. **Input defaults unchanged.** `inp_cf_mode=0`, `inp_cf_q75_atr_pct_34=0`, `inp_cf_q75_adx_34=0`, ATR=4, ADX=14, M1 chart, unchanged.
4. **No rule/threshold/δ_DI change.**
5. **M0 setup population unchanged.** Same number of setups reaching Step3→4 (historical 431/431 StepNo=4 vs CF Step3→4 A rows in the current population), same lifecycle events, same `PositionOpenDataset` StepNo counts (1=4864, 2=2970, 3=1149, 4=431, 5=191).
6. **M0 position-open records unchanged.** `IndicatorShiftUsed` remains `0` on `PositionOpenDataset`; only a new C2/M2 evidence field/file may be added.
7. **M0 `CF_events_m0.csv` unchanged** (if D7 Option B is chosen). If D7 Option A were chosen, this invariant would be explicitly waived only by Owner.
8. **No new dependency on live time in M0 decision path** except evidence capture; no new tick/latency-sensitive behavior.
9. **All new C2/M2 logic gated** by `inp_cf_mode > 0` (or an explicit C2/M2 enable flag) so it cannot execute in M0.
10. **Regression evidence:** a side-by-side M0 baseline (pre-repair) vs repaired M0 run must show identical trade outputs. **This is a validation / pre-coding acceptance gate, not executed now.**

**Recommended Option:** Enforce these invariants as a written acceptance gate before coding. `RECOMMENDED — PENDING OWNER APPROVAL`.

**Owner Approval Required:** YES.

---

## 16. Recommended Decisions

| ID | Recommended | Status |
|---|---|---|
| D1 | `PriceRef_Shift1 = Close[1]`, no Bid/Ask fallback | RECOMMENDED — PENDING OWNER APPROVAL |
| D2 | `T_decision = tick.time`; keep `TimeCurrent()` as `DecisionServerTime` | RECOMMENDED — PENDING OWNER APPROVAL |
| D3 | Confirm Symbol/TF/periods; repair Shift 0→1; hard-code `FeatureShift=1` | RECOMMENDED — PENDING OWNER APPROVAL |
| D4 | Strict `<` | RECOMMENDED — PENDING OWNER APPROVAL |
| D5 | Option B (C2/M2 decision latched; PlaceEntry retry separate) | RECOMMENDED — PENDING OWNER APPROVAL |
| D6 | Option A (isolated C2/M2 module) | RECOMMENDED — PENDING OWNER APPROVAL |
| D7 | Option B (dedicated `C2M2_Step34_Decisions_*`) | RECOMMENDED — PENDING OWNER APPROVAL |
| D8 | Required field list in §11; optional diagnostics allowed | RECOMMENDED — PENDING OWNER APPROVAL |
| D9 | Owner-only; no option selected | NOT DECIDED — OWNER |
| D10 | No threshold derivation/change | LOCKED |
| D11 | Historical Shift-0 never relabeled | RECOMMENDED — PENDING OWNER APPROVAL |
| D12 | M0 invariants as pre-coding acceptance gate | RECOMMENDED — PENDING OWNER APPROVAL |

---

## 17. Decisions Requiring Owner Approval

1. **D1** — PriceRef no-fallback / `INVALID` decision policy.
2. **D2** — `tick.time` vs `TimeCurrent()` authority.
3. **D3** — confirm Shift=1 repair (facts confirmed; behavior change needs sign-off).
4. **D4** — strict `<` vs `<=`.
5. **D5** — Latch Option A vs B.
6. **D6** — C2/M2 isolation architecture.
7. **D7** — Export architecture (independent C2/M2 export).
8. **D8** — approval of optional diagnostic fields (required list already in spec).
9. **D9** — δ_DI (Owner only; **no value selected here**).
10. **D10** — confirm no-threshold lock.
11. **D11** — historical Shift-0 non-relabeling principle.
12. **D12** — M0 preservation acceptance invariants.

---

## 18. Items Explicitly Not Decided

- **δ_DI value** — not selected.
- **Threshold value(s)** — not determined.
- **Latch invalid-input behavior for D1** — the `INVALID` decision policy (allow/veto/no-decision) is not fixed here.
- **M1/M3 semantics** — not changed or redefined; any change to them is out of the C2/M2 scope.
- **Retry behavior details** beyond D5 Option B — exact number of placement retries, timeout, and reset are not specified.
- **Exact file naming/provenance manifest schema** — only a proposed pattern (`C2M2_Step34_Decisions_*`) is given; exact schema waits for design phase.
- **Temporal authorities in edge cases at bar boundaries** — dependent on D2/D4 decisions.
- **Any historical data recoding** — prohibited.
- **Coding start** — not authorized.

---

## 19. Pre-Coding Gate

Coding may **not** begin until all of the following are resolved/approved by the Owner:

1. **D1** approved → PriceRef policy decided (including invalid-input decision behavior).
2. **D2** approved → `T_decision` authority fixed.
3. **D3** approved → Shift=1 mapping for C2/M2 confirmed.
4. **D4** approved → strict `<` invariant confirmed.
5. **D5** approved → latch semantics fixed.
6. **D6** approved → C2/M2 isolation architecture fixed.
7. **D7** approved → export architecture fixed (M0 untouched).
8. **D8** approved → minimum + optional evidence fields confirmed.
9. **D9** Owner resolves δ_DI (or explicitly declares "evidence-only, no δ_DI").
10. **D10** confirmed → no threshold derivation/change.
11. **D11** confirmed → historical Shift-0 data stays Shift-0.
12. **D12** confirmed → M0 preservation invariants and regression acceptance gate agreed.
13. **Documented evidence hierarchy** confirmed → Shift-1 C2/M2 evidence is a new authority; no code is permitted until the new evidence export is specified and accepted.

After the gate, the next step is only to produce the **implementation design** (pseudocode / file-level diff plan) for Owner review. **No code, no test, no backtest, no threshold, no freeze, no production change may occur before the gate passes.**

---

**FINAL**

```
DECISION LIST READY FOR OWNER REVIEW
```

Production remains **M0**. No `.mq5` changed, no new code, no test/backtest/OOS/TRUE_FORWARD executed, no threshold changed, no δ_DI selected, no freeze, no rule registered.
