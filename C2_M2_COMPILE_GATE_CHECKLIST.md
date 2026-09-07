# C2/M2 — Compile Gate Checklist

**Purpose:** precise, auditable compile procedure for the repaired C2/M2 source, to be executed by the Owner in a real MT5/MetaEditor environment.
**Artifact:** `C2_M2_COMPILE_GATE_CHECKLIST.md`
**Production:** M0 (unchanged). No source change, no rule change, no threshold change, no Backtest/Smoke Run/Evidence Run/OOS/TRUE_FORWARD/Freeze/Production change.

---

## 1. Exact `.mq5` file to compile

| Item | Value |
|---|---|
| Repository path | `v2_research_export_v2_cf/v2_research_export_v2_cf.mq5` |
| File name | `v2_research_export_v2_cf.mq5` |
| Base commit | `29dfb1529293c10b364014464977a150074da25f` |
| Expected source SHA-256 | `842eec36547538b719bd8798a766dd535423196bb1a519185dac904226c61f07` |
| Source line count | 52,106 |
| Content | Repair V1: isolated C2/M2 Step3→4 Shift-1 latched gate + `C2M2_Step34_Decisions_*` evidence export |

Do **not** compile:
- `v2.mq5`
- `v2_research_export_v2/v2_research_export_v2.mq5`
- `v2_research_export_v2-new/v2_research_export_v2.mq5`

Only the CF source file above is the repaired implementation.

---

## 2. Expected source SHA verification (before compile)

Windows PowerShell (or preferred shell at the exact file):

```powershell
Get-FileHash -Algorithm SHA256 "path\to\v2_research_export_v2_cf\v2_research_export_v2_cf.mq5"
```

Expected output SHA-256:

```
842eec36547538b719bd8798a766dd535423196bb1a519185dac904226c61f07
```

If this does **not** match, stop and report a source mismatch; do not compile.

---

## 3. Placement in MT5

Recommended MetaTrader 5 path:

```
C:\Users\<USER>\AppData\Roaming\MetaQuotes\Terminal\<INSTANCE_HASH>\MQL5\Experts\v2_research_export_v2_cf\v2_research_export_v2_cf.mq5
```

Notes:
- `<INSTANCE_HASH>` is the terminal instance folder under `...\Terminal\`.
- Use `MetaTrader 5 → File → Open Data Folder` to find the correct `MQL5\Experts` path.
- Any `Experts` subfolder is acceptable for compilation because the file has no project-relative includes.
- Do **not** place it in `MQL5\Scripts` or `MQL5\Indicators`; it is an Expert Advisor (`.mq5` EA) with `#include <Trade\Trade.mqh>`.
- The .mq5 file must be copied byte-for-byte from the repository; if the file is placed at the repository path, copy it to the data-folder path above.

---

## 4. Dependencies / includes

| # | Dependency | Source line | Required? |
|---|---|---|---|
| 1 | `#include <Trade\Trade.mqh>` | line 15 | **Yes** — MetaTrader 5 standard include |
| 2 | MQL5 standard language / standard functions (`MqlTick`, `TimeCurrent`, `FileOpen`, `CopyBuffer`, `iATR`, `iADX`, `iClose`, `iTime`, `PeriodSeconds`, `MathIsValidNumber`, etc.) | built-in | **Yes** |
| 3 | `stdlib.mqh`, `Arrays.mqh`, custom library | — | **No** |
| 4 | External DLL | — | **No** |
| 5 | Broker-specific library | — | **No** |

No custom or third-party include is required. `Trade\Trade.mqh` is bundled with MetaEditor.

---

## 5. MetaEditor / environment settings

| Setting | Required value | Rationale |
|---|---|---|
| Platform | MetaTrader 5 (MT5) | EA is MQL5, not MQL4 |
| MetaEditor version | Current MT5 build (record the exact build number) | reproducibility |
| 64-bit terminal | Recommended | matches standard MT5 builds |
| Source encoding | UTF-8 (source contains Persian comments) | prevents decoding errors in comments/strings |
| Line endings | Keep the file as delivered (LF/CRLF both acceptable for compiler) | avoid source change |
| Project | None (single file) | no `.mqproj` needed |
| Chart timeframe | M1 | not a compile setting; only relevant at run time |
| Input reset / backtest settings | not touched | this is compile-only |

Do **not** run the Strategy Tester from this checklist. This is compile-only.

---

## 6. Compile command / procedure

Recommended procedure (MetaEditor GUI):

1. Open MetaEditor via MetaTrader 5 (or `MetaEditor64.exe`).
2. `File → Open` → `v2_research_export_v2_cf.mq5`.
3. Verify the source SHA-256 matches §2 before pressing Compile.
4. `Compile → Compile` (keyboard: `F7`), or toolbar Compile button.
5. Wait for the compilation result in the `Toolbox → Errors/Warnings` panel.
6. **Do not** press `Run`, `Test`, `Backtest`, or `Optimize`.

Optional command-line equivalent (if a `MetaEditor64.exe` CLI is available on the Owner's machine):

```
MetaEditor64.exe /compile:"<path>\v2_research_export_v2_cf.mq5"
```

Note: MetaEditor CLI path/build availability varies by installation; if CLI is unavailable, the GUI `F7` procedure is the authoritative method.

---

## 7. Minimum pass criteria

### 7.1 Errors

```
Errors = 0
```

Any `Error` line in the `Toolbox` output is **FAIL**.

### 7.2 Warnings — list separately

| Warning type | Required evaluation |
|---|---|
| Pre-existing warnings outside C2/M2 area | Record; must be unchanged from the baseline source if such warnings exist; do not fail on them unless tied to this repair |
| Warnings inside `C2/M2` new block (lines 19337–19485) | **Severity check required** — must be reviewed one-by-one |
| Warnings inside `ManagePositions` branch (lines 42580–42597) | **Severity check required** |
| `return value ... not used` | Must be reviewed; generally non-blocking |
| `implicit conversion` / `possible loss of data` | Must be reviewed; if from C2/M2 new code, treat as **pending/fail unless justified** |
| `variable declared but not used` | Must be reviewed; treat as minor |
| `function not used` | If it refers to `C2M2_*` functions and mode never calls them, not a compile failure; report as informational |

A compile is **PASS** only if:
- Errors = 0,
- no new critical warning is introduced by the C2/M2 repair block,
- all warnings are separately documented with line numbers and justification.

If the compiler reports any warning that refers to the C2/M2 new code, the Owner must report it in the compile log and it must be classified before evidence validation.

---

## 8. Artifacts/logs to store for an auditable compile

Save the following **without modifying the source**:

| Artifact | Required |
|---|---|
| Source `.mq5` file (post-compile, byte-identical to commit) | yes |
| Compiled `.ex5` file (`v2_research_export_v2_cf.ex5`) | yes |
| SHA-256 of source `.mq5` | yes |
| SHA-256 of compiled `.ex5` | yes |
| MetaEditor output log (e.g. `MetaEditor.log`, or full `Toolbox → Errors/Warnings` export) | yes |
| MetaTrader/MetaEditor release version and build number | yes |
| Terminal data-folder path used | yes |
| Compile date/time (UTC + local) | yes |
| Error count | yes |
| Warning list with line numbers and warning type | yes |
| Note whether the source SHA matched §2 before compile | yes |
| Evidence that no Strategy Tester / Backtest / Smoke run was started during compile | yes |

Suggested log filename:

```
C2M2_COMPILE_LOG_V1_<date>.log
```

Store alongside the repository artifact and include in any future evidence audit.

---

## 9. On compile failure

If compile fails:

1. **Do not modify the source automatically.**
2. Record the **exact** compiler output:
   - error code/text
   - line/column
   - warning context (if any)
   - MetaEditor version
   - source SHA-256 at time of failure (to prove no source drift)
3. Classify each error:
   - Syntax error (e.g. duplicate identifier, missing semicolon, unclosed brace)
   - Type/usage error (e.g. reference/array usage not supported)
   - Include error (e.g. `Trade\Trade.mqh` not found)
   - Encoding error
   - Unexpected pre-existing error
4. Map errors to the suspected code area:
   - `CSetupTracker` latch fields (lines 12622–12623, 12661–12662)
   - New `C2/M2` module (lines 19337–19485)
   - `ManagePositions` branch (lines 42580–42597)
5. **Only propose** a correction in the report; do not apply it.
6. Stop after reporting the failure and proposed correction. Do NOT run compile again until Owner reviews.

Proposed-correction format:

```
Error:
Line:
Suggested fix (description only):
Risk if applied:
Files/files to change:
```

---

## 10. Final reported values (to be filled by Owner after compile)

| Field | Value |
|---|---|
| Source SHA-256 checked | `842eec36547538b719bd8798a766dd535423196bb1a519185dac904226c61f07` |
| MetaEditor version | __ |
| MT5 build | __ |
| Errors | __ |
| Pass/Fail | __ |
| Compile log path | __ |
| `.ex5` SHA-256 | __ |
| Compile date/time | __ |
| Warnings reviewed | __ |

---

**End of checklist. No source change, no rule change, no threshold change, no backtest/smoke/evidence run, no OOS/TRUE_FORWARD, no freeze, no production change.**
