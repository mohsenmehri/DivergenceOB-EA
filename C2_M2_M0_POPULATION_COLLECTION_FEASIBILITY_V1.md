# C2/M2 — M0 Shift-1 Population Collection Feasibility (V1)

**Date:** 2026-09-07
**Status:** REVIEW ONLY — NO CODE CHANGE PERFORMED
**Production:** M0 (unchanged)

---

## Question

Can the current repaired source collect the approved M0 Shift-1 Step3→4 population with `inp_cf_mode = 0`, while keeping the independent C2/M2 Shift-1 instrumentation active and NOT applying any veto?

## Answer

**NO — the current code cannot do this as-is.**

`inp_cf_mode = 0` bypasses the C2/M2 Shift-1 module entirely. To collect the M0 population with the same Shift-1 instrumentation, a **minimal, diagnostic-only instrumentation change** is required (described in §4). This is not a strategy/rule/threshold change.

---

## 1. Exact code path (current repaired source)

File: `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5`
Source SHA at this diagnosis: `128a5bc9b2df4d960b2d7632a38f247cd2de588ebaa59c1874a0f50bed479ed3`

### Gate in `ManagePositions` (around line 42594)
```cpp
bool is_c2m2 = (inp_cf_mode == 2 && cur_step == 3 && next == 4);
if(is_c2m2)
   cf_veto = C2M2_Step34ShouldVeto(idx, is_bull, tk);
else
   cf_veto = CF_ShouldVeto(cur_step, next, is_bull);   // Shift 0 path
   CF_LogEvent(...);
```

- With `inp_cf_mode = 0`, `is_c2m2 = false`.
- The code therefore uses the **legacy Shift-0** path (`CF_ShouldVeto` + `CF_LogEvent`).
- The isolated C2/M2 exporter is never called.

### Guard inside `C2M2_Step34ShouldVeto` (line 19422)
```cpp
if(inp_cf_mode != 2) return false;             // isolation guard
```
- Even if invoked, mode 0 returns immediately.
- No Shift-1 decision row is logged.

### Exporter (`C2M2_LogStep34Decision`, line 19363)
- Opens `C2M2_Step34_Decisions_m{inp_cf_mode}.csv` with `FILE_COMMON`.
- Writes all required Shift-1 fields — this part is already correct.
- First column currently is always `C2M2_DECISION`; it does **not** write `Event=A`.
- `Rule` is always `M2`.
- `Mode` is written as `inp_cf_mode` (so mode 0 would write `Mode=0` if invoked).
- Provenance columns exist (`SourceCommit`, `SpecVersion`, `Build`), but `SourceCommit` is currently hard-coded to `29dfb15` (the original baseline), not the current collection build.

---

## 2. Why a dedicated collection mode is needed

| Requirement | Current code | Needed for M0 population |
|---|---|---|
| Record Shift-1 Step3→4 decisions | only for `inp_cf_mode=2` | must run under M0/no-veto |
| Do not apply veto | only possible with thresholds `0.0` in M2 | must be unconditional no-veto |
| Record `Event=A` | exporter writes `C2M2_DECISION` | must label accepted M0 decision rows as `A` |
| Do not let C2/M2 affect trading | mode 2 with threshold 0 does not veto, but is classified M2 | must be explicit `M0_COLLECT` / no rule application |
| First valid per Setup | already handled by `c2m2_step34_latch` (FIRST/ALLOW) | same behavior, but no veto path |

The clean design: a **collection-only switch** that is orthogonal to mode. With that switch:
- `cf_veto` is always forced `false`.
- The C2/M2 step is logged once per setup as `Event=A`, with all Shift-1 fields.
- The existing latch is used to guarantee one record per `(SetupID, StepFrom=3, StepTo=4)`, but it never blocks placement.

---

## 3. Run range: Full vs DEV only

**Use DEV only:**

```
2014-01-01 00:00:00  →  2023-12-31 23:59:59
```

Reasons:
1. The approved threshold derivation uses **DEV only** (`C2_M2_FORMAL_REGISTRATION_SPECIFICATION.md` §6/§7/§16).
2. Running only DEV guarantees **no OOS data appears in the collection export**, eliminating any ambiguity.
3. Full-range collection is unnecessary for Q75 on DEV; it would also pull OOS decision rows into the export, which is undesirable pre-freeze.

Regression range from the M0 population: 431 unique `Event=A, StepFrom=3, StepTo=4` setups; DEV subset is 366 setups. A DEV-only run should yield `n = 366` expected rows (confirmable after collection).

---

## 4. Minimum instrumentation required (NO code changed in this step)

To make the M0 collection runnable, the implementation needs one **isolated, non-strategy change** (proposed only, not applied here):

1. **Add a collection-only input**, e.g.:
   ```
   input bool inp_c2m2_collect_shift1_m0 = false;
   ```
   or a mode value, e.g. `inp_cf_mode = -1` (M0-collection) — Owner/method must pick the naming.

2. **Enable the C2/M2 Shift-1 observer in M0**, e.g. change the call condition:
   ```cpp
   bool is_c2m2 = (inp_cf_mode == 2 && cur_step == 3 && next == 4)
                  || (inp_c2m2_collect_shift1_m0 && cur_step == 3 && next == 4);
   ```
   and in `C2M2_Step34ShouldVeto`, remove/relax the `inp_cf_mode != 2` guard for collection mode.

3. **Force no-veto in collection mode**:
   - `C2M2_Step34ShouldVeto` must return `false` in collection mode regardless of threshold/ADX/DI.
   - It must NOT call `CF_ShouldVeto`/`CF_LogEvent` (keep M0 order path untouched).

4. **Label the row as `Event=A` for collection mode**:
   - Extend `C2M2_LogStep34Decision` with an event label parameter (or a collection-mode path that writes `A` in the first column).

5. **Set correct Rule/provenance for collection**:
   - `Rule = M0_COLLECT` (or `NONE`) instead of `M2`.
   - `Mode = 0`.
   - `SpecVersion`, `Build`, and `SourceCommit` should be set to the **actual collection build/commit**, not the original `29dfb15`.

6. **Record all already-supported fields** (already in exporter):
   - `SetupID`, `StepFrom`, `StepTo`, `T_decision`, `DecisionServerTime`
   - `FeatureShift=1`
   - `BarOpenTime_Shift1`, `BarCloseTime_Shift1`
   - `ATR_Shift1`, `ATRPct_Shift1`, `ADX_Shift1`, `DIPlus_Shift1`, `DIMinus_Shift1`, `PriceRef_Shift1`
   - `strict_before_check`
   - `LatchStatus=FIRST`
   - provenance/source commit columns

7. **Keep strategy/order logic unchanged**:
   - Do not alter entry conditions, ladder geometry, margin/market/spread gates, `PlaceEntry`, `current_step`, M1/M3 paths, or `CF_ShouldVeto`/`CF_LogEvent` for mode 0.

---

## 5. Proposed input set (after the minimal instrumentation)

```ini
Symbol=XAUUSD
Timeframe=M1
inp_cf_mode=0
inp_c2m2_collect_shift1_m0=true
inp_cf_q75_atr_pct_34=0.0
inp_cf_q75_adx_34=0.0
inp_cf_q75_atr_pct_23=0.0
inp_atr_period=4
inp_adx_period=14
```

Keep all other inputs as used in the M0 regression run (default touch confirm, trend guard disabled, etc.). Do NOT set any M2 threshold.

---

## 6. Expected output

If the minimal instrumentation is made and the DEV-only run is executed, the output should be:

```
C2M2_Step34_Decisions_m0.csv
```

Expected rows:
- `n = 366` (DEV-only, M0 Event=A Step3→4 first-valid population)
- `Event = A`
- `FeatureShift = 1`
- `strict_before_check = true`
- `LatchStatus = FIRST`
- `Mode = 0`
- `Rule = M0_COLLECT` (or `NONE`)
- all Shift-1 fields populated
- provenance columns populated

Note: currently the exporter would name the file `_m0.csv` for mode 0 (already correct), but it would not be invoked without the instrumentation.

---

## 7. Conclusion

- **Current code: cannot collect with `inp_cf_mode=0` as-is.**
- Required: only the **isolated collection-only instrumentation** described above.
- No strategy, rule, threshold, M2, M1, M3, or production behavior change is required.
- **Recommended run: DEV only**, `2014-01-01 00:00:00 → 2023-12-31 23:59:59`.
- No code change was made in this step.

---

**Status:** Feasibility review complete; instrumentation proposal only. Production remains M0. No OOS, no freeze, no threshold derivation, no full run, no performance verdict.
