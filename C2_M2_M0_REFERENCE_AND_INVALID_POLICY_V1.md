# C2/M2 — M0 Reference Resolution + 9 INVALID Policy (V1)

Status: alignment/normalization only. **NO NEW BACKTEST RUN, NO CODE CHANGE,
NO THRESHOLD CHANGE, NO PERFORMANCE VERDICT.**

---

## 1. M0 Reference Resolution

Two M0 data sets exist in the discussion:

| Data set | Setups | Position rows | End time marker |
|---:|---:|---|
| Owner-provided M0 reference | 4,864 | 9,605 | cutoff `2026-08-18 23:59:58` implied (M2-matched) |
| Repo M0 artifact `v2_research_export_v2-new/` | 4,865 | 9,606 | `PositionOpenSummary` Time `2026.08.19 23:59` |

**Conclusion (evidence-based): these are different M0 run variants, not the
same run.** The repo artifact’s own summary says 2026-08-19; the owner-provided
reference is the pre-extra-setup variant. The difference is exactly +1 setup
and +1 position row.

---

## 2. 4864 vs 4865 discrepancy

### Repo artifact `v2_research_export_v2-new` (committed in `0e91848`, source unchanged from `ad4f6e4`)

| Item | Value |
|---|---|
| Manifest path | `v2_research_export_v2-new/RX_XAUUSD_RX_1388707200_XAUUSD_Manifest.json` |
| Manifest `rows` | `4865` |
| Manifest SHA-256 | `67d408af4c5e674ee1a469ea312d988796ff52cb1a9cb8db6b5c851b2b411995` |
| `SetupID_XAUUSD.txt` | `4866` (next ID) |
| `Stats_XAUUSD.txt` | `NextSetupID=4866`, `Stat0=4865` |
| `PositionOpenSummary` All Opened | `9606` |
| M0 source file | `v2_research_export_v2-new/v2_research_export_v2.mq5` |
| M0 source SHA-256 | `d961dcb9113749aa8a814fd88e60a87075e66c27094a239b1e0e18d9d9bc99f6` |
| M0 source commit | `ad4f6e400191f28e303d9715f2e6c8e8f76714c0` (unchanged later; also present at `0e91848`) |
| M0 config (broker/deposit/leverage/model/spread from MT5 report) | `NOT AVAILABLE` (report is Git-LFS, not fetchable in this environment) |

### Owner-provided M0 reference

| Item | Value |
|---|---|
| Setups | `4,864` |
| Position rows | `9,605` |
| Step1/2/3/4/5 | `4,864 / 2,970 / 1,149 / 431 / 191` |
| Censored at EOT | `setup #4864` |
| Commit / source SHA | `NOT AVAILABLE` (not supplied; no matching non-LFS metadata found) |
| Broker / symbol / timeframe / cutoff | `NOT AVAILABLE` except implied symbol/timeframe/cutoff from task |

**Exact per-row classification of what the extra M0 SetupID is**

- SetupID `4865` is not merely a counter: the M0 artifact has a real manifest
  row for it (`rows=4865`), and the M0 dataset opens an extra position row
  (`9,606` vs `9,605`).
- Whether SetupID `4865` is a normal full setup, a partial setup, or a
  censored EOT setup is **NOT PROVABLE** from non-LFS metadata. The M0
  `PositionOpenSummary` time `2026.08.19 23:59` is inconsistent with the
  M2 cutoff `2026-08-18 23:59:58`; this is *consistent with* SetupID `4865`
  being an extra setup captured after the M2-matched window, but that is an
  inference, not proven. No claim is made as fact.
- The extra position row is likewise **consistent with** belonging to the same
  extra setup, but is **NOT PROVABLE** without the LFS dataset.

---

## 3. Official M0 Reference

`OFFICIAL M0 REFERENCE` (for matched M2 comparison):

```
Setups = 4864
Position rows = 9605
Step1/2/3/4/5 = 4864 / 2970 / 1149 / 431 / 191
Censored at EOT = setup #4864
End/cutoff = 2026-08-18 23:59:58 (M2-matched)
```

Why: the M2 run is cut at `2026-08-18 23:59:58`. The owner-provided M0 counts
line up with that cut and are the only M0 data set that avoids the +1 setup
(+1 position) introduced by the stored `v2_research_export_v2-new` artifact,
whose summary extends to `2026.08.19 23:59`. The stored repo artifact is useful
as a *M0 baseline dataset*, but it is a different (extended) run variant.

**Caveat:** owner-provided M0 commit/source SHA/config are `NOT AVAILABLE`;
therefore the official reference is selected on count/time alignment, not on a
fully verifiable configuration. This is a “best available reference”, not a
fully proven matched baseline.

---

## 4. 9605 vs 9606 discrepancy

| Item | 9605 | 9606 |
|---|---|---|
| Data set | owner-provided M0 reference | repo artifact `v2_research_export_v2-new` |
| Difference | base | +1 position row |
| Likely source | M0 run ending before the extra setup | M0 run whose summary is `2026.08.19 23:59`; +1 setup, +1 Step1 position |
| Proof | — | **NOT PROVABLE** from non-LFS repo files |

Both the setup count and position-row count differ by exactly 1 in the same
direction. This strongly indicates the stored M0 artifact contains one extra
setup/position (probably after the M2 matched cutoff). The exact row is in the
Git-LFS dataset and could not be inspected here.

---

## 5. 9 INVALID behavior

Read from `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5`
(SHA-256 `327952595e1b1b04feedc8fb373874e7ee5624de6fbe007e97d75a3929734f9b`).

Code path `C2M2_Step34ShouldVeto`:

1. On `!ok_data || !strict_before`, the function sets
   `c2m2_step34_latch = C2M2_LATCH_INVALID` and logs `C2M2_Decision=INVALID`,
   `LatchStatus=FIRST`.
2. It then `return false;` — **no veto**.
3. On later calls, the latch check
   `if(c2m2_step34_latch == C2M2_LATCH_INVALID) return false;` keeps it
   no-veto permanently for that setup/step.
4. Therefore INVALID **is latched**, **allows Step4**, and is **fail-open**.
5. The 9 boundary rows have `BarCloseTime_Shift1 == T_decision`, so
   `strict_before_check=false`. They bypass the veto and appear in Step4/Step5
   all-clear records.

**Is it tester-only?**

- The trigger is `bar_close == t_decision` where `t_decision = tick.time`.
- This is a **bar boundary timestamp condition**. It can occur in the Strategy
  Tester (as observed 9 times), and it can also occur in live/production if a
  server tick’s time equals the M1 bar close time. It is **not** logically
  tester-only; it is timing-boundary-dependent. The exact live frequency is not
  measured here.

**Specification status**

- `C2_M2_IMPLEMENTATION_REPAIR_V1.md` states the implemented behavior:
  strict-before failure → `INVALID`, latched, **fail-open**.
- `C2_M2_IMPLEMENTATION_REPAIR_SPEC_V1.md` / `C2_M2_IMPLEMENTATION_REPAIR_DECISION_LIST_V1.md`
  earlier marked the “what should the gate do on invalid data” behavior as an
  **open owner decision** (allow / veto / no-decision).

So: current **code** is fail-open, but the formal **specification** is
ambiguous about whether fail-open was owner-approved or just a
recommendation/implementation choice. This is why the policy is surfaced below.

---

## 6. FAIL-OPEN vs FAIL-CLOSED

| Option | Behavior | Pros | Cons |
|---|---|---|---|
| **A — FAIL-OPEN** | `INVALID` allows Step4, logs the row | preserves M0-like execution; avoids blocking setups on rare boundary ticks; existing behavior | violates strict-before invariant; allows execution without a valid temporal proof; can include false execution; audit shows an invalid record but it still trades |
| **B — FAIL-CLOSED** | `INVALID` blocks Step4 | protects against execution on unverifiable temporal data; stricter no-lookahead guarantee; better false-execution protection | may drop setups that are actually valid but hit the exact boundary; changes behavior vs current run; requires code change (not done) |

**Recommendation (proposal only, not a final decision):** If the priority is
**protection against false execution** and **auditability of a strict
no-lookahead rule**, **Option B — FAIL-CLOSED** is the more appropriate design:
an unverified temporal decision should not permit execution. However, this is a
proposal only, and switching to it requires an explicit owner decision plus a
code change (which must not be made in this task).

---

## 7. Owner Decision Required / Resolved

- **9 INVALID policy:** because the pre-implementation spec explicitly left
  the invalid-input decision open, while the implementation record documents
  fail-open, the final policy is:

```
OWNER DECISION REQUIRED
```

  Current implemented behavior (do-not-change-without-approval): **FAIL-OPEN**.
  Recommended proposal for a future change: **FAIL-CLOSED** (not final).

- **Official M0 reference:** **RESOLVED for comparison purposes** as the
  owner-provided 4,864 / 9,605 reference (see Section 3), with the caveat that
  its commit/source SHA and full MT5 config are `NOT AVAILABLE`.

---

## 8. Exact prerequisite for matched M0 rerun

Before any M2-vs-M0 comparison:

1. Confirm owner-provided M0 reference vs stored `v2_research_export_v2-new`
   is truly one extra setup/position outside the M2 cutoff (needs the M0 LFS
   dataset or the original MT5 report).
2. Use the **same M2 source build**
   `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5`
   (SHA-256 `327952595e1b1b04feedc8fb373874e7ee5624de6fbe007e97d75a3929734f9b`)
   with `inp_cf_mode=0` and `inp_c2m2_collect_shift1_m0=false`.
3. Use the exact M2 MT5 settings: XAUUSD, M1, Every-tick real ticks,
   `2014-01-01` → `2026-08-18 23:59:58`, deposit 500,000, leverage 1:100,
   profile D_FULL, reset SetupID=true, fast-backtest=false, export dataset=true.
4. Generate the full M0 `PositionOpenDataset` and lifecycle at the same cutoff,
   then do SetupID-level parity against M2 before any performance verdict.

---

```
NO NEW BACKTEST RUN
NO CODE CHANGE
NO THRESHOLD CHANGE
NO PERFORMANCE VERDICT
```
