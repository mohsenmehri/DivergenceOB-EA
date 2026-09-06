# C2 / M2 — PRE-REGISTRATION DESIGN PROPOSAL (DRAFT v1)

**Date:** 2026-09-06 · **State:** `DRAFT — NOT APPROVED — NOT REGISTERED — NOT FROZEN`
**Scope:** design only. No test, no freeze, no threshold derivation, no code/strategy/production change has been performed or is implied by this document.
**Candidate hypothesis (owner-supplied):** Step3→Step4 candidate VETO based on (i) high ATR-percentage regime, combined with (ii) adverse trend/momentum state.

> Items marked **[OWNER-DECISION]** cannot be fixed by the analyst; they must be chosen by the project owner before approval.

---

## 1. Exact population

Unit of analysis: **Step3→Step4 decision attempts** observed in the M0 (control) run's event export:
`Event=A AND StepFrom=3 AND StepTo=4 AND Mode=0`, one attempt per setup (in the observed M0 export this invariant held: n=431 equal to distinct setups; a run violating it is rejected per §17).
- **Inclusion:** every such attempt with DecisionTime inside the experiment window ending at the registered cutoff.
- **Exclusion:** attempts with missing/NaN feature fields (each is counted and reported; no imputation).
- **Join key (for causal pairing with any candidate run):** setup identity by immutable fingerprint (Step1 decision time + entry price + direction + strategy tag) — never raw SetupID.
- The identical population definition applies to the candidate (C2) run.

## 2. Decision timestamp and feature timestamp

- **Decision timestamp (t_d):** the exact tick at which the attempt event is logged (= the instant the EA would invoke the entry-placement call).
- **Feature timestamp (t_f):** features are computed exclusively from market state at t ≤ t_d. Invariant asserted during analysis: `t_f ≤ t_d` for every row. Any violation ⇒ automatic rejection of the run (§17).

## 3. Bar convention — must choose ONE, never mix

| Option | Definition | Note |
|---|---|---|
| **[A] closed-bar shift=1** | ATR/ADX/DI computed on the last *completed* M1 bar | live-parity, reproducible, no intrabar noise; **but incompatible with reusing the value 0.06754475** (that value was derived from shift-0-flavoured logged values) ⇒ would require a new derivation (§4) |
| **[B] forming-bar shift=0** | computed on the current forming bar at the decision tick | matches the implemented M1 behaviour and the logged `ATRPct100` provenance; noisier, intrabar-state dependent |

**Proposal:** primary = **[A] closed-bar shift=1** (deployment-faithful); the logged M0 `ATRPct100` (shift-0-flavoured) is retained only as a *legacy exploratory descriptor*, never as the threshold basis. → **[OWNER-DECISION]** (see Ambiguity AM-1).

## 4. ATR definition and threshold source

- **Feature 1 (volatility regime):** `ATRpct = ATR(14, M1, Wilder-SMA as per MQL5 iATR) / Close × 100`, computed under the §3 convention at t_d.
- **Threshold source (single, pre-committed, literal number in the approved registration):**
  - If convention [B]: the value may be referenced from `M1_THRESHOLD_FREEZE_POSTHOC.json` (**0.06754475**, permanently labelled POST-HOC; it is *not* the historical threshold).
  - If convention [A]: a **new `C2_THRESHOLD_FREEZE.json`** must be produced BEFORE any outcome inspection: Q75 (Hyndman–Fan type-7, formula stated) of ATRpct over the DEV attempt population, with full integrity checks, and committed with its commit-hash — the registration then cites that literal value.
- No other source (e.g., OOS refit, grid search, marginal-best selection) is permitted; deriving more than one candidate value and choosing post hoc is prohibited.

## 5. ADX / DI / momentum definitions (exact)

- `ADX = ADX(14, M1)` and `DI+ , DI− = DI components of ADX(14, M1)` under the §3 convention at t_d.
- **Adverse trend/momentum state (direction-conditional, written literally):**
  - For a SELL basket: `ADVERSE ⇔ (ADX ≥ τ_ADX) AND (DI+ − DI− ≥ δ)`.
  - For a BUY basket: `ADVERSE ⇔ (ADX ≥ τ_ADX) AND (DI− − DI+ ≥ δ)`.
  - τ_ADX and δ are single pre-committed values from the same freeze artifact as §4 (derived on DEV decision-time distributions only; e.g., evidence-agnostic quantiles must be stated literally).
- No alternative momentum proxies (MACD-hist, RSI, EMA-slope) may enter the primary rule; they are permitted only as pre-declared secondary descriptions.

## 6. Exact logical equation of the rule

```
VETO(t_d) ⇔ ( ATRpct(t_d) ≥ τ_ATR )  AND  ADVERSE(t_d)
```

with τ_ATR from §4, `ADVERSE` from §5, direction mapped per setup direction. Nothing else (no ORs, no extra terms, no sessions/filters) enters the primary rule.

## 7. Persistence semantics — must choose ONE

| Option | Meaning | Note |
|---|---|---|
| **[P1] per-decision-tick** | veto re-evaluated each tick while the attempt is pending | known M1 failure mode (89% lapse ⇒ delay-dominated); mechanism ≠ risk avoidance |
| **[P2] latched per attempt** | once vetoed at the first decisive tick, the attempt is dead until the setup invalidates | matches the research question; **requires a code change ⇒ separate owner authorization (currently prohibited)** |

**Proposal:** **[P2]** is the only semantics under which the hypothesis "avoid high-vol+adverse entries" is actually testable; therefore this registration is **BLOCKED on code authorization** if [P2] is chosen. → **[OWNER-DECISION]** (Ambiguity AM-2).

## 8. Veto semantics vs gates/order

Pipeline order at t_d (must be logged explicitly):
1. feature computation (§3–§5) → 2. **C2 veto decision** → 3. existing market gates (spread/margin/session) → 4. PlaceEntry.
- The veto considers ONLY the entry decision; it never touches management, exits, or other legs.
- Event taxonomy logged: `A` (attempt proceeds) / `V-cohort` (first veto of an attempt AND its final disposition) / `P` / `F`. Tick-dwell V-spam is collapsed to cohorts by construction (analysis-side convention, no code change required for counting).
- The veto MUST be evaluated before other gates so exclusions are attributable (no "vetoed vs margin-rejected" ambiguity); gating order is asserted from the log.

## 9. DEV / OOS split

- DEV = `DecisionTime < 2024-01-01 00:00:00`; OOS = `≥ 2024-01-01` up to the registered cutoff.
- OOS is **read-only**: never used for thresholds, δ/τ_ADX selection, or any design choice. All parameter-bearing decisions come from DEV alone; the decision-read is OOS-first in the final report.

## 10. TRUE_FORWARD requirement

No production consideration before ALL of:
- ≥ 3 calendar months of true-forward live/paper operation with identical instrumentation and frozen parameters;
- ≥ 60 eligible attempts (see §15) accumulated forward; 
- forward behaviour consistent with the registered mechanism dwell/disposition profile;
- no parameter change during the period. Until then, even a clean backtest PASS = research-stage only.

## 11. Primary outcome metrics (exactly these; nothing else is "primary")

- **P1 — paired causal net:** Σ (net_C2 − net_M0) over the **affected cohort** (attempts meeting the rule) on the common aligned universe, paired by fingerprint, both runs' exits observed inside the window; censoring rules pre-defined (pairs with unmet exit-observability are excluded and counted).
- **P2 — loss-tail:** Σ net of the worst-decile *attempt-cohort setups* in M0 vs C2 (same ranks fixed from M0), plus the SL$-mass share carried by the eligible cohort.
- Reporting: point estimates + cluster-robust bootstrap 95% CIs (block: attempt-day), one-sided α=0.025 for P1 (benefit direction) as the confirmatory gate.

## 12. Secondary outcome metrics (descriptive; BH-FDR controlled as a family)

PF; MaxClosedDD (closed-equity, method pinned); SL-exit count/rate; BE-rescue rate; TP-family shift counts; delayed-fill statistics; enrollment-drift decomposition (aligned vs drift PnL); yearly/DEV-OOS breakdowns; effect size (Cohen's d on pairs); the integrity invariant (§13-C1).

## 13. Acceptance criteria — fixed BEFORE any test

- **C1 (integrity, hard):** non-affected aligned setups are bit-identical across runs (100%); Mode/Rule/scoping correct; threshold-exchange bracket proof (min-ATR(V) ≥ τ > max-ATR(no-veto)); feature-timestamp invariant holds.
- **C2 (coverage, hard):** all export streams (events/lifecycle/PO) reach the registered cutoff; final-day setups present; any truncation ⇒ run unusable (no assumption-based patching).
- **C3 (primary):** P1 one-sided CI-95 lower bound > 0 **and** P2 indicates tail reduction without DD or SL-count worsening beyond pre-set margins (margins to be filled by owner: e.g., integer counts & dollar bounds) **[OWNER-DECISION]**.
- **C4 (stability):** yearly Δ sign consistency in ≥ 8 of 10 DEV years and OOS Δ ≥ 0 (bounds literally specified) **[OWNER-DECISION]**.
- **C5 (sample, hard):** §15 minima met in both DEV and OOS; otherwise verdict = INCONCLUSIVE BY DESIGN, regardless of point estimates.
Verdicts allowed: `ACCEPTED (research-stage)` / `NOT ACCEPTED — INCONCLUSIVE` / `REJECTED`. Production remains M0 in all cases until §10 completes.

## 14. Bias protections (each with its enforced mechanism)

| Threat | Protection |
|---|---|
| lookahead | t_f ≤ t_d invariant, asserted per-row; code path audited |
| leakage | thresholds derived from decision-time distributions only, DEV only, committed before outcome access (commit-order proof) |
| multiple comparisons | exactly one primary endpoint; secondary battery under BH-FDR; analysis script committed BEFORE unblinding the candidate-run data |
| threshold tuning on OOS | prohibited by construction; any evidence ⇒ run disqualified (§17) |
| selection bias | single pre-registered rule variant; enrollment-drift audit + sensitivity (effect reported with and without drift cohorts; drift share cap in §17); control-cohort identity invariant (C1) |
| incomplete coverage | C2 gates as hard preconditions; nothing assumption-based enters evidence |

## 15. Minimum sample sizes (fixed ex ante)

Given observed per-setup sd ≈ 73$ on the affected cohort:
- affected n ≥ 150 (DEV) **and** n ≥ 60 (OOS) are required for the confirmatory read of P1;
- with n ≈ 118 the 95% CI half-width on mean Δ is ≈ ±13$/setup (Σ ≈ ±1,500$ on 118): the registration must therefore state a *minimum detectable effect* explicitly and accept that smaller true effects ⇒ INCONCLUSIVE BY DESIGN;
- attempt dependence (lag-1 autocorr ≈ +0.16, day-clustering ≈1.02/day) is handled by day-block bootstrap in §11; effective-n is reported.

## 16. Required instrumentation and provenance

- Export set: CF-events (A/V-cohort/P/F with final dispositions), lifecycle, PositionOpenDataset, manifests, Stats/SetupID files — each with byte SHA-256 + git blob SHA registered on the data branch BEFORE analysis unblinding.
- Environment record: MT5 build, tester config hash, feed/data-version identifier, backtest period, spread/swap/commission model — stored in the run folder manifest at registration time.
- Analysis script committed to the working branch BEFORE the candidate-run data is first accessed; any later analysis change is a new, labelled amendment.
- All artifacts (this registration, threshold freeze, run manifests, verdicts) carry literal commit hashes in a single provenance chain.

## 17. Explicit rejection conditions (any one ⇒ REJECTED)

1. Any C1 integrity invariant fails (incl. threshold-bracket or timestamp violations).
2. Coverage gates (C2) fail or any fabrication/patching is detected.
3. P1 CI-95 upper bound < 0 with a harmful point estimate beyond the pre-set harm margin.
4. Loss tail not reduced **and** (DD **or** SL-count worsen beyond margins).
5. Benefit concentration: > 50% of P1 benefit from ≤ 2 setups ⇒ fragile ⇒ reject.
6. Mechanism mismatch: observed candidate behaves delay-dominated again (median veto dwell < 15 min with ≥ 50% lapse) while registered as persistent [P2].
7. DEV/OOS sign reversal of P1 beyond the pre-set tolerance band.
8. Drift-attributed share of the global effect > 25%.
9. Any §15 minimum unmet ⇒ INCONCLUSIVE BY DESIGN (not evidence of harm or benefit).

---

## UNRESOLVED METHODOLOGICAL AMBIGUITIES (require owner resolution before approval)

- **AM-1 — Bar convention vs the frozen number.** Choosing §3-[A] (closed-bar) invalidates reuse of 0.06754475 (needs a new freeze on a new population); choosing [B] keeps live-parity doubt (forming-bar noise) but reuses provenance. These are mutually exclusive; one must be fixed.
- **AM-2 — Persistence vs code authorization.** The testable semantics [P2] (latch) requires a code change, currently prohibited. Options: (i) owner authorizes the minimal latch change; (ii) register under [P1] knowing the delay-bias and pre-setting mechanism criteria (§17-6 handles mismatch); (iii) park C2/M2 until (i).
- **AM-3 — Absolute vs relative threshold under regime drift.** A DEV-frozen absolute τ_ATR faces eligibility drift (25% DEV → 61% in 2026 observed): the backtest then tests "threshold robustness × rule efficacy" jointly. A relative (within-period quantile) variant is a different rule. One must be chosen; mixing is prohibited.
- **AM-4 — Power ceiling.** The fixed window caps affected n ≈ 118 < §15 minima; achieving them requires TRUE_FORWARD accumulation (time) or accepting wider CIs with an explicit MDE statement. Resource/strategy decision.
- **AM-5 — "Adverse" definition for a mean-reversion basket.** DI-based adversity (trend against position) vs basket-relative adverse excursion (floating PnL path) are different concepts; §5 fixed the former for registration, but the owner may intend the latter — if so, §5 must be rewritten BEFORE approval (mixing mid-study is prohibited).
- **AM-6 — DEV reuse.** The same DEV window supports parameter derivation and DEV reporting → irreducible in-sample optimism; mitigation here is single-shot freeze + OOS-first decision rule. If the owner requires a cleaner read, a fresh untouched window must be designated as confirmatory.

---

**FOOTER — NOTHING IS FROZEN BY THIS DOCUMENT:** no rule registered, no threshold derived or committed, no code/strategy/production change, Production = M0. This draft becomes binding only after explicit owner approval, at which point a separate `C2_M2_PREREGISTRATION.md` (hash-chained) is created and every [OWNER-DECISION] is fixed in literal form.
