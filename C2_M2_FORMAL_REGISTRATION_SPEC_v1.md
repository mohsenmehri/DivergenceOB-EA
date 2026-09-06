# C2 / M2 — FORMAL REGISTRATION SPECIFICATION v1

**Date:** 2026-09-06 · **Phase:** FORMAL REGISTRATION SPECIFICATION (Pre-Freeze Audit pending)
**Status at issuance:** `C2/M2 = APPROVED FOR REGISTRATION SPECIFICATION · C2/M2 = NOT YET FROZEN · Production = M0`
Chain: prereg `1d976a6` → D3 approval `ad9f231` → D4-v2 `1d0b390` → memo `a25202f` → sheet v1 `7355ced` → sheet v2 FINAL `606c07f` → **Owner Approval of D4-1…D4-12** → **this specification**.
**Permits:** specification, formal definition, provenance, audit checklist, registration schema.
**Still forbidden:** threshold calculation · numeric threshold extraction · freeze · backtest · OOS analysis · TRUE_FORWARD · parameter sweep · rule tuning · code change · production change. **No new numbers are derived anywhere in this document.**

---

## 1. Official rule name & version

- **Rule name:** C2/M2 — *Latched Step3→4 Volatility-Trend Veto*
- **Rule ID:** `C2_M2_S34_LATCHED_ATR_ADX_DI_V1`
- **Version:** 1.0 (registration lineage: v1.0 = this specification; any amendment increments minor version and must be hash-chained and owner-signed)
- All documents, artifacts, logs, and reports referring to this rule MUST cite the Rule ID and version.

## 2. Population (exact)

- Source run family: **M0 baseline** (control) exports; identical definition applies to any candidate run.
- Rows: event rows satisfying `Event = "A" AND StepFrom = 3 AND StepTo = 4 AND Mode = 0`.
- **One first-valid decision per Setup:** if an export contains more than one qualifying decision row for a setup, the population element is the **earliest** such row; the uniqueness gate (§14) counts and reports any violation of expected uniqueness, with no silent repair.
- Identity: setup fingerprint = (Step1 decision time, entry price, direction, strategy tag); raw SetupID is never a join key.

## 3. Decision timestamp & feature timestamp (exact)

| Symbol | Definition |
|---|---|
| `t_d` (decision time) | the tick time at which the Step3→4 placement condition fires with valid state — in the M0 export, the row's `DecisionTime`; in any candidate run, the logged first-valid decision tick |
| `t_f` (feature time) | the latest information instant legally usable by any feature = `t_d` itself, restricted by the information set of §4 |
| Invariant | every feature value must be computable from information satisfying `t ≤ t_d` under §4; a row violating this invariant invalidates the run (§18) |

## 4. Shift=1 / closed-bar information set (exact)

- Admissible bars at `t_d` on M1: all bars **fully completed strictly before `t_d`** — formally, bars whose bar-open time ≤ `t_d − 60 seconds`; the current forming bar and any later bar are inadmissible.
- Every feature (ATR, ADX, DI+, DI−, prices used in ratios) is computed **exclusively from admissible bars**; all features of one decision share the same admissible set (“single information set” rule).
- Feature availability: an indicator requiring k bars is *well-defined* at t_d only if ≥ k admissible bars exist (trivial in this study window, but the rule must be written and checked); otherwise the row enters the missingness audit path (§14), never silent imputation.
- The exact timestamp arithmetic (bar-open vs bar-close convention of the data source) is written literally into the freeze artifact to make the information set reproducible.

## 5. ATRPct (exact)

```
ATRpct(t_d) = ATR(14, M1, Wilder/MQL5 iATR, shift=1 at t_d) / Close(M1, shift=1 at t_d) × 100
```
Both numerator (Wilder-smoothed true range, 14-bar) and denominator come from the SAME last admissible bar; units = percent.

## 6. ATR threshold methodology (exact)

- Statistic: **Q75, Hyndman–Fan type-7** — `h = (n−1)·0.75 + 1` (1-based fractional rank over ascending DEV values), linear interpolation between bounding order statistics; both neighbors recorded.
- Population: DEV attempt population (§2 restricted to §12 DEV window), single-shot extraction at freeze time; the freeze-time n is recorded and must reconcile with the registered population (prior validated reference: n_DEV = 366, to be re-verified, any deviation explained).
- No alternative statistic, value, or comparison against outcomes is permitted at any point.

## 7. ADX threshold methodology (exact)

- Feature: `ADX(14, M1, shift=1 at t_d)`.
- Statistic: Q75, type-7, identical procedure and population as §6; literal recorded at freeze.
- Registered note (carried from D4-3): choosing this quantile for ADX is a **structural, outcome-blind choice**; no outcome-linked evidence exists for ADX at any quantile — the artifact must state this verbatim.

## 8. Direction-aware DI (exact)

- Components: `DI+(t_d), DI−(t_d)` from `ADXD(14, M1, shift=1 at t_d)` (the DI outputs of the ADX computation, same information set).
- Let `δ_DI` be the dominance margin (a registered parameter; proposed form = 0, value fixed at freeze by owner — see §15).
- Definition:
  - setup direction = **SELL** ⇒ `DI_ADVERSE ⇔ (DI+ − DI−) > δ_DI`
  - setup direction = **BUY** ⇒ `DI_ADVERSE ⇔ (DI− − DI+) > δ_DI`
- Setup direction is the baseline strategy's own direction for that attempt (immutable fingerprint attribute), never inferred from outcome data.

## 9. Final Boolean rule (exact)

```
VETO(t_d) ⇔ [ ATRpct(t_d) ≥ τ_ATR ]  AND  [ ADX(t_d) ≥ τ_ADX  AND  DI_ADVERSE(t_d) ]
```
- A single conjunction at one decision instant; direction-mapping per §8; no OR/NOR/else branches; no other term may ever be added without a new version (§1).

## 10. LATCHED semantics (documenting confirmed Decision 3)

- **First valid decision (t₀):** per setup, the first `t_d` satisfying §2–§4.
- **VETO state:** if `VETO(t₀)` is true, a per-setup persistent veto state is set for that setup.
- **Post-VETO:** *Step4 of THAT Setup is never allowed* — a per-Setup prohibition tied to that setup's own veto; **not** a general Step4 ban on other setups. The setup's step chain stops at Step3 (no Step5+ exists, steps being sequential).
- **No reopen / no re-arm:** the veto state never clears for the life of the setup; per-tick re-evaluation is outside the registered semantics.
- Scope: the veto touches nothing else — management, exits, other legs, other setups proceed per baseline.
- Implementation note (status): latched semantics requires minimal code support in the player; the **owner's code authorization is the remaining execution precondition** — registration proceeds as specification; test execution stays blocked until that authorization exists.

## 11. Gate order & veto position (exact)

Pipeline at `t₀` (instrumented in this order; order asserted from logs in any run):
1. **feature computation** (§4–§8) → 2. **C2 veto evaluation** (§9) → 3. **existing market gates** (spread / margin / session / account) → 4. **PlaceEntry invocation**.
- The veto is evaluated **before** other gates so exclusions are causally attributable (a veto decision can never be masked by a market-gate rejection); any candidate run whose logs imply a different order is rejected (§18).
- If placed, the entry proceeds through the unchanged baseline gates (3)–(4); the veto never modifies discounts, prices, sizes, or management.

## 12. DEV / OOS boundaries (exact)

- **DEV:** `t_d < 2024-01-01 00:00:00` — the ONLY window from which any parameter-bearing artifact may be derived (exploratory/selection zone).
- **OOS:** `2024-01-01 00:00:00 ≤ t_d ≤ registered cutoff (2026-08-18 23:59:58)` — confirmatory zone; fully quarantined for selection; read-only; first-class reporting zone at evaluation time.
- **TRUE_FORWARD:** `t_d > cutoff` under live/paper operation per the registered requirement — the only zone that can ever support production consideration.
- Labels are mandatory on every figure: `exploratory-DEV` / `confirmatory-OOS` / `TRUE_FORWARD`.

## 13. Leakage controls (exact, binding)

1. Derivation reads obey a fixed whitelist: identity fields, decision fields, feature fields only; **outcome/PnL/lifecycle columns are forbidden** at derivation time (assert + log).
2. **Outcome quarantine:** no outcome data may be accessed before the freeze commit (commit-order proof in the chain).
3. **Single-shot freeze:** exactly one freeze act fixes (τ_ATR, τ_ADX, δ_DI) + convention + population + formulas; no value is ever adjusted afterwards; defect-fix amendments are owner-signed, hash-chained, and never value-raising/lowering.
4. **No retuning:** no sweep, no sensitivity-based selection, no veto-rate targeting; joint rates are reported consequences only.
5. **First-decision alignment:** derivation population ≡ deployment population (§2, §10).
6. **OOS quarantine pre-freeze:** nothing OOS-derived precedes the freeze commit.
7. **Convention exactness:** every feature carries its shift=1 flag; mixing information sets voids artifacts (§18).
8. **Family discipline:** the parameter family = exactly three literals (τ_ATR, τ_ADX, δ_DI); any additional candidate is a new version requiring new authorization.

## 14. Uniqueness and coverage/data-quality gates (exact)

- **G-U (uniqueness):** per-setup first-valid-decision count == 1; violations listed, counts reconciled, no silent drops.
- **G-S (stream span):** events/lifecycle/PO spans reach the registered window ends; expected tail setups present per reference.
- **G-R (reconciliation):** event rows / lifecycle opens / PO legs / manifest counts reconcile (equality or explained deltas listed).
- **G-C (feature completeness):** per-column missing/NaN/non-numeric counts on the whitelist, each classified by cause — instrumentation gap / structural N/A / truncation artifact — before any action; every exclusion is listed with cause and its population impact reported; zero-impact claims must be justified, else escalated (§18).
- **G-T (time integrity):** parse-clean timestamps, monotonic per setup, no out-of-window values.
- All gates produce a written gate report shipped with the data drop; **all** must PASS before any analysis or freeze-time derivation.

## 15. Provenance & required freeze artifacts (schema)

**`C2_THRESHOLD_FREEZE.json`** (created only at the freeze act — NOT now) must contain:
`rule_id, version` · population definition + SHA-256 of its extraction set · convention block (shift=1 + timestamp arithmetic of §4) · DEV/OOS windows · per-feature formulas (§5–§8) · **`τ_ATR`, `τ_ADX`, `δ_DI` literals** · derivation-method (§6) + derivation-script SHA-256 · both bounding order statistics per quantile · DEV descriptive blocks (n, min, q25, med, q75, q90, mean, span) · joint DEV eligibility rate (descriptive only; explicitly **not a target**) · integrity-gate results (§14) · owner sign-off block · commit hash.
**Provenance chain:** every artifact cites its parent commits; data blobs are pinned by SHA-256; the chain on record: `234dc6b → f4fd889 → 164ea76 → 3912e21 → … → 606c07f (sheet v2) → this spec`.
**No artifact containing any C2/M2 parameter may ever reference, reuse, compare against, or anchor on the M1 threshold** (see §19).

## 16. Acceptance / Rejection framework (structure only)

Per the approved D4-10 six-dimension framework — causal paired delta (primary endpoint concept), loss-tail behavior, robustness, coverage, sample adequacy, no-search-artifact — with **no numeric margins set here** (margin literals enter the freeze act / final registration). Verdict taxonomy: `ACCEPTED (research-stage)` / `NOT ACCEPTED — INCONCLUSIVE` / `REJECTED`; sample shortfalls ⇒ `INCONCLUSIVE BY DESIGN` (never “no effect”, never “harm proven insufficiently”).

## 17. Conditions for a VALID registration

A registration act is valid iff ALL hold:
1. Every section of this specification is instantiated with literals where required (τ_ATR, τ_ADX, δ_DI, margin literals, code-authorization reference).
2. No contradiction with confirmed decisions (D3=D4-8 latch, D4-1 shift=1, D4-2 ATRPct, D4-3 DEV-Q75, D4-4 population, D4-5 windows, D4-6 DI-form, D4-7 conjunction, D4-9 provenance order, D4-10 framework, D4-11 gates, D4-12 boundary).
3. Freeze commit precedes all candidate-run artifacts and all OOS-derived material (commit-order proof).
4. All gate definitions are operational and have produced PASSing gate reports on the extraction set.
5. Owner approval of the freeze act is recorded with its hash.

## 18. Conditions under which registration is REJECTED or HALTED

1. Any missing literal or gate definition required by §17-1.
2. Any deviation from the confirmed D4 decisions, or mixing of information sets (§4/§13-7).
3. Any evidence of outcome-linked derivation or post-hoc value adjustment.
4. Any unsatisfiable gate on the real extraction set (e.g., systematic non-uniqueness): HALT and escalate to owner — never patch silently.
5. Feature-timestamp invariant violation discovered in any material (§3).
6. Any C2/M2 artifact found to reference the M1 threshold (§19).
7. Execution boundary: if code authorization for the latched semantics does not exist, **test execution HALTs** — the specification itself may be valid, but no run may be produced.

## 19. C2/M2 vs M1 — registered separation (binding)

| Aspect | M1 (closed line) | C2/M2 (this specification) |
|---|---|---|
| Shift / information set | shift=0 (forming-bar, as implemented) | **shift=1 (closed-bar), §4** |
| Rule form | ATRpct ≥ τ only | **conjunction with trend/DI adversity, §9** |
| Persistence | per-tick re-evaluation (delay-dominated, observed) | **latched single first-valid decision, §10** |
| Threshold | `0.06754475` — belongs **EXCLUSIVELY to M1 provenance** (label: POST-HOC RE-FROZEN) | **must NOT enter C2/M2 in any form** — not as value, prior, sanity anchor, comparison row, or fallback |
| Experiment status | export incomplete; `PERFORMANCE VERDICT = NOT ISSUED`; files are provenance-only | no run has occurred; nothing to reuse |
| Evidence status | exploratory-only, voided as basis for acceptance/rejection | tabula rasa; starts at this specification |

Any derived number traceable to M1 artifacts appearing inside a C2/M2 document or artifact voids that artifact (§18-6).

---

## Closing registered status

```text
C2/M2 = APPROVED FOR REGISTRATION SPECIFICATION
C2/M2 = NOT YET FROZEN
PRODUCTION = M0
```

**Next step (gated):** the **PRE-FREEZE AUDIT** will verify this specification's internal consistency, the operationality of all gates, the completeness of the freeze-artifact schema, and the absence of any leakage path — BEFORE any freeze or experimental execution. **Nothing may be frozen, computed on parameters, backtested, or executed until that audit is explicitly approved by the Owner.**
