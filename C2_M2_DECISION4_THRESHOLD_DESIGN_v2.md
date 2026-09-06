# C2 / M2 — DECISION 4: THRESHOLD-DEFINITION DESIGN v2 (supersedes v1; DESIGN-ONLY)

**Date:** 2026-09-06 · **State:** `DESIGN ONLY — NO NUMERIC THRESHOLD CHOSEN — NO FREEZE — NO CODE CHANGE — NO BACKTEST`
Chain: prereg draft `1d976a6` → Decision-3 + Decision-4 design v1 `ad9f231` → **this v2**.

> v1 remains archived for provenance. Where v1 and v2 conflict, **v2 governs**.

---

## A. Registered framework (owner-approved frame, 2026-09-06)

### A1. Proposed population
- **M0 baseline run**, event rows with `Event=A AND StepFrom=3 AND StepTo=4 (Mode=0)`;
- exactly **one valid decision per Setup** — the population is the set of **first valid Step3→4 decision ticks** per setup (latch-aligned with the approved Decision-3 semantics).

### A2. Proposed window
- **DEV only** (`DecisionTime < 2024-01-01 00:00:00`) for any threshold selection;
- **OOS fully quarantined** for threshold selection: no reads, no counts, no summaries, no influence.

### A3. Proposed method
- **Q75 on the feature distribution** — status: **PROPOSED METHOD only**; the owner's final method choice (incl. a different fixed quantile) is pending. Under no circumstance may any value be influenced by outcomes, performance, veto-rate targets, or OOS.

### A4. Freeze-before-OOS checklist
**ALL** of the following must be frozen (in one hash-chained `C2_THRESHOLD_FREEZE` artifact) **before any OOS run or OOS access**:
1. shift convention (bar convention) — **OPEN (AM-1)**;
2. ATR threshold value — **NOT chosen**;
3. ADX threshold value — **NOT chosen**;
4. adverse-direction definition (exact direction-mapped formula) — **OPEN**;
5. the Boolean rule (final form; working form from the prereg draft: `VETO ⇔ ATRpct ≥ τ_ATR AND ADVERSE`) — **final wording pending owner**;
6. latched semantics — **APPROVED (Decision 3)**, recorded here by reference;
7. DEV/OOS windows — registered boundary `2024-01-01 00:00:00` (cutoff pinned by experiment registration);
8. evaluation endpoints — registered primary/secondary metrics (prereg §11–§12); acceptance margins **owner-set, OPEN**.

### A5. Leakage controls (binding)
1. **Outcome columns forbidden in derivation** — derivation reads only a fixed whitelist of identity/decision/feature columns; assertion + log.
2. **OOS forbidden before freeze** — nothing OOS-derived may precede the freeze commit (commit-order proof in the chain).
3. **Retuning forbidden** — single-shot freeze; the only permissible amendment is an owner-signed, hash-chained defect fix; no value is ever adjusted after observing run artifacts or joint effects.
4. **Population alignment** — derivation population ≡ deployment population: first valid decision tick per setup (first-decision / latched).
5. **Full provenance** — population definition + its hash, derivation-script SHA-256, integrity-gate results, artifact commit hashes in a single chain; every figure in the freeze artifact is reproducible from named blobs.

---

## B. Items removed or demoted (owner directive, 2026-09-06)

- **Cross-feature correlation figure (previously cited ρ≈0.04):** removed from the specification. It is **not** evidence of independence and must not appear as such; at most a *future descriptive review* item, if ever.
- **DI-gap default (previously δ = 0):** removed as any kind of definitive choice. **δ is an OPEN owner decision**; no default is asserted anywhere in this design.
- **"20-row manual convention audit" criterion:** removed from the specification (arbitrary); may exist only as an informal future check, not a registered gate.
- **0.06754475:** retained **strictly as M1 provenance** (see `M1_THRESHOLD_FREEZE_POSTHOC.json`). For **C2/M2 it is NOT selected, NOT approved, NOT frozen — in any branch**; no language in this design may present it as a C2/M2 candidate. Selection of any C2/M2 literal is a separate, explicit owner action inside a future freeze act.

---

## C. Convention-conditional note (design only — no values)

The same protocol applies under either bar-convention branch once AM-1 is decided:
- the derivation population (A1), DEV window (A2), whitelist/gates (A3–A5) are branch-invariant;
- branch choice only changes **how feature values are obtained** (as-logged at decision tick vs recomputed from fully closed bars before `t_d`);
- **no threshold literal exists for either branch until a future `C2_THRESHOLD_FREEZE` act** (owner-approved, single-shot, hash-chained).

## D. Open owner decisions (all remain OPEN)

| ID | Item |
|---|---|
| AM-1 | shift/bar convention (forming-bar vs closed-bar) |
| AM-3 | absolute DEV-frozen threshold vs relative/within-period variant |
| — | final threshold method (Q75 proposed; pending final decision) |
| — | ATR threshold literal · ADX threshold literal (both NOT chosen) |
| — | adverse-direction definition formula · δ (OPEN; no default) |
| — | final Boolean rule wording |
| — | acceptance margins for the registered endpoints |
| — | code authorization for the approved latched semantics (precondition of any C2 test run) |

**FOOTER:** Nothing in this document freezes, selects, registers, or tests anything. No backtest was run. No code/strategy/production change. **Production = M0. C2/M2 = DESIGN ONLY / NOT REGISTERED / NOT FROZEN.**
