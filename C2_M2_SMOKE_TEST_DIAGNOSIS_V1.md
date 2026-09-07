# C2/M2 Smoke-Test Diagnosis V1

**Date:** 2026-09-07
**Target branch analyzed:** `agent-regression-data-real` @ `e716f7b608232b8955588c3447fe9c78f0866a07`
**Output folder analyzed:** `v2_research_export_v2_cfv2/`
**Production:** M0 (unchanged)

---

## Root Cause

Two independent causes explain why `C2M2_Step34_Decisions_m2.csv` was not produced:

### Root cause 1 — Run was M0 (primary)
The run executed with `inp_cf_mode=0`:
- `20260907.log` (decoded from UTF‑16LE): `inp_cf_mode=0`
- `inp_cf_q75_atr_pct_34=0.0`
- `inp_cf_q75_adx_34=0.0`
- The repaired C2/M2 module is gated on `inp_cf_mode == 2` (`C2M2_Step34ShouldVeto` guard, and the caller `is_c2m2 = (inp_cf_mode == 2 && cur_step == 3 && next == 4)`).
- With `inp_cf_mode == 0`, `C2M2_Step34ShouldVeto` and `C2M2_LogStep34Decision` are **never reached** → no `C2M2_Step34_Decisions_m2.csv`.

Repair file evidence (`v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` at our `arena/01a06d3c-divergenceob-ea` branch):
- `bool C2M2_Step34ShouldVeto` → `if(inp_cf_mode != 2) return false;`
- `ManagePositions` → `bool is_c2m2 = (inp_cf_mode == 2 && cur_step == 3 && next == 4);`

### Root cause 2 — Non-common export path (latent defect)
The C2/M2 export opened into the **terminal-local, non-common** MQL5 Files path, unlike every output file that appeared in `v2_research_export_v2_cfv2/`:

- `C2M2_LogStep34Decision` originally:
  ```cpp
  fh = FileOpen(path, FILE_WRITE | FILE_CSV | FILE_ANSI | FILE_SHARE_READ, ';');
  ```
  No `FILE_COMMON`.
- Every recovered CSV/text in `v2_research_export_v2_cfv2/` is created via `SafeOpenCSV` / common-file functions which use `FILE_COMMON` (e.g. `SafeOpenCSV` → `FileOpen(filename, FILE_WRITE | FILE_BIN | FILE_COMMON)`).
- The previous shared CF log (`CF_LogEvent`) has the same non-common open, and indeed `CF_events_m0.csv` is **also absent** from `v2_research_export_v2_cfv2/`; it was only present in the older `v2_research_export_v2_cf/` output.
- Conclusion: even with `inp_cf_mode=2`, the C2/M2 evidence file would be written to a non-common local path and not recovered in the result folder. This is a diagnostic/export bug, not an entry/strategy/threshold rule issue.

### Root cause 3 — Binary provenance (secondary blocker)
The log says it tested `Experts\v2_research_export_v2_cf-v2.ex5`, but:
- No `v2_research_export_v2_cf-v2.mq5` is present in commit `e716f7b`.
- The repository's repaired source file is `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` (on `arena/01a06d3c-divergenceob-ea`, commit `8681ec6`), and it is **not** present in `e716f7b`.
- Therefore the source used to compile `v2_research_export_v2_cf-v2.ex5` is **not verifiable from the commit**.

---

## Evidence

### 1. Run is M0 / C2/M2 not enabled
`v2_research_export_v2_cfv2/20260907.log` (UTF‑16LE decoded):
```
inp_cf_mode=0
inp_cf_q75_atr_pct_34=0.0
inp_cf_q75_adx_34=0.0
testing of Experts\v2_research_export_v2_cf-v2.ex5 from 2014.01.01 00:00 to 2014.03.02 00:00
```

### 2. No C2/M2 decision/export in output
`git ls-tree -r e716f7b | grep -i c2m2` → **empty**.
`git grep -i -n 'C2M2' e716f7b -- .` → **empty**.

### 3. Run outputs standard but no C2/M2 / CF event file
Files in `v2_research_export_v2_cfv2/`:
- `PositionOpenDataset_XAUUSD.csv` (present)
- `RX_...Features.csv`, `RX_...Labels.csv`, `RX_...SetupLifecycleEvents.csv` (present)
- `MarketEdge_*.csv` (present)
- `PositionOpenSummary_XAUUSD.txt` (present)
- `Stats_XAUUSD.txt` (present)
- `SetupID_XAUUSD.txt` (present)
- `ReportTester-91306235.xlsx` (present)
- `C2M2_Step34_Decisions_m2.csv` → **NOT present**
- `CF_events_m0.csv` → **NOT present**

### 4. Run counts (population was M0, tiny)
`PositionOpenDataset_XAUUSD.csv`:
- Step1 = 60, Step2 = 37, Step3 = 15, Step4 = 3, Step5 = 2
- All `IndicatorShiftUsed=0`

`RX_...Features.csv`:
- 60 rows, `M1_Shift=1`, `M1_Forming=0`, `M1_Valid=1` (this is the existing Step1 RX snapshot, not C2/M2).

`RX_...SetupLifecycleEvents.csv`:
- `STEP1_OPEN=60`, `STEP2_OPEN=37`, `STEP3_OPEN=15`, `STEP4_OPEN=3`, `STEP5_OPEN=2`.
- No `C2M2_DECISION` event.

### 5. Export path discrepancy
- `SafeOpenCSV` uses `FILE_COMMON` → all standard outputs are produced in the recovered output folder.
- `C2M2_LogStep34Decision` and `CF_LogEvent` use `FileOpen(..., FILE_WRITE | FILE_CSV | FILE_ANSI | FILE_SHARE_READ, ';')` (no `FILE_COMMON`).
- Result: neither `CF_events_m0.csv` nor `C2M2_Step34_Decisions_m2.csv` appears in the recovered folder.

---

## Code Change (minimal, diagnostic/export only)

File: `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5`
Changed function: `C2M2_LogStep34Decision`

Before:
```cpp
fh = FileOpen(path, FILE_WRITE | FILE_CSV | FILE_ANSI | FILE_SHARE_READ, ';');
if(fh != INVALID_HANDLE)
   FileWriteString(fh, header...);
```

After:
```cpp
fh = FileOpen(path, FILE_WRITE | FILE_CSV | FILE_ANSI | FILE_SHARE_READ | FILE_COMMON, ';');
if(fh == INVALID_HANDLE)
{
   Print("C2M2_EXPORT_OPEN_FAILED|", path);
   return;
}
FileWriteString(fh, header...);
```

This is the **only** source change. It:
- Adds `FILE_COMMON` so the C2/M2 evidence file is written to the same common Files area recovered by the run result folder (matching `SafeOpenCSV`).
- Adds a `C2M2_EXPORT_OPEN_FAILED|` diagnostic log line if the file cannot be opened.
- Does **not** change strategy, entry logic, thresholds, rule definition, M0/M1/M3 behavior, `FeatureShift=1`, `PriceRef=Close[1]`, strict-before check, or latch semantics.

### Source SHAs
| Item | SHA-256 |
|---|---|
| Baseline repaired source (before this diagnostic fix) | `842eec36547538b719bd8798a766dd535423196bb1a519185dac904226c61f07` |
| New repaired source (after diagnostic fix) | `128a5bc9b2df4d960b2d7632a38f247cd2de588ebaa59c1874a0f50bed479ed3` |
| Source line count | 52,114 |

---

## Compile Result

**NOT EXECUTED — BLOCKED.** No MetaEditor / MQL5 compiler (`metaeditor`, `runmeta`, `mql5`, `wine`) is available in this sandbox. Static review only:
- Brace balance: raw `{` = `}`, repaired region `{` = `}`.
- Diff is a single localized hunk inside `C2M2_LogStep34Decision`.
- No identifiers/function signatures touched.
- `FILE_COMMON` is already used throughout the same file, so the API use is consistent.

---

## Exact Smoke-Test Inputs

The following must be set to make the C2/M2 isolated path run:

| Input | Value | Purpose |
|---|---|---|
| EA file | `v2_research_export_v2_cf.mq5` | actual repaired source (the committed file), compiled to `.ex5`; do not use an unverifiable renamed `.ex5` |
| Symbol | `XAUUSD` | same as regression population |
| Timeframe | `M1` | EA is chart-locked to M1 |
| Date range | e.g. `2014.01.01 00:00` → `2014.03.02 00:00` | contains 15 Step3 / 3 Step4 in the M0 run |
| `inp_cf_mode` | **2** | activates the isolated C2/M2 path (M0/M1/M3 are left untouched) |
| `inp_cf_q75_atr_pct_34` | keep as-is / **0.0** | no threshold derivation; with 0.0, gate records `ALLOW` rows but still validates the full decision/latch/export path |
| `inp_cf_q75_adx_34` | keep as-is / **0.0** | same as above |
| Testing mode | real ticks / strategy tester | current output was a tester run |
| Visual/testing | no | not needed for smoke diagnostics |

Important: the smoke test must compile **the committed repaired source**. If the binary is named `v2_research_export_v2_cf-v2.ex5`, it must be compiled from this exact `.mq5` (renaming is acceptable for the binary, but the source provenance must be recorded) and the run must still have `inp_cf_mode=2`.

---

## Expected Output Files

After a correct `inp_cf_mode=2` run over the same short range:

| File | Expected content |
|---|---|
| `C2M2_Step34_Decisions_m2.csv` | 3 decision rows (one per 3→4 in the short range) with `FeatureShift=1`, `strict_before_check=true`, `C2M2_Decision=ALLOW` (with threshold 0.0), `LatchStatus=FIRST`, and all approved evidence fields |
| `PositionOpenDataset_XAUUSD.csv` | standard Step counts (unchanged from M0: Step4=3; `IndicatorShiftUsed` remains 0) |
| `RX_...SetupLifecycleEvents.csv` | `STEP4_OPEN=3`, no `C2M2_DECISION` in RX events (C2/M2 decision is separate) |
| `20260907.log` | must contain `inp_cf_mode=2`; if the export fails it must contain `C2M2_EXPORT_OPEN_FAILED|C2M2_Step34_Decisions_m2.csv` |

---

## Remaining Blockers

1. **Compile gate not executed here** — must be run in real MT5/MetaEditor.
2. **Run must be actual mode=2** — previous run was M0 and therefore not a C2/M2 test.
3. **Binary provenance unverifiable** — the previous run used `v2_research_export_v2_cf-v2.ex5`, but no corresponding source exists in `e716f7b`; compile from the committed repaired source and record its SHA.
4. **Threshold placeholders remain 0.0** — with no thresholds set, the smoke test proves the Shift-1/latch/export pipeline but records `ALLOW` rows (no veto). A true veto/freeze still requires authorized threshold derivation (not performed here).
5. **Historical Shift-0 data must never be relabeled** — this fix does not alter historical data.

---

## Status

- No new threshold derived.
- No rule definition changed.
- No M0/M1/M3 behavior changed.
- No Freeze / OOS / TRUE_FORWARD / backtest run.
- Production remains **M0**.
- The only code change is the **C2/M2 diagnostic export path fix** (`FILE_COMMON` + open-failure log).
