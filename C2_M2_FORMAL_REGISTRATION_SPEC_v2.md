# C2 / M2 — FORMAL REGISTRATION SPECIFICATION v2 (supersedes v1)

**Date:** 2026-09-06 · **Phase:** FORMAL REGISTRATION SPECIFICATION (Pre-Freeze Audit pending)
**Status at issuance:** `C2/M2 = APPROVED FOR REGISTRATION SPECIFICATION · C2/M2 = NOT YET FROZEN · Production = M0`
Chain: prereg `1d976a6` → D3/D4-v1 `ad9f231` → D4-v2 `1d0b390` → memo `a25202f` → sheet v1 `7355ced` → sheet v2 `606c07f` → Owner Approval → spec v1 `871fe50` → **this v2 (owner-corrected; supersedes v1)**.
**Owner corrections applied:** (1) final gate order; (2) shift=1 definition without unproven-origin arithmetic; (3) δ_DI reclassified as proposed/default parameter (never frozen by this document).
**Forbidden throughout this phase:** threshold calculation · numeric threshold extraction · freeze · backtest · OOS analysis · TRUE_FORWARD · parameter sweep · rule tuning · code change · production change. **No new numbers are derived anywhere in this document.**

---

## 1. Official rule name & version

- **Rule name:** C2/M2 — *Latched Step3→4 Volatility-Trend Veto*
- **Rule ID:** `C2_M2_S34_LATCHED_ATR_ADX_DI_V1`
- **Version:** 1.0 (specification lineage amended by hash-chained, owner-signed documents; each amendment increments minor version)
- All artifacts, logs, and reports referring to this rule MUST cite the Rule ID and version.

## 2. Population (exact)

- Source run family: **M0 baseline** exports; identical definition applies to any candidate run.
- Rows: event rows satisfying `Event = "A" AND StepFrom = 3 AND StepTo = 4 AND Mode = 0`.
- **One first-valid decision per Setup** (see §10 for the definition of first-valid); if an export ever contains more than one qualifying decision row per setup, the population element is the **earliest**; the uniqueness gate (§14) counts and reports violations with no silent repair.
- Identity: setup fingerprint = (Step1 decision time, entry price, direction, strategy tag); raw SetupID is never a join key.

## 3. Decision timestamp & feature timestamp (exact)

| Symbol | Definition |
|---|---|
| `t_d` (decision time) | the tick time of a logged Step3→4 decision row — the instant the player evaluates the entry decision for that attempt |
| `t_f` (feature time) | the latest information instant legally usable by any feature = `t_d` itself, restricted by the information set of §4 |
| Invariant | every feature value must be computable from information satisfying `t ≤ t_d` under §4; violation invalidates the run (§18) |

## 4. Shift=1 / closed-bar information set (exact; auditable; no external-origin claims)

- **Definition (shift-based, primary):** for a decision at `t_d` on timeframe M1, let `b0(t_d)` be the (unique) M1 bar whose interval **contains** `t_d` (i.e., `open(b0) ≤ t_d < close(b0)`); `b0` is the FORMING bar and is **inadmissible**. The admissible set = all bars strictly preceding `b0(t_d)` — exactly the bars referenced by MQL5 shift indices `≥ 1` at `t_d`.
- **Derived arithmetic equivalent (origin explicitly stated):** because timeframe M1 has a registered period of 60 seconds, `open(b0) = floor(t_d, 60s)`, so the admissible set equals `bars with open_time ≤ floor(t_d,60s) − 60s`. This arithmetic is presented **only** as a consequence of the registered timeframe and the platform's bar-indexing model — not as an independently sourced invariant.
- Every feature (ATR, ADX, DI+, DI−, and any prices inside ratios) is computed **exclusively** from the admissible set; all features of one decision share the same set (“single information set”).
- Feature availability: an indicator needing k bars is well-defined only if ≥ k admissible bars exist at `t_d` (trivially true in this window; the rule must still be written and checked); otherwise the row enters the missingness-audit path (§14), never silent imputation.
- The freeze artifact pins: the exact definition text above, the bar-history source, and a sample of independently re-derived rows (audit gate).

## 5. ATRPct (exact)

```
ATRpct(t_d) = ATR(14, M1, Wilder/MQL5 iATR, shift=1 at t_d) / Close(M1, shift=1 at t_d) × 100
```
Numerator and denominator come from the SAME last admissible bar (§4); units = percent.

## 6. ATR threshold methodology (exact)

- Statistic: **Q75, Hyndman–Fan type-7** — `h = (n−1)·0.75 + 1` (1-based fractional rank over ascending DEV values), linear interpolation between bounding order statistics; both neighbors recorded.
- Population: DEV attempt population (§2 restricted to §12 DEV window), single-shot extraction at freeze time; the freeze-time n is recorded and must reconcile with the registered population (prior validated reference: n_DEV = 366, to be re-verified; any deviation explained).
- No alternative statistic, value, or outcome comparison is permitted at any point.

## 7. ADX threshold methodology (exact)

- Feature: `ADX(14, M1, shift=1 at t_d)`.
- Statistic: Q75, type-7, identical procedure and population as §6; literal recorded at freeze.
- Registered note (carried): the ADX quantile is a **structural, outcome-blind choice**; no outcome-linked evidence exists for ADX at any quantile — this sentence must appear verbatim in the freeze artifact.

## 8. Direction-aware DI (exact; δ proposed/default only)

- Components: `DI+(t_d), DI−(t_d)` from the same `ADXD(14, M1, shift=1 at t_d)` computation (same information set).
- `δ_DI` = dominance margin: a **proposed/default specification parameter (proposed form: 0)**. **It is NOT FROZEN** by this specification; it becomes literal only at the freeze act by explicit owner decision.
- Definition:
  - setup direction = **SELL** ⇒ `DI_ADVERSE ⇔ (DI+ − DI−) > δ_DI`
  - setup direction = **BUY** ⇒ `DI_ADVERSE ⇔ (DI− − DI+) > δ_DI`
- Setup direction is the baseline strategy's own immutable direction attribute for that attempt (never inferred from outcomes).

## 9. Final Boolean rule (exact)

```
VETO(t_d) ⇔ [ ATRpct(t_d) ≥ τ_ATR ]  AND  [ ADX(t_d) ≥ τ_ADX  AND  DI_ADVERSE(t_d) ]
```
Single conjunction at one decision instant; direction-mapping per §8; no OR/NOR/else branches; any change = new version (§1).

## 10. LATCHED semantics (documenting confirmed Decision 3; gate-order-aligned)

- **First valid decision (t₀):** per setup, the first decision instant `t_d` at which ALL execution preconditions pass — i.e., the entry candidate has cleared **Market Gate → Spread Gate → Margin Gate** (§11) AND the §4 information-set features are well-defined. Decisions failing any execution gate never become population members and can never be a first-valid decision.
- **VETO state:** if `VETO(t₀)` is true, a per-setup persistent veto state is set for that setup.
- **Post-VETO:** *Step4 of THAT Setup is never allowed* — a per-Setup prohibition tied to that setup's own veto; **not** a general Step4 ban on other setups. The setup's step chain stops at Step3 (no Step5+, steps being sequential).
- **No reopen / no re-arm:** the veto state never clears for the life of the setup.
- Scope: nothing else is touched — management, exits, other legs, other setups proceed per baseline.
- Implementation note (status unchanged): latched semantics requires minimal code support in the player; **owner's code authorization remains the only execution precondition** — registration proceeds as specification; test execution stays blocked until authorization exists.

## 11. Gate order & veto position (FINAL — owner-corrected)

The exact required pipeline at a decision instant:

```
Market Gate  →  Spread Gate  →  Margin Gate  →  First Valid Decision  →  LATCHED VETO  →  PlaceEntry(next=4)
```

Registered semantics:
1. **Only execution-gate-eligible candidates enter the VETO decision** — Market/Spread/Margin gates run BEFORE the veto; a candidate rejected by any of them never reaches feature evaluation for veto purposes (it is not a population member, and no veto state is set by it).
2. **The first valid decision IS the eligible and complete decision** — the first t_d after these gates pass (§10).
3. **If the rule returns VETO=true at that decision, Step4 is never allowed for that Setup** (per §10; per-Setup only).
4. **If VETO=false, the path continues to `PlaceEntry(next=4)`** through the unchanged baseline placement mechanics — the veto adds or alters nothing about prices, sizes, discounts, or management.
5. **Latched is per-Setup only; no reopen exists** (§10).
6. Instrumentation requirement: gate outcomes and the veto decision are logged in this order for every decision instant; the logs must make the order verifiable (a V row implies Market/Spread/Margin PASS at that tick); any candidate run whose logs imply a different order is rejected (§18). Audit note: the required instrumentation of this exact order is part of the pending code authorization.

## 12. DEV / OOS boundaries (exact)

- **DEV:** `t_d < 2024-01-01 00:00:00` — sole derivation window for parameter-bearing artifacts (exploratory/selection zone).
- **OOS:** `2024-01-01 00:00:00 ≤ t_d ≤ registered cutoff (2026-08-18 23:59:58)` — confirmatory zone; quarantined for selection; read-only.
- **TRUE_FORWARD:** `t_d > cutoff`, live/paper, per the registered requirement — the only zone that can ever support production consideration.
- Mandatory labels on every figure: `exploratory-DEV` / `confirmatory-OOS` / `TRUE_FORWARD`.

## 13. Leakage controls (exact, binding)

1. Derivation whitelist reads only: identity fields, decision fields, feature fields; **outcome/PnL/lifecycle columns forbidden** at derivation (assert + log).
2. **Outcome quarantine:** no outcome access before the freeze commit (commit-order proof).
3. **Single-shot freeze:** exactly one act fixes (τ_ATR, τ_ADX, δ_DI) + convention + population + formulas; no later adjustments; defect-fix amendments are owner-signed, hash-chained, value-neutral.
4. **No retuning:** no sweep, no sensitivity-based selection, no veto-rate targeting; joint rates are reported consequences only.
5. **First-decision alignment:** derivation population ≡ deployment population (§2, §10).
6. **OOS quarantine pre-freeze.**
7. **Convention exactness:** every feature carries its shift=1 flag; mixing information sets voids artifacts (§18).
8. **Family discipline:** parameter family = exactly three literals (τ_ATR, τ_ADX, δ_DI); anything additional = new version + new authorization.

## 14. Uniqueness and coverage/data-quality gates (exact)

- **G-U (uniqueness):** per-setup first-valid-decision count == 1 (under §10's definition); violations listed, counts reconciled, no silent drops.
- **G-S (stream span):** events/lifecycle/PO spans reach registered window ends; expected tail setups present per reference.
- **G-R (reconciliation):** event rows / lifecycle opens / PO legs / manifest counts reconcile (equality or explained deltas listed).
- **G-C (feature completeness):** per-column missing/NaN/non-numeric counts on the whitelist, each classified by cause (instrumentation gap / structural N/A / truncation artifact) before any action; exclusions listed with cause + population impact; zero-impact claims must be justified, else escalated (§18).
- **G-T (time integrity):** parse-clean, monotonic per setup, in-window.
- All gates produce a written gate report shipped with the data drop; **all** must PASS before any analysis or freeze-time derivation.

## 15. Provenance & required freeze artifacts (schema)

**`C2_THRESHOLD_FREEZE.json`** (created only at the freeze act — NOT now) must contain:
`rule_id, version` · population definition + SHA-256 of its extraction set · convention block (§4 primary shift definition + stated origin of the arithmetic equivalent) · DEV/OOS windows · per-feature formulas (§5–§8) · **`τ_ATR`, `τ_ADX`, `δ_DI` literals** · derivation method (§6) + derivation-script SHA-256 · both bounding order statistics per quantile · DEV descriptive blocks · joint DEV eligibility rate (descriptive only — **never a target**) · gate-order declaration (§11) as implemented in the player · integrity-gate results (§14) · owner sign-off block · commit hash.
**Provenance chain:** `234dc6b → f4fd889 → 164ea76 → 3912e21 → … → 606c07f → 871fe50(v1) → this v2`.
**No artifact containing any C2/M2 parameter may reference, reuse, compare against, or anchor on the M1 threshold** (§19).

## 16. Acceptance / Rejection framework (structure only)

D4-10 six-dimension framework (causal paired delta · loss-tail · robustness · coverage · sample adequacy · no-search-artifact) with **no numeric margins here** (margin literals enter the freeze act / final registration). Verdicts: `ACCEPTED (research-stage)` / `NOT ACCEPTED — INCONCLUSIVE` / `REJECTED`; sample shortfalls ⇒ `INCONCLUSIVE BY DESIGN`.

## 17. Conditions for a VALID registration

1. Every section instantiated with required literals (τ_ATR, τ_ADX, δ_DI, margin literals, code-authorization reference).
2. No contradiction with confirmed decisions (D3/D4-1…D4-12 including the FINAL gate order of §11).
3. Freeze commit precedes all candidate-run artifacts and all OOS-derived material (commit-order proof).
4. All gates operational on the extraction set with PASSing reports.
5. Owner approval of the freeze act recorded with its hash.

## 18. Conditions under which registration is REJECTED or HALTED

1. Any missing literal or gate definition required by §17-1.
2. Any deviation from confirmed decisions or final gate order; any information-set mixing (§4/§13-7).
3. Any evidence of outcome-linked derivation or post-hoc value adjustment.
4. Any unsatisfiable gate on the real extraction set: HALT + escalate, never silent patch.
5. Feature-timestamp invariant violation (§3).
6. Any C2/M2 artifact referencing the M1 threshold (§19).
7. Execution boundary: without code authorization for latched semantics + §11 instrumentation, **test execution HALTs**; the specification may stand valid, but no run may be produced.

## 19. C2/M2 vs M1 — registered separation (binding)

| Aspect | M1 (closed line) | C2/M2 (this specification) |
|---|---|---|
| Shift / information set | shift=0 (forming-bar, as implemented) | **shift=1 (closed-bar), §4** |
| Rule form | ATRpct ≥ τ only | **conjunction with trend/DI adversity, §9** |
| Persistence | per-tick re-evaluation (delay-dominated, observed) | **latched single first-valid decision, §10** |
| Veto position | attempt-level (implementation-era) | **after Market/Spread/Margin gates, before PlaceEntry, §11** |
| Threshold | `0.06754475` — belongs **EXCLUSIVELY to M1 provenance** (POST-HOC RE-FROZEN label) | **must NOT enter C2/M2 in any form** — not as value, prior, sanity anchor, comparison row, or fallback |
| Experiment status | export incomplete; `PERFORMANCE VERDICT = NOT ISSUED`; provenance-only | no run has occurred; nothing to reuse |
| Evidence status | exploratory-only, voided as accept/reject basis | tabula rasa; starts at this specification |

Any M1-traceable number inside a C2/M2 artifact voids that artifact (§18-6).

---

## Closing registered status

```text
C2/M2 = APPROVED FOR REGISTRATION SPECIFICATION
C2/M2 = NOT YET FROZEN
PRODUCTION = M0
```

**Halting here as instructed.** Next step (gated): **PRE-FREEZE AUDIT** — internal consistency of this specification, operationality of all gates, completeness of the freeze-artifact schema, absence of leakage paths. **No freeze, threshold calculation, backtest, OOS, code change, or production change occurs until that audit is explicitly approved by the Owner.**
