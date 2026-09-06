# M1 FEATURE / RULE RESEARCH — OFFICIAL CLOSE-OUT

**Date:** 2026-09-06
**Status:** `M1 FEATURE RESEARCH = CLOSED`
**Registered by:** project owner directive; recorded by Arena agent
**Provenance chain:** threshold re-freeze `234dc6b` → experiment close-out `f4fd889` → research approval `164ea76` → **this close-out**

---

## 1. Scope of the closed phase

This document formally closes the **M1 Feature / Rule Research Analysis** phase. The phase produced
*exploratory* observations only. Nothing in this phase constitutes performance evidence, a production
decision input, or a validated rule.

## 2. Registered exploratory results (ALL exploratory, NONE confirmatory)

The following are registered **strictly as exploratory/observed descriptors of the available incomplete
export** and must never be quoted as confirmatory evidence:

| # | Item | Exploratory value |
|---|---|---|
| 1 | Eligible-cohort SL-enrichment vs base rate | 1.74× (47.1% vs 27.1%) |
| 2 | Share of SL-exit PnL mass inside eligible cohort | ≈38% (−10,180 of −26,928) |
| 3 | Toxicity gradient across attempt-ATR quartiles (Q1→Q4) | mean −22.6 / −32.2 / −40.9 / −86.4; SL-rate 8%→12%→15%→29% |
| 4 | Q4/Q1 contrast | ≈3.8× (exploratory descriptive ratio) |
| 5 | AUC-style separation of ATRPct100 vs harmful outcome | ≈0.65 (exploratory) |
| 6 | Association p-value of gradient | p≈7e-5 (exploratory, in-sample, NOT confirmatory) |
| 7 | Affected-cohort paired Δ (n=117, observed exits) | Σ=−499.37; blocked-13 +415.22; delayed-104 −914.59 |
| 8 | Veto mechanism | delay-dominated (89% lapse; median dwell ≈7 min) |
| 9 | ADX vs ATRPct100 at decision point | near-orthogonal (ρ≈+0.04) — channel exists, relevance unproven |
| 10 | Regime non-stationarity | eligibility rate 13%→61% (2026); DEV/OOS distribution shift |
| 11 | Control-cohort isolation | 4,723/4,723 non-affected aligned setups bit-identical |

> **Explicit non-claim:** items 4–6 (Q4/Q1, AUC≈0.65, p≈7e-5) are **in-sample exploratory descriptors**,
> derived and evaluated on the same DEV-biased window. They are **not** confirmatory statistics,
> **not** evidence of edge, and must not be presented as such anywhere.

## 3. Insufficiency statement

**The current Feature Research is NOT SUFFICIENT for any Production decision.** Reasons: DEV-side
in-sample optimism (selection and evaluation share the same window); OOS sample too small (n=65
attempts / 25 affected); regime non-stationarity of the frozen absolute threshold (eligibility 25.1%
DEV → 61% in 2026); direction asymmetry (eligible SELL n=11); missing-window dependence of all
global aggregates (`DATA-LIMITED / ASSUMPTION-BASED`); and the observed veto mechanism differing from
design intent (delay-dominated, not persistent-block).

## 4. Canonical registered state

```text
M1 FEATURE RESEARCH = CLOSED
M1 PERFORMANCE      = NOT PROVEN / NOT ISSUED
ATR                 = CANDIDATE FEATURE / RISK-REGIME DESCRIPTOR (not a validated management rule)
M2/C2               = EXPLORATORY / NOT REGISTERED / NOT FROZEN
PRODUCTION          = M0
PRODUCTION CHANGE   = NONE

M0                  = PRODUCTION
M1                  = APPROVED FOR RESEARCH CONTINUATION (research track only)
M1 threshold        = 0.06754475 (FROZEN)
M1 observed export  = TRUNCATED AT 2026-08-17 22:27:04
Cutoff              = 2026-08-18 23:59:58
Coverage to cutoff  = DECISION ASSUMPTION ONLY — never to be presented as observed fact
DATA COMPLETENESS   = FAIL
PERFORMANCE VERDICT = NOT ISSUED
EDGE                = NOT PROVEN
```

## 5. Standing prohibitions (continuing obligations)

- No threshold extraction, retune, sweep, or optimization — of any existing or new value.
- No new rule registration or freeze.
- No code / strategy / production parameter change; no latch or new behavior.
- No production deployment of M1 (or any derivative).
- No fabrication of data/trades/PnL/events for the unobserved window; no row-by-row equality claims.

## 6. Unlock condition for the NEXT phase

The next phase may start **only after a NEW pre-registration** (with owner authorization) that
specifies, at minimum: population, feature(s), decision-time convention (forming vs completed bar),
DEV/OOS protocol, single pre-committed threshold value source, acceptance criteria, and a
TRUE_FORWARD evaluation plan. Without a new pre-registration, nothing beyond audit/provenance work
may be performed.

### Unprovable without TRUE_FORWARD (carried over)

Live forming- vs completed-bar ATR behavior; slippage/spread realism of delayed fills; regime
robustness beyond 2026-08; persistence of the blocked-cohort effect (+415.22, n=13) under latch
semantics; enrollment-drift cost in deployment; anything about the unobserved window
2026-08-17 22:27:04 → 2026-08-18 23:59:58.

## 7. Data & provenance references

| Item | Reference |
|---|---|
| Data branch | `agent-regression-data-real` @ `f349093b5137f63ed7e9cf8b4833d86f76b9421c` (unchanged since 2026-09-06 12:11 local) |
| M1 events blob | `28e58f270cd5864458fdd065b0c2a1d5a2e97440` |
| M0 events blob | `f0435f88f536c6f4534505daa6d6546f02e88cd2` |
| Freeze artifact | `M1_THRESHOLD_FREEZE_POSTHOC.json` @ `234dc6b` |
| Experiment close-out | `M1_EXPERIMENT_CLOSEOUT.json` @ `f4fd889` |
| Research approval | `M1_RESEARCH_APPROVAL.json` @ `164ea76` |
| Labeled artifacts | any global-at-cutoff aggregate ⇒ `DATA-LIMITED / ASSUMPTION-BASED` |

**END OF CLOSE-OUT — M1 FEATURE RESEARCH = CLOSED**
