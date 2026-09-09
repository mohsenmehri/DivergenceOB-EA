# C2/M2 — INVALID Fail-Closed Repair V1

Status: **CODE CHANGE COMPLETE — FAIL-CLOSED** (static validation only; no run).

## Decision

Owner final decision: **INVALID Policy = FAIL-CLOSED**.
This repair applies the minimal change to the isolated C2/M2 Step3→4 path only.

## Source provenance

| Item | Value |
|---|---|
| Branch | `arena/01a06d3c-divergenceob-ea` |
| HEAD before commit | `3569caba8681e3ca340cfa1606eb01b8ad0fe388` |
| File | `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` |
| Source SHA-256 BEFORE | `327952595e1b1b04feedc8fb373874e7ee5624de6fbe007e97d75a3929734f9b` |
| Source SHA-256 AFTER | `7cbd5c9b8198e9d2fd71bca35df1e06b94ba11b16b716da1984d24bbea4b2cc7` |
| Commit | added after this report |

## Exact diff

```diff
@@ -19434,9 +19434,11 @@ bool C2M2_Step34ShouldVeto(int setup_idx, bool is_bull, const MqlTick &tk)
    if(g_setups[setup_idx].current_step != C2M2_STEP_FROM) return false;

    // LATCH (Owner-approved D5 Option B): independent of PlaceEntry retry.
+   // INVALID is FAIL-CLOSED (Owner final decision): an INVALID decision
+   // latches and blocks Step4; placement retry must NOT turn it into ALLOW.
    if(g_setups[setup_idx].c2m2_step34_latch == C2M2_LATCH_VETO)    return true;
    if(g_setups[setup_idx].c2m2_step34_latch == C2M2_LATCH_ALLOW)   return false;
-   if(g_setups[setup_idx].c2m2_step34_latch == C2M2_LATCH_INVALID) return false;
+   if(g_setups[setup_idx].c2m2_step34_latch == C2M2_LATCH_INVALID) return true;

@@ -19451,7 +19453,7 @@ bool C2M2_Step34ShouldVeto(int setup_idx, bool is_bull, const MqlTick &tk)
          0.0, 0.0, 0.0, 0.0, 0.0, 0.0, false, "INVALID", "FIRST",
          "C2M2_DECISION", "M2", "C2M2_Step34_Decisions_m" + IntegerToString(inp_cf_mode) + ".csv",
          C2M2_PROVENANCE_SOURCE_COMMIT, C2M2_PROVENANCE_SPEC_VERSION, C2M2_PROVENANCE_BUILD);
-      return false;                              // fail-open, never blocks ladder
+      return true;                               // FAIL-CLOSED: INVALID blocks Step4
@@ -19481,7 +19483,7 @@ bool C2M2_Step34ShouldVeto(int setup_idx, bool is_bull, const MqlTick &tk)
          atr, atr_pct, adx, pdi, mdi, price_ref, strict_before, "INVALID", "FIRST",
          "C2M2_DECISION", "M2", "C2M2_Step34_Decisions_m" + IntegerToString(inp_cf_mode) + ".csv",
          C2M2_PROVENANCE_SOURCE_COMMIT, C2M2_PROVENANCE_SPEC_VERSION, C2M2_PROVENANCE_BUILD);
-      return false;                              // fail-open, never blocks ladder
+      return true;                               // FAIL-CLOSED: INVALID blocks Step4
```

Only `C2M2_Step34ShouldVeto` was changed. No ATR/ADX/DI, thresholds, Shift,
latch enum, rule values, or legacy path logic changed.

## Static validation (no compiler)

```
COMPILE NOT AVAILABLE — STATIC VALIDATION ONLY
```

Static checks performed on the working source:

| Check | Result |
|---|---|
| Braces / parentheses balance unchanged vs base | PASS (same counts) |
| Diff limited to `C2M2_Step34ShouldVeto` (3 INVALID return sites + comment) | PASS |
| `C2M2_FEATURE_SHIFT` still used for features | PASS |
| `strict_before = (bar_close < t_decision)` strict `<` unchanged | PASS |
| INVALID detection still sets `C2M2_LATCH_INVALID` and logs `INVALID`, `FIRST` | PASS |
| INVALID latch re-check now returns `true` (blocks Step4) | PASS |
| CopyIndicatorBuffers failure now returns `true` (block) | PASS |
| `!ok_data || !strict_before` now returns `true` (block) | PASS |
| Valid ALLOW path unchanged (`return veto` with `veto=false`) | PASS |
| Valid VETO path unchanged (`return veto` with `veto=true`) | PASS |
| Does not convert INVALID to ALLOW on retry (latch is permanent) | PASS |
| Legacy `CF_ShouldVeto` M0/M1/M3 guards unchanged | PASS |
| Legacy `CF_LogEvent` path unchanged | PASS |
| `is_c2m2` routing unchanged; only branch for `mode==2 && step3->4` | PASS |

## Smoke test (source-level)

No backtest was run. Source-level smoke checks:

| Case | Expected | Static result |
|---|---|---|
| Shift=1 | `C2M2_FEATURE_SHIFT=1`, all reads at index 1 | PASS |
| strict `<` | `bar_close < t_decision` | PASS |
| INVALID detected | `!ok_data || !strict_before`, plus `CopyIndicatorBuffers` failure | PASS |
| INVALID → NO Step4 | returns `true`, caller sets `cf_veto=true` → ladder returns before `PlaceEntry` | PASS |
| LATCH unchanged | INVALID still latched `C2M2_LATCH_INVALID`; retry cannot turn to ALLOW | PASS |
| Valid ALLOW | valid data, `veto=false` → `return false` | PASS |
| Valid VETO | valid data, `veto=true` → `return true` | PASS |
| Legacy M0/M1/M3 untouched | guards and `else` CF path byte-identical | PASS |

## Final status

```
INVALID POLICY = FAIL-CLOSED
NO FULL BACKTEST
NO M0 BACKTEST
NO THRESHOLD CHANGE
NO STRATEGY CHANGE
NO LEGACY PATH CHANGE
```
