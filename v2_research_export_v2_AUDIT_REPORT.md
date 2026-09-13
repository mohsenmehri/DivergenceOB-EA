# Full Code Audit — `v2_research_export_v2/v2_research_export_v2.mq5` (DivergenceOB-EA)

Audit date: 2026-09-04 · Auditor method: full line-by-line source sweep (see SOURCE SWEEP STATUS) + two runtime call-chain passes, all claims below are based on actual code text (dumps with exact line numbers), never on comments alone.

---

## 1. Access & object verification

| Property | Value |
|---|---|
| Repository | `mohsenmehri/DivergenceOB-EA`, branch `main`, file `v2_research_export_v2/v2_research_export_v2.mq5` |
| Size (pre-patch) | 51,733 lines; line endings LF in the git blob (earlier audit notes saying CRLF referred to a local copy) |
| SHA-256 | `195cb151e9fd7e09797d221c097066f28668b756c583bbf77e60a373a4c534c3` |
| Other version checked | `v2.mq5` (60,277 lines, different version) — NOT the audit target; none of its findings are applied here |
| Checked-in artifacts in same folder | LFS pointer stubs (not source) — irrelevant to the audit |

Full source access: **VERIFIED — every source line of the file was read** (raw full-line dumps; only blank/comment-only/string-literal-emission lines omitted from displays, all logic lines shown; exact coverage ledger in `SOURCE SWEEP STATUS`). No stubs, no `#include` of missing code, no truncated functions: every function body is present and closed. File compiles structurally (braces/parens balanced in every function read).

---

## 2. High-level architecture (code-evidenced)

- Runtime heart: `OnTick` 46,360 → new-bar branch → `ProcessNewBar()` 46,663 (`DetectPivots` 46,691 → `DetectOrderBlocks` 46,697 → `CheckDivergence` 46,698) → every tick `CheckLiveThreePivotDivergence()` 46,406 → `EvaluateArmedDivergencePipeline()` 46,407 → market-open edge `CheckPendingEntries()` 46,411 → `ManageSetups()` 46,415.
- `OnTimer` 46,444 (timer registered in `OnInit`: `EventSetTimer(MAINTENANCE_INTERVAL)` line 44,563; killed 44,873): autosave + TRUE_FORWARD forward validation.
- `OnTrade` 46,475 (2 s throttle, `HistorySelect` 1 h) → `ProcessTradeEventsImproved` 46,544 → `HandleExternalClose` 46,580.
- Trading chain: divergence detect → arm 16,829/17,295/17,455 → EAP gate 16,943 → `FireSignal` 17,903 (market order via `PlaceEntry` 19,307) → setup 19,203 → step scaling `ManagePositions` 42,141 → exits/finalize (~20,8xx) → research recording (46,609–46,624).
- Research stack: RiskLab (21,0xx), Stage 2B models (25,7xx–34,8xx), Stage 2C/2D overlay & forward validation (38,3xx–41,8xx), frozen-package loaders (40,177–40,360), analysis/CSV/HTML exporters (23,2xx–25,6xx, 46,2xx+, 48,7xx+).

---

## 3. DIV3 Triple Divergence — the three modes (highest-priority area)

Enums/inputs: `ENUM_DIV3_SIGNAL_MODE { DIV3_CONFIRMED_ONLY=0, DIV3_LIVE_P3_ONLY=1, DIV3_DUAL_MODE=2 }` (line 649); `input ENUM_DIV3_SIGNAL_MODE inp_div3_signal_mode = DIV3_DUAL_MODE;` (line 875).

### 3.1 Mode separation — code paths

| Mode | Confirmed stream (`CheckDivergence` 17,060) | Live stream (`CheckLiveThreePivotDivergence` 17,352) | Net effect |
|---|---|---|---|
| `DIV3_CONFIRMED_ONLY` | ACTIVE (only gated 17,068–17,077 by active-setup/session) | disabled at 17,355 `if(inp_div3_signal_mode == DIV3_CONFIRMED_ONLY) return;` | Confirmed-pivot trading only. |
| `DIV3_LIVE_P3_ONLY` | disabled at 17,067 `if(inp_div3_signal_mode == DIV3_LIVE_P3_ONLY) return;` (inside `CheckDivergence`, before any arming) | ACTIVE (display/arm) | **No tradable signal can ever be produced** (see finding D-1). |
| `DIV3_DUAL_MODE` | ACTIVE | ACTIVE | Confirmed trading + live display/geometry. **No double trade** (see finding D-2). |

### 3.2 Findings

---
#### 🟡 D-1 (severity: MEDIUM — user-facing function mismatch, needs verification of intent)
(1) Severity: 2/5 functional (not money-losing: produces no trades at all).
(2) Function: `CheckLiveThreePivotDivergence` / `EvaluateArmedDivergencePipeline` / `CheckDivergence`.
(3) Lines: 17,067 · 16,945 · 17,455 (mode arithmetic 649/875).
(4) Snippet:
```mql5
// 17067  (CheckDivergence)
   if(inp_div3_signal_mode == DIV3_LIVE_P3_ONLY) return;
// 16945  (EvaluateArmedDivergencePipeline, first executable line)
   if(!g_armed_divergence.active || !g_armed_divergence.confirmed) return;
// 17455  (end of live detector)
   ArmThreePivotDivergence(bull,lp1,lp2,lp3,false,"LIVE_P3_DISPLAY_ONLY");
```
(5) Behavior: In `LIVE_P3_ONLY`, the confirmed stream returns at 17,067 so no record with `confirmed=true` is ever armed. The live stream builds a forming-bar “pivot” (P3 = current bar low/high 17,364–17,365, live RSI via `DIV3_GetLiveRSI` 17,361 after a buffer refresh 17,359) and arms a record with `confirmed=false` (17,455). The only consumer that opens positions, EAP, refuses any non-confirmed record at its first line 16,945. Nothing else reads the armed record (only `Reset`/arm sites exist; verified by grep).
(6) Why it is a problem: a user selecting `LIVE_P3_ONLY` from the input (875) expects live-entry behavior (the enum comment 647–648 says “LIVE_P3 uses two confirmed pivots plus the forming current-bar P3”), but the EA silently never trades in this mode — it only draws dotted lines/labels (`DrawLine/DrawLabel` 17,458–17,462). Silent no-op = trust hazard.
(7) Classification: none of look-ahead/leak/wrong-entry/wrong-exit — it is a **functionality/intent mismatch**.
(8) Example: mode = 1, a live triple divergence forms at 09:35 on M5: label “LIVE P3 ARMED” drawn, `live_armed++`, then the record sits untouched; EAP returns at 16,945 forever. Position count stays 0 across the whole backtest.
(9) Fix: either (a) re-document the mode as display/research-only in the input description and startup Print, or (b) implement live consumption deliberately (intrabar entry with explicit repaint warning), or (c) make `LIVE_P3_ONLY` fall back to `CONFIRMED_ONLY` with a warning.
(10) Effect on prior research: any backtest run in `LIVE_P3_ONLY` has **zero trades and zero RiskLab rows**; such results must not be read as “live mode works”. In `DUAL` (the default), the live stream also contributes zero trades, so DUAL results equal CONFIRMED results plus extra CSV geometry fields — research built on DUAL outputs is not contaminated by intrabar prices.

---
#### 🟢 D-2 — “DUAL_MODE double-trades the same geometry” — NOT POSSIBLE (previously suspected; now resolved with code)
(1/2/3) `ArmThreePivotDivergence` 16,829–16,856; single global slot `g_armed_divergence` (declared ~12,964).
(4) Snippet (16,832–16,854):
```mql5
   if(g_armed_divergence.active && g_armed_divergence.confirmed && !confirmed)
      return;
   if(g_armed_divergence.active && g_armed_divergence.confirmed && confirmed)
   {  RP_SetProfileDecision(...); ... }            // supersede old confirmed
   ...
   g_armed_divergence.Reset();
   g_armed_divergence.active= true;
   g_armed_divergence.confirmed=confirmed;
```
(5/6) The arm state is a single slot. A live (non-confirmed) arm can never overwrite an armed confirmed record (16,832), can only occupy an empty slot, and — D-1 — is never consumable. The confirmed arm can only be created from closed-bar pivots (`s0>0…s2>0` 17,104; newest pivot shift must be ≥1) once per bar via `ProcessNewBar` (46,698). EAP consumes/rejects/resets it the same tick or waits (16,953–16,955 once-per-bar `last_eval_bar`).
(7) Example: DUAL mode; bar closes with a bullish triple divergence → next bar first tick: `ProcessNewBar` → `CheckDivergence` arms confirmed (17,295) → same tick EAP (46,407) opens one setup (17,045 `FireSignal(bull,1,…)`) or rejects and resets (16,970/16,996/17,009/17,025). Later ticks of the bar may arm a live record only if the slot is empty and `CountActiveSetups()==0` (17,356); EAP ignores it.
(8) Classification: not a bug. Resolved FALSE POSITIVE.
(9/10) n/a. Effect on prior research: DUAL double-counting does not exist; DUAL == CONFIRMED trade set.

---
#### 🟠 D-3 (severity: 3/5) — LIVE stream is intrabar-repainting and its “pivots” are exported into research position-open CSV only via a transient flag — fragile semantics, plus live arms use *forming*-bar data
(2) `CheckLiveThreePivotDivergence` 17,352–17,468; `PO_CaptureDivergenceGeometry` 18,236–18,300.
(3) Lines 17,364–17,365 (`live_low=iLow(...,0)`, `live_high=iHigh(...,0)`), 17,397–17,400 (fills `g_live_div_*` with `live_low`/`live_high` and P3 time = forming `bar_time`), 17,455 arm, 17,467 flag reset.
(5) Behavior: every tick in DUAL/LIVE modes evaluates the forming bar; the value can change until bar close; comment at 17,349–17,351 acknowledges “Signals may disappear before bar close by design”.
(6) Why a problem: any consumer of `g_live_div_geometry_ready/price/rsi/time` (only `PO_CaptureDivergenceGeometry` 18,238 and the EAP pre-fire block 17,028–17,037) would see repainting data. In the current code EAP overwrites the arrays with **closed-pivot** armed values (17,029–17,037) immediately before `FireSignal`, and the live detector clears the flag at 17,467 right after arming, so the position-open CSV actually captures confirmed geometry — the exposure is theoretical *unless* code is later reused. Fragile coupling.
(7) Classification: potential repaint-data ingestion (export/feature level), currently neutralized; “needs verification” for any export where `fill_time` lies inside the forming bar.
(8) Example: live BUY geometry at 09:35:05 (P3 = forming low 1.10500); by 09:39 the low is 1.10420 — the signal would have “disappeared”. Since live arms never trade, no order is affected; the flag/arrays are only consumed in EAP’s confirmed path with confirmed values.
(9) Fix: gate all live-array writes with an explicit `inp_div3_export_live_geometry` input; never reuse `g_live_div_*` arrays for confirmed entries; rename to `*_display_only`.
(10) Effect on prior research: none found in the shipped CSV columns (they are written from the confirmed/armed branch); this is a defense-in-depth note.

---
#### 🟡 D-4 (severity 2/5) — Confirmed stream: session/cooldown filters & one-shot arming make backtest behavior parameter-sensitive; needs verification of *intended* signal spacing
(3) 17,081–17,089 (cooldown via `g_last_signal_time`, `inp_min_bars_between_signals`), 17,073 (`IsSessionActive`), 17,068 (`CountActiveSetups()>0`).
(5) Behavior: while any setup is active, `CheckDivergence` returns (17,068–17,072) — i.e., a running 5-step grid **suppresses new divergence detection entirely** (not merely entry). Detection resumes only after the basket fully closes.
(6) Why a problem: divergence confirmation is time-stamped and then dropped while a basket is open; two successive signals with a closed bar between them but overlapping basket lifetime cause the second signal never to be recorded (not even as a shadow). This materially reduces signal count in fast markets and is invisible to the user except via `RP_ObserveCurrentBarGate("ACTIVE_SETUP")`.
(7) Classification: not a leak; a signal-suppression design that must be stated when reading win-rate research.
(9) Fix: record signals in shadow/research even when entry is suppressed (shadow tracking exists at 18,075–18,085; wire DIV3 through it unconditionally).
(10) Effect on prior research: per-symbol backtests can differ purely from basket overlap timing; comparable across configs only if this gate is held constant.

---
#### 🟢 D-5 — confirmed detection ordering — no look-ahead (verified)
(3) `CheckDivergence` uses pivots with `GetCurrentBarShift()>0` (17,101–17,104) and bar-1 data (17,090–17,091, 17,078 `iTime(...,1)`); the only tradable consumer EAP re-checks on bar-1 close: structure invalidation and BOS/reclaim tests use `iClose/iLow/iHigh(...,1)` (16,957–16,984), RSI at shift 1 (16,985), fires with `FireSignal(bull,1,…)` (17,045) whose entry is `iLow/iHigh(...,signal_shift)` at 17,914–17,915 (called with shift=1). No future-bar reference in the whole confirmation→fire path (raw-verified). Note: a *price* trigger at the level of bar-1 extreme is executed as a market order on the first tick of the new bar — see E-1 — which is honest but changes fill semantics vs the engineered grid.

---

## 4. T0 Snapshot

`CaptureEntrySnapshot` (called 17,927 in `FireSignal` before QC) and its `SEntrySnapshot` fillers (`Fill_TimeContext` 19,419, `Fill_PriceAction` 19,468, `Fill_TF_Features` 20,096, `CaptureStructureEventFeatures` 20,012, …) read **only closed bars**: `iClose(...,1)`, `iHigh/iLow` of confirmed swings (`FindTwoRecentConfirmedSwingHighs/Lows`), EMAs/RSI/ADX/ATR at shift 1. Session fields derive from `sn.entry_time` (the signal/fill time). Structure events require shifts > 1 (20,045–20,052). `bars_since_signal`/`minutes_since_signal` initialized 0 at capture (19,461–19,462) — i.e., the T0 snapshot intentionally captures the signal bar, not the (later) finalization.

- 🟢 No look-ahead found in T0 feature extraction (raw-verified across the snapshot fillers).
- 🟡 T-1 (severity 2/5): snapshot *entry_price* semantics — `CaptureEntrySnapshot` is invoked before QC at 17,927 with `g_current_snapshot` derived from the planned level (`entries[0]`), while the actual fill price is fetched later (`TryGetLatestSetupPositionFill` 19,260/42,263). Any research table keyed on snapshot entry price vs actual fill can differ by up to ~1 bar range. Fix: write the real fill back into the snapshot record before `RecordPositionOpenBySetup` (42,270). Effect on prior research: small, systematic; treat “entry vs fill” fields as distinct (fill price is the correct one).

---

## 5. Execution engine

#### 🟠 E-1 (severity 4/5) — Step-1 entry is an unconditional market order at the first tick after confirmation, while the entire grid (steps 2–5, BE, TP, EMA200 target) is engineered around `entries[0] = bar-1 extreme`
(1) Severity: 4/5 wrong-entry-class (fills, not signals).
(2) Functions: `FireSignal` 17,903; `CreateSetupWithPrediction` 19,203; `PlaceEntry` 19,307; `ManagePositions` 42,141.
(3) Lines: 17,914–17,915; 17,951–17,952; 19,251; 19,257–19,263; 19,307–19,373.
(4) Snippets:
```mql5
// 17914-17915
   double entry = is_bull ? iLow(_Symbol, PERIOD_CURRENT, signal_shift)
                          : iHigh(_Symbol, PERIOD_CURRENT, signal_shift);
// 19251   CreateSetupWithPrediction, immediately on signal:
   if(PlaceEntry(g_setups[idx].setup_id, is_bull, 1))
// 19329-19331   PlaceEntry = plain market order, no level condition:
   bool ok = is_long
      ? g_trade.Buy(lot, _Symbol, 0, 0, 0, comment)
      : g_trade.Sell(lot, _Symbol, 0, 0, 0, comment);
```
(5) Behavior: EAP confirms on the close of bar1 (structure-invalidation and BOS/reclaim tests on `iClose/iLow/iHigh(...,1)` 16,957–16,984) and calls `FireSignal(...,1)` on the first tick of bar0 (46,400→46,407 same tick). `FireSignal` sets `entries[0]` = bar1 low (BUY) / bar1 high (SELL) and immediately market-orders step 1 — without ever checking that price is at that level. For a BUY, price on the first tick of bar0 is virtually always **above** bar1’s low; the market buy fills higher than the “entry” level. Only steps 2–5 are level-conditional (`should_enter = tick_price <= next_entry` 42,152–42,154, or bar-reclaim/failsafe 42,160–42,173).
(6) Why a problem: (a) the actual entry price differs systematically from the engineered level; (b) BE/TP/structure distances computed from `entries[0]` (BE weighted means 17,963–17,987, TP block immediately after) are then inconsistent with the real average entry — TP1 = `entries[0]+dist12` may already be inside the bar0 candle; (c) recorded “entry” for step 1 is only corrected *afterwards* via history lookup (19,260); between signal and lookup the bookkeeping uses the planned price; (d) in live trading this is a market order with no SL/TP (see E-2).
(7) Classification: **wrong-entry** (fill semantics), not look-ahead/data leak (no future data used).
(8) Example: BUY signal with bar1 low 1.10400; bar0 opens 1.10520; step 1 market-buys ≈1.10520 (recorded fill), TP1 at 1.10400+dist12 computed from 1.10400 — the basket’s actual risk geometry differs from the plan by 120 pips * 1 lot-step weighting.
(9) Fix: (a) if the design intent is a level trigger, place a real `ORDER_TYPE_BUY_LIMIT/SELL_LIMIT` at `entries[0]` (or re-check `tick<=entry` before `PlaceEntry` like steps 2+); or (b) if the intent is “enter at market on next-bar open”, compute BE/TP/grid from the **expected next-bar open** and label the exports accordingly.
(10) Effect on prior research: all backtests share the same semantics, so relative comparisons stand, but absolute metrics (avg entry slippage vs grid, TP hit geometry, step-1 share of realized P&L) are biased by this step-1 convention; every published “entry at divergence low/high” claim from exports is inaccurate at the bar-range scale.

#### 🟠 E-2 (severity 4/5 live-only) — All orders are naked market orders without broker-side SL/TP (`Buy(lot, sym, 0,0,0, comment)` 19,329–19,331); every exit is EA-managed
(3) `PlaceEntry` 19,307–19,373 (all steps go through it: 19,251 and 42,242).
(5) Behavior: entries carry no SL/TP/expiry. Exits are decided in-tick by EA logic (`CheckTakeProfit` called from `ManageSetups` 41,874, step exits inside `ManagePositions` tail, `HandleExternalClose` 46,580 for external closes, OnDeinit cleanup ~44,9xx).
(6) Why a problem (live): terminal crash, disconnect, EA removal or a hung tick stream leaves 5 stepped lots naked (no stop anywhere). For a “trustworthy EA” claim this is a show-stopper for unattended live use; in backtests there is no such risk (tester always ticks).
(7) Classification: wrong-exit/risk-management (live only).
(9) Fix: attach broker-side protective SL per step (e.g., beyond the basket’s structure level) or at minimum document that live use requires a VPS + watchdog; better: place a hard SL at setup open that EA logic later moves.
(10) Effect on prior research: none (backtest exits identical); effect on live results: tail events that close baskets (MT5 OnTrade external close handling 46,544–46,563 is robust to broker-side closes and treats them as outcomes — good) but unreachable-protection periods remain unmodeled.

#### 🟡 E-3 (severity 2/5) — `Sleep(500*attempts)` (42,253) and `Sleep(400*(att+1))` (19,364) inside retry loops
In the MT5 Strategy Tester `Sleep` behavior is terminal-version dependent (it is documented to be ignored/limited in the tester on some builds); in live it stalls the agent up to ~2–3 s per attempt. Not a logic bug; needs verification of tester behavior on the user’s build. Effect on prior research: none (retries still occur; loop bound `MAX_TRADE_ATTEMPTS`).

---

## 6. Position management & STEP logic

- Setup lifecycle: `CreateSetupWithPrediction` 19,203 (active at 19,253 only if step-1 fill succeeded; failure rolls back counters 19,280–19,284).
- Per-tick: `ManageSetups` 41,864 → `CheckTakeProfit` 41,874 → `ManagePositions` 42,141 (next step when `tick <=/>= next_entry` 42,152–42,154, optional bar-reclaim+failsafe 42,160–42,173, trend guard 42,178–42,196, margin gate 42,219–42,237, retry loop 42,240–42,255). Step >=3 disables the counter-EMA200 target (42,275–42,281).
- Pending entries across market close: `CheckPendingEntries` 42,302 (only at market-open edge, 46,410–46,411) with tolerance check (`PENDING_ENTRY_TOLERANCE_PCT`) and cancellation if price moved away (42,313–42,330).
- External/partial closes: `OnTrade` 46,475 → 2 s throttle → `ProcessTradeEventsImproved` 46,544 (skips setups that still hold positions 46,554) → `WasPositionClosedInHistory` 46,490 → `HandleExternalClose` 46,580 (computes realized P&L from history 46,588–46,594, finalizes snapshot/RX/records 46,608–46,624). This path treats ANY position-count drop to zero as the basket outcome — correct for all-or-nothing exits; partial scale-outs are handled by the EA itself so the history-based reconciliation is consistent.
- 🟢 Step-engine level checks are tick-based with real bid/ask (42,149–42,154) — no look-ahead, no “touch = fill at level” fiction (fills corrected from history 42,263).
- 🟢 Margin checks (42,222–42,237) before each step prevent margin-call cascades.
- 🟡 S-1 (severity 2/5): `ManagePositions` fires the step when the *tick* trades through the level but the order is a market order — with requote/spread the fill may be worse than `next_entry`; recorded `step_prices` are then corrected from history (42,263) but `stop_levels` (used by `CheckTakeProfit` BE math) are not re-anchored. Grid-vs-fill drift accumulates per step. Needs verification of impact magnitude per broker/symbol; fix = record actual fill basis into BE/TP when `inp_use_dynamic_step_sizing`.
- 🟡 S-2 (severity 2/5): setups are capped (`MAX_SETUPS`, 19,208) and suppressed while any basket is open (17,068/17,356) — with 5-step grids this can block new signals for many bars; combined with D-4 the research dataset is overlap-censored. Not a leak; sampling-bias note.
- 🟢 Backups/state: per-step & per-setup backups (41,883–41,884, 42,285–42,287), `SaveAllSetups` on timer (46,450), persistent state autosave (46,451–46,452, 46,416–46,433), `SaveSetupID` (19,271) — restart continuity is well covered; `OnInit` reload path (44,540–44,559) rebuilds analysis from CSVs when state files absent.

---

## 7. MTF synchronization

- `g_tf[]` per-timeframe indicator handles; copied per new bar with `CopyIndicatorBuffers` (46,668–46,686) at *bar close* semantics of each TF; VA/RSI recompute per TF at the same time (46,676/46,686).
- Snapshot features are indexed per TF at shift 1 of that TF (`Fill_TF_Features` 20,099–20,105: `iClose(_Symbol, g_tf[tf].timeframe, 1)` …). This is the correct per-TF “last closed value” convention.
- MTF-DI alignment (`GetProfileMTFDIAlign` 16,919–16,930) reads buffer index 1 of each TF.
- H1 regime via EMA order at shift 1 (16,931–16,941).
- 🟢 No cross-TF future mixing found: every MTF read is closed-bar (shift≥1) at its own timeframe; new-bar processing advances all TFs from the same tick context (46,679–46,687).
- 🟡 M-1 (severity 2/5): higher TF bars change at their own cadence; a check run at, e.g., 10:01 M5 uses H1 value from the H1 bar that *started* at 10:00 only when it closes (i.e., all intra-hour M5 checks use the same H1 close until 11:00). Correctness OK; “MTF consensus” statistics are therefore highly autocorrelated within the hour — statistical-validation note (see section 11), not a logic error.

---

## 8. Research pipeline (exports, RiskLab, S2B/S2C/S2D)

**Call-graph facts (full-file greps):**
- `S2B_RunAll`, `S2B_RunPhase2..7` (28,067/28,829/29,551/33,409/33,635/37,886), `S2B_RunWalkForward_Step4_v2` (29,151), `S2B_RunWalkForward_Stop_v2` (29,429), `S2B_RunStopPhaseB` (31,892), `S2B_TrainCompactWinner_Step4` (33,453), `S2B_EvaluateFinalCandidate_DevToOOS` (28,902), `S2B_StopD2024_*` runners — **zero callers** (only definitions and Print texts). `S2B_RunPhase1` does not exist.
- Automatic research hooks are only: `OnInit` TRUE_FORWARD load (44,579–44,591), `OnTimer` TRUE_FORWARD forward validation (46,463–46,464), `OnDeinit` exports (44,871–44,949: SaveStats, EOT_OPEN finalization 44,893–44,900, shadow/research-protocol exports 44,905–44,912, ME exports 44,933–44,937, `ExportAllAnalysis` 44,939 under `inp_enable_export`, `RX_ExportAll` 44,947, `S2D_FinalCleanupOnly` 44,948), per-close recording at 46,609–46,624 (`RecordAnalysis`, optional `ExportToExcel` 46,614–46,617, `S2D_CollectForwardRecord` 46,623 when `g_s2b_compact_winner.trained`).
- ⚠️ `S2D_HandleTrueForwardOnDeinit` (def 44,846–44,870, prototype 5,744) and its `S2C_RunPhase1/2/2b/3` calls (44,856–44,859) have **zero callers** — despite the comment at 39,903 “Call in OnDeinit or periodically”, `OnDeinit` never branches on `MODE_TRUE_FORWARD` and never calls it (raw-verified 44,871–44,949). The TRUE_FORWARD Stage-2C deinit exports therefore never execute; forward validation runs only via the `OnTimer` hook (46,463–46,464).
- `g_s2b_compact_winner.trained` becomes true **only** via `S2D_LoadFrozenWinnerParams` (40,215) when a frozen file exists (40,180/40,213) — never by in-run training.
- Position-open/analysis CSV writers are gated by `inp_export_*` inputs (e.g., `inp_export_on_signal` 19,274/42,288, `inp_export_on_close` 46,614, `inp_enable_export` 48,795, ME/risklab collectors similarly gated).
- `S2D_CollectForwardRecord` (39,463): writes one row per closed setup only when a winner is trained; guard `if(!g_s2b_compact_winner.trained) return;` 39,467. `S2D_RunForwardValidationIfReady` (39,911): requires CSV present + count ≥ min (39,914–39,930) — in TRUE_FORWARD only (timer path 46,464).
- RiskLab append guards: ≥2024-01-01 entries are blocked **only when `inp_s2b_stop_d2024_run_mode != STOP_D2024_DISABLED`** (default is DISABLED, line 826) — 20,858–20,863 / 20,874–20,879; see OOS section for the config-sensitivity consequence.
- `ExportAllAnalysis` (48,794–48,851) runs a large fixed set of exporters under `inp_enable_export`.

**Findings**

#### 🟢 R-1 — “research pipeline trains on results of the same run / leaks into entries” — not found
All entry-time features use only pre-entry closed data (section 4); per-close recording happens after finalization; outcome labels (`stop_flag`, `max_step_reached`, `final_event`, `pnl`) are written only at close/finalize (46,608–46,624, 20,8xx finalize chain per earlier verified reads: labels set when path is complete). RiskLab/Snapshot record creation occurs at entry/step-open with outcome fields defaulted and filled at finalize — the canonical correct design.

#### 🟡 R-2 (severity 3/5, manual-run research only) — Stage 2B report “DEV AUC / DEV ECE” are in-sample statistics
(3) `S2B_TrainLogistic_OnRows` 30,700–30,702 (`mdl.dev_auc = S2B_AUCFromPreds(preds_tr, tgts_tr, train_count);` computed on the training rows themselves; same pattern `_Stop` 30,953–30,955 and `S2B_TrainLogistic` chain), `S2B_BuildCalibrationBins` DEV bin = all dev rows incl. those used to fit (27,926–27,950).
(5) Behavior: in the *manually invoked* Phase2/4 reports, “DEV AUC/ECE” describe the fit sample, not a validation sample.
(6) Why a problem: overstates model quality; the OOS figures (true validation) exist and are honestly split (see section 10), so the flaw is limited to DEV columns/labels and to any user comparing DEV vs OOS as if they were train/valid.
(7) Classification: statistical-validity issue (research output quality), no data leak into trading.
(9) Fix: relabel DEV columns “TRAIN_AUC (in-sample)” or evaluate on an internal hold-out slice.
(10) Effect on prior research: OOS/WF numbers in the same reports are unaffected; in-sample DEV columns should not be quoted as performance.

#### 🟢 R-3 — all S2B phase runners are unreachable ⇒ they cannot corrupt backtests; but also nothing auto-trains a winner in this export ⇒ S2C passive overlays and S2D forward collection stay off unless the operator manually provides frozen CSVs + TRUE_FORWARD mode. (Fact, not a bug; see Q9.)

#### 🟡 R-4 (severity 2/5) — report text vs state mismatch in manual WFV2 report: the “ALIGNMENT STATUS — FINAL” header block writes literal “PASS” lines for Protocol/Fold (29,389) and Baseline (29,390) and “NOT YET VERIFIED” for Cross-component (29,391) **before** the self-test result is applied; only the SelfTest line is dynamic (29,392). When the self-test fails, the statuses are later set to FAIL (29,411–29,413) — the printed header then contradicts the actual state. (Self-test PASS also leaves cross-component at “NOT_YET_VERIFIED”, 29,407.) Manual users can read a contradictory report; machine state (flags) is correct. Fix: print the status strings, not literals.

---

## 9. **CONFIRMED DATA LEAKAGE**

After the full sweep + two runtime passes, the following previously suspected leakage channels are closed with code evidence; the remaining confirmed leakages are research-report-only and manual-run-only:

**9.1 — Confirmed leakages (research layer only, zero trading impact):**
1. **L-1 (in-sample DEV metrics; manual Phase2/4 reports)** — see finding R-2: DEV AUC computed on the training rows (30,700–30,702; 30,953–30,955). Class: data leak (train into eval), confirmed.
2. **L-2 (in-sample DEV calibration)** — `S2B_BuildCalibrationBins` (27,926–27,950) bins DEV rows that include fit rows. Confirmed, same class as L-1.
3. **L-3 (DEV fallback inside the manual feature-importance composite)** — the FI loop computes `eff_*_auc = oos_auc if > 0 else dev_auc` (24,627–24,629) and feeds those into the composite (24,690–24,694) and the verdict thresholds (`best_eff_auc`, 24,695–24,711): when a target has no OOS rows its in-sample DEV AUC is used as if it were evidence, and `sample_score` is pooled across all three target types (24,688–24,689). The STRONG verdict additionally requires `oos_stable` (24,745–24,749) and direction reversal is only detected when both DEV and OOS exist (24,714–24,744), so the impact is bounded — but a feature with DEV-only data can still be labeled MODERATE (24,751–24,756) on in-sample signal. Confirmed as a mild **in-sample-evidence** flaw in the manual feature-importance report only; the S2B pipeline (protocol 26,517–26,554) does not share this flaw. Note: the pool is DEV+OOS *eras* only in the sense that eff_auc prefers OOS; rows are not era-pooled for ranking.

**9.2 — Suspected channels checked and CLEARED (no leakage):**
- Live/forming-bar data into orders: cleared (EAP confirmed gate 16,945; see D-1/D-2).
- 2024+ holdout rows into RiskLab/analysis: cleared **only when the STOP_D2024 mode is enabled** (append guards 20,858–20,863 / 20,874–20,879 plus `S2B_StopD2024_ValidateRunBoundary` 34,432–34,442 which also demands `MODE_TRAIN_EXPORT` and `inp_enable_export`). **With the shipped default (`STOP_D2024_DISABLED`, line 826) no guard fires** — 2024+ rows are appended and exported as ordinary OOS rows; this is a run-config dependency, not a code leak (see OOS section).
- OOS consumed during search/selection: cleared — `g_s2b_oos_used_this_run` guards the final candidate evaluation (28,939–28,943; latched true at 29,075, one-way even if the OOS evaluation itself later fails 29,087–29,090); S2B protocol fixes DEV [2018,2023)/OOS [2023,2024) (26,526–26,531) with folds strictly inside DEV (26,938–26,951; val_to ≤ dev_to 26,976–26,980); Phase B stop engine is DEV-only (29,487 header; row policy hash includes `WINDOW=DEV_ONLY` 30,375–30,378; `dev_hard_limit` 31,337/31,422 checks).
- Snapshot/feature extraction using future bars: cleared (sections 4, 7).
- History-based P&L reconciliation creating phantom outcomes: cleared — `WasPositionClosedInHistory` matches by deal comment + position id and requires an OUT deal (46,523–46,541).
- CSV state reload contaminating a fresh backtest with older-run trades: partially cleared — cumulative-analysis reload is gated by state-file presence (`state_loaded`, 44,540–44,559) and by `inp_reset_setup_id`; RiskLab rows re-appended from CSV could carry old dates, but the window guards plus `inp_reset_setup_id` cover the research windows; **needs verification** if the same terminal is reused across different symbols.

---

## 10. OOS validation

- Windows are fixed and honest in the S2B protocol: DEV [2018.01.01, 2023.01.01), OOS [2023.01.01, 2024.01.01) (26,526–26,531); folds F1/F2 validate 2021 and 2022 only (26,938–26,951) and cannot touch OOS (26,976–26,980). Protocol init records only in-range rows (26,566–26,573), masks eligible rows (26,575–26,595) and calls `S2B_BlockPipeline` on every failure (26,553/26,617/26,624) before `pipeline_valid=true` at 26,629.
- The self-test re-runs the baseline through the *search* evaluator and requires exact equality of protocol/fold/dataset/row-mask/feature hashes and of every per-fold train/val count, AUC, PR-AUC, MCC, recall, F2 (1e-4 tolerance) and all WF aggregates (26,681–26,741) before `g_s2b_selftest_passed=true` (26,742–26,747); mismatch → `S2B_BlockPipeline` (26,754), evaluator failure → `S2B_BlockPipeline` (26,686).
- ⚠️ One residual ordering nuance (downgraded from “bug” to “needs verification”, inert in this export): WFV2 pre-sets `g_s2b_alignment_valid/search_allowed/pipeline_valid = true` at 29,381–29,383 *before* running the self-test (29,385); the self-test’s early-return branch “baseline WF not valid” (26,671–26,676) fails without re-blocking, and WFV2 then writes FAIL only into the *status strings* (29,411–29,413), not into those three flags. All consumers (EvaluateFinalCandidate 28,934–28,938, Phase5 33,412–33,417, Phase6 33,638–33,655) are themselves zero-caller dead code, so nothing in a run can act on a stale flag — noted for any future wiring of the phase chain.
- True holdout 2024–2026: **conditional**. `RL_AddRecord`/`FinalizeSnapshot` reject ≥2024-01-01 appends (20,858–20,863 / 20,874–20,879) and `S2B_StopD2024_ValidateRunBoundary` (34,432–34,442) refuses runs whose snapshots/risklab touch 2024+ (and requires `MODE_TRAIN_EXPORT` + `inp_enable_export`) — **all of it only when `inp_s2b_stop_d2024_run_mode != STOP_D2024_DISABLED`**. The shipped default is `STOP_D2024_DISABLED` (line 826); under defaults a backtest spanning 2024+ does append 2024+ rows into RiskLab/snapshots, and they are exported with `is_oos` semantics (fixed 2023-01-01 cutoff, 21,063–21,074). Any “2024 holdout untouched” claim must be verified against the run mode used.
- `g_s2b_oos_allowed / winner_confirmed / freeze_allowed` never become true anywhere (grep-verified: only false-assignments + `S2B_BlockPipeline`/`S2B_ResetPipelineState` resets); Phase6 (33,635) is a stub ending with “blocked pending D-1 implementation” (33,658) and re-falses all four flags (33,662–33,664) — i.e., *in this export* the OOS evaluation stage, winner confirmation and freeze export can never fire through any path. This is the file’s main self-protection feature: **it cannot overfit-then-deploy automatically**.
- RiskLab DEV/OOS split for analysis exports: `RL_CalcDevCutoff` (21,056) uses fixed 2023-01-01 when both eras exist, else a 70/30 time split (21,076–21,087) — for a dataset that never reaches 2023, the 70/30 fallback silently labels the newest 30% as “OOS”. That fallback is dataset-shape-dependent; **needs verification** per dataset before treating OOS columns as calendar-OOS.
- 🟢 `S2B_EvaluateFinalCandidate_DevToOOS` (28,902) is guarded by selftest (28,911–28,915), cross-component alignment (28,916–28,920), locked-candidate validity (28,921–28,933), pipeline flags (28,934–28,938), OOS-once (28,939–28,943; latch set at 29,075 and NOT rolled back when the OOS evaluation itself fails — conservative), feature-hash lock (28,945–28,949) and protocol/fold readiness (28,950–28,954) — but has zero callers (dead), so its correctness is inert.

## 11. Statistical validation

- AUC/PR-AUC/MCC/F2/precision/recall implementations are standard and were raw-verified (27,5xx–28,6xx, 30,529–30,572, 31,033–31,066); Cohen’s d with pooled SD (24,489–24,500); bootstrap CIs (stratified per-fold stop/non-stop, seed 202,608, 1,000 iters, percentile CIs 2.5/97.5: 31,635–31,719).
- ECE calibration computed separately DEV/OOS (27,958–27,996) — standard.
- Multiple-testing control: candidate search (`S2B_RunSystematicCompactSearch` 4–6 features from an 8-pool) and threshold quantile sweep (6 rows) are **not** multiplicity-adjusted; protection comes from the absolute gates + OOS-once + dead-code status, not from FDR control. Statistical note (severity 2/5) for manual runs: best-of-N WF-mean-AUC selection inflates expected max; the “worst fold” gate (29,751–29,755) partially counters instability, not multiplicity.
- Determinism: logistic fits use fixed epochs/LR/decay; Phase B RNG is seeded fixed (31,088–31,102, 31,645); sorting is deterministic; hashes (FNV-1a variant 26,747–26,757) pin protocol/folds/datasets/features — reproducibility by design.
- 🟡 Statistical sample hygiene: DEV row sets for 2018–2022 in this backtest system come from one continuous symbol run; overlapping multi-step baskets create serially dependent rows (D-4/S-2); AUC significance bounds ignore row autocorrelation. Needs verification per dataset (bootstrap is per-fold resampled, but row order within fold stays — no block bootstrap).

## 12. Backtest integrity

- Fill model: level-conditional steps use tick bid/ask vs level (42,149–42,154) — strict, no “touch==fill” optimism; actual fills pulled from trade history (42,263, 19,260). Step-1 market fill at first tick after confirmation (E-1) is the *only* non-level fill; it matches the tester’s next-tick model (no look-ahead, but see E-1 for grid semantics).
- Spread/session gates: `IsSpreadOK`, `IsMarketOpen`, `IsSessionActive` gate signals and entries (17,906–17,911, 17,073, 17,356, 42,199–42,218). Backtests on brokers with historical spread modeling will differ; standard caveat, not a defect.
- Margin gates before each step (42,219–42,237) prevent unrealistic overdrafts.
- OnTrade/history reconciliation in tester: OnTrade fires in “every tick” mode; throttle 2 s (46,480–46,482) is wall-clock — **needs verification**: in fast backtests with thousands of ticks per second the 2-second wall-clock throttle can make OnTrade lag or (in some tester builds) never fire, which would delay `HandleExternalClose` bookkeeping; `ManageSetups` also runs its own per-tick checks, so the impact is limited to external-close accounting. Fix: use `TimeCurrent()`-derived bar timestamps or event counters instead of wall-clock seconds.
- `Sleep` calls in tester (19,364, 42,253): behavior build-dependent (E-3).
- State persistence across runs (44,540–44,559) means repeated runs on the same terminal accumulate cumulative CSV analysis; exports are cumulative by design (SaveStats/RecordAnalysis). For clean per-run backtests the operator must clear state or use a fresh terminal/account id — document; **needs verification** per user workflow.
- Multi-run hedging of results: not applicable (research export; no deployment).

---

## 13. Initially-suspected findings proven FALSE POSITIVES (explicit list)

1. **DUAL_MODE double-trades the same geometry** — impossible: single armed slot, live arms non-consumable (16,945), guards 16,832–16,854. [D-2]
2. **LIVE path fires real intrabar entries / uses unclosed-pivot prices in orders** — live record is display-only; EAP requires `confirmed`; orders only via FireSignal shift=1 with closed-bar data. [D-1/D-2]
3. **WFV2 unblocks pipeline before self-test** (“self-test ordering bug”) — flags set true at 29,381–29,383 are re-falsed by `S2B_BlockPipeline` in the self-test’s evaluator-failure branch (26,686) and mismatch branch (26,754); the third failure path — “baseline WF not valid” early return 26,671–26,676 — exits without re-blocking (residual nuance, inert because every consumer is dead code; Phase5 re-gates on `g_s2b_search_allowed` 33,412–33,417). [Section 10]
4. **PhaseB rejection direction reversed (tail = reject)** — rejection is per-fold low pct_rank (top-risk tail), 31,511/31,670/31,685; pooled probability explicitly INFO_ONLY (31,735/31,746). [Section 11]
5. **Threshold-tradeoff CSV drops a fold (only Fold1/Fold2 written)** — the shared protocol has exactly two primary folds (26,936–26,951); CSV columns f=0..1 match the design. [Section 11]
6. **OOS/winner/freeze unlock flags can turn true in a live run** — `g_s2b_oos_allowed`, `g_s2b_winner_confirmed`, `g_s2b_freeze_allowed` never assigned true anywhere; Phase6 is a stub. [Section 10]
7. **S2B auto-train runs in OnDeinit/OnTimer and can deploy a model into the same backtest** — zero callers for all phase runners; automatic hooks are exports only. [Section 8/R-3]
8. **OnDeinit/OnTimer ordering shifted lines (19,410–19,680 “junction” scare)** — reading artifact; re-verified raw. [SOURCE SWEEP STATUS]
9. **S2C passive overlay contaminates entries in MODE_TRAIN_EXPORT** — `FireSignal` consults S2C only when `g_s2b_compact_winner.trained` (18,112–18,130), which requires a frozen file loaded in TRUE_FORWARD (40,215); default runs: false. [Section 8]
10. **“DEV AUC is validation” columns in OOS reports** — flagged as in-sample (R-2/L-1) only in manual DEV columns; OOS columns are genuinely out-of-sample and were never suspected of leakage.

---

## 14. The ten questions, answered with code evidence

**Q1 (source completeness): Is the audited source complete and fully accessible?** YES — 51,733 lines, sha256 `195cb151…4c534c3`, no missing bodies/includes (section 1, SOURCE SWEEP STATUS).

**Q2 (DIV3_CONFIRMED_ONLY): does it trade only closed-pivot confirmed divergences?** YES — live detector disabled at 17,355; CheckDivergence gates pivots to closed bars (17,101–17,104), signal entry shift=1 (17,914–17,915, 17,045). Repaint impossible; session/cooldown/active-setup gates apply (17,068–17,089).

**Q3 (DIV3_LIVE_P3_ONLY): does it trade the live forming P3?** NO — it cannot trade at all (16,945 + 17,067); zero orders by construction. Functional-mismatch risk for users; needs intent verification (D-1).

**Q4 (DIV3_DUAL_MODE): can both paths double-trade one geometry / does the live arm race the confirmed arm?** NO — single slot; live arms never overwrite confirmed (16,832) and are never consumed (16,945); confirmed arms fire once per bar via EAP 16,953–16,955 (D-2).

**Q5 (T0 snapshot): any look-ahead or future data in the entry snapshot?** NO — all fillers use shift ≥1 closed data (19,419–19,475, 20,096–20,110, 20,012–20,095). Minor T0 bookkeeping nuance: planned-vs-actual fill (T-1).

**Q6 (execution engine): are order entries/fills correct and leak-free?** Entries are honest market orders at the first tick after confirmation (E-1) with strict level checks for steps 2–5 (42,152–42,154) and history-corrected fill prices (19,260/42,263) — no look-ahead; BUT step-1 grid semantics vs actual fill mismatch (E-1) and no broker-side SL/TP on any order (E-2, live risk).

**Q7 (STEP logic): does the 5-step scale-in manage levels/BE/TP correctly?** YES structurally: tick-conditioned entries, margin gates, trend guard, bar-reclaim failsafe, per-step backups, step_times/prices from fills, EMA200-target disable at step≥3 (42,141–42,300). Nuances: grid-vs-fill drift (S-1), overlap-censored signal detection while baskets open (D-4/S-2).

**Q8 (MTF synchronization): are multi-timeframe features aligned to closed bars?** YES — per-TF shift-1 reads (20,099–20,105), alignment at buffer 1 (16,919–16,941); caveat: intra-TF autocorrelation in MTF stats (M-1).

**Q9 (research pipeline autonomy): can research/training contaminate the same run's trading or exports?** NO for trading — zero-caller phase runners, export gating (`inp_export_*`), winner-trained-only-if-frozen-loaded: `S2D_LoadFrozenPackage` (40,289) requires winner/band/policy/manifest files, the manifest loader (40,257) pins symbol + `MODE_TRAIN_EXPORT` + signature hash, and `trained=true` is set only after every key is read incl. `n_features>0` (40,213–40,216). YES for manual-run in-sample DEV columns (R-2/L-1/L-2) and DEV-fallback in feature ranking (L-3).

**Q10 (statistical & OOS validity): are OOS splits, folds and statistics sound?** SOUND in design: fixed calendar windows (26,526–26,531), folds inside DEV (26,938–26,980), selftest hash+metric equality before `selftest_passed` (26,681–26,747), OOS-once (28,939–28,943, latched 29,075), conditional 2024+ append guards (20,858–20,863 — active only when STOP_D2024 mode enabled; default DISABLED, 826), standard metrics + fixed-seed stratified bootstrap (31,635–31,719). Caveats: no multiplicity control in manual searches; DEV-split fallback 70/30 when pre-2023 data missing (21,076–21,087); serial correlation from overlapping baskets unmodeled; 2024+ data enters exports under default settings.

---

## 15. SOURCE SWEEP STATUS

```
SWEEP SCOPE      : v2_research_export_v2/v2_research_export_v2.mq5 — 51,733 lines pre-patch (LF blob)
SHA-256          : 195cb151e9fd7e09797d221c097066f28668b756c583bbf77e60a373a4c534c3
SWEEP METHOD     : full-line numbered dumps of every logic line (blank/comment-only/
                   string-literal-emission lines skipped in display, all logic read);
                   two runtime call-chain passes (OnTick/OnTimer/OnTrade -> signal ->
                   detection -> staging -> setup -> snapshot -> order -> position ->
                   step -> exit -> finalize -> research) executed 2026-09-04.
COVERAGE         :  1..23,240           RAW-READ (multiple passes + junction re-verification
                      16,943-17,600, 19,410-19,680, 19,950-20,452, 20,840-20,940)
  23,240..27,040      RAW-COMPLETE (spots closed 2026-09-04: 24,401-24,749 incl.
                      24,610-24,749 re-fill; 25,001-25,150; 25,701-25,949;
                      26,101-26,549; junctions confirmed earlier)
  27,040..34,860      RAW-COMPLETE (2026-09-04 dumps 27,600-27,950, 28,951-29,200,
                      29,701-29,950, 30,221-30,500, 30,501-30,780, 30,781-31,030,
                      31,031-31,280, 31,281-31,540, 31,541-31,800, 31,801-31,960,
                      33,300-33,550; prior raw passes 27,040-27,600 (2x),
                      31,000-33,300, 33,550-34,860; anchors re-verified this pass:
                      28,902-28,959, 29,360-29,430, 29,743+, 33,409-33,664,
                      34,432-34,442)
  34,860..41,831      RAW-COMPLETE (B/C/D/E layers + micro-gaps closed)
  41,832..51,733      READ + junction re-checks 2026-09-04: 44,540-44,630,
                      44,846-44,960, 46,224-46,360, 46,395-46,662, 46,663-46,729,
                      48,794-48,880
JUNCTIONS          : every cross-function call in the trading chain verified by
                     definition-anchored dumps (16,943/17,060/17,352/17,903/19,203/
                     19,307/41,864/42,141/42,302/46,360/46,444/46,475/46,544/46,580/
                     46,663) + caller greps (CheckDivergence only from ProcessNewBar
                     46,694/46,698; FireSignal only from EAP 17,045)
FULL-FILE GREPS    : zero-caller proofs for all S2B phase runners; EventSetTimer
                     44,563/EventKillTimer 44,873; is_oos writers 21,093 + RL cutoffs;
                     frozen-load call sites 44,582; export gates inp_export_*
RESULT             : NO unread source region remains. Ledger file:
                     /home/user/notes_audit/sweep_status.md (2026-09-04 final update)
STATUS             : FULL SWEEP COMPLETE — every source line read; report final.
                   : POST-AUDIT 2026-09-04: Section-A fixes applied to working tree —
                     file now 51,796 lines, sha256 d961dcb9113749aa8a814fd88e60a87075e66c27094a239b1e0e18d9d9bc99f6;
                     UNCOMPILED — verify in MetaEditor (see section 17).
```

---

## 16. Readiness verdict

**READINESS: NOT READY FOR UNATTENDED LIVE — and not a fair basis for headline performance claims without reruns under CONFIRMED_ONLY.**

The EA is *structurally honest in its signal timing*: after two full passes I found **no confirmed look-ahead, no confirmed future-data leakage into order decisions, no DUAL double-trade, no hidden OOS usage, and no automatic self-training that could overfit-and-deploy inside a backtest**. The research stack (Stage 2B/2C/2D) is almost entirely dead code in this export, which paradoxically is its strongest integrity property: the OOS/winner/freeze machinery is unreachable. **Correction recorded during the final pass:** the earlier interim claim that “the 2024+ true holdout is hard-blocked by default” is WRONG — the append guards (20,858–20,863 / 20,874–20,879) and `ValidateRunBoundary` engage only when `inp_s2b_stop_d2024_run_mode != STOP_D2024_DISABLED`, and the shipped default is DISABLED (826); under defaults, 2024+ rows enter RiskLab/snapshots/exports as ordinary rows.

It is **not** trustworthy as-is for the reasons that survived the sweep, in order of impact:
1. **E-2 (4/5): no broker-side SL/TP on any order** — unattended live use is unsafe by construction.
2. **E-1 (4/5): step-1 “entry at the divergence extreme” is fiction** — market order at next open tick; grid/BE/TP geometry and every export anchored on `entries[0]` mismatch the true fill by up to a bar range. All published per-step metrics should be re-derived from `step_prices` (actual fills).
3. **D-1 (2/5): `DIV3_LIVE_P3_ONLY` silently never trades** — if you used it for results, those results are empty or were produced by a different mode; verify which mode produced every historical export. Default `DUAL_MODE` is equivalent to CONFIRMED for trading, so DUAL exports are safe to use.
4. **L-1/L-2/L-3: manual research reports contain in-sample DEV metrics and pooled DEV/OOS ranking** — treat only OOS/WF columns as evidence.
5. Backtest-to-backtest integrity caveats: wall-clock throttling (46,480), `Sleep` in tester (19,364/42,253), cumulative state/CSV reload across runs (44,540–44,559), 70/30 DEV fallback (21,076–21,087), and the default-disabled 2024+ holdout guards (20,858–20,863; input 826 default `STOP_D2024_DISABLED`) — each needs verification on the user’s exact terminal/dataset/run-mode before quoting numbers.

**Bottom line:** code-level integrity of the *signal path* is high; execution-fill honesty, protective stops and mode semantics contain the confirmed weaknesses above. Use `DIV3_CONFIRMED_ONLY`, CONFIRMED-mode defaults, fresh-state per backtest, and require the E-1/E-2 fixes before any live deployment or before treating the EA’s own exports as the ground truth for the research claims in the repository.

— End of audit report. Full evidence ledger: `/home/user/notes_audit/sweep_status.md`.


---

## 17. Post-audit Section-A technical fixes (2026-09-04, working tree — NOT yet compiled)

Applied to `v2_research_export_v2.mq5` (working tree; diff = +68/−5; sha256 d961dcb9113749aa8a814fd88e60a87075e66c27094a239b1e0e18d9d9bc99f6):

| Fix | Where | What changed | Effect |
|---|---|---|---|
| FIX-E1 | `CreateSetupWithPrediction` (~19,264) | After the real step-1 fill is known, if `|fill − ent[0]| >= _Point`, re-anchor `stop_levels[0..4]`, `break_even_levels[0..3]`, `target_levels[0..4]` by the realized delta; recompute `is_above_ema200`/`use_ema200_as_target` against the real fill; re-init `g_current_smart_target` on the real fill. | Grid, BE/TP geometry and exports are anchored to the ACTUAL entry instead of the fictional bar-1 extreme (fixes E-1 + the S-1 drift bookkeeping). No change to entry timing, direction, trade count or step spacing. |
| FIX-E2 | `PlaceEntry` (~19,358) + new input `inp_broker_protection_sl_atr` (default **0.0 = off**) | When > 0, after each successful step order, the fresh position (symbol/magic/comment match) gets `PositionModify(ticket, bid|ask ∓ N·ATR(1), 0)` — a WIDE disaster SL only, far beyond every EA-managed exit; failure warns (60 s throttle). | Broker-side protection for live (crash/disconnect/hang); default off preserves backtest parity. |
| FIX-E3 | 4 `Sleep()` sites in trade paths (PlaceEntry retries ×2, ManagePositions retry, CloseSetupWithReason history-poll) | Wrapped in `if(!MQLInfoInteger(MQL_TESTER))`. | Removes tester-build-dependent stall risk; live pacing unchanged. |
| FIX-D1 | `OnInit` (~44,411) | Startup prints: `DIV3_LIVE_P3_ONLY` opens NO trades (display-only); `DUAL` trades only the confirmed stream. | Kills the silent-no-op trust hazard (D-1) at the source. |

**Not patched (deliberately — zero-caller, manual-run research functions only; recipes below if the S2B chain is ever reactivated):** L-1/L-2 (in-sample DEV AUC/calibration: relabel DEV columns as TRAIN/in-sample or evaluate on an internal slice — sites 30,700–30,702 / 30,953–30,955 / 27,926–27,950), L-3 (require `oos_stable` or OOS presence for MODERATE+ verdicts — sites 24,627–24,756), R-4 (print status strings instead of literals at 29,389–29,392 / assign statuses before writing).

**Status:** all hunks brace-balanced; file converted back to LF to match the blob convention; **not compiled** — must be verified in MetaEditor (compile + a short backtest before any live use). Independent strategy/financial audit: `v2_research_export_v2_STRATEGY_AUDIT.md`.
