# C2/M2 — Verification Evidence Report V1 (Code + Data)

**تاریخ:** 2026-09-12 (Asia/Tehran) · **ماهیت:** صرفاً Verification — بدون هیچ تغییر کد/threshold/feature/backtest/optimization.
**قاعده:** هیچ نتیجه‌ای بر اساس حدس یا تفسیر نیست؛ هر ادعا فقط با Code Evidence و Data Evidence از آبجکت‌های git.
**Status scope:** PROVEN / PARTIAL / NOT PROVEN / FALSE.
**منبع کد مرجع:** `v2_research_export_v2_cf.mq5` @ commit `4759646` (شاخهٔ داده) — شماره‌خطوط از همین blob.
**مرجع‌های داده:** R2=`3352b18:M0/` ‏(matched M0)، R4=`0e91848:v2_research_export_v2_cfv2/` ‏(M2)، R3=`2f04d01:…/C2M2_M0_Shift1_Population_DEV.csv` ‏(population)؛ R1=`662a7ec:v2_research_export_v2_cf/` ‏(legacy).

---

## ۱. MAE / MFE / MaxDD — UpdateDrawdown در Step1..Step5

### ۱.۱ Code Evidence (مسیر اجرا)

| مرحله | مسیر کدی (line refs) | ماهیت |
|---|---|---|
| Trigger | `OnTick` (:46832) → `ManageSetups()` (:46887) → per setup فعال: `CheckTakeProfit(i)` (:42306) | در **هر تیک**، برای **هر setup فعال** — بدون وابستگی به شمارهٔ Step |
| Gate داخل `CheckTakeProfit` (:43061) | ابتدا guard: `is_active` و `IsMarketOpen` (:43062-43063) | اجرا فقط روی setup فعال و بازار باز |
| شاخهٔ Step≥4 | `if(g_setups[idx].current_step >= 4) { … UpdateDrawdown(idx); … return; }` (:43115-43122) | **تک‌خط UpdateDrawdown صریح در شاخهٔ Step≥4** — با کامنت تطبیقی قطعی (:43118-43120): «CF fix (research-only, Freeze v2): **keep MAE/MFE/MaxDD fresh at Step>=4.** Tracking/export only — logging fix != strategy change; no trade impact.» |
| مسیر Step≤3 | بعد از بازگشت شاخهٔ بالا: `UpdateDrawdown(idx);` (:43164) | هم‌پوشانی هر دو مسیر: به‌ازای هر Step از 1 تا 5 در هر تیک |
| بدنهٔ `UpdateDrawdown(int idx)` (:43470-43509) | `cf = BuildComment(setup_id)`؛ حلقه بر `PositionsTotal()` با فیلتر `_Symbol`+magic+`POSITION_COMMENT == cf`؛ `profit = Σ (POSITION_PROFIT + POSITION_SWAP)` | floating **سبدِ پله‌ها (basket)** — جمع همهٔ پوزیشن‌های همان setup (همهٔ Stepهای باز) |
| به‌روزرسانی‌ها | peak/lowest profit (:43488-43491)؛ MaxDD = max(peak−profit، |profit| در صورت منفی) با نسبت به entry_balance (:43493-43501)؛ MFE `if(profit>mfe_money) …` (:43504-43505)؛ MAE `if(profit<0 && -profit>mae_money) mae_money=−profit` (:43508-43509) با کامنت «مثبت ذخیره می‌شود (magnitude)» | تعریف دقیق: MFE=ماکزیمم سود شناور؛ MAE=ماکزیمم |زیان شناور| (magnitude)؛ MaxDD=drawdown از قلهٔ شناور |
| Reset | :2639-2640 و :12657-12658 (`mfe=mae=0`) | یک مقدار per setup در `g_setups[idx]` |
| Persist/export | snapshot copy :21325-21326؛ RXL emit :13157/:13233؛ POSummary emit :15009-15010 | خروجی در `RX_..._Labels.csv` و PO rows |

### ۱.۲ Data Evidence (فایل‌های واقعی)

بررسی subset **Step≥4** (join روی PO↔RXL با `SetupID`):

| اجرا | setups با MaxStep≥4 | legs آن‌ها | join به RXL | MFEMoney کامل | MAEMoney کامل | MaxDDMoney کامل | MFE>0 | MAE>0 | MaxDD>0 |
|---|---|---|---|---|---|---|---|---|---|
| M0 (`R2`) | 431 | 1,915 | 431/431 | **431/431** | **431/431** | **431/431** | 421 | 431 | 431 |
| M2 (`R4`) | 376 | 1,673 | 376/376 | **376/376** | **376/376** | **376/376** | 367 | 376 | 376 |

- کل جمعیت RXL نیز کامل است: 4,864/4,864 (M0) و 4,782/4,782 (M2) — **0 empty و 0 zero** برای هر سه فیلد.
- نمونهٔ رکورد واقعی (deep، 5 پله): M0 SetupID **1032** → `MFEMoney=0.09, MAEMoney=299.90, MaxDDMoney=299.99, MaxDDPct=0.060091`؛ M2 SetupID **1015** → `MFEMoney=0.09, MAEMoney=299.90, MaxDDMoney=299.99, MaxDDPct=0.060098`.
- قرارداد علامت در داده نیز تأیید شد: `MAEMoney < 0` در هیچ ردیفی مشاهده نشد (magnitude).

### ۱.۳ Mapping — Setup یا Position/Leg؟

**PROVEN: سطح Setup.** مبنای کد: فیلدها روی `g_setups[idx]` (یک رکورد per setup) و نه per-position؛ basket از روی comment مشترک `BuildComment(setup_id)` ... یک مقدار واحد در RXL per SetupID (RXL دقیقاً 4,864/4,782 ردیف = یک ردیف per setup، کلید `[RunID,SetupID]` طبق Manifest). در PO dataset، مقادیر `SetupMaxDDMoney/SetupMaxDDPct` روی legrows فقط **تکثیر (replication)** از همان مقدار setup-level هستند (مثلاً هر ۵ leg ردیف SetupID=1032 مقادیر یکسانی دارند).

> نکته: MAE/MFE «از زمان فعال‌سازی setup» محاسبه می‌شود؛ چون update با وقفهٔ تیک روی سبد باز است، قبل از اولین ورود مقدار=0 می‌ماند (هیچ موقعیتی با comment matching نیست).

---

## ۲. DeepStop — تعداد دقیق و Reconcile قاعده

### ۲.۱ قاعدهٔ کد (Code Evidence)

`MQ5@4759646 :18604` داخل `PO_GroupMatch`:
```
case PO_GROUP_DEEPSTOP: return (r.setup_final_stop && r.setup_max_step >= 4);
```
هر دو فیلد ورودی (`SetupFinalStop`, `SetupMaxStep`) به‌صورت per-leg replicated در `PositionOpenDataset` موجودند — یعنی classification با همان قاعدهٔ کد، روی فایل دیتاست قابل‌محاسبه است.

### ۲.۲ Data Evidence (شمارش روی فایل‌های واقعی)

| اجرا | تعداد Setup با ‏FinalStop=1‏ (setup-level) | تعداد Leg آن‌ها | DeepStop Setup یکتا (FinalStop∧MaxStep≥4) | تعداد Leg دیپ | POSummary claim | Match |
|---|---|---|---|---|---|---|
| M0 (`R2`) | 88 | 383 | **70** | **332** | `STOP rows=383` / `DEEPSTOP rows=332` | ✓ exact |
| M2 (`R4`) | 90 | 365 | **53** | **257** | `STOP rows=365` / `DEEPSTOP rows=257` | ✓ exact |

- «383/332 و 365/257» در هدر summary نسبت به **ردیف‌های Leg** عدد می‌تواند (نه setup). این باعث تناقض‌های گذشتهٔ setup-vs-rows بود و اکنون **PROVEN** شده است.
- Parity لایه‌ای: `MarketEdge.StrategyStopFlag=1` شمارش = 88 (M0) و 90 (M2) = دقیقاً تعداد setupهای FinalStop هر اجرا ✓ (نکته: این flag «هر نوع stop» است، نه DeepStop — کامنت کد :2722 «SL / SL$ / STOP»).

---

## ۳. Entry Snapshot — زمان ایجاد، Shift، و وجود per-step

### ۳.۱ Code Evidence — دو لایهٔ snapshot جداگانه

**لایهٔ A — «Entry Snapshot» معنایی (per-setup، در زمان سیگنال):**
- `struct SEntrySnapshot` تعریف‌شده (:2136)؛ گلوبال `SEntrySnapshot g_current_snapshot;` (:9250) و آرایهٔ تجمیع `g_snapshots[]` (:5611).
- تابع ایجاد: **`CaptureEntrySnapshot(int setup_id, bool is_bull)` (:20884)** که در `FireSignal(bool, signal_shift=1, "CONFIRMED")` فراخوانی می‌شود (:17943) — یعنی **در هنگام تولید سیگنال (T0)، پیش از ورود**. یک فراخوانی دوم فقط-تحقیقاتی در `RP_RegisterConfirmedDivergence` (:16778) با الگوی copy/restore (بی‌اثر بر execution) وجود دارد.
- **Shift داخل این capture: 1** — نمونه‌های verbatim داخل بدنه (:~20940): `double atr_main = g_tf[0].GetBufferValue(g_tf[0].buffer_atr, 1);` و `ref_price = iClose(_Symbol, PERIOD_CURRENT, 1);` — آخرین‌بار بسته‌شده.
- valid نهایی regime سیگنال: `signal_shift` پیش‌فرض 1 = CONFIRMED؛ در الگوی LIVE_FORMING_P3 می‌تواند 0 باشد (print «RepaintRisk», :17936).

**لایهٔ B — رکورد snapshot ورود per-leg (در زمان fill):**
- تابع **`RecordPositionOpenBySetup` (:18345)** برای **هر پلهٔ 1..5 جداگانه** فراخوانی می‌شود: Step1 از مسیر ورود اولیه (:19661)؛ Stepهای 2..5 از `ManagePositions` هنگام fill پله (:42734) — با guard `step_no<1 || >5 return` (:18349-18350).
- **Shift = 0 (forming) در لحظهٔ fill** — verbatim :18391-18392: `// Refresh and capture the forming values that existed at actual fill time.` سپس `CopyIndicatorBuffers(...)` + `GetBufferValue(buffer_rsi|adx|cci|atr|plus_di|minus_di|ema_*, 0)` برای هر TF (:18396-18407)؛ guard صحت‌سنجی روی مقادیر (rsi/adx/atr/ema>0) قبل از ست `has_tf[tf]=true` (:18408-18411).
- تعریف ستون دیتاست: `IndicatorShiftUsed` ثابت 0 با کامنت کد «0 = live state captured at actual fill time» (:2841, assignment :18373, write :18731).

### ۳.۲ Data Evidence

- `PositionOpenDataset_XAUUSD.csv`: M0 = **9,605** ردیف (StepNo dist {1:4,864, 2:2,970, 3:1,149, 4:431, 5:191})؛ M2 = **9,380** ردیف ({4,782, 2,921, 1,132, 376, 169}) — یعنی **برای Step1 تا Step5 ردیف مستقل وجود دارد** (هر leg یک snapshot at-fill).
- ستون `IndicatorShiftUsed=0` در **تمام** ردیف‌های هر دو فایل (0 استثنا).
- منحصر‌بودن: dup روی (SetupID×StepNo) = 0 در هر دو اجرا.

### ۳.۳ نتیجه‌گیری دقیق برای مورد ۳

- Entry Snapshot «معنایی» (SEntrySnapshot) **یک‌بار per setup در زمان سیگنال** ایجاد می‌شود — نه per step؛ shift=1.
- Per-step snapshotها **فقط در قالب ردیف‌های PositionOpenDataset** دارای وجود پایدار هستند (به‌جای دیتاست جداگانه) — per leg، shift=0 at-fill. تشکیلات feature-side (formation shift-1) اما در `RX Features` به‌صورت یک ردیف per setup نمود می‌یابد (۶ ستون `*_Shift=1` ∀).
- ادعای «فقط PositionOpenDataset داریم» برای per-step دقیق است؛ اما ادعای «وجود یک snapshot واحد shift-1 at-signal» هم صحیح است — **دو لایه مجزا با دو shift متفاوت**.

---

## ۴. D1 / W1 — چه featureهایی واقعاً ذخیره شده‌اند

### ۴.۱ Code Evidence — تفکیک handle-based از price-derived

**Indicator handle با D1/W1: هیچ (FALSE).** جست‌وجوی کامل `iRSI|iADX|iCCI|iATR|iMA` × `PERIOD_D1|PERIOD_W1` در کل MQ5 → **صفر نتیجه**. تمامی کاربردهای `PERIOD_D1/W1` مربوط به دو mapping enum/string (:13469, :13480-13481) و **خواندن قیمت خام** (iOpen/iHigh/iLow + iBarShift) است.

**Price-derived D1/W1 «در کد موجود» (درون `SEntrySnapshot`):**
| تابع (خط) | فیلدها | ورودی خام |
|---|---|---|
| `CaptureLiquiditySweepFeatures` (:20139, block :20204-20215) | `distance_to_prior_day_high_atr`, `distance_to_prior_day_low_atr` | `iHigh/iLow(PERIOD_D1, cur_day_bar+1)` — روز قبیلِ کامل |
| `Fill_Distance` (:20731, block :20738-20765) | `dist_day_open/high/low_atr`, `day_range_position`, `dist_week_open/high/low_atr`, `week_range_position` | `iOpen/iHigh/iLow(PERIOD_D1/W1, bar(entry_time))` — روز/هفتهٔ جاری تا entry_time |

### ۴.۲ Data Evidence — در دیتاست‌ها: هیچ (FALSE for storage)

Scan هدر همهٔ خانواده‌های dataset هر دو اجرا (`RX Features` 358 ستون، `MarketEdge` 303، `PositionOpenDataset` 370، `Step1OutcomeComparison` 279، `RX Labels` 33) بر اساس الگوهای `PriorDay|DistDay|DistWeek|DayRange|WeekRange|D1_|W1_|Daily|Weekly` → **صفر ستون D1/W1 در هیچ‌یک**.

**تنها ردِ تشخیصی در کد:** enumٔ Stage2B با نام `S2B_F_DIST_PRIOR_DAY_HIGH_ATR` ↔ عنوان `"DistanceToPriorDayHighATR"` (:27453/:27479/:27570) که از `g_snapshots[si].distance_to_prior_day_high_atr` می‌خواند — یعنی «در حافظه، برای ماشینری StopFlag Stage2B». خروجی‌های Stage2B (مثلاً `Stage2B_Stop_D2024_*`) به‌صورت `FILE_COMMON` تولید می‌شوند و در درخت‌های run قابل‌ردیابی git **وجود ندارند** → محتوایشان NOT PROVEN. در فایل‌های run چهارگانهٔ ما ذخیره‌سازی D1/W1 رخ نداده است.

---

## ۵. 366 / 360 / 358

| عدد | اتصال قطعی به artifact | نوع (population یا schema count) | وضعیت |
|---|---|---|---|
| **366** | `legacy/CF_events_m0.csv` @ commit R1 `662a7ec` — شمارش first-per-setup رخداد A(3→4) با `DecisionTime < 2024.01.01` (کل: 431 → DEV 366 / OOS 65؛ DEV cutoff با داده: آخرین DEV `2023.12.15 09:04:34`، اولین OOS `2024.01.03 17:41:54`) | **population/event count** (نه schema) — هیچ schema با عرض 366 وجود ندارد | PROVEN |
| **358** | **دو اتصال تأییدشده:** (الف) رکوردهای `C2M2_M0_Shift1_Population_DEV.csv` @ commit R3 `2f04d01` — population واقعی τ (SHA و Q75 replay exact: τ_ATR=0.04668375، τ_ADX=47.50305)؛ (ب) **عرض schema فایل‌های `RX_..._Features.csv` هر دو اجرا = 358 ستون** (M0 و M2 هر دو). مبنای تحلیلی تنها (الف) است | **population count (a)** + اتفاقا **schema column count (b)** — هر دو PROVEN؛ ربط‌دهی غلط این دو به هم، منشأ آمبوی است | PROVEN |
| **360** | در هیچ artifact قابل‌ردیابی (md/docs/datasets/counts) به‌عنوان count/population/schema یافت نشد. تنها مقادیر cell-level تصادفی مثل `SetupID=360`، `RecordID=360`، `SetupDurationMin=360` در PO/CF/RXL — با معنای نمایشی unrelated | هیچ | **NOT PROVEN (NOT FOUND)** |

### ۵.۱ اختلاف ۸ — فقط با evidence قطعی

- اتصال داده‌ای: DEV-A34 مجموعه (366) − population (358) = دقیقاً ۸ SetupID: `{169, 470, 1072, 1431, 2656, 2779, 3112, 3360}` (مجموعهٔ ۷۳تایی مذکور در sweeps شامل 65 مورد OOS است که از پنجرهٔ population خارج‌اند — مقایسهٔ درست DEV-only است).
- مکانیزم حذف (Code Evidence): دو گیت در کد: `C2M2_Step34ShouldVeto` (:19430; گیت :19477) و `C2M2_CollectM0Step34Decision` (:19513; گیت :19557-19558) با منطق `if(!ok_data || !strict_before) return false;` و کامنت «Invalid/strict-before-failing ticks are ignored and no latch is set» → collect تک‌شلیک‌آمیز روی رخداد اولِ هر setup؛ **وقتی ورود به Step4 رخ دهد دیگر امکان ثبت نیست** و latch FROM ریجکت پس‌زا می‌شود (only-before-entry «later tick can be used»).
- علت دقیق tick-سطح برای هر ۸ مورد: نیازمند snapshot لحظه‌ای وضعیت strict-before روی همان تیک (dataset ثبت نشده) → **NOT PROVEN**. (الگوی مشاهده‌ای—not due to sec :00 علت نیست: در CF هر ۸ DecisionTime إلى `:00` ختم می‌شوند ولی ۲۲ setup DEV دیگر با همان الگو «داخل» population هستند — پس الگو باعث نمی‌شود؛ قبلاً falsified.)

---

## ۶. جدول نهایی: CLAIM | CODE EVIDENCE | DATA EVIDENCE | STATUS

| # | CLAIM | CODE EVIDENCE | DATA EVIDENCE | STATUS |
|---|---|---|---|---|
| 1 | UpdateDrawdown در هر تیک برای setup active از Step1 تا Step5 اجرا می‌شود | OnTick:46832→ManageSetups:46887→CheckTakeProfit(i):42306→UpdateDrawdown:43121 شاخهٔ `current_step>=4` و :43164 مسیر Step≤3 | القا از کد؛ داده با وجود مقادیر کامل در هر MaxStep سازگار — ببینید سطر ۲ | **PROVEN** |
| 2 | MAE/MFE/MaxDD در Step≥4 به‌صورت کامل ثبت شده‌اند | شاخهٔ Step≥4 صریح :43115-43122 + comment «keep MAE/MFE/MaxDD fresh at Step>=4» (:43118-43120) + تعریف محاسبه :43470-43509 | M0: 431/431 کامل (مقادیر نمونه SetupID=1032)؛ M2: 376/376 (SetupID=1015)؛ کل‌جرم: 4,864/4,864 و 4,782/4,782 بدون empty/zero | **PROVEN** |
| 3 | mapping این مقادیر setup-level است (نه per-position) | grepذخیره روی `g_setups[idx].mfe/mae/maxdd` (struct :2398-9)؛ basket با `POSITION_COMMENT==BuildComment(setup_id)` (:43475-43485)؛ emit per setup :13157/:13233 | RXL دقیقاً 1 ردیف per SetupID؛ PO rows مقادیر تکراری یکسان per setup (مثال: هر ۵ leg=1032 مقادیر برابر) | **PROVEN** |
| 4 | MAE به‌صورت magnitude مثبت ذخیره می‌شود | کامنت کد :43507 «مثبت ذخیره می‌شود (magnitude)» + منطق `mae_money = -profit;` فقط وقتی profit<0 (:43508-43509) | `MAEMoney < 0`: 0 ردیف در هر دو اجرا | **PROVEN** |
| 5 | تعداد DeepStop setup یکتا = M0: 70 / M2: 53 | rule کد `final_stop && max_step>=4` (:18604) | شمارش روی PO rows با شرط اتحاد (SetupFinalStop=1 ∧ max StepNo≥4): 70/53 exact؛ پله‌ها مستقلاً: 88/90، legs 383/332 و 365/257؛ match با POSummary | **PROVEN** |
| 6 | اعداد 383/332 و 365/257 تعداد Setup نیستند؛ Leg rows هستند | generator هدر POSummary با group_name «SETUP_STOP/SETUP_DEEPSTOP» بر اساس row iteration (:19004-19200‌ها) | Σ legs روی مجموعه‌ها در دیتاست: 383/332 (M0)، 365/257 (M2) — برابر مقدار چاپ‌شده؛ deep⊂stop (70⊂88، 53⊂90) | **PROVEN** |
| 7 | Entry Snapshot «معنایی» یک‌بار per setup در زمان سیگنال با shift=1 ایجاد می‌شود | `CaptureEntrySnapshot` :20884 فراخوانی از `FireSignal` :17943 (`signal_shift=1`، CONFIRMED)؛ `GetBufferValue(...,1)`/`iClose(...,1)` داخل بدنه | حمل به RXF/ME (FeatureShift=1، `bar=last_completed` در Manifest؛ 6×*_Shift=1 در RXF) | **PROVEN** |
| 8 | snapshot مستقل برای هر Step1..Step5 وجود دارد — در قالب رکوردهای PositionOpenDataset با shift=0 at-fill | `RecordPositionOpenBySetup(setup_idx, step_no, …)` :18345؛ guard step 1..5 :18349؛ call sites :19661/:42734؛ verbatim «forming values at actual fill time» + reads shift 0 (:18390-18420)؛ ستون `IndicatorShiftUsed=0` (:2841, :18373) | 9,605 ردیف M0 / 9,380 ردیف M2 با dist {1..5}؛ dup(SetupID×StepNo)=0؛ IndicatorShiftUsed=0 در 100% ردیف‌ها | **PROVEN** |
| 9 | ستون D1/W1 «indicator-handle» در dataset ذخیره شده | — (نبود handle: `iRSI/iADX/iCCI/iATR/iMA` × D1/W1 = 0 hit) | — (نبود ستون: scan هدر 5 dataset اصلی = 0) | **FALSE** (نه در کد handle‌ای، نه در dataset) |
| 10 | ستون D1/W1 «price-derived» در dataset ذخیره شده | فیلدهای موجود در `SEntrySnapshot` (:20204-20215، :20738-20765) | scan هدر همهٔ datasetها = 0 ستون؛ تنها رد: enum حافظهٔ Stage2B (:27453/:27479/:27570) و خروجی‌های FILE_COMMON غایب از tracked refs | **FALSE** (به‌عنوان ستونِ dataset قابل‌ردیابی) |
| 11 | 366 = population/event count رخداد A3→4 در DEV (نه schema count) | منطق event-logger legacy (هدر CFEvents :18643+ / CF_ShouldVeto shift-0) | `CF_events_m0.csv` @662a7ec: 19,087 ردیف؛ first-per-setup A34=431؛ DEV(<2024.01.01)=366؛ OOS=65 | **PROVEN** |
| 12 | 358 = population واقعی τ (و اتفاقاً عرض schema RxF) | گیت collect `C2M2_CollectM0Step34Decision` :19513 با latch FIRST/ALLOW؛ برچسب‌های ردیف `M0_COLLECT/0/true/ALLOW/FIRST` | `C2M2_M0_Shift1_Population_DEV.csv` @2f04d01: 358 ردیف؛ SHA=docs-hash ✓ و Q75 replay exact (τ_ATR/τ_ADX)؛ RXF cols=358 (هر دو اجرا) | **PROVEN** |
| 13 | 360 به‌عنوان population/schema count به artifact معتبری متصل است | — | sweep whole-word: هیچ؛ فقط cell-value تصادفی (SetupID=360 و …) | **NOT PROVEN** |
| 14 | علت tick-سطح حذف ۸ setup مفقود از population | مکانیزم گیت (:19557-19558) PROVEN؛ علت هر ۸ مورد نیازمند snapshot تیک (موجود نیست) | فقط الگوی مصادفی `:00` — قبلاً falsified | **NOT PROVEN** |
| 15 | سیاست «لا تغيير کد» رعایت شده | — | diff گزارش فقط MD اضافه‌شده؛ بدون تغییر mq5/params | **PROVEN** |

---

*پایان Verification Evidence Report V1 — بدون حدس/تفسیر؛ هر وضعیت فقط یکی از PROVEN / PARTIAL / NOT PROVEN / FALSE.*
