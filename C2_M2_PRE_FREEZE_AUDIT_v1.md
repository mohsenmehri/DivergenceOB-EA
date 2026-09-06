# C2 / M2 — PRE-FREEZE AUDIT v1

**Date:** 2026-09-06 · **Audited object:** `C2_M2_FORMAL_REGISTRATION_SPEC_v2.md` @ commit `48ad17e`
**Audit scope:** specification correctness only. No computation, no data re-verification, no threshold material, no execution of any kind. Constraints honored throughout: no threshold calc/freeze, no backtest, no OOS, no TRUE_FORWARD, no sweep/tuning, no code change, no production change.
**Chain:** prereg `1d976a6` → D3/D4-v1 `ad9f231` → D4-v2 `1d0b390` → memo `a25202f` → sheet v2 `606c07f` → spec v1 `871fe50` → spec v2 `48ad17e` → **this audit**.

---

## A. Item-by-item findings (15 required items)

| # | Item | Verdict | Location | Note |
|---|---|---|---|---|
| 1 | Gate order exact: `Market → Spread → Margin → First Valid Decision → LATCHED VETO → PlaceEntry(next=4)` | **PASS** | §11 | Exact pipeline + 6 registered semantics; instrumentation/order-verifiability required (§11-6); violation = run rejection (§18). |
| 2 | Shift=1 precise; no forming-bar/shift-0 in spec | **PASS** | §4 (primary), §19 (separation table only) | Admissible set = bars strictly preceding the bar containing t_d (MQL5 shift ≥ 1); the "60s" arithmetic is stated only as a derivative of the registered timeframe. §19 mentions shift-0 solely as prohibited contrast — not contamination. |
| 3 | PriceRef(Shift1) exact, deterministic, population-uniform | **PASS with formal correction C-1** | §5 | The ATRPct denominator is the Close of the last admissible bar (reproducible per t_d; constant within a bar; one definition across the population). **C-1:** the price reference is not a named definition — see Correction C-1. |
| 4 | ATRPct precise & reproducible | **PASS** | §5 | Wilder iATR(14) and Close from the same last admissible bar; platform-canonical algorithm; independent re-derivation gate required (§4 final bullet, §15). |
| 5 | ADX threshold + DEV-Q75 methodology precise; nothing numeric | **PASS** | §6–§7 | Q75 type-7 exactly specified (rank formula, neighbors recorded); literals explicitly pending the freeze act; the "structural/outcome-blind ADX quantile" sentence is mandated verbatim; no numbers derived anywhere in the spec. |
| 6 | DI direction-aware unambiguous (BUY/SELL) | **PASS** | §8 | Explicit mirrored formulas; strict `>` with δ_DI; equality ⇒ not adverse (deterministic); δ_DI held as proposed/default parameter (proposed form 0) — **not frozen** (per owner correction). |
| 7 | Boolean exactly `ATR_condition AND Adverse_condition` | **PASS** | §9 | Single conjunction; no side branches; change requires version increment. |
| 8 | LATCHED semantics (first valid decision / per-Setup / post-VETO Step4 ban / no reopen) | **PASS** | §10 | All four components exact and gate-order-aligned (t₀ = post-gates eligible complete decision); per-Setup prohibition; no re-arm; baseline elsewhere untouched. |
| 9 | Population exact (M0 → Event=A → StepFrom=3 → StepTo=4 → first valid decision/setup → DEV only) | **PASS with substantive correction C-2** | §2 (+§6, §10–§11, §14 G-U) | Definition is exact and latch-aligned. **C-2:** consistency between the M0 exporter's `Event=A` emission point and the FINAL gate order must be verified before derivation — see Correction C-2. |
| 10 | DEV/OOS boundary & cutoff consistent | **PASS** | §12 | DEV `< 2024-01-01 00:00:00`; OOS to registered cutoff `2026-08-18 23:59:58`; TRUE_FORWARD beyond; labels mandatory. |
| 11 | Leakage / duplicates / missing features / coverage / timestamp integrity | **PASS** | §13 (1–8), §14 (G-U/S/R/C/T) | Leakage controls binding (whitelist reads, outcome quarantine, single-shot freeze, no retune, alignment, OOS quarantine, convention exactness, family discipline). Gates cover duplicates (G-U), missingness with cause classification + no silent drops (G-C), coverage span/reconciliation (G-S/G-R), time integrity (G-T). |
| 12 | No M1 information (shift=0, the M1 threshold literal) inside C2/M2 | **PASS with formal correction C-3** | §19, §15, §18-6 | Prohibition architecture is complete (voiding clause, no-anchor clause). **C-3:** the M1 threshold *literal* currently appears textually inside §19's prohibition row — replace by reference token (see Correction C-3). |
| 13 | Provenance & Freeze Artifact Schema | **PASS** | §15 | Complete schema (rule/version, population hash, convention + stated-origin arithmetic, windows, formulas, 3 literals, method + script SHA-256, quantile neighbors, DEV block, joint eligibility as descriptive-only, gate-order declaration, gate results, owner sign-off, commit hash). |
| 14 | Acceptance/Rejection framework; no numeric margins added | **PASS** | §16 (+§17-1) | Six-dimension framework preserved; margins explicitly deferred to the freeze act as owner literals; verdict taxonomy incl. INCONCLUSIVE BY DESIGN. No margin invented in spec or audit. |
| 15 | Remaining ambiguity/contradiction register | **Addressed** | §B below | Two formal + one substantive items (C-1..C-3); one documented edge rule (see B-1); code authorization pending (registered execution precondition, §18-7) — not an ambiguity; δ_DI pending by design — not an ambiguity. |

## B. Residual ambiguity register

- **B-1 (resolved in-text, no defect):** a decision tick exactly on a bar boundary (`t_d` == bar open) is governed by the interval rule `open(b0) ≤ t_d < close(b0)` ⇒ `b0` = that tick's own (forming) bar ⇒ all prior bars admissible. Deterministic; consistent with the 60s arithmetic.
- **B-2 (owned by C-2):** unknown whether the M0 exporter emitted `Event=A` rows *at* post-execution-gate instants or pre-gate. Affects population identity, not rule design; resolved by the added verification gate.
- **B-3 (registered, not an ambiguity):** owner code authorization for latched semantics + §11 instrumentation remains the sole execution precondition (§18-7 halts execution until granted).

## C. Corrections (exact location + proposed fix)

### C-1 — Named price reference (formal; §4/§5)
**Finding:** ATRPct embeds its denominator informally; reproducibility/audit wording should name it.
**Proposed fix (spec amendment, v-next):** add to §5 first line: *"Let `PriceRef(t_d) = Close(M1, last admissible bar at t_d)`; then `ATRpct(t_d) = ATR(14,M1,shift=1) / PriceRef(t_d) × 100`."* and add to §4: *"PriceRef is deterministic per decision and constant for all decisions inside the same admissible-bar boundary; the same definition applies uniformly to the whole population."*

### C-2 — Population↔gate-order emission verification gate (substantive; §2/§14)
**Finding:** The FINAL gate order makes the population = decisions that PASS Market/Spread/Margin gates (§10–§11). The calibration population is defined from M0-export `Event=A` rows (§2). It is currently an *instrumentation assumption* that the M0 exporter emitted `Event=A` exactly at post-execution-gate instants; if it emitted them pre-gate, pre-gate failures would be invisible and the frozen threshold population would drift from the deployment population.
**Proposed fix (spec amendment, v-next):** add gate **G-P (population-emission)** to §14: *"Before any freeze-time derivation: (a) from the player's implementation + its logs, verify that `Event=A` rows are emitted exactly at first-valid post-execution-gate decision instants; (b) if not, re-filter the extraction set to post-gate decisions via gate-outcome logs, or re-instrument the exporter before any derivation; (c) freeze-time n must then reconcile with the registered population under the verified emission rule; deviations are reported, never silently repaired."* Consequence chain: τ_ATR/τ_ADX literals and n_DEV references (§6) become conditional on G-P = PASS.

### C-3 — Remove the M1 threshold literal from C2/M2 text (formal hygiene; §19)
**Finding:** §19's prohibition row embeds the M1 threshold literal in C2/M2 text (inside a prohibition — necessary in spirit, but the audit rule is "no M1 information inside C2/M2").
**Proposed fix (spec amendment, v-next):** replace the literal in §19 with the reference token: *"`«M1-THRESHOLD»` — redacted; defined solely in `M1_THRESHOLD_FREEZE_POSTHOC.json` (M1 provenance)"*, keeping all prohibition language intact. (This audit document itself refrains from reproducing the literal.)

---

## D. Audit verdict

```text
PRE-FREEZE AUDIT = PASS WITH CORRECTIONS
```

- The specification is internally consistent, complete on all 15 required items, and free of computation, freeze, or execution content.
- Corrections **C-1 (formal), C-2 (substantive verification gate), C-3 (formal hygiene)** must be applied as a hash-chained specification amendment **before any freeze act**. C-2 is the binding one: no derivation may proceed until G-P passes.
- No numeric margins, thresholds, or parameters were introduced by this audit.

**Halting here as instructed; awaiting the Owner's next decision (apply corrections → re-audit/confirm → then, and only then, the freeze act). Production = M0.**
