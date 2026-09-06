# C2 / M2 — OWNER DECISION SHEET v2 (FINAL, D4-1 … D4-12) · DESIGN ONLY

**Date:** 2026-09-06 · **State:** FINAL sheet — **awaiting Owner approval. All Proposed Defaults remain OPEN.**
Chain: prereg `1d976a6` → D3/D4-v1 `ad9f231` → D4-v2 `1d0b390` → memo `a25202f` → sheet v1 `7355ced` → **this v2 (supersedes v1)**.
*Incorporates Owner corrections 2026-09-06 (items 1–4 below). No computation, execution, freeze, registration, or change of any kind.*

---

## D4-1 — Shift Convention

> Key insight: the shift is **not an engineering reproducibility knob** — it *redefines the feature* and the **information set available at decision time**. `ATR(t | completed bars ≤ t)` (shift=1) vs `ATR(t | including current forming bar)` (shift=0) are **different features** with different causal content, different distributions, and hence different thresholds. All downstream artifacts (threshold literal, DEV distributions, run-time comparison) are only valid for the chosen information set.

| Aspect | shift=0 (forming bar) | shift=1 (closed bar) |
|---|---|---|
| Real-time operational fidelity | matches what a live EA touches at the tick; zero lag | 1-bar lag at decision; live EA reproduces it straightforwardly (closed bars are stable) |
| Reproducibility | depends on the exact tick path; intrabar range growth makes values time-within-bar dependent | **higher** than forming-bar, **conditional on** precise definitions of (a) decision timestamp and (b) feature availability (which bars count as complete at t) — it does **not** by itself guarantee full reproducibility |
| Auditability | value depends on exact tick sequence ⇒ hard to re-derive independently | **higher** than forming-bar under the same conditional definitions (re-derivation from OHLC is possible once they are fixed) |
| Timing semantics | decision uses "now-state"; borderline values can flap during a bar | decision uses "last-complete-state"; stable within a bar |
| M1 compatibility | matches historical M1 logs (M1 results are exploratory/void regardless) | new feature basis; M1 provenance explicitly non-transferable (registered) |
| Look-ahead/leakage risk | none intrinsically, but intrabar state in *tester* vs *live* can differ silently (feed modeling) | none; information set provably causal |

**Proposed default (OPEN): shift=1.** Reason: the veto is a risk gate at basket horizons; ≤1-bar staleness is negligible, while the information set becomes time-within-bar-stable and independently re-derivable *provided* the timestamp and feature-availability rules are written exactly into the specification. Choosing shift=0 would instead require registering feed-model assumptions. **Consequence if chosen:** feature formulas, DEV distributions, threshold literals, source data (bar-history), and the freeze artifact all reference closed-bar definitions, **with explicit timestamp and feature-availability rules**; old shift-0 values are retired.

## D4-2 — ATR Representation / Threshold Type

| Option | Pros | Cons |
|---|---|---|
| raw ATR (points) | simplest | price-level non-stationarity (gold's level changed materially across the window) ⇒ threshold meaning decays |
| ATRPct (×100/price) | **reduces the direct effect of price scale — without guaranteeing full price-neutrality**; single feature, no extra degrees of freedom; matches project's historical slot semantics | volatility-regime drift remains (the ATRPct distribution itself shifts over time) |
| relative/rolling normalized (causal) | veto-rate ~stationary under regime drift; matches DEV/OOS discipline | adds lookback parameter + warmup rules + arrival-path dependence + strict causality audit; more failure modes |

**Proposed default (OPEN): ATRPct with an absolute DEV-frozen value; rolling normalization = DEFERRED.** Reason: ATRPct captures most of the scale problem cheaply; residual regime-drift is then tested *deliberately* and reported inside acceptance criteria (an explicit joint test), rather than hidden inside adaptive machinery whose own parameters would need separate registration. **Consequence if chosen:** threshold literal is a single DEV-frozen constant; the drift-sensitivity statement belongs in the acceptance section.

## D4-3 — Threshold Method (defensibility of DEV-Q75 as PROPOSED METHOD)

- **Defensible if** signed ex ante as a structural, outcome-blind coverage choice: distribution-free, exactly defined (type-7), one value per feature, fixed in a single freeze act.
- **Not defensible if** compared against outcomes, chosen among alternatives post hoc, or adjusted for hit-rate/performance (all prohibited by standing clauses).
- Registered alongside: conjunction-mechanics statement (joint veto-rate is a measured consequence, never a target); evidence-asymmetry statement (ATR gradient is exploratory-registered; ADX has no outcome-linked evidence at any quantile — its quantile is a structural guess); family-of-two discipline (both literals in one act).
**Proposed default (OPEN): DEV-Q75 (type-7) as Proposed Method, subject to Owner signature.** **Consequence:** the freeze act derives exactly two literals (τ_ATR, τ_ADX) under this method after D4-1/D4-2 are fixed.

## D4-4 — Population

Proposed: M0 baseline · `Event=A, StepFrom=3, StepTo=4 (Mode=0)` · **one first-valid decision per Setup** · latch-aligned.
- **Uniqueness gate (exact definition):** per Setup, count of first-valid-decision rows == 1 (all violations listed; any setup with 0 or ≥2 rows is reported; population count must reconcile; no silent dropping — see D4-11).
- **Coverage gate (exact definition):** population spans the registered window's expected attempt regions (earliest/latest present against the M0 reference); feature completeness = 100% non-empty/non-NaN on whitelisted feature columns; decision timestamps parse-clean and ordered.
- **Latched alignment:** the deployment trigger (first tick at which the Step3→4 placement condition fires with valid features) must equal the calibration row's tick — asserted by instrumentation in any candidate run.
**Proposed default (OPEN): as proposed.** **Consequence:** population definition + hash enters the freeze artifact.

## D4-5 — DEV / OOS

- DEV = 2014–2023 attempts; OOS = 2024 → registered cutoff. **DEV is exploratory; OOS is confirmatory — never blended in one claim.**
- No OOS results are computed in this phase. At evaluation time: OOS-first reporting; DEV figures always carry the in-sample-selection label.
**Proposed default (OPEN): split as defined.** **Consequence:** report templates carry the exploratory/confirmatory label field.

## D4-6 — Adverse Trend/Momentum Definition (options compared, no outcome-based choice)

| Option | Content | Pros | Cons |
|---|---|---|---|
| **A. ADX + direction-aware DI** | `ADX ≥ τ_ADX` AND (SELL: DI+>DI−; BUY: DI−>DI+) | trend strength + direction against position; interpretable; minimal family | two parameters; DI is not a speed/reversal measure |
| **B. MACD histogram** | momentum against position | single family | noisy on M1; overlapping info; unproven threshold |
| **C. RSI extreme** | extreme against intended reversal | matches the strategy's premise | duplicates the entry signal's own logic ⇒ redundancy |
| **D. Combinations** | A in primary rule; B/C only as pre-declared secondary *descriptors* | prevents rule-sprawl | — |

**Proposed default (OPEN): Option A.** **δ is an OPEN owner decision; no default is asserted.** **Consequence if chosen:** prereg §5 takes the A-formula; δ finalized by Owner later.

## D4-7 — Boolean Rule (structure review only)

`VETO ⇔ ATR_condition AND adverse_condition` — a pure **conjunction** at one decision instant, direction-mapped.
- Consequences reviewed: (i) veto-rate < either marginal rate ⇒ affected cohort shrinks; power re-check at evaluation; (ii) AND is unforgiving — a weak leg removes true-positive attempts without adding any; (iii) same structure both directions, only the adverse leg flips; (iv) no OR/NOR variants registered; any future structural change = new amendment.
**Proposed default (OPEN): conjunction as written; literals and the adverse definition remain undecided.** **Consequence:** wording finalized at registration.

## D4-8 — LATCH Semantics (**CONFIRMED in Decision 3 — this section only documents his semantics**)

**Confirmed earlier by the Owner (Decision 3). This section only documents the semantics:**
- **First-valid decision (t₀):** the first tick at which, for a given setup, the Step3→4 placement condition fires with the chosen information-set features well-defined, and the EA would otherwise invoke entry placement.
- **Post-VETO:** *after the first valid VETO for a Setup, Step4 is not allowed anymore for THAT Setup.* This is a per-Setup prohibition tied to that Setup's own veto event — **it is NOT a general Step4 prohibition for other setups.** The step chain of that Setup stops at Step3 (no Step5+, steps being sequential); management/exits/other legs of that Setup continue per baseline; if the Setup later invalidates, it dies naturally without Step 4.
- Re-arm: none (per-tick re-evaluation is outside the registered semantics).
**Status: CONFIRMED (Decision 3).** Consequence: code authorization remains the only execution blocker.

## D4-9 — DEV Reuse / Provenance

DEV was used for exploration/hypothesis generation (registered). Therefore:
- From DEV, at most: exploratory figures with mandatory labels; parameter-bearing artifacts frozen from DEV before any outcome exposure.
- A confirmatory claim requires, cumulatively: (1) single-shot hash-chained freeze before any candidate run and before OOS access; (2) a run passing all integrity/coverage gates; (3) OOS-first evaluation against pre-registered acceptance criteria; (4) TRUE_FORWARD accumulation per the registered requirement before any production consideration.
**Proposed default (OPEN): as stated.** **Consequence:** every report template carries the label field; no DEV-based confirmatory statement is ever possible.

## D4-10 — Acceptance / Rejection Criteria Framework

| Dimension | Confirmatory question | Proposed default (structure) |
|---|---|---|
| causal paired delta | does the affected cohort improve vs its M0 twin? | primary endpoint: paired ΣΔ with block-bootstrap CI; one-sided benefit test |
| loss-tail behavior | does the harm mass shrink? | worst-decile mass & SL$-mass share inside the eligible cohort must not worsen |
| robustness | year/regime stability? | yearly sign-consistency report; OOS read required non-harmful |
| coverage | exports complete? | all D4-11 gates pass as preconditions |
| sample adequacy | enough affected pairs? | minimum-n gate; unmet ⇒ INCONCLUSIVE BY DESIGN |
| no search artifact | single-shot pipeline? | one freeze, one run family, script committed pre-unblinding, drift sensitivity reported |
**Proposed default (OPEN): framework as tabulated; numeric margins/minima remain Owner-set proposals.** **Consequence:** registration cannot complete without these literals.

## D4-11 — Missing / Coverage Gate

Gates (all must PASS before any analysis or freeze use):
1. **Stream-span:** events/lifecycle/PO terminal timestamps ≥ reference coverage; final-day setup presence.
2. **Reconciliation:** event rows, lifecycle opens, PO legs, manifest counts reconcile (equality or explained deltas listed).
3. **Uniqueness gate** (D4-4).
4. **Feature completeness audit:** per-column missing/NaN/non-numeric counts on whitelisted columns, with cause classified before any action (instrumentation gap / structural N/A / truncation artifact). **No row is auto-dropped;** each exclusion is listed with cause and its population impact reported; zero-impact claims must be justified, else escalated to Owner.
5. **Time-integrity:** parse-clean, monotonic per setup, no future dates.
**Proposed default (OPEN): gates 1–5 as written.** **Consequence:** every data drop ships with a gate report file.

## D4-12 — Production Boundary

```text
Production = M0
C2/M2     = DESIGN ONLY
Until explicit Owner Approval: Registration = NO | Freeze = NO | Backtest = NO | Code Change = NO | OOS Analysis = NO
```

---

## FINAL TABLE

| Decision | Options | Proposed Default (all OPEN) | Reason | Consequence | Owner Status |
|---|---|---|---|---|---|
| **D4-1** Shift | 0 forming / 1 closed | **1 (closed)** — OPEN | redefines feature+info-set; higher reproducibility/auditability vs forming-bar *conditional on exact timestamp & feature-availability rules* | closed-bar basis for all features/stats/literals + explicit availability rules; shift-0 values retired | **AWAITING APPROVAL** |
| **D4-2** Threshold type | raw ATR / ATRPct absolute / rolling-normalized | **ATRPct absolute; rolling DEFERRED** — OPEN | ATRPct reduces direct price-scale effect (no full price-neutrality guarantee); residual drift handled by explicit test | τ_ATR = one DEV-frozen constant + drift statement in acceptance | **AWAITING APPROVAL** |
| **D4-3** Method | DEV-Q75 type-7 / other fixed quantile | **DEV-Q75 type-7** — OPEN | structural, outcome-blind, exactly defined, vocabulary-coherent | freeze act derives exactly 2 literals | **AWAITING APPROVAL** |
| **D4-4** Population | M0 A(3→4) first-valid, latch-aligned | **as proposed** — OPEN | calibration ≡ deployment decision point | population def+hash in freeze artifact | **AWAITING APPROVAL** |
| **D4-5** DEV/OOS | DEV 2014–23 / OOS 2024→cutoff | **as proposed** — OPEN | provenance-locked boundary; exploratory vs confirmatory separation | mandatory label fields | **AWAITING APPROVAL** |
| **D4-6** Adverse definition | A: ADX+DI dir-aware / B / C / D | **A** — OPEN (**δ OPEN, no default**) | trend-level interpretability; minimal family | prereg §5 takes A-form | **AWAITING APPROVAL** |
| **D4-7** Boolean structure | conjunction / others | **ATR ∧ adverse** — OPEN | owner's hypothesis; falsifiable, minimal | wording finalized at registration | **AWAITING APPROVAL** |
| **D4-8** Latch | — | **LATCHED** | **CONFIRMED in Decision 3; semantics documented only:** after first valid VETO for a Setup, Step4 of **that Setup** is prohibited (no general Step4 ban for other setups) | t₀ + per-Setup permanence registered | **CONFIRMED** |
| **D4-9** DEV reuse | OOS-first + TRUE_FORWARD chain | **as stated** — OPEN | DEV never yields confirmatory claims | label discipline + chain requirements | **AWAITING APPROVAL** |
| **D4-10** Accept/reject framework | 6-dimension structure | **framework as tabulated; margins = proposals** — OPEN | only Owner can price tolerances | registration blocked until margin literals | **AWAITING APPROVAL** |
| **D4-11** Coverage/missing gates | gates 1–5 | **as written** — OPEN | hard-codes the truncation lesson; no silent drops | gate report per data drop | **AWAITING APPROVAL** |
| **D4-12** Production boundary | M0 / design-only | **as written** | standing law | none | **CONFIRMED & STANDING** |

**Awaiting Owner approval:** D4-1, D4-2, D4-3, D4-4, D4-5, D4-6, D4-7, D4-9, D4-10 (incl. margin literals), D4-11 — plus the carried-forward code authorization for the latched semantics. **All Proposed Defaults remain OPEN until explicit approval.**
**Confirmed & standing:** D4-8 (LATCHED — Decision 3), D4-12 (Production = M0).

*This sheet changes nothing; it awaits signature.*
