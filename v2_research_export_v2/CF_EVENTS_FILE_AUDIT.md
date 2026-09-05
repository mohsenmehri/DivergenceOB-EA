# CF_events_m0.csv — CODE AUDIT (frozen v2_research_export_v2_cf.mq5)

Audit date: 2026-09-04 · Audited file sha256: `567b94a2bb15fbd5f41c4d9fd4b601b0f8a5370e99b17e00e95b492965e91e3b` (== frozen blob, == user's `v2_research_export_v2_cf.mq5`, 2,184,911 bytes). No code/production changes made. `check_m0_gate.py` untouched.

## CF_events_m0 generation:

- **Expected filename:** `CF_events_m0.csv` — constructed at line 19250:
  `"CF_events_m" + IntegerToString(inp_cf_mode) + ".csv"` → mode 0 ⇒ `CF_events_m0.csv`.
  This is the ONLY occurrence of `CF_events` in the entire file (grep: 1 hit). No reader, no copy, no rename anywhere.
- **Expected path:** relative filename, no subfolder ⇒ terminal's sandbox-local `MQL5\Files` of the tester agent ⇒ on your machine:
  `<MT5 Data Folder>\Tester\Agent-*\MQL5\Files\CF_events_m0.csv`
  (NOT `Common\Files`, NOT `MQL5\Files` of the main terminal when running under the Strategy Tester).
- **FileOpen flags (line 19253):** `FILE_WRITE | FILE_CSV | FILE_ANSI | FILE_SHARE_READ`, delimiter `';'`.
  No `FILE_READ` (truncates on open), **no `FILE_COMMON`** — this is the key difference vs every other export in this build, which all use `FILE_COMMON`.
- **Creation condition:** lazy, on the FIRST successful call of `CF_LogEvent()` (static handle, line 19247–19256), then header row written once (27 columns: `Event;SetupID;DecisionTime;StepFrom;StepTo;Direction;Bid;Ask;SpreadPoints;ATR;ATRPct100;ATRRegime;ADX;DIPlus;DIMinus;RSI;MACDHist;EMA200DistATR;EMA200Slope5ATR;FloatingPnLNoCost;OpenPositions;BasketBE;Session;OpenHour;Mode;Rule;FillPrice;FillTime`).
  First call in a mode-0 run = first Step-1 entry attempt inside `CreateSetupWithPrediction()` (line 19382), i.e. early in the backtest (first setup ≈ 2014-01-03) — guaranteed if ≥1 setup occurs. Rows written as `';'`-separated line + `"\n"`; `FileFlush` every 500 rows (line ~19352); handle never explicitly closed (no `FileClose` for this file anywhere; MQL5 closes at EA unload).
  Note: open failure is SILENT (no `Print`/`GetLastError` anywhere in the CF block), but the open is retried on every subsequent `CF_LogEvent` call (handle stays `INVALID_HANDLE` ⇒ retry path).
- **M0 mode behavior (inp_cf_mode=0):** `CF_LogEvent` has NO mode gate — **A/P/F rows ARE produced in M0**.
  `CF_ShouldVeto()` (line 19214) returns `false` immediately for mode ≤ 0 (line 19216: `if(inp_cf_mode <= 0) return false;`) ⇒ veto never fires in M0 ⇒ no `V` rows, `Rule` column is always `NONE` (advance decisions) or `M0` (P/F rows), `Mode` column = `0`.
- **A event:** decision/attempt to enter — two sites:
  1. Step-1 attempt (line 19382, in `CreateSetupWithPrediction`, BEFORE order send): `A; <SetupID>; StepFrom=0; StepTo=1; Rule=NONE` → expected 4,865 rows in M0 (one per setup).
  2. Ladder-advance decision (line 42433, in `ManagePositions`, after market/spread/margin gates, BEFORE order send): event `A` (or `V` if a veto fired — never in M0), `StepFrom=cur_step`, `StepTo=next` (2→5).
- **P event:** (line 42457, in `ManagePositions`) after a ladder order actually FILLS (PlaceEntry success): `P` row for the same advance; `StepFrom/StepTo` same as the A row. One `P` per successful ladder add (steps 2–5). Step-1 has no P/F (only A).
- **F event:** (line 42470, same success block, right after P) final fill record carrying `FillPrice` and `FillTime` of the same ladder add. Expected counts in M0 ≈ P rows ≈ total ladder fills (9,606 opened positions − 4,865 step-1 = ~4,741 ladder fills ⇒ ≈4,741 P + ≈4,741 F; ≈9,606 A) — sanity anchor for validation once the file is found.
- **Actual reason file is absent:** NOT a code bug and NOT a mode/config gate. The CF file is the **only** output of this build written WITHOUT `FILE_COMMON` → the Strategy Tester redirects it into the per-agent sandbox `Tester\Agent-*\MQL5\Files\` instead of `Common\Files`, where every collected export (RX_*, PositionOpenDataset, PositionOpenSummary, ReportTester xlsx, …) is written. Your output folder mirrors the FILE_COMMON exports; the sandbox file was simply never collected/copied from the agent folder. (Not verified at 100% until you search the agent folder — step below.)
- **Verification steps (no moves, no changes):** search on your machine, in order:
  1. `<MT5 Data Folder>\Tester\Agent-*\MQL5\Files\CF_events_m0.csv` (wildcard — several agent dirs may exist),
  2. `<MT5 Data Folder>\Tester\*\MQL5\Files\CF_events_m0.csv` and `Tester\Files\`,
  3. if found: copy it to your output folder only after you report back (per your instruction nothing is moved yet) — file should be ~1.5–2 MB, header + ~19k rows.
  4. If NOT found in any agent dir: the sandbox was cleaned or the open failed; then a rerun is the only path (details below), but first we would inspect the tester session's own logs.
- **Is rerun required? NO** — pending the agent-folder search. Only if the file is genuinely absent from all `Tester\Agent-*\MQL5\Files` dirs: rerun = YES, and before rerunning we report exactly what must change (nothing in strategy; the freeze would need a documented, minimal, logging-only amendment such as FILE_COMMON + failure Print — NOT done without your approval).
- **Is code change required? NO** (not for provenance; optional future diagnostic improvement only, and only with your approval + re-freeze).
- **Does M0 Gate remain blocked? YES** — gate still needs: CF_events_m0.csv + real content of M0 Labels/PO CSVs (LFS egress).

## Appendix — evidence

- File verified byte-identical to frozen build (sha256 above).
- All CF call sites: defs 19214/19242; calls 19382, 42433, 42457, 42470. `CF_ShouldVeto` M0 short-circuit line 19216.
- Single `CF_events` string occurrence (19250); no `FileCopy`, no `FileIsExist`/read-back, no OnDeinit close for this file.
- No FILE_COMMON on the CF file open; FILE_COMMON confirmed on the standard export writers (e.g. 4380/5099/6596/11909/41998/44376/46440/47152/47654 regions).

No production changes. No threshold changes. No rule changes.
