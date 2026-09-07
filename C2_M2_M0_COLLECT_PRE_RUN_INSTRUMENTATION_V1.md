# C2/M2 — M0 Shift-1 Population Collection: Pre-Run Instrumentation Review (V1)

Status: **IMPLEMENTATION COMPLETE — PRE-RUN REVIEW ONLY**
Run performed: **NONE. NO BACKTEST. NO OOS. NO THRESHOLD. NO FREEZE.**
Production mode: **UNCHANGED — M0.**

This document reports the four review items requested before any tool run:
source diff, source SHA-256, exact changes, compile requirement, and the M0-preservation invariants.

---

## 1. Provenance

| Item | Value |
|---|---|
| Branch | `arena/01a06d3c-divergenceob-ea` |
| Repo HEAD before instrumentation | `061a1bb47c5bc7768f66cc9dec291d5c5e8b7135` (C2/M2: M0 population collection feasibility v1) |
| Source file | `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` |
| Instrumented source SHA-256 | `327952595e1b1b04feedc8fb373874e7ee5624de6fbe007e97d75a3929734f9b` |
| Pre-instrumentation source SHA-256 | `128a5bc9b2df4d960b2d7632a38f247cd2de588ebaa59c1874a0f50bed479ed3` |

The M0 collection export stamps `SourceCommit = 061a1bb47c5bc7768f66cc9dec291d5c5e8b7135`.
This is the actual upstream source commit (not `29dfb15`). The exact instrumented
source SHA-256 is the one above, recorded here because a run-stamped value can
only be post-commit and would still precede the commit that contains it.

---

## 2. Source Diff (against repo HEAD `061a1bb` / SHA-256 `128a5bc9…`)

```diff
--- /tmp/baseline.mq5	2026-09-07 13:04:08.957880613 +0000
+++ v2_research_export_v2_cf/v2_research_export_v2_cf.mq5	2026-09-07 13:05:31.530297677 +0000
@@ -969,6 +969,9 @@
 input double   inp_cf_q75_atr_pct_34  = 0.0; // FROZEN DEV_Q75 of M1_ATR_PCT_PRICE on M0 decision-attempt rows StepFrom=3 (from rule_freeze.json)
 input double   inp_cf_q75_adx_34      = 0.0; // FROZEN DEV_Q75 of M1_ADX on M0 decision-attempt rows StepFrom=3 (M2 only)
 input double   inp_cf_q75_atr_pct_23  = 0.0; // FROZEN DEV_Q75 of M1_ATR_PCT_PRICE on M0 decision-attempt rows StepFrom=2 (M3 only)
+// M0 Shift-1 population collection (Threshold Derivation). Observation-only:
+// does NOT apply any C2/M2 veto and does NOT use CF_ShouldVeto/CF_LogEvent.
+input bool     inp_c2m2_collect_shift1_m0 = false;
 
 input int      inp_rsi_period        = 14;
 input int      inp_adx_period        = 14;
@@ -19343,9 +19346,15 @@
 #define C2M2_DI_ADVERSE_DELTA             0.0
 #define C2M2_STEP_FROM                    3
 #define C2M2_STEP_TO                      4
-#define C2M2_PROVENANCE_SOURCE_COMMIT     "29dfb1529293c10b364014464977a150074da25f"
+#define C2M2_PROVENANCE_SOURCE_COMMIT     "061a1bb47c5bc7768f66cc9dec291d5c5e8b7135"
 #define C2M2_PROVENANCE_SPEC_VERSION      "C2_M2_IMPLEMENTATION_REPAIR_SPEC_V1"
 #define C2M2_PROVENANCE_BUILD             "C2M2_REPAIR_V1"
+// M0 collection provenance: actual upstream source commit, NOT the 29dfb15
+// baseline. The exact instrumented-source SHA-256 is recorded in the pre-run
+// instrumentation report (C2_M2_M0_COLLECT_PRE_RUN_INSTRUMENTATION_V1.md).
+#define C2M2_M0COLLECT_SOURCE_COMMIT      "061a1bb47c5bc7768f66cc9dec291d5c5e8b7135"
+#define C2M2_M0COLLECT_SPEC_VERSION       "C2_M2_M0_POPULATION_COLLECTION_V1"
+#define C2M2_M0COLLECT_BUILD              "C2M2_M0COLLECT_V1"
 
 enum ENUM_C2M2_LATCH
 {
@@ -19365,17 +19374,18 @@
                             int feature_shift, datetime bar_open, datetime bar_close,
                             double atr, double atr_pct, double adx, double pdi, double mdi,
                             double price_ref, bool strict_before, string decision,
-                            string latch_status)
+                            string latch_status, string event_label, string rule_label,
+                            string output_file, string source_commit, string spec_version,
+                            string build)
 {
    if(setup_idx < 0 || setup_idx >= g_setup_cnt) return;
    static int fh = INVALID_HANDLE;
    if(fh == INVALID_HANDLE)
    {
-      string path = "C2M2_Step34_Decisions_m" + IntegerToString(inp_cf_mode) + ".csv";
+      string path = output_file;
       // Diagnostic export fix: write to FILE_COMMON like every other run export
       // (SafeOpenCSV uses FILE_COMMON), so the file is recovered in the result
-      // folder. Previously it opened the terminal-local non-common path and the
-      // C2/M2 evidence was not returned with the run outputs.
+      // folder.
       fh = FileOpen(path, FILE_WRITE | FILE_CSV | FILE_ANSI | FILE_SHARE_READ | FILE_COMMON, ';');
       if(fh == INVALID_HANDLE)
       {
@@ -19390,7 +19400,7 @@
          "SourceCommit;SpecVersion;Build\n");
    }
    if(fh == INVALID_HANDLE) return;
-   string row = "C2M2_DECISION;"
+   string row = event_label + ";"
       + IntegerToString(g_setups[setup_idx].setup_id) + ";"
       + IntegerToString(step_from) + ";" + IntegerToString(step_to) + ";"
       + TimeToString(t_decision, TIME_DATE | TIME_SECONDS) + ";"
@@ -19407,10 +19417,10 @@
       + (strict_before ? "true" : "false") + ";"
       + decision + ";" + latch_status + ";"
       + (is_bull ? "BUY" : "SELL") + ";"
-      + IntegerToString(inp_cf_mode) + ";M2;"
-      + C2M2_PROVENANCE_SOURCE_COMMIT + ";"
-      + C2M2_PROVENANCE_SPEC_VERSION + ";"
-      + C2M2_PROVENANCE_BUILD;
+      + IntegerToString(inp_cf_mode) + ";" + rule_label + ";"
+      + source_commit + ";"
+      + spec_version + ";"
+      + build;
    FileWriteString(fh, row + "\n");
    static int c2m2_rows = 0;
    c2m2_rows++;
@@ -19438,7 +19448,9 @@
       g_setups[setup_idx].c2m2_step34_latch_time = t_decision;
       C2M2_LogStep34Decision(setup_idx, C2M2_STEP_FROM, C2M2_STEP_TO, is_bull, tk,
          t_decision, t_server, C2M2_FEATURE_SHIFT, 0, 0,
-         0.0, 0.0, 0.0, 0.0, 0.0, 0.0, false, "INVALID", "FIRST");
+         0.0, 0.0, 0.0, 0.0, 0.0, 0.0, false, "INVALID", "FIRST",
+         "C2M2_DECISION", "M2", "C2M2_Step34_Decisions_m" + IntegerToString(inp_cf_mode) + ".csv",
+         C2M2_PROVENANCE_SOURCE_COMMIT, C2M2_PROVENANCE_SPEC_VERSION, C2M2_PROVENANCE_BUILD);
       return false;                              // fail-open, never blocks ladder
    }
 
@@ -19466,7 +19478,9 @@
       g_setups[setup_idx].c2m2_step34_latch_time = t_decision;
       C2M2_LogStep34Decision(setup_idx, C2M2_STEP_FROM, C2M2_STEP_TO, is_bull, tk,
          t_decision, t_server, C2M2_FEATURE_SHIFT, bar_open, bar_close,
-         atr, atr_pct, adx, pdi, mdi, price_ref, strict_before, "INVALID", "FIRST");
+         atr, atr_pct, adx, pdi, mdi, price_ref, strict_before, "INVALID", "FIRST",
+         "C2M2_DECISION", "M2", "C2M2_Step34_Decisions_m" + IntegerToString(inp_cf_mode) + ".csv",
+         C2M2_PROVENANCE_SOURCE_COMMIT, C2M2_PROVENANCE_SPEC_VERSION, C2M2_PROVENANCE_BUILD);
       return false;                              // fail-open, never blocks ladder
    }
 
@@ -19482,10 +19496,77 @@
    C2M2_LogStep34Decision(setup_idx, C2M2_STEP_FROM, C2M2_STEP_TO, is_bull, tk,
       t_decision, t_server, C2M2_FEATURE_SHIFT, bar_open, bar_close,
       atr, atr_pct, adx, pdi, mdi, price_ref, strict_before,
-      veto ? "VETO" : "ALLOW", "FIRST");
+      veto ? "VETO" : "ALLOW", "FIRST",
+      "C2M2_DECISION", "M2", "C2M2_Step34_Decisions_m" + IntegerToString(inp_cf_mode) + ".csv",
+      C2M2_PROVENANCE_SOURCE_COMMIT, C2M2_PROVENANCE_SPEC_VERSION, C2M2_PROVENANCE_BUILD);
    return veto;
 }
 
+// ============================================================================
+// C2/M2 M0 SHIFT-1 POPULATION COLLECTION (threshold-derivation population).
+// Observation-only. It does NOT call CF_ShouldVeto/CF_LogEvent and never
+// applies a veto. It records the first valid Event=A Step3->4 decision per
+// setup with FeatureShift=1 and strict BarCloseTime_Shift1 < T_decision.
+// ============================================================================
+bool C2M2_CollectM0Step34Decision(int setup_idx, bool is_bull, const MqlTick &tk)
+{
+   if(inp_cf_mode != 0) return false;              // M0 collection only
+   if(!inp_c2m2_collect_shift1_m0) return false;   // switch off by default
+   if(setup_idx < 0 || setup_idx >= g_setup_cnt) return false;
+   if(g_setups[setup_idx].current_step != C2M2_STEP_FROM) return false;
+
+   // Latch: only ALLOW is used here (collection invariant). Once collected,
+   // later ticks are suppressed; no re-evaluation / duplicate row.
+   if(g_setups[setup_idx].c2m2_step34_latch == C2M2_LATCH_ALLOW) return false;
+   if(g_setups[setup_idx].c2m2_step34_latch == C2M2_LATCH_VETO ||
+      g_setups[setup_idx].c2m2_step34_latch == C2M2_LATCH_INVALID) return false;
+
+   datetime t_decision = tk.time;                  // T_decision = tick.time
+   datetime t_server   = TimeCurrent();
+
+   // DEV-only guard for the derivation population. The collection file must
+   // contain only decisions inside the approved DEV window; OOS rows are never
+   // written and have no effect on M0 execution.
+   datetime t_dev_start = StringToTime("2014.01.01 00:00:00");
+   datetime t_dev_end   = StringToTime("2023.12.31 23:59:59");
+   if(t_decision < t_dev_start || t_decision > t_dev_end) return false;
+
+   int mi = TF_IDX_MAIN;                           // M1
+   if(!g_tf[mi].CopyIndicatorBuffers(MIN_BUFFER_DEPTH))
+      return false;                                // do not latch; later tick may succeed
+
+   datetime bar_open  = iTime(_Symbol, g_tf[mi].timeframe, C2M2_FEATURE_SHIFT);
+   datetime bar_close = bar_open + PeriodSeconds(g_tf[mi].timeframe);
+   double price_ref   = iClose(_Symbol, g_tf[mi].timeframe, C2M2_FEATURE_SHIFT);
+   double atr         = g_tf[mi].GetBufferValue(g_tf[mi].buffer_atr, C2M2_FEATURE_SHIFT);
+   double adx         = g_tf[mi].GetBufferValue(g_tf[mi].buffer_adx, C2M2_FEATURE_SHIFT);
+   double pdi         = g_tf[mi].GetBufferValue(g_tf[mi].buffer_plus_di, C2M2_FEATURE_SHIFT);
+   double mdi         = g_tf[mi].GetBufferValue(g_tf[mi].buffer_minus_di, C2M2_FEATURE_SHIFT);
+   double atr_pct     = (price_ref > 0.0 && atr > 0.0) ? (atr * 100.0 / price_ref) : 0.0;
+
+   bool strict_before = (bar_close < t_decision);
+   bool ok_data = (C2M2_IsValidNumber(price_ref) && price_ref > 0.0 &&
+                   C2M2_IsValidNumber(atr)     && atr  > 0.0 &&
+                   C2M2_IsValidNumber(adx)     && adx >= 0.0 &&
+                   C2M2_IsValidNumber(pdi)     && pdi >= 0.0 &&
+                   C2M2_IsValidNumber(mdi)     && mdi >= 0.0);
+
+   // Only valid rows become the population. Invalid/strict-before-failing
+   // ticks are ignored and no latch is set, so a later valid tick can be used.
+   if(!ok_data || !strict_before)
+      return false;
+
+   g_setups[setup_idx].c2m2_step34_latch      = C2M2_LATCH_ALLOW;
+   g_setups[setup_idx].c2m2_step34_latch_time = t_decision;
+   C2M2_LogStep34Decision(setup_idx, C2M2_STEP_FROM, C2M2_STEP_TO, is_bull, tk,
+      t_decision, t_server, C2M2_FEATURE_SHIFT, bar_open, bar_close,
+      atr, atr_pct, adx, pdi, mdi, price_ref, strict_before,
+      "ALLOW", "FIRST",
+      "A", "M0_COLLECT", "C2M2_M0_Shift1_Population_DEV.csv",
+      C2M2_M0COLLECT_SOURCE_COMMIT, C2M2_M0COLLECT_SPEC_VERSION, C2M2_M0COLLECT_BUILD);
+   return false;                                  // never vetoes
+}
+
 bool CreateSetupWithPrediction(bool is_bull, double &ent[], double &be[], double &tp[], double dist)
 {
    if(!IsMarketOpen()|| !IsSpreadOK()) return false;
@@ -42590,9 +42671,17 @@
       // =====
       // C2/M2 Repair V1 (Owner-approved D1-D12, d_DI=0): Step3->4 M2 is isolated
       // from the shared CF_ShouldVeto/CF_LogEvent path so M0/M1/M3 are unchanged.
+      // M0 Shift-1 population collection also uses a separate observation-only
+      // path and never applies a veto.
       bool cf_veto   = false;
+      bool is_m0collect = (inp_cf_mode == 0 && inp_c2m2_collect_shift1_m0 &&
+                           cur_step == 3 && next == 4);
       bool is_c2m2   = (inp_cf_mode == 2 && cur_step == 3 && next == 4);
-      if(is_c2m2)
+      if(is_m0collect)
+      {
+         cf_veto = C2M2_CollectM0Step34Decision(idx, is_bull, tk);
+      }
+      else if(is_c2m2)
       {
          cf_veto = C2M2_Step34ShouldVeto(idx, is_bull, tk);
       }

```

---

## 3. Exact Changes

1. **New input (default off)**
   - `input bool inp_c2m2_collect_shift1_m0 = false;`
   - Placed with the CF inputs. Default `false` keeps the M0 path byte-for-byte equivalent at the callsite branch level.

2. **C2/M2 shared logger parameterized**
   - `C2M2_LogStep34Decision(…)` now accepts `event_label`, `rule_label`, `output_file`,
     `source_commit`, `spec_version`, `build`.
   - Existing M2 callsites pass `event=C2M2_DECISION`, `rule=M2`,
     output `C2M2_Step34_Decisions_m<mode>.csv`, and the real source commit.
   - The M2 export behavior is unchanged; only the provenance string changed from
     placeholder `29dfb15` to the actual parent commit.

3. **New observation-only collector**
   - `C2M2_CollectM0Step34Decision(setup_idx, is_bull, tk)`.
   - Runs only when `inp_cf_mode == 0 && inp_c2m2_collect_shift1_m0 && current_step==3 && next==4`.
   - Reads all features from `FeatureShift = 1` (M1 buffer index 1):
     `bar_open = iTime(...,1)`, `bar_close = bar_open + PeriodSeconds(M1)`,
     `price_ref = Close[1]`, `ATR[1]`, `ADX[1]`, `+DI[1]`, `-DI[1]`,
     `ATRPct = ATR*100/Close[1]`.
   - `T_decision = tick.time` and `DecisionServerTime = TimeCurrent()`.
   - `strict_before = (bar_close < t_decision)`.
   - DEV-only guard: only decisions in `2014-01-01 00:00:00 … 2023-12-31 23:59:59`
     can be collected.
   - First-valid latch: once a valid row is emitted the setup latch is set to
     `C2M2_LATCH_ALLOW`; later ticks for the same Step 3→4 attempt are suppressed.
     Invalid data or `strict_before == false` simply returns without latching so a
     later valid tick can be captured.

4. **Collection CSV**
   - New export: `C2M2_M0_Shift1_Population_DEV.csv` (FILE_COMMON).
   - Header already contains the required fields plus the existing `C2M2_Decision`
     and `LatchStatus` observability columns:
     `Event,SetupID,StepFrom,StepTo,T_decision,DecisionServerTime,FeatureShift,BarOpenTime_Shift1,BarCloseTime_Shift1,ATR_Shift1,ATRPct_Shift1,ADX_Shift1,DIPlus_Shift1,DIMinus_Shift1,PriceRef_Shift1,strict_before_check,C2M2_Decision,LatchStatus,Direction,Mode,Rule,SourceCommit,SpecVersion,Build`.
   - Row values for collection: `Event=A`, `Mode=0`, `Rule=M0_COLLECT`,
     `C2M2_Decision=ALLOW`, `LatchStatus=FIRST`, `Direction=BUY/SELL`.

5. **Call-site routing**
   - `ManagePositions` now checks `is_m0collect` before `is_c2m2`.
   - Collection path calls only `C2M2_CollectM0Step34Decision(…)`.
   - It does **not** call `CF_ShouldVeto` or `CF_LogEvent`.
   - It always returns `false`, so `cf_veto` remains `false` and execution continues
     to `PlaceEntry` exactly as M0 does.

6. **Provenance**
   - Removed the hard-coded `29dfb1529293c10b364014464977a150074da25f`.
   - M2 and M0-collection exports now use the real parent source commit
     `061a1bb47c5bc7768f66cc9dec291d5c5e8b7135`.
   - Build/version labels: `C2_M2_IMPLEMENTATION_REPAIR_SPEC_V1` /
     `C2M2_REPAIR_V1` (M2) and `C2_M2_M0_POPULATION_COLLECTION_V1` /
     `C2M2_M0COLLECT_V1` (M0 collection).

---

## 4. Compile Requirement

- **No MQL5/MetaEditor compiler is available in this sandbox.** The compile gate
  was NOT performed and remains required in MetaEditor before any run.
- Static checks performed on the edited source:
  - All four callsites and the signature of `C2M2_LogStep34Decision` agree on 25 arguments.
  - Parenthesis balance unchanged from baseline (0 net delta).
  - Brace balance delta matches baseline (no new structural imbalance).
  - No references to `CF_ShouldVeto`/`CF_LogEvent` were added to the collection path.
- Required compile in MetaEditor on the same terminal/broker configuration used
  for the actual collection run (MQL5, Windows/MT5 build compatible with the
  existing indicators).

---

## 5. M0-Preservation Invariants

1. **Default off**: with `inp_c2m2_collect_shift1_m0=false`, `is_m0collect` is
   always false and the exact baseline branch order applies.
2. **No veto**: the collector unconditionally returns `false`; `cf_veto` stays
   `false`; the ladder continues to `PlaceEntry` exactly as M0 does.
3. **No M0 rule/threshold change**: the collection path never reads
   `inp_cf_q75_atr_pct_34`, `inp_cf_q75_adx_34`, or `inp_cf_q75_atr_pct_23`.
4. **No M0 management change**: the only EA state it writes is the existing
   `c2m2_step34_latch`/`c2m2_step34_latch_time` fields, which are read only by
   the M2 module (`C2M2_Step34ShouldVeto`, guarded by `inp_cf_mode==2`).
   In a mode-0 collection run these fields are inert for execution.
5. **Isolated export**: it writes only `C2M2_M0_Shift1_Population_DEV.csv`
   (FILE_COMMON, same pattern as existing exports). It does not alter any other
   export or order path.
6. **Expected diagnostic difference**: because the user requires the collection
   path NOT to use `CF_ShouldVeto`/`CF_LogEvent`, a collection run
   (`inp_cf_mode=0`, switch=true) will not write `CF_events_m0.csv`. This is a
   diagnostic-export difference only, not an execution/management difference.
7. **Production unchanged**: production remains **M0**; no new Rule freeze, no
   threshold change, no OOS use.

---

## 6. Stop Point

No backtest was run. The next action, if approved, is a MetaEditor compile +
DEV-only (`2014-01-01 00:00:00`→`2023-12-31 23:59:59`) collection run with
`inp_cf_mode=0` and `inp_c2m2_collect_shift1_m0=true`. Actual population count will
be reported after the run; **366 is not an invariant** and any deviation will be
explained from the collected evidence.
