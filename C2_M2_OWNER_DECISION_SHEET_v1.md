# C2 / M2 — OWNER DECISION SHEET v1 (D4-1 … D4-12) · DESIGN ONLY

**Date:** 2026-09-06 · **State:** awaiting Owner decisions. Nothing computed, executed, frozen, registered, or changed.
Chain: prereg `1d976a6` → D3/D4-v1 `ad9f231` → D4-v2 `1d0b390` → memo `a25202f` → **this sheet**.
*No new numbers are produced here; observations are referenced only as already-registered qualitative facts.*

---

## D4-1 — Shift Convention

> Key insight: the shift is **not an engineering reproducibility knob** — it *redefines the feature* and the **information set available at decision time**. `ATR(t | completed bars ≤ t)` (shift=1) vs `ATR(t | including current forming bar)` (shift=0) are **different features** with different causal content, different distributions, and different thresholds. All downstream artifacts (threshold literal, DEV distributions, run-time comparison) are only valid for the chosen information set.

| Aspect | shift=0 (forming bar) | shift=1 (closed bar) |
|---|---|---|
| Real-time operational fidelity | matches what a live EA touches at the tick; zero lag | 1-bar lag at decision; live EA reproduces it trivially (closed bars are stable) |
| Reproducibility | requires identical tick path; intrabar range growth makes values time-within-bar dependent | bit-identical in any environment with the same bar history |
| Auditability | value depends on exact tick sequence ⇒ hard to re-derive independently | independently re-derivable from OHLC |
| Timing semantics | decision uses "now-state"; borderline values flap during a bar | decision uses "last-complete-state"; stable within a bar |
| M1 compatibility | matches historical M1 logs (but M1 results are exploratory/void anyway) | new feature basis; M1 provenance explicitly non-transferable (registered) |
| Look-ahead/leakage risk | none intrinsically, but intrabar state in *tester* vs *live* can differ silently (feed modeling) | none; information set provably causal |

**Proposed default: shift=1.** Reason: the veto is a risk gate on basket-horizon horizons; a ≤1-bar staleness is negligible, while determinism of the information set converts the rule from a tick-path artifact into a portable feature. Choosing shift=0 would require registering feed-model assumptions. **Consequence if chosen:** feature formulas, DEV distributions, threshold literals, source data (bar-history vs event-log columns), and the freeze artifact all reference closed-bar definitions; old shift-0 values are retired.

## D4-2 — ATR Representation / Threshold Type

| Option | Pros | Cons |
|---|---|---|
| raw ATR (points) | simplest | price-level non-stationarity (gold tripled across the window) ⇒ threshold meaning decays; **defensible only if window short** |
| ATRPct (×100/price) | level-invariant; single feature, zero extra degrees of freedom; matches project's historical slot semantics; maximal reproducibility | regime drift remains (distribution of ATRPct shifts) |
| relative/rolling normalized (causal) | veto-rate ~stationary under regime drift; philosophically matches DEV/OOS discipline | adds lookback parameter + warmup rules + arrival-path dependence + strict causality audit; more failure modes |

**Proposed default: ATRPct with an absolute DEV-frozen value; rolling normalization = DEFERRED.** Reason: ATRPct already removes price-level non-stationarity; regime-drift robustness is then tested *deliberately* as part of the rule's acceptance criteria (an explicit, honest joint test), instead of being hidden inside adaptive machinery whose own parameters would need separate registration. The rolling form doubles the hyperparameter surface (lookback, warmup, estimator) and is deferred to a possible future separate registration. **Consequence if chosen:** threshold literal is a single DEV-frozen constant; acceptance criteria must include the drift-sensitivity statement (per AM-3).

## D4-3 — Threshold Method (defensibility of DEV-Q75 as PROPOSED METHOD)

- **Defensible if** it is signed ex ante as a structural, outcome-blind coverage choice: distribution-free, exactly defined (type-7), one value per feature, fixed in a single freeze act.
- **Not defensible if** it is compared against outcomes, selected among alternatives later, or adjusted for hit-rate/performance (all prohibited by standing clauses).
- Register with it: conjunction-mechanics statement (joint veto-rate is a measured consequence, never a target); evidence-asymmetry statement (ATR gradient exploratory-registered; ADX has no outcome-linked evidence at any quantile — the ADX quantile is a structural guess); family-of-two discipline (both literals in one act).
**Proposed default: DEV-Q75 (type-7) as Proposed Method, subject to Owner signature.** Reason: maximally simple, faithful to the project's design vocabulary, statistically well-defined at the available DEV sample sizes. **Consequence:** the freeze act contains exactly two literals (τ_ATR, τ_ADX) derived under this method, after D4-1/D4-2 are fixed.

## D4-4 — Population

Proposed: M0 baseline · `Event=A, StepFrom=3, StepTo=4 (Mode=0)` · **one first-valid decision per Setup** · latch-aligned.
- **Uniqueness gate (exact definition):** per Setup, count of first-valid-decision rows == 1 (report all violations; any setup with 0 or ≥2 such rows is listed and the population count reconciles; violations are not silently dropped — see D4-11).
- **Coverage gate (exact definition):** population spans the registered window endpoints (earliest and latest expected attempt windows present against M0 reference); feature completeness = 100% non-empty/non-NaN on all whitelisted feature columns; decision timestamps strictly ordered and parse-clean.
- **Latched alignment:** the deployment trigger (first tick at which the Step3→4 placement condition fires) must equal the calibration row's tick — asserted by instrumentation in any candidate run.
**Proposed default: as proposed (validated in memo `a25202f`).** Reason: calibration population ≡ deployment decision point is the design's core causal property. **Consequence:** population definition + hash becomes part of the freeze artifact.

## D4-5 — DEV / OOS

- DEV = 2014–2023 attempts; OOS = 2024 → registered cutoff. **DEV is exploratory; OOS is confirmatory — never blended in one claim.**
- No OOS results are computed in this phase. At evaluation time: OOS-first reporting; DEV figures always carry the in-sample-selection label; a DEV number may never be quoted without that label.
**Proposed default: split as defined (already registered boundary).** Reason: preserves the project's provenance-locked boundary and the confirmatory meaning of OOS. **Consequence:** all reporting templates need the exploratory/confirmatory label field.

## D4-6 — Adverse Trend/Momentum Definition (options compared, no outcome-based choice)

| Option | Content | Pros | Cons |
|---|---|---|---|
| **A. ADX + direction-aware DI** | `ADX ≥ τ_ADX` AND (for SELL: DI+>DI−; BUY: DI−>DI+) | measures trend strength + direction against the position; two components, interpretable; matches historical slot design | two parameters (τ_ADX, dominance form); DI says nothing about speed/momentum reversal |
| **B. MACD histogram sign/size** | momentum sign against position (optionally ≥ τ_MACD) | single-parameter family; captures momentum not trend | noisy on M1; overlapping information with price position; another unproven threshold |
| **C. RSI extreme** | `RSI ≥ τ_overbought` for SELL (mirror for BUY) | directly captures the strategy's reversal premise (entry was already divergence-based) | RSI is part of the entry signal already ⇒ redundancy risk; threshold unproven |
| **D. Permitted combinations** | A only in the primary rule; B/C allowed only as pre-declared secondary *descriptors*, never in the veto equation | preserves single composite; prevents rule-sprawl | — |

**Proposed default: Option A (ADX + direction-aware DI), δ=0 as *proposed* form of dominance (pure sign), explicitly NOT final.** Reason: it is the only option whose two components are independently interpretable at a *decision-point trend* level (rather than duplicating the entry's own divergence logic), keeping the family's hyperparameter count minimal. **Consequence if chosen:** §5 of the prereg takes the A-formula; δ is Owner-final later (no default asserted anywhere in binding text).

## D4-7 — Boolean Rule (structure review only)

`VETO ⇔ ATR_condition AND adverse_condition` — a pure **conjunction** at one decision instant, direction-mapped.
- Reviewed consequences: (i) veto-rate < either marginal rate (multiplicative-ish shrink) ⇒ the affected cohort shrinks; power must be re-checked at evaluation; (ii) miscalibration of either leg compounds (AND is unforgiving — a useless leg can only *remove* true-positive attempts but never add them); (iii) symmetry: the same structure for BUY/SELL, only the adverse leg flips sign; (iv) no OR/NOR/variant forms are registered — any future change to logical structure is a new amendment.
**Proposed default: conjunction as written; literals and adverse definition remain undecided.** Reason: matches the owner-supplied hypothesis exactly; simplest falsifiable form. **Consequence:** the prereg's §6 formula is finalized wording-wise at registration.

## D4-8 — LATCH Semantics (CONFIRMED — not reopened)

Precise definitions to register:
- **First-valid decision (t₀):** the first tick at which, for a given setup, (a) the Step3→4 placement condition fires, (b) features for the chosen D4-1 information set are well-defined, and (c) the EA would otherwise invoke entry placement (pre-gate state); t₀ is logged with setup fingerprint.
- **Post-VETO semantics:** if VETO(t₀) fires, **Step4 never opens for that setup** — permanently for the life of the setup; the step chain stops advancing at Step3 (no Step5+ can occur, since steps are sequential); the veto touches nothing else (management/exits/other legs continue per baseline); if the setup later invalidates/fails, it dies naturally without step 4.
- Re-arm: none. Per-tick re-evaluation is outside the registered semantics (retired to a hypothetical sensitivity track only).
**Proposed default: already CONFIRMED by Owner (Decision 3).** Consequence: code authorization (D4-10/OD-10) remains the only execution blocker.

## D4-9 — DEV Reuse / Provenance (what a confirmatory claim requires)

DEV has been used for exploration and hypothesis generation (registered). Therefore:
- **From DEV, at most:** exploratory figures with mandatory labels; the parameter-bearing artifacts (convention, literals) frozen from DEV before any outcome exposure.
- **A confirmatory claim requires, cumulatively:** (1) the single-shot freeze (hash-chained) before any candidate run and before OOS access; (2) a candidate run passing all integrity/coverage gates; (3) OOS-first evaluation against pre-registered acceptance criteria; (4) TRUE_FORWARD accumulation per the registered requirement before any production consideration.
- No DEV-based confirmatory statement is possible now or later; DEV never upgrades.
**Proposed default: as stated (already the registered chain).** Reason: only a fresh, post-freeze evidence stream can carry confirmatory weight. **Consequence:** every future report template has the label field (exploratory-DEV / confirmatory-OOS / TRUE_FORWARD).

## D4-10 — Acceptance / Rejection Criteria Framework (structure only; margins are proposals, not science)

| Dimension | Confirmatory question | Proposed default (structure) |
|---|---|---|
| causal paired delta | does the affected cohort improve vs its M0 twin? | primary endpoint: paired ΣΔ with block-bootstrap CI; one-sided benefit test |
| loss-tail behavior | does the harm mass shrink? | worst-decile setup mass & SL$-mass share inside the eligible cohort must not worsen |
| robustness | is the effect year/regime-stable? | sign-consistency report across years; OOS-read required non-harmful |
| coverage | are exports complete? | all D4-11 gates pass as preconditions |
| sample adequacy | enough affected pairs? | minimum-n gate; if unmet ⇒ INCONCLUSIVE BY DESIGN (never "no effect") |
| no search artifact | is the pipeline single-shot? | one freeze, one run family, script committed pre-unblinding, drift sensitivity reported |
- Rejection directions: harm beyond margins; integrity failure; mechanism mismatch; benefit-fragility (concentration cap concept, margin owner-set later).
**Proposed default: framework as tabulated; all numeric margins/minima remain Owner-set proposals inside the future registration.md.** Reason: only the Owner can price risk tolerances; the analyst provides structure. **Consequence:** registration cannot complete without these literals.

## D4-11 — Missing / Coverage Gate (pre-analysis gates; missingness handled explicitly, never silently)

Gates (all PASS required before any analysis or freeze use):
1. **Stream-span gate:** events/lifecycle/PO spans cover the registered window (terminal timestamps ≥ reference coverage; final-day setup presence).
2. **Row/setup-count reconciliation:** event rows, lifecycle opens, PO legs, and manifest counts mutually reconcile (exact equality or explained deltas listed).
3. **Uniqueness gate** (D4-4 definition).
4. **Feature completeness audit:** per-column missing/NaN/non-numeric counts on the whitelisted columns, with **cause classified** before any action: (a) instrumentation gap, (b) structural N/A (e.g., feature undefined at that tick), (c) truncation/export artifact. **Rows are never auto-dropped;** each excluded row is listed with cause, and its population impact is reported as a population-bias note. Zero-impact classification must be justified; otherwise the case is escalated to the Owner.
5. **Time-integrity:** parse-clean timestamps, monotonic per setup, no future dates.
**Proposed default: gates 1–5 exactly as written.** Reason: the previous phase demonstrated that silent truncation silently changes evidence — this gate-family hard-codes the lesson. **Consequence:** every data drop arrives with a gate report file.

## D4-12 — Production Boundary

```text
Production = M0
C2/M2     = DESIGN ONLY
Until explicit Owner Approval: Registration = NO | Freeze = NO | Backtest = NO | Code Change = NO | OOS Analysis = NO
```

---

## FINAL TABLE

| Decision | Options | Proposed Default | Reason | Consequence (what changes in final spec) | Owner Status |
|---|---|---|---|---|---|
| **D4-1** Shift | 0 forming / 1 closed | **1 (closed)** | redefines feature+info-set; determinism; bar-auditability; staleness negligible for basket gate | all features/DEV-stats/threshold literals = closed-bar basis; shift-0 values retired | **AWAITING APPROVAL** |
| **D4-2** Threshold type | raw ATR / ATRPct absolute / rolling-normalized | **ATRPct absolute; rolling DEFERRED** | level-invariance free; drift handled via explicit test, not adaptive machinery | τ_ATR = one DEV-frozen constant; drift statement in acceptance section | **AWAITING APPROVAL** |
| **D4-3** Method | DEV-Q75 type-7 / other fixed quantile | **DEV-Q75 type-7** | structural, outcome-blind, exactly defined, project-vocabulary-coherent | freeze act derives exactly 2 literals by this method | **AWAITING APPROVAL** |
| **D4-4** Population | M0 A(3→4) first-valid only, latch-aligned | **as proposed** | calibration ≡ deployment decision point | population definition+hash enters freeze artifact; uniqueness/coverage gates per §D4-4 | **AWAITING APPROVAL** |
| **D4-5** DEV/OOS | DEV 2014–23 / OOS 2024→cutoff | **as proposed** | provenance-locked boundary; exploratory vs confirmatory separation | label fields mandatory in all reports | **AWAITING APPROVAL** |
| **D4-6** Adverse definition | A: ADX+DI dir-aware / B: MACD / C: RSI / D: combos | **A (δ=0 as proposed-form only)** | trend-level interpretability; minimal parameter family; no duplication of entry logic | prereg §5 takes A-form; δ Owner-final later | **AWAITING APPROVAL** |
| **D4-7** Boolean structure | conjunction (proposed) / other logic | **ATR ∧ adverse** | owner's own hypothesis; falsifiable, minimal | final wording at registration | **AWAITING APPROVAL** |
| **D4-8** Latch | (CONFIRMED — not reopened) | **LATCHED** | Decision 3 | t₀ + post-VETO permanence definitions registered | **CONFIRMED** |
| **D4-9** DEV reuse | OOS-first + TRUE_FORWARD chain | **as stated** | DEV can never yield confirmatory claims | label discipline + chain requirements in templates | **AWAITING APPROVAL** |
| **D4-10** Accept/reject framework | 6-dimension structure | **framework as tabulated; margins = proposals** | only Owner can price tolerances | registration blocked until margin literals exist | **AWAITING APPROVAL** |
| **D4-11** Coverage/missing gates | gates 1–5 | **as written** | hard-codes the truncation lesson | every data drop ships with a gate report | **AWAITING APPROVAL** |
| **D4-12** Production boundary | M0 / design-only | **as written** | standing law | none | **CONFIRMED & STANDING** |

**Awaiting Owner approval:** D4-1, D4-2, D4-3, D4-4, D4-5, D4-6, D4-7, D4-9, D4-10 (incl. margin literals), D4-11 — and the carried-forward code-authorization for the latched semantics.
**Confirmed & standing:** D4-8 (LATCHED), D4-12 (Production = M0).

*This sheet changes nothing; it awaits signature. No numbers were derived; no execution of any kind occurred.*
