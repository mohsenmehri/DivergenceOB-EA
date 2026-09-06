# C2 / M2 — OWNER DECISION MEMO v1 (advisory; DESIGN ONLY)

**Date:** 2026-09-06 · **State:** advisory memo; nothing executed, computed, frozen, registered, or changed.
Chain: prereg draft `1d976a6` → Decision-3/4 v1 `ad9f231` → Decision-4 v2 `1d0b390` → **this memo**.
*All quantitative references cite previously registered artifacts only; no new numbers were produced.*

---

## OD-1 — Shift Convention (forming-bar shift=0 vs closed-bar shift=1)

**Shift 0 (forming bar at the decision tick)**
- Pros: matches the historically implemented veto behaviour; zero staleness — uses all information up to the tick; existing M1-era event logs are directly comparable (no re-derivation of feature values).
- Cons: the forming bar's range grows during the bar ⇒ the same setup at two ticks inside one bar can see materially different ATRpct; **path-dependent, noise-amplifying** (observed in M1 as per-tick veto churn / borderline flapping); exact reproduction outside the identical tick sequence is impossible ⇒ **weakest cross-environment reproducibility** (tester vs live vs research workbook, whose capture policy is shift=1 per the registered manifest).
- Production semantics: "act on live, possibly half-formed information" — borderline decisions are unstable between environments.

**Shift 1 (last closed bar)**
- Pros: feature value is **constant within a bar and exactly reproducible from completed-bar history** in any environment (tester/live/research); immune to intrabar feed microstructure and tick-path dependence; aligns with the project's registered research-capture policy (shift=1); maximal auditability and determinism.
- Cons: up to one M1 bar of staleness (≤ ~1 minute at the decision tick); breaks value-comparability with the old shift-0-flavoured M1 logs (must never be mixed).
- Production semantics: "decide on closed information" — standard, defensible practice for risk gates whose horizon (basket lifecycle, minutes–hours) dwarfs the staleness cost.

**Analyst recommendation: SHIFT=1 (closed bar).** The rule's decision horizon is basket-level; the ≤1-bar staleness is negligible, while reproducibility and the elimination of threshold-flapping are first-order benefits — and they directly address the failure mode observed in the M1 run (delay-dominated churn around the boundary). Decision belongs to the Owner.

## OD-2 — ATR threshold TYPE (absolute ATRPct vs relative/normalized)

- ATRPct is already price-level-invariant (ATR/price), so **nominal XAUUSD level drift (≈1200$→≈4300$) is neutralized by the feature itself**; the remaining non-stationarity is the **volatility-regime distribution**: registered observation — eligibility of a DEV-frozen absolute Q75 drifted from the ~25% design rate to ≈61% in 2026. An absolute DEV-frozen threshold is thus a *joint bet*: rule efficacy × regime persistence.
- **Relative/normalized (causal rolling reference):** threshold expressed against trailing history only (e.g., a causal rolling quantile or a ratio to a long-run trailing reference; the exact lookback and warmup rules would themselves be part of the registered formula). Keeps the veto *rate* approximately stationary by construction; better aligned with this project's DEV/OOS discipline; but adds path-dependence on attempt-arrival order, estimation noise, warm-up handling, and strict causality audit obligations (any use of future bars = look-ahead).
- **Which is more logical here:** if the intended rule semantics is *"veto the currently-most-volatile quartile of moments"*, the **relative (causal)** form is methodologically more logical given the documented regime drift; if the owner instead intends an explicit **transfer test** of a 2014–2023 calibration carried forward, the **absolute** form is coherent — but the report must then treat threshold-robustness as part of what is being tested (registered as AM-3).
- Analyst recommendation: **relative-causal** (with the lookback defined in the formula at registration); decision belongs to the Owner.

## OD-3 — Threshold METHOD (is Q75 defensible for ATR and ADX?)

**Defensible, *as a pre-committed structural choice*:** distribution-free (no parametric assumption — important given the registered right-skew of ATRPct); single, exactly-defined statistic (type-7 formula already specified); embodies a marginal risk-coverage intention (~top quartile) rather than any outcome fit; consistent with the project's historical design vocabulary (the frozen build's input comments themselves named DEV_Q75-style sources for these slots).
**Risks / caveats (to record, not resolve, here):**
1. **Conjunction ≠ product of marginals:** two marginal Q75s do not yield a designed joint veto-rate; the joint rate is a *consequence to be measured and reported*, never a tuning target (registered anti-tuning clause applies).
2. **Evidence asymmetry between the two features:** an ATR-quartile risk gradient exists as *exploratory* registered evidence; **no outcome-linked evidence exists for ADX at any quantile** — choosing Q75 for ADX is a structural guess and must be minuted as such.
3. **Quantile-chosen-by-humans = hyperparameter:** Q75 is defensible only if one value is signed ex ante with a stated rationale; comparing quantiles against outcomes would be tuning (prohibited).
4. **Finite-sample quantile noise:** with DEV n in the few-hundreds, the Q75 estimator has material (but documented) standard error; the two neighboring order statistics are recorded at freeze time so the interpolation is fully reproducible.
5. **Regime mixing:** pooling heterogeneous 2014–2023 regimes into one quantile inherits the OD-2 decision.
6. **Two thresholds = family of two:** both must be fixed in a single freeze act; any alternative tested later requires a new amendment.
**Conclusion:** Q75 is *acceptable* provided the owner signs it as a structural, evidence-agnostic choice for BOTH features, with the joint-rate-as-consequence clause. No value is derived here.

## OD-4 — Population & DEV Window: validation of the proposed design

Proposed: M0 baseline · `Event=A, StepFrom=3, StepTo=4 (Mode=0)` · **one first valid decision per Setup** · DEV only.
- **Alignment (strongest property):** calibration population ≡ deployment decision point under the approved latched semantics (first valid decision tick). Registered observation supports transportability: pre-veto first-decision features were bitwise identical across runs where undisturbed.
- **Uniqueness invariant** (one attempt per setup) held on the registered M0 export (431/431) and must be re-verified as a hard gate on every data drop, with violations counted and reported.
- **M0 as reference** is correct: candidate-run first-decision ticks coincide with M0's (pre-veto state identical), so the M0 attempt set is the lawful calibration set; the C2 run's own logs then serve only for evaluation.
- **DEV exclusivity for selection** is sound; registered caveat preserved: DEV is used for derivation *and* DEV-side reporting ⇒ irreducible in-sample optimism; decision-read must be OOS/TRUE_FORWARD-first (already in the prereg).
- **Stratum limitation (to record, not a defect):** this population is survivor-conditioned (baskets that reached Step3→4). Thresholds govern exactly this stratum and must not be generalized to other steps or to setup entry.
- **Adequacy:** DEV n in the few-hundreds suffices for a point estimate of the quantile with documented error; OOS adequacy is a separate *gate* (C5), never a derivation input — correct design.
**Verdict: design is VALID for its stated purpose**, conditional on the recorded gates (uniqueness, field purity, timestamp sanity, feature completeness) being re-run on the actual freeze-time extraction.

## OD-5 — Owner Decision Checklist (current OPEN state)

| # | Decision | Options | Analyst note | State |
|---|---|---|---|---|
| OD-1 | Shift convention | shift=0 / shift=1 | recommend **shift=1** | **OPEN** |
| OD-2 | ATR threshold type | absolute DEV-frozen / relative causal-rolling | recommend **relative-causal** if "top-quartile moments" is the intent; absolute = explicit transfer-test | **OPEN (AM-3)** |
| OD-3 | Threshold method | Q75 type-7 (proposed) / other fixed quantile | must be signed ex ante with rationale; no outcome-linked choice | **OPEN** |
| OD-4 | τ_ATR literal | — | blocked until OD-1..OD-3 | **NOT CHOSEN** |
| OD-5 | τ_ADX literal | — | blocked until OD-1..OD-3 | **NOT CHOSEN** |
| OD-6 | adverse-direction definition + δ | direction-mapped DI formula (or basket-based variant) | exact formula + δ value; **no default exists** | **OPEN** |
| OD-7 | Boolean rule final wording | working: `ATRpct ≥ τ_ATR ∧ ADVERSE` | sign-off only after OD-1..OD-6 | **OPEN** |
| OD-8 | Population & DEV window | as validated in §OD-4 | confirm + re-verify gates at freeze time | **OPEN (confirm)** |
| OD-9 | Acceptance margins (C3/C4 numbers) | owner-set bounds | required before any run | **OPEN** |
| OD-10 | Code authorization for latched semantics | yes/no | hard precondition of any C2 test run | **OPEN** |
| AM-5 | "Adverse" semantics | trend-against-position (DI) vs basket-relative | mutually exclusive; choose before OD-6 fix | **OPEN** |
| AM-6 | DEV-reuse mitigation | accept OOS-first discipline / designate fresh confirmatory window | already mitigated in-part by registration | **OPEN** |

**Forbidden during this phase (restated):** backtest · code change · threshold calculation · threshold freeze · rule registration · OOS analysis · production change.
**Production = M0. C2/M2 = DESIGN ONLY / NOT REGISTERED / NOT FROZEN.**
