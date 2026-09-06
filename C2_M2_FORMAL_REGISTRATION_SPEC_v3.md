# C2 / M2 — FORMAL REGISTRATION SPECIFICATION v3 (supersedes v2; audit corrections applied)

**Date:** 2026-09-06 · **Phase:** AFTER PRE-FREEZE AUDIT (R1 corrections resolved)
**Status at issuance:** `PRE-FREEZE AUDIT = PASS WITH CORRECTIONS RESOLVED · C2/M2 = APPROVED FOR REGISTRATION SPECIFICATION · C2/M2 = NOT YET FROZEN · Production = M0`
Chain: prereg `1d976a6` → D3/D4-v1 `ad9f231` → D4-v2 `1d0b390` → memo `a25202f` → sheet v2 `606c07f` → spec v1 `871fe50` → spec v2 `48ad17e` → audit v1 `576c76e` → **this v3 (owner-accepted corrections applied; supersedes v2)**.
**Amendments applied (owner-directed R1):** A-1 named unique PriceRef (§4/§5); A-2 G-P registered as Pre-Freeze Validity Gate (§14) with explicit consequence chain (§6/§15/§17); A-3 M1 literal fully redacted from C2/M2 text (§19). **No threshold values, no extraction-time population counts, and no other literals have been computed, extracted, or frozen at any point; every such quantity is a future freeze-act output.**
**Forbidden (standing):** threshold calculation · freeze · backtest · OOS · TRUE_FORWARD · sweep/tuning · code change · production change.

---

## 1. Official rule name & version

- **Rule name:** C2/M2 — *Latched Step3→4 Volatility-Trend Veto*
- **Rule ID:** `C2_M2_S34_LATCHED_ATR_ADX_DI_V1` · **Rule version:** 1.0 (amended by owner-signed hash-chained amendments)
- All artifacts, logs, and reports referring to this rule MUST cite the Rule ID and version.

## 2. Population (exact)

- Source run family: **M0 baseline** exports; identical definition applies to any candidate run.
- Rows: event rows satisfying `Event = "A" AND StepFrom = 3 AND StepTo = 4 AND Mode = 0`.
- **One first-valid decision per Setup** (§10); if an export ever contains more than one qualifying decision row per setup, the population element is the **earliest**; the uniqueness gate (§14) counts and reports violations with no silent repair.
- Identity: setup fingerprint = (Step1 decision time, entry price, direction, strategy tag); raw SetupID is never a join key.
- Population identity is **conditional** on G-P = PASS (§14): extraction and any downstream derivation are blocked until the emission-point verification completes.

## 3. Decision timestamp & feature timestamp (exact)

| Symbol | Definition |
|---|---|
| `t_d` (decision time / `T_decision`) | the tick time of a logged Step3→4 decision row — the instant the player evaluates the entry decision for that attempt |
| `t_f` (feature time) | the latest information instant legally usable by any feature = `t_d` itself, restricted by the information set of §4 |
| Invariant | every feature value must be computable from information satisfying `t ≤ t_d` under §4; violation invalidates the run (§18) |

## 4. Shift=1 / closed-bar information set & PriceRef (exact; auditable; A-1)

- **Admissible set (primary definition):** for a decision at `t_d` on timeframe M1, let `b0(t_d)` be the (unique) M1 bar whose interval **contains** `t_d` (i.e., `open(b0) ≤ t_d < close(b0)`); `b0` is the FORMING bar and is **inadmissible**. The admissible set = all bars strictly preceding `b0(t_d)` — exactly the bars referenced by MQL5 shift indices `≥ 1` at `t_d`. A decision tick with timestamp exactly equal to a bar's open time belongs to that bar (it is forming); all prior bars are admissible. This follows deterministically from the interval rule.
- **Derived arithmetic equivalent (origin explicitly stated):** because timeframe M1 has a registered period of 60 seconds, `open(b0) = floor(t_d, 60s)`, so the admissible set equals `bars with open_time ≤ floor(t_d,60s) − 60s`. This arithmetic is presented **only** as a consequence of the registered timeframe and the platform's bar-indexing model — not as an independently sourced invariant.
- **PriceRef (unique named definition):** `PriceRef(t_d) = Close` **of the last fully-closed M1 bar before `T_decision`** — i.e., the Close of the last admissible bar at `t_d`. **This is the one and only price type permitted as the volatility ratio's reference; no other price type (Bid/Ask/open/high/low/typical/…) is admissible.** PriceRef is deterministic per decision and constant for all decisions inside the same admissible-bar boundary; the definition applies uniformly across the whole population.
- Every feature (ATR, ADX, DI+, DI−, and PriceRef) is computed **exclusively** from the admissible set; all features of one decision are referenced to the **same last admissible bar** (“single bar reference” for shift=1).
- Feature availability: an indicator needing k bars is well-defined only if ≥ k admissible bars exist at `t_d`; otherwise the row enters the missingness-audit path (§14 G-C), never silent imputation.
- The freeze artifact pins: the exact definition text above, the bar-history source, and a sample of independently re-derived rows (audit gate).

## 5. ATRPct (exact; A-1)

```
ATRpct(t_d) = ATR(14, M1, Wilder/MQL5 iATR, shift=1 at t_d) / PriceRef(t_d) × 100
```
Numerator and denominator come from the SAME last admissible bar (§4); PriceRef per §4 (Close only); units = percent.

## 6. ATR threshold methodology (exact; future outputs only)

- Statistic: **Q75, Hyndman–Fan type-7** — `h = (n−1)·0.75 + 1` (1-based fractional rank over ascending DEV values), linear interpolation between bounding order statistics; both neighbors recorded.
- Population: DEV attempt population (§2 restricted to §12 DEV window), single-shot extraction **at the freeze act, and only after G-P = PASS (§14)**.
- **No threshold value (τ_ATR, τ_ADX, δ_DI) and no extraction-time population size (n_DEV) has been computed, extracted, or frozen.** The exploratory-phase reference count (366 research attempts in the DEV window, previously registered as knowledge artifact number 366) is an **expectation only**, MUST be re-verified during freeze-time extraction under G-P/G-U/G-S/G-R/G-C/G-T, with any deviation reported and explained — never silently repaired. All such literals are future freeze-act outputs and appear in this document as symbols only.

## 7. ADX threshold methodology (exact)

- Feature: `ADX(14, M1, shift=1 at t_d)`.
- Statistic: Q75, type-7, identical procedure and population as §6; literal recorded at freeze (future output).
- Registered note (carried): the ADX quantile is a **structural, outcome-blind choice**; no outcome-linked evidence exists for ADX at any quantile — this sentence must appear verbatim in the freeze artifact.

## 8. Direction-aware DI (exact; δ proposed/default only)

- Components: `DI+(t_d), DI−(t_d)` from the same `ADXD(14, M1, shift=1 at t_d)` computation (same bar reference).
- `δ_DI` = dominance margin: a **proposed/default specification parameter (proposed form: 0)**. **It is NOT FROZEN** by this specification; it becomes literal only at the freeze act by explicit owner decision.
- Definition:
  - setup direction = **SELL** ⇒ `DI_ADVERSE ⇔ (DI+ − DI−) > δ_DI`
  - setup direction = **BUY** ⇒ `DI_ADVERSE ⇔ (DI− − DI+) > δ_DI`
- Setup direction is the baseline strategy's own immutable direction attribute for that attempt (never inferred from outcomes). Strict `>` ⇒ exact equality is NOT adverse (deterministic tie-break).

## 9. Final Boolean rule (exact)

```
VETO(t_d) ⇔ [ ATRpct(t_d) ≥ τ_ATR ]  AND  [ ADX(t_d) ≥ τ_ADX  AND  DI_ADVERSE(t_d) ]
```
Single conjunction at one decision instant (`ATR_condition AND Adverse_condition`); direction-mapping per §8; no OR/NOR/else branches; any change = new version (§1).

## 10. LATCHED semantics (documenting confirmed Decision 3; gate-order-aligned)

- **First valid decision (t₀):** per setup, the first decision instant `t_d` at which ALL execution preconditions pass — i.e., the entry candidate has cleared **Market Gate → Spread Gate → Margin Gate** (§11) AND the §4 information-set features are well-defined. Decisions failing any execution gate never become population members and can never be a first-valid decision.
- **VETO state:** if `VETO(t₀)` is true, a per-setup persistent veto state is set for that setup.
- **Post-VETO:** *Step4 of THAT Setup is never allowed* — a per-Setup prohibition tied to that setup's own veto; **not** a general Step4 ban on other setups. The setup's step chain stops at Step3 (no Step5+, steps being sequential).
- **No reopen / no re-decision:** the veto state never clears and is never re-evaluated for the life of the setup.
- Scope: nothing else is touched — management, exits, other legs, other setups proceed per baseline.
- Implementation note (status unchanged): latched semantics requires minimal code support in the player; **owner's code authorization remains the only execution precondition** — registration proceeds as specification; test execution stays blocked until authorization exists.

## 11. Gate order & veto position (FINAL — owner-corrected @ v2)

The exact required pipeline at a decision instant:

```
Market Gate  →  Spread Gate  →  Margin Gate  →  First Valid Decision  →  LATCHED VETO  →  PlaceEntry(next=4)
```

Registered semantics:
1. **Only execution-gate-eligible candidates enter the VETO decision** — Market/Spread/Margin gates run BEFORE the veto; a candidate rejected by any of them never reaches feature evaluation for veto purposes (it is not a population member, and no veto state is set by it).
2. **The first valid decision IS the eligible and complete decision** — the first t_d after these gates pass (§10).
3. **If the rule returns VETO=true at that decision, Step4 is never allowed for that Setup** (per §10; per-Setup only).
4. **If VETO=false, the path continues to `PlaceEntry(next=4)`** through the unchanged baseline placement mechanics.
5. **Latched is per-Setup only; no reopen exists** (§10).
6. Instrumentation requirement: gate outcomes and the veto decision are logged in this order for every decision instant; the logs must make the order verifiable (a V row implies Market/Spread/Margin PASS at that tick); any candidate run whose logs imply a different order is rejected (§18). Audit note: the required instrumentation of this exact order is part of the pending code authorization.

## 12. DEV / OOS boundaries (exact)

- **DEV:** `t_d < 2024-01-01 00:00:00` — sole derivation window for parameter-bearing artifacts (exploratory/selection zone).
- **OOS:** `2024-01-01 00:00:00 ≤ t_d ≤ registered cutoff (2026-08-18 23:59:58)` — confirmatory zone; quarantined for selection; read-only.
- **TRUE_FORWARD:** `t_d > cutoff`, live/paper — the only zone that can ever support production consideration.
- Mandatory labels on every figure: `exploratory-DEV` / `confirmatory-OOS` / `TRUE_FORWARD`.

## 13. Leakage controls (exact, binding)

1. Derivation whitelist reads only: identity fields, decision fields, feature fields; **outcome/PnL/lifecycle columns forbidden** at derivation (assert + log).
2. **Outcome quarantine:** no outcome access before the freeze commit (commit-order proof).
3. **Single-shot freeze:** exactly one act fixes (τ_ATR, τ_ADX, δ_DI) + convention + population + formulas; no later adjustments; defect-fix amendments are owner-signed, hash-chained, value-neutral.
4. **No retuning:** no sweep, no sensitivity-based selection, no veto-rate targeting; joint rates are reported consequences only.
5. **First-decision alignment:** derivation population ≡ deployment population (§2, §10), verified under G-P (§14).
6. **OOS quarantine pre-freeze.**
7. **Convention exactness:** every feature carries its shift=1 flag with the §4 single-bar-reference rule; mixing information sets voids artifacts (§18).
8. **Family discipline:** parameter family = exactly three literals (τ_ATR, τ_ADX, δ_DI); anything additional = new version + new authorization.

## 14. Integrity & validity gates (exact; A-2 adds G-P)

- **G-P (Pre-Freeze Validity Gate — population emission, NEW):** it must be **verifiable** — from the player's implementation and/or its instrumentation — that `Event=A` rows are emitted **exactly at first-valid post-execution-gate decision instants**, i.e., that the recorded `Event=A` population is exactly the set of candidates that PASSED Market Gate → Spread Gate → Margin Gate (§11). **Nothing in this phase performs any re-filtering, derivation, or calculation for this gate; it is registered as a pure verification condition.** Consequence chain, binding: **G-P = PASS is a precondition for ANY threshold derivation; if the implementation/instrumentation cannot prove this relation, the freeze act does not take place (HALT, §18).**
- **G-U (uniqueness):** per-setup first-valid-decision count == 1 (under §10's definition); violations listed, counts reconciled, no silent drops.
- **G-S (stream span):** events/lifecycle/PO spans reach registered window ends; expected tail setups present per reference.
- **G-R (reconciliation):** event rows / lifecycle opens / PO legs / manifest counts reconcile (equality or explained deltas listed).
- **G-C (feature completeness):** per-column missing/NaN/non-numeric counts on the whitelist, each classified by cause (instrumentation gap / structural N/A / truncation artifact) before any action; exclusions listed with cause + population impact; zero-impact claims must be justified, else escalated (§18).
- **G-T (time integrity):** parse-clean, monotonic per setup, in-window.
- All gates produce a written gate report shipped with the data drop; **all** must PASS before any analysis or freeze-time derivation.

## 15. Provenance & required freeze artifacts (schema)

**`C2_THRESHOLD_FREEZE.json`** (created only at the freeze act — NOT now) must contain:
`rule_id, version` · population definition + SHA-256 of its extraction set · convention block (§4 primary shift definition + stated origin of the arithmetic equivalent + PriceRef definition) · DEV/OOS windows · per-feature formulas (§5–§8) · **`τ_ATR`, `τ_ADX`, `δ_DI` literals (freeze-act outputs)** · derivation method (§6–§7) + derivation-script SHA-256 · both bounding order statistics per quantile (freeze-time records) · extraction-time n (freeze-time record, reconciled under gates) · DEV descriptive blocks · joint DEV eligibility rate (descriptive only — **never a target**) · gate-order declaration (§11) as implemented in the player · integrity-gate results (**including G-P**, §14) · owner sign-off block · commit hash.
**Provenance chain:** `234dc6b → f4fd889 → 164ea76 → 3912e21 → … → 606c07f → 871fe50(v1) → 48ad17e(v2) → 576c76e(audit v1) → this v3`.
**No artifact containing any C2/M2 parameter may reference, reuse, compare against, or anchor on the M1 threshold** (§19).

## 16. Acceptance / Rejection framework (structure only)

D4-10 six-dimension framework (causal paired delta · loss-tail · robustness · coverage · sample adequacy · no-search-artifact) with **no numeric margins here** (margin literals enter the freeze act / final registration). Verdicts: `ACCEPTED (research-stage)` / `NOT ACCEPTED — INCONCLUSIVE` / `REJECTED`; sample shortfalls ⇒ `INCONCLUSIVE BY DESIGN`.

## 17. Conditions for a VALID registration

1. Every section instantiated with required literals (τ_ATR, τ_ADX, δ_DI, margin literals, code-authorization reference) — all as freeze-act / final-registration outputs; none exists yet.
2. No contradiction with confirmed decisions (D3/D4-1…D4-12 including the FINAL gate order of §11).
3. Freeze commit precedes all candidate-run artifacts and all OOS-derived material (commit-order proof).
4. All gates operational on the extraction set with PASSing reports — **including G-P**.
5. Owner approval of the freeze act recorded with its hash.

## 18. Conditions under which registration is REJECTED or HALTED

1. Any missing literal or gate definition required by §17-1.
2. Any deviation from confirmed decisions or final gate order; any information-set or price-reference mixing (§4/§5/§13-7).
3. Any evidence of outcome-linked derivation or post-hoc value adjustment.
4. Any unsatisfiable gate on the real extraction set: HALT + escalate, never silent patch — with an explicit clause: **G-P ≠ PASS ⇒ no freeze, no derivation, HALT (§14 G-P).**
5. Feature-timestamp invariant violation (§3).
6. Any C2/M2 artifact referencing the M1 threshold (§19).
7. Execution boundary: without code authorization for latched semantics + §11 instrumentation, **test execution HALTs**; the specification may stand valid, but no run may be produced.

## 19. C2/M2 vs M1 — registered separation (binding; A-3: literal redacted)

| Aspect | M1 (closed line) | C2/M2 (this specification) |
|---|---|---|
| Shift / information set | shift=0 (forming-bar, as implemented) | **shift=1 (closed-bar), §4** |
| Rule form | ATRpct ≥ τ only | **conjunction with trend/DI adversity, §9** |
| Persistence | per-tick re-evaluation (delay-dominated, observed) | **latched single first-valid decision, §10** |
| Veto position | attempt-level (implementation-era) | **after Market/Spread/Margin gates, before PlaceEntry, §11** |
| Threshold | `«M1-THRESHOLD»` — **redacted; defined solely in `M1_THRESHOLD_FREEZE_POSTHOC.json` (M1 provenance, POST-HOC RE-FROZEN label)** | **must NOT enter C2/M2 in any form** — not as value, prior, sanity anchor, comparison row, or fallback |
| Experiment status | export incomplete; `PERFORMANCE VERDICT = NOT ISSUED`; provenance-only | no run has occurred; nothing to reuse |
| Evidence status | exploratory-only, voided as accept/reject basis | tabula rasa; starts at this specification |

Any M1-traceable number inside a C2/M2 artifact voids that artifact (§18-6). The M1 literal must never appear inside C2/M2 text except in redacted/token form (this row).

---

## Closing registered status

```text
PRE-FREEZE AUDIT = PASS WITH CORRECTIONS RESOLVED
C2/M2 = APPROVED FOR REGISTRATION SPECIFICATION
C2/M2 = NOT YET FROZEN
PRODUCTION = M0
```

**Halting here as instructed.** Threshold calculation, freeze, backtest, OOS, TRUE_FORWARD, sweep/tuning, code change and production change remain forbidden and untriggered.
