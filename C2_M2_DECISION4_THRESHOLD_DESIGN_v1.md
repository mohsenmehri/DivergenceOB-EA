# C2 / M2 — DECISION 3 (APPROVED) + DECISION 4: THRESHOLD-DEFINITION DESIGN (DESIGN-ONLY v1)

**Date:** 2026-09-06 · **State:** `DESIGN ONLY — NO NUMERIC THRESHOLD CHOSEN — NO FREEZE — NO BACKTEST`
Chain: prereg draft `1d976a6` → **this document**.

---

## PART A — DECISION 3, REGISTERED (owner-approved 2026-09-06)

| Item | Registered value |
|---|---|
| Veto persistence semantics (primary) | **LATCHED**: the rule is evaluated ONCE at the **first valid Step3→4 decision tick** of a setup; if vetoed, **Step4 never opens for that setup** (attempt dead unless setup invalidates) |
| Per-tick evaluation | demoted to a **possible future sensitivity analysis only** — never the primary semantics |
| Status consequences | latch semantics requires a minimal code change ⇒ **a separate owner code-authorization is a hard precondition of any C2 test run**. NO code change is authorized or performed by this document. No backtest, no registration, no freeze, Production = M0. |

---

## PART B — DECISION 4: THRESHOLD-DEFINITION DESIGN

### B1. Population for BOTH thresholds (τ_ATR and τ_ADX) — identical by construction

- Universe: the registered C2 population — **Step3→Step4 decision-attempt rows** (`Event=A, StepFrom=3, StepTo=4, Mode=0`) of the M0 control run; **one row per setup, taken at the FIRST decision tick** (matching the latched semantics of Part A; if a future exporter ever emits >1 decision row per setup, the population is defined as the FIRST row per setup and the invariant is checked and counted).
- Sampling window: **DEV only** — `DecisionTime < 2024-01-01 00:00:00` (observed M0: n = 366).
- OOS rows (`≥ 2024-01-01`, up to the registered cutoff) are **quarantined** for threshold purposes: never read, never summarized, never even counted for selection.
- Both features are read at **the same decision timestamps** on the **same rows** — a conjunction cannot have marginals calibrated on different populations, or its joint veto-rate is undefined.

### B2. Bar-convention branch (Decision 4 is convention-conditional; AM-1 still open)

The extraction protocol is identical; only the feature VALUES differ by convention:

| Branch | Feature values used | Consequence for τ_ATR | Consequence for τ_ADX / DI |
|---|---|---|---|
| **[B] shift=0 (as-logged)** | the M0 event-log columns `ATRPct100`, `ADX`, `DIPlus`, `DIMinus` exactly as the EA logged them at decision time | the already post-hoc-registered value **0.06754475** is the *legal candidate literal* (it was derived from this exact DEV population/window under this convention; stays labelled POST-HOC) | derive from DEV distribution of logged `ADX`; δ for the DI term fixed at **0** (pure dominance: DI+>DI− for SELL, DI−>DI+ for BUY) — identical to the semantics of the reference code path |
| **[A] shift=1 (closed-bar)** | features recomputed from M1 price history at `t_d` using only bars fully completed before `t_d` | a NEW derivation on the same 366 rows, producing a NEW literal in a NEW freeze artifact | same, new literal; δ=0 |

**Only ONE branch may be carried into the freeze.** If branch [A] is chosen, 0.06754475 is retired as a candidate and may never be reused.

### B3. Computation method (identical for both thresholds)

1. Column-read whitelist for derivation: `SetupID, DecisionTime, StepFrom, StepTo, Mode, Event, Direction` + the TWO feature columns for the active branch. **No outcome/PnL/lifecycle column is read** (assert + log in the derivation run).
2. Integrity gates (all must PASS, recorded in the artifact): unique attempt per setup; zero empty/NaN/non-numeric/negative feature values; timestamps parse; counts reconcile to the registered population (n = 366 DEV under [B]; identical n under [A], same SetupIDs).
3. Statistic: **Q75, Hyndman–Fan type-7 linear interpolation** — formula printed literally in the artifact: `h=(n−1)·p+1; Q = v[⌊h⌋]·(1−frac) + v[⌊h⌋+1]·frac` (1-based), sorted ascending; both neighboring order statistics recorded for reproducibility.
4. Deterministic reference implementation (python, committed BEFORE the numbers are generated); its file SHA-256 is recorded.
5. Output per feature: literal threshold value + full DEV descriptive block (n, min, q25, median, q75, q90, mean, start, end).

### B4. What freezes BEFORE OOS is touched (single artifact: `C2_THRESHOLD_FREEZE.json`)

Frozen **before** any candidate run, any OOS access, and any figure that could correlate with outcomes:
`rule_id` · population definition (+ hash) · branch/convention · DEV window bounds · per-feature formulas · the literal values `τ_ATR`, `τ_ADX`, `δ=0` · derivation-script SHA-256 · all integrity checks · joint DEV eligibility rate (descriptive consequence only) · commit-hash of this registration.
**Not frozen / prohibited:** no quantile selection by veto-rate targeting, no value adjustment after observing joint effects, no second candidate value, no re-derivation at any later point (defect-fix amendments only, owner-signed, hash-chained).

### B5. Leakage & selection-bias controls (binding)

1. **Single-shot freeze with commit-order proof:** the freeze artifact's commit must precede every C2-run artifact (verified timestamps in the provenance chain).
2. **Outcome quarantine during derivation:** whitelist-only reads (B3-1); the derivation script physically cannot see PnL.
3. **Code-path identity:** the threshold computation and the run-time veto consume the same feature definition; the post-run **bracket proof** is mandatory: `min ATRpct over vetoed-first-decision rows ≥ τ_ATR > max ATRpct over non-vetoed first-decision rows` (and identically for ADX with the conjunction semantics); any violation ⇒ run disqualified.
4. **Family discipline:** exactly two thresholds + δ=0 constitute the entire candidate family; ANY additional value/variant = new amendment + new freeze BEFORE its test.
5. **Independence witness (descriptive):** DEV cross-correlation of the two features recorded at freeze time (decision-time only) to document orthogonality; never used to reweight or re-select.
6. **OOS quarantine:** until freeze-commit is pushed, no person or script accesses OOS attempt rows; the analysis script is committed before unblinding any candidate-run data.
7. **Anti-tuning clause:** the joint veto-rate and its DEV/OOS behaviour are REPORTED, never TARGETED; adjusting a quantile to reach a desired rate or any outcome-linked criterion is prohibited.
8. **First-decision alignment (latch):** population rows are the first valid decision ticks; the C2 instrumentation must label the first decisive tick per setup with the same taxonomy, so calibration population ≡ deployment population.
9. **Adequacy statement inside the artifact:** n(DEV)=366 and the implied minimum-detectable-effect record (unpaired planning figures from the registered sd≈73$/setup) — recorded so underpowered reads fail C5 by design.
10. **Convention audit:** no silent mixing: every feature value in the freeze artifact carries its shift-flag; a row-level sample (e.g., 20 attempts) is recomputed independently at registration review.

---

## OPEN OWNER ITEMS (unchanged, still blocking approval of any test)

1. **AM-1:** choose branch [A] or [B] (decides whether a NEW numeric freeze is required or the existing literal is the legal candidate).
2. **AM-3:** absolute DEV-frozen threshold (this design assumes it) vs relative/within-period variant — a different rule, would rewrite B2–B4.
3. **Code authorization** for the latched semantics (Part A consequence) — precondition of any C2 run.

**FOOTER:** No numeric threshold was chosen or frozen by this document (branch [B] merely *references* the already-existing post-hoc literal); no backtest was run; no code/strategy/production change. Production = M0.
