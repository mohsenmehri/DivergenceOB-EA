# C2/M2 — Data/Artifact Audit & Reconciliation (V1)

**تاریخ گزارش:** 2026-09-12 (Asia/Tehran)
**نوع مأموریت:** صرفاً Audit و Reconciliation مبتنی بر شواهد. **هیچ تغییر کدی، تغییر threshold، اجرای backtest جدید یا پیشنهاد استراتژی انجام نشده است.**
**قاعدهٔ خروجی:** برای هر ادعا — مسیر فایل، نام ستون، تعداد رکورد، و در صورت امکان نمونهٔ رکورد. پایان هر مورد با یکی از: **PASS / PARTIAL / BLOCKED / NOT PROVEN**.
**دامنهٔ اجراها (read-only):** تمام محتوای زیر از آبجکت‌های git مخزن `mohsenmehri/DivergenceOB-EA` استخراج و روی نسخه‌های محلی آن‌ها محاسبه شده است. ابزار `git-lfs` در محیط اجرا نصب نیست؛ بنابراین فایل‌های LFS-pointer به‌صورت «BLOCKED» با oid ثبت‌شده گزارش می‌شوند.

---

## ۱. خلاصه اجرایی یافته‌ها (Headline)

| # | ادعا/سؤال | نتیجهٔ تأییدشده |
|---|---|---|
| A | مبدأ **366** | = تعداد رخدادهای first-per-setup پیمانی A3→4 در پنجرهٔ DEV (DecisionTime < `2024.01.01`) در فایل legacy `CF_events_m0.csv` — **اجماع shift-0**. Match دقیق. |
| B | مبدأ **358** | = تعداد رکوردهای `C2M2_M0_Shift1_Population_DEV.csv` — **population واقعی استخراج τ با shift-1**. بازتولید Q75 به‌صورت byte-exact تأیید شد. |
| C | رابطهٔ 358 و 366 | 358 ⊂ 366 با **دقیقاً ۸ SetupID مفقود** `{169,470,1072,1431,2656,2779,3112,3360}`؛ مکانیزم حذف در کد piece آی‌دی‌محور تأیید (single-shot collect). علت tick-سطح هر ۸ مورد از artifactهای موجود → **NOT PROVEN**. |
| D | مبدأ **360** | در هیچ artifact قابل‌ردیابی git یافت نشد → **NOT FOUND/NOT PROVEN**. |
| E | اصطلاح STOP/DEEPSTOP در POSummary | مقادیر 383/332 و 365/257 «تعداد رکوردهای leg (سطر)» هستند، نه setup. بازسازی با قاعدهٔ کد exact match دارد (§۸). |
| F | قاعدهٔ DeepStop | از کد استخراج و روی داده بازتولید شد: `setup_final_stop && setup_max_step >= 4` (MQ5@4759646:18604) — setup-level DeepStop **قابل‌محاسبه است**. |
| G | MAE/MFE setup-level | **موجود است** (RX Labels، کامل در هر دو اجرا) با تعریف کاملاً کدشده (MQ5@4759646:43505-43509). یافتهٔ قبلی «غایب» بودنِ این فیلدها بر اساس شواهد جدید اصلاح می‌شود (§۹ — تصمیم نهایی با مالک). |
| H | ادعاهای متضاد Shift=0/1 | **ناسازگاری مستندسازی واقعی PROVEN**: رکوردهای PO با ستون `IndicatorShiftUsed=0` (= live state at fill، کد:2835-2841/18373) ولی هدر POSummary ادعای `(shift=1)` دارد؛ exporter آمار‌ها همان رکوردهای shift-0 را می‌خواند (کد:19020-19023). جزئیات §۵. |
| I | خط‌سیر τ | population SHA-256 برابر هش مستند: `09d0a302…` ✓؛ برچسب‌ها: SourceCommit `061a1bb4…`, Spec `C2_M2_M0_POPULATION_COLLECTION_V1`, Build `C2M2_M0COLLECT_V1` ✓؛ freeze commit `c995cdf` (τ_ATR=0.04668375, τ_ADX=47.50305) ✓. |

---

## ۲. ثبت اجراها (Run Registry) — مسیر git، کامیت، تاریخ، وضعیت محتوا

| ID | نقش اجرا | مرجع git | تاریخ کامیت | وضعیت محتوای فایل‌ها |
|---|---|---|---|---|
| **R1** | اجرای legacy CF (shift-0) | `FETCH_HEAD = 662a7ec3…` روی شاخهٔ `agent-regression-data-real`، پوشهٔ `v2_research_export_v2_cf/` | 2026-09-07 15:28:51 +0330 («add») | real content برای فایل‌های متنی (`CF_events_m0.csv`, `PO_cf.csv`, …). log اصلی `20260905.log`. |
| **R2** | اجرای matched M0 (reference) | `3352b18:M0/` (orphan root commit) | 2026-09-10 11:27:53 +0330 | real content؛ ۱۷ فایل (فهرست §۳). |
| **R3** | اجرای population-collection (M0 Shift-1) | `2f04d019…:v2_research_export_v2_cfv2/` | 2026-09-08 08:05:59 +0330 | **population CSV = real text (95,671B، SHA تأیید)**؛ POSummary/Stats/Manifest/SetupID real؛ **بقیهٔ bulk CSVها = Git-LFS pointer (BLOCKED، §۱۰)**. |
| **R4** | اجرای M2 (decision) | `0e918480…:v2_research_export_v2_cfv2/` | 2026-09-08 12:29:09 +0330 | real content برای همهٔ CSVها و xlsx؛ `20260908.log` = LFS pointer (BLOCKED). |

**روابط بین درخت‌ها (با blob-hash سنجیده‌شده):**
- R3 ↔ R2: چهار فایل کوچک byte-identical (`PositionOpenSummary`=5462499, `Stats`=655b0dd, `Manifest`=23cfa09, `SetupID_XAUUSD.txt`=95c50b3). تمام bulk CSVهای R3 pointer هستند → هم‌محتوایی با R2 **قابل‌اثبات نیست** (هش‌ها متفاوت/نامقطوع).
- R1 ↔ R2: `PositionOpenSummary_XAUUSD.txt` و `Stats_XAUUSD.txt` **byte-identical** (md5: `e0c2b311…`, `c1cebdd3…`). برتری SetupID-space: 4,864/4,864 setup بین R1 (`PO_cf.csv`) و R2 (matched M0) یکسان با `OpenTime` و `FillPrice` یکسان به‌ازای همهٔ IDها (0 اختلاف) → **همان پایهٔ backtest**؛ بنابراین مقایسهٔ A3→4 بین R1 و R3/R2 روی فضای SetupID مشترک معتبر است.
- R4: درخت مستقل با bulk CSVهای real و اعداد متفاوت (4,782 setup).

---

## ۳. موجودی Datasetها (مسیر / ستون / رکورد / کلید / توزیع Step / Shift / زمان)

### ۳.۱ — R2 (M0 matched, `3352b18:M0/`)

| فایل | رکورد داده | ستون | SetupID یکتا | نکات کلیدی |
|---|---|---|---|---|
| `PositionOpenDataset_XAUUSD.csv` | 9,605 | 370 | 4,864 | leg-level؛ StepNo dist: {1:4,864، 2:2,970، 3:1,149، 4:431، 5:191} (Σ=9,605 ✓)؛ `IndicatorShiftUsed=0` در **تمام** ردیف‌ها؛ dup روی (SetupID×StepNo)=0؛ PositionID یکتا per leg |
| `RX_..._Features.csv` | 4,864 | 358 | 4,864 | هر ۶ ستون `*_Shift` = **1 در تمام ردیف‌ها**؛ `H4_ADX` empty=47 (کمبود سوابق H4) |
| `RX_..._Labels.csv` | 4,864 | 33 | 4,864 | `IsCensored=1` در همه (CENSORED_EOT_OPEN)؛ `EventPath` empty=80 (A34 inactive؛ قبلاً شناخته‌شده)؛ `MFEMoney/MAEMoney/MaxDDMoney/MaxDDPct` **کامل** (0 empty، 0 zero) |
| `RX_..._SetupLifecycleEvents.csv` | 28,447 | 8 | — | RunID=`RX_1388707200_XAUUSD` |
| `RX_..._Step1OutcomeComparison.csv` (S1C) | 2,325 | 279 | ⊂PO | SingleStepN=1,894 / Deep45N=431 |
| `MarketEdge_..._XAUUSD.csv` | 18,892 | 303 | — | `FeatureShift_M1=1` در همه؛ `WasExecuted=1`→4,864؛ `StrategyStopFlag=1`→**88** |
| `PositionOpenIndicatorStats_XAUUSD.csv` | آمار تجمیعی pivot | — | — | تولید از روی همین PO rows (کد §۵) |
| `Stats_XAUUSD.txt` | 256B | — | — | محتوا: `NextSetupID=4865; Stat0=4864 …Stat2=1894 …Stat6=191` |
| `PositionOpenSummary_XAUUSD.txt` | 5,876B | — | — | STOP rows=383، DEEPSTOP rows=332 (§۸) |
| بقیه (`ProfileStats`, `SetupID.txt`, `ReportTester-91306235.xlsx`, `DatasetQualityReport.html`) | — | — | — | — |

### ۳.۲ — R4 (M2, `0e918480…:v2_research_export_v2_cfv2/`)

| فایل | رکورد داده | SetupID یکتا | نکات کلیدی |
|---|---|---|---|
| `PositionOpenDataset_XAUUSD.csv` | 9,380 | 4,782 | StepNo dist: {1:4,782، 2:2,921، 3:1,132، 4:376، 5:169} (Σ=9,380 ✓)؛ `IndicatorShiftUsed=0` در همه؛ `ReportTester-91306235.xlsx` = real (PK) |
| `RX_..._Features.csv` | 4,782 | 4,782 | ۶ ستون `*_Shift`=1 در همه |
| `RX_..._Labels.csv` | 4,782 | 4,782 | MAE/MFE کامل (0 empty/0 zero)؛ EventPath empty=80 |
| `MarketEdge_..._XAUUSD.csv` | 18,892 | — | **device universe سیگنال با R2 یکسان (18,892 ردیف)**؛ WasExecuted=1→4,782؛ `StrategyStopFlag=1`→**90** |
| `RX_..._SetupLifecycleEvents.csv` | 27,957 | 8 cols | — |
| `Step1OutcomeComparison.csv` | 2,237 | 279 cols | — |
| `PositionOpenSummary_XAUUSD.txt` | — | — | STOP rows=365، DEEPSTOP rows=257 (§۸) |

> نکتهٔ روش: مجموعهٔ سیگنال MarketEdge بین دو اجرا عوض نمی‌شود (18,892 سیگنال؛ همان detector)؛ تفاوت M0↔M2 روی *قبول‌سازی اجرا* است (4,864 در برابر 4,782).

### ۳.۳ — R3 (collect, `2f04d019…:v2_research_export_v2_cfv2/`)

| فایل | اندازهٔ blob | وضعیت |
|---|---|---|
| `C2M2_M0_Shift1_Population_DEV.csv` | 95,671B | **real** ✓ SHA-256=`09d0a3028520a499624a82f0…` (برابر مستندات) |
| `PositionOpenSummary_XAUUSD.txt`, `Stats_XAUUSD.txt`, `RX_..._Manifest.json`, `SetupID_XAUUSD.txt` | real | byte-identical با R2 |
| `PositionOpenDataset/IndicatorStats/RX Features/RX Labels/ME/S1C×2/ProfileStats/ReportTester.xlsx/20260907.log` | 128–134B | **LFS pointer — BLOCKED** (oidها §۱۰) |

### ۳.۴ — R1 (legacy `_cf`, `662a7ec3…`)

| فایل | رکورد | کلیدها |
|---|---|---|
| `CF_events_m0.csv` | 19,087 | Event∈{A:9,605, P:4,741, F:4,741}؛ `Mode='0'` در همه؛ `Rule∈{NONE:9,605, M0:9,482}`؛ A3→4 first-per-setup=**431** (DEV=366، OOS=65) |
| `PO_cf.csv` | 9,605 legs | ایکّر با R2 per (SetupID, OpenTime, FillPrice) — مطابقت کامل |
| POSummary/Stats | — | byte-identical با R2 |

---

## ۴. تطبیق تعاریف‌های Dataset با کد منبع

منبع کد مرجع: `v2_research_export_v2_cf.mq5` @ blob شاخهٔ `arena/01a06d3c` tip **`4759646`** (build ذخیره‌شده در git). شماره‌خطوط از همان blob.

| تعریف ادعاشده | شواهد کدی | نتیجه |
|---|---|---|
| `PositionOpenDataset` (entry-level، leg rows) | header ساخته‌شده در :18643-18652؛ فیلد `indicator_shift_used` با کامنت «0 = live state captured at actual fill time» (:2841) و مقداردهی `rec.indicator_shift_used = 0;` در :18373 و خروجی :18731 | تعریف = «snapshot زندهٔ forming-bar در لحظهٔ fill»؛ **PASS** (کد↔داده سازگار) |
| `RX Features` (formation snapshot shift-1) | Manifest (`..._Manifest.json`) capture_policy: `bar=last_completed, shift=1, slope_lookback=5`؛ schema_version `RX_STEP1_DATASET_V1`؛ join_key `[RunID,SetupID]`؛ na_representation=`empty_field`؛ ۶ ستون `M1/M5/M15/M30/H1/H4_Shift`=1 در همهٔ 4,864/4,782 ردیف | **PASS** |
| «Entry Snapshot» به‌عنوان موجودیت مستقل | فایل جداگانه‌ای به این نام وجود ندارد؛ معادل عملیاتی آن همان رکوردهای PO dataset است (SignalSource ∈ {CONFIRMED, LADDER}) | **PARTIAL** (نام=مفهوم، نه دیتاست مستقل) |
| `MarketEdge` | 303 ستون، 18,892 ردیف در هر دو اجرا؛ `FeatureShift_M1=1`؛ فیلد `StrategyStopFlag` با کامنت ساختاری `SL / SL$ / STOP` (:2722) | تعریف = «وقوع hit هر نوع stop»، نه DeepStop به‌معنای خاص (**مهم**: رجوع به §۸) |
| `C2M2 Step34 Decisions` / population | header دقیق population (§پیوست-نمونه‌ها)؛ ثابت‌های `C2M2_FEATURE_SHIFT=1`, `C2M2_LATCH_ALLOW`؛ emission تنها پس از قبول | **PASS** |

---

## ۵. سنجش Shift با نمونهٔ واقعی — و **ناسازگاری مستندسازی PROVEN**

**سه دستگاه با Shift مشخصِ رکوردی/کدی:**

۱. **لایهٔ Entry-legs (PO dataset): shift-0 (live at fill).**
   - شواهد رکوردی: ستون `IndicatorShiftUsed=0` در **تمام** 9,605 و 9,380 ردیفِ هر دو اجرا (0 استثنا).
   - شواهد کدی: کامنت صریح «0 = live state captured at actual fill time» (:2841)؛ مقداردهی :18373.
۲. **لایهٔ Formation/Labels (RXF/ME/C2M2): shift-1.**
   - رکوردی: هر ۶ ستون `*_Shift` در RXF = 1 (نمونهٔ واقعی بخش پیوست-۳)؛ `FeatureShift_M1=1` در هر 18,892 ردیف ME (هر دو اجرا)؛ ستون `FeatureShift=1` در هر 358 ردیف population و در C2M2 decisions.
   - مستندی: Manifest — `capture_policy.shift=1`, `bar=last_completed`.
۳. **لایهٔ legacy CF: shift-0 (از کد).**
   - `CF_ShouldVeto` با `GetBufferValue(..., 0)` و `iClose(..., 0)` — literal shift-0؛ فایل CF فاقد ستون shift است؛ این یافته code-level است، نه رکوردی.

**⚠ ناسازگاری متخذ PROVEN (اتفاقاً همان چیزی که «ادعاهای متضاد» معروف است):**
- هدر فایل `PositionOpenSummary_XAUUSD.txt` صریحاً می‌نویسد:
  `Note : Indicator snapshot uses latest closed bar at position open (shift=1)`
- ولی (الف) همهٔ رکوردهای dataset، که exporter آمار‌های summary از همان‌ها تولید می‌کند — exporter در :18974-19023 مستقیماً روی آرایهٔ `g_position_open_records[]` (همان رکوردهای shift-0) پیمایش می‌کند — shift-0 هستند؛ (ب) نتیجه: **آمارهای IndicatorStats عملاً از snapshotهای shift-0 اتخاذ شده‌اند، درحالی‌که note آن‌ها را shift-1 توصیف می‌کند.**
- بارزش: این یک ناسازگاری *مستندات↔داده↔کد* است؛ نه انشعاب در عملکرد ترید. پیشنهادی برای تغییر ارائه نشده است (خارج از scope مأموریت) — صرفاً ثبت شد.

---

## ۶. Join ها و کنترل‌های یکپارچگی

| چک | روش | نتیجه |
|---|---|---|
| یکنواختی فضای setup درون هر اجرا | set-equality روی (RXF, RXL, LC, ME-executed, PO) | R2: برابری کامل با 4,864 ID؛ R4: 4,782 ID — **PASS** |
| S1C ⊂ PO | subset-check ronyه SetupID + step | R2: 2,325 ⊂ 4,864؛ R4: 2,237 ⊂ 4,782 — **PASS** |
| تکرار (SetupID×StepNo) | groupby-count>1 | 0 در هر دو اجرا — **PASS** |
| یکتایی PositionID per leg | unique check | یکتا — PASS |
| هم‌خطایی R1↔R2 روی فضای SetupID | join روی (SetupID, OpenTime, FillPrice) | 4,864/4,864 مطابقت کامل، **0 اختلاف** — **PASS** |
| parity متقابل StopFlag | ME.StrategyStopFlag=1 vs union-PO.SetupFinalStop=1 | R2: 88=88؛ R4: 90=90 — **PASS** |
| سیاست NA/empty | نمونه‌گیری و شمارش | جدول §۶.۱ — **PASS** |

### ۶.۱ سیاست‌های NA/Empty مستند

| Dataset | ستون | سیاست مشاهده‌شده |
|---|---|---|
| PO | `ResearchP3Time=1970.01.01`, `ResearchJoinStatus=PROTOCOL_DISABLED` (همهٔ ردیف‌ها) | sentinel ثابت — پروتکل P3 غیرفعال |
| RXF | `H4_ADX` empty=47 | کمبود سوابق H4 در ابتدای بازه |
| RXL | `EventPath` empty=80 (هر دو اجرا) | رخداد A34 inactive |
| ME | Exec-fields/`BarsToTarget` empty 7,142 (R2) / 7,? (R4) | تنها برای سیگنال‌های اجراشده/censored |
| Manifest | `na_representation=empty_field` | CSV با BOM/`, `/crlf |

---

## ۷. آشتی 366 / 358 / 360 — جدول نهایی

| عدد | origin قطعی | محتوای واقعی |
|---|---|---|
| **366** | `legacy/CF_events_m0.csv` (R1) | first-per-setup رخداد A3→4 با `DecisionTime < 2024.01.01`: 431 دسته‌بندی‌شده → DEV=366، OOS=65. آخرین DEV A34 = `2023.12.15 09:04:34`؛ اولین OOS = `2024.01.03 17:41:54`. |
| **358** | `pop/C2M2_M0_Shift1_Population_DEV.csv` (R3) | population واقعی shift-1 (تمام ستون‌ها: ALLOW / Mode=0 / Rule=M0_COLLECT / FeatureShift=1 / Latch=FIRST / strict_before_check=true)؛ بازه: `2014.01.08 00:14:04 → 2023.12.15 09:04:34` = دقیقاً آخرین DEV فوق ✓؛ type-7 Q75 روی این 358 ردیف → **τ_ATR=0.04668375 / τ_ADX=47.50305** (بازتولید exact). |
| **360** | — | جست‌وجوی کامل در artifactهای ردیابی‌شده (`.md` در HEAD `0cc723b`، tip شاخهٔ داده `4759646`، origin/main و هدرهای دیتاست‌ها) → **صفر نتیجه. مبدأ NOT FOUND → NOT PROVEN**. |

**دلتا 366→358 = دقیقاً ۸ SetupID:** `{169, 470, 1072, 1431, 2656, 2779, 3112, 3360}`
- مکانیزم حذف (اجماع کد): در MQ5@4759646 حدود خطوط 19556–19564: فقط تیک‌های «valid» وارد population می‌شوند؛ «Invalid/strict-before-failing ticks are ignored and no latch is set» — `if(!ok_data || !strict_before) return false;`؛ collect تک‌شلیک است و پس از ورود به پلهٔ 4 دیگر امکان ثبت وجود ندارد.
- ترانه به‌زودی: وقوع microscopy بین tick و BarClose در retained data وجود ندارد → علت میان‌تیک دقیق هر ۸ setup از artifactهای حاضر **NOT PROVEN** (قبلاً فرضیهٔ sec==00 falsify شده: ۳۰ رویداد A34 با `:00` وجود دارند که ۲۲ تای آن‌ها داخل 358 هستند).
- حافظهٔ نمونهٔ CF برای سه مورد از ۸ (پیوست-۴) تأیید می‌کند DecisionTime آن‌ها با `:00` ختم می‌شود (الگو، نه علت).

**جمع‌بندی 366/358/360:** 366 **PASS** / 358 **PASS** / Δ(8) mechanism **PASS** + per-ID cause **NOT PROVEN** / 360 **NOT PROVEN**.

---

## ۸. STOP و DEEPSTOP — از اصطلاح‌گیجی تا Match

**اختلاف کلیدی اصطلاحات (علت بسیاری از Nهای متناقض):**
`PositionOpenSummary_XAUUSD.txt` می‌گوید:
```
Setup STOP rows      : 383        (R2)   /   365 (R4)
Setup DEEPSTOP rows  : 332        (R2)   /   257 (R4)
```
✅ این اعداد **تعداد ردیف‌های leg** هستند، نه تعداد setup. بازسازی روی PO dataset:

| اجرا | Stop setups (union `SetupFinalStop=1`) | Σ legs آن‌ها | DeepStop setups (rule کد) | Σ legs دیپ |
|---|---|---|---|---|
| R2 (M0 matched) | 88 | **383 = claim ✓** | 70 | **332 = claim ✓** |
| R4 (M2) | 90 | **365 = claim ✓** | 53 | **257 = claim ✓** |

**قاعدهٔ DeepStop (از کد، :18604):**
```
case PO_GROUP_DEEPSTOP: return (r.setup_final_stop && r.setup_max_step >= 4);
```
با دو ستون مستقیم داخل PO dataset (`SetupFinalStop`, `SetupMaxStep`؛ هر دو replicated روی هر leg پس از finalization) → **DeepStop setup-level کاملاً قابل‌محاسبه و RECONCILED** است:
- R2: 88 / 70 ; R4: 90 / 53 .
- parity با ME: `StrategyStopFlag=1` در ME = تعداد stop setups (any-stop) → 88 (R2) / 90 (R4) ✓. (`StrategyStopFlag` ≠ DeepStop؛ کامنت کد = «SL / SL$ / STOP»).

**جمع‌بندی:** `DeepStop = FinalStop ∧ MaxStep≥4`: PASS (کد + داده + aggregate + متن مستند، کمی‌سازی‌شده روی هر دو اجرا).

---

## ۹. MAE/MFE setup-level — تصحیح یافتهٔ پیشین (با شواهد)

**یافتهٔ مستقیم داده:** ستون‌های `MFEMoney / MAEMoney / MaxDDMoney / MaxDDPct` در هر دو `RX_..._Labels.csv` **کامل** هستند: R2: 4,864/4,864 بدون empty و بدون 0؛ R4: 4,782/4,782. قرارداد علامت: `MAEMoney ≥ 0` در همه (counter منفی=0) — یعنی magnitude.
نمونهٔ واقعی (R2، SetupID=4): `MFEMoney=2.13, MAEMoney=21.33, MaxDDMoney=21.36, MaxDDPct=0.004272, IsCensored=0, CensorReason="", EventPath=""`.

**منشأ محاسبه از کد (MQ5@4759646):**
- reset: :2639-2640 و :12657-12658 (`mfe_money = mae_money = 0` per setup).
- update — :43480-43509: سود شناور سبد (`profit = Σ POSITION_PROFIT + POSITION_SWAP` برای پوزیشن‌های همان setup) در هر تیک ارزیابی می‌شود؛
  `mfe: if(profit > g_setups[idx].mfe_money) → mfe_money=profit` (:43504-43505)
  `mae: if(profit<0 && -profit > mae_money) → mae_money = -profit` (:43508-43509) — «مثبت ذخیره می‌شود (magnitude)».
  MaxDD = بزرگ‌ترین میان «peak−current» و «worst |loss|» (:43493-43501).
- emit: RXL :13157/:13233؛ POSummary :15009-15010؛ snapshot copy :21325-21326.

**نتیجه:** تعریف دقیق setup-level در دسترس است: «ماکزیمم/میینمم سود شناور سبد از زمان فعال‌شدن setup تا پایان؛ MAE به‌صورت magnitude ذخیره می‌شود». → endpoint سابق «MAE/MFE setup-level در دسترس نیست» با شواهد فوق لازم است بازنگری شود (پیشنهاد ارتقا: **با شرایط** — قرارداد علامت magnitude + واحد money—not ATR—not points). «ارتقای رسمی ماتریس gate = تصمیم مالک».

---

## ۱۰. Provenance (هش / کامیت / تاریخ / Build)

| آیتم | مقدار قطعی |
|---|---|
| population CSV، SHA-256 | `09d0a3028520a499624a82f0…` = هش مذکور در مستندات FULL_BACKTEST ✓ |
| برچسب‌های درون population | `SourceCommit=061a1bb47c5bc7768f66cc9dec291d5c5e8b7135`؛ `SpecVersion=C2_M2_M0_POPULATION_COLLECTION_V1`؛ `Build=C2M2_M0COLLECT_V1` |
| کامیت instrumentation | `32d8b9b` — 2026-09-07 13:06:54 +0000 («M0 Shift-1 population collection instrumentation (pre-run review)») |
| کامیت freeze τ | `c995cdf` — 2026-09-08 04:41:26 +0330 («threshold freeze v1 (τ_ATR=0.04668375, τ_ADX=47.50305)») |
| کامیت population data | `2f04d01` — 2026-09-08 08:05:59 +0330 |
| کامیت R4 (M2 run) | `0e91848` — 2026-09-08 12:29:09 +0330 |
| کامیت R2 (M0 matched) | `3352b18` — 2026-09-10 11:27:53 +0330 |
| کامیت legacy `_cf` | `662a7ec` (FETCH_HEAD `agent-regression-data-real`) — 2026-09-07 15:28:51 +0330 |
| کامیت fail-closed INVALID | `4759646` — 2026-09-09 05:54:20 +0000 |
| run logها | LFS pointer (مثال: `20260908.log` → oid sha256:`19e8d3…`، ~462MB) — بدون ابزار LFS **BLOCKED**؛ بنابراین «timestamp شروع اجرا» از log قابل‌اثبات نیست (به‌جای آن تاریخ کامیت‌ها بالاتر). |
| bulk CSVهای R3 | LFS pointers — oids: `PositionOpenDataset`=4d013f… (27MB)، `RX Features`=08f8d6…، `ReportTester`=63a4c9… → **BLOCKED** |

**بازتولید τ (reproducibility PASS):** روی 358 ردیف population، type-7 Q75 از دو ستون `ATRPct_Shift1` و `ADX_Shift1` → مقادیر exact 0.04668375 و 47.50305 — مطابق freeze commit و مستندات.

---

## ۱۱. پوشش Step2→Step5 و ماتریس دست‌رس‌پذیری Entry-Snapshot

### ۱۱.۱ پوشش لایه‌ها برای تحلیل chain مراحل

| مرحله | دیتاست پوشش‌دهنده | وضعیت |
|---|---|---|
| Step 0→1 (تشکیل/شروع) | ME + RXF + LCEvents(RunID/SetupID); PO legموارد StepNo=1 | PASS |
| Step 1→2 / 2→3 | PO (StepNo dist) + LCEvents + C2M2 file (R4) | PASS |
| Step 3→4 (منطق C2/M2) | population (R3، shift-1، 358 DEV) + C2M2 decisions (R4) + legacy CF (R1، shift-0، 431/366/65) | PASS (هر دو timeline) |
| Step 4→5 (پلهٔ آخر) | PO Step5 + Stats (=191 R2 / 169 R4 محاسبه با stepdist ✓) | PASS |
| لایهٔ outcome | RXL (censor/labels/MAE/MFE/DD) + S1C | PASS |

### ۱۱.۲ Entry-Snapshot Canonical — ماتریس به‌روز (شامل تصحیح §۸/§۹)

| خاصیت canonical | وضعیت | اساس |
|---|---|---|
| entry price/time/pivot lineage | PASS | PO legs + Join کامل با ME/RX |
| indicator features at entry | PASS با قرارداد صریح: **shift-0 live-at-fill** (ناسازگاری note هدر §۵ — ثبت مستنداتی) | PO `IndicatorShiftUsed=0` |
| formation features shift-1 closed-bar | PASS | RXF 6×Shift=1 + Manifest |
| MAE/MFE setup-level | **قابل‌استفاده (cاندید ارتقا به PASS-conditions)** | RXL + کد §۹ — تصمیم مالک |
| DeepStop setup-level | **RECONCILED — برخلاف ارزیابی قبلی «غیرقابل‌رسیدگی»** | rule :18604 + دو ستون موجود (§۸) — تصمیم مالک |
| Strategy/any-stop flag | PASS | ME `StrategyStopFlag` / PO `SetupFinalStop` (parity برقرار) |
| علت tick-سطح 8-ID drop | NOT PROVEN | §۷ |

---

## ۱۲. ثبت اصلاحات برای artifactهای پیشین (بدون اقدام مالکانه؛ فقط پیشنهاد شواهدی)

1. اصطلاح‌گیجی STOP/DEEPSTOP در POSummary «rows = leg rows» است، نه setup. ورودی قبلی مبتنی بر 88/70 برای M0 و 90/53 برای M2 است؛ اعداد 383/332 و 365/257 رویظر respective sums هستند و هر دو طرف مطابقت دارند (§۸).
2. فیلدهای MAE/MFE «غایب» تلقی شده بودند؛ در RXL موجود و کامل‌اند (§۹). اظهارنظر نهایی ماتریس با مالک.
3. آمار M2 واقعی (R4): stop=90 setups / 365 legs؛ deep=53 / 257. هر ذکر divergent (مثل 353/262) در پیش‌نویس‌های غیررسمی با آبجکت‌های فعلی **سازگار نیست** و منشأ آن در tracked refs یافت نشد.
4. note هدر POSummary دربارهٔ shift-1 با محتوای dataset ناسازگار است (§۵).

---

## ۱۳. وضعیت نهایی موارد ۱–۹

| # | مورد | وضعیت |
|---|---|---|
| 1 | موجودی و شناسنامهٔ هر ۴ خانوادهٔ dataset + CodeEvolution | **PASS** (§۲–§۴؛ با entry برای رژیم legacy) |
| 2 | مسیر / نام ستون / تعداد رکورد / setup یکتا / بازهٔ زمانی / Step dist / Shift / schema / NA policy per dataset | **PASS** (§۳، §۶.۱) |
| 3 | تطبیق تعاریف با کد منبع (PO/RXF/EntrySnapshot/ME/C2M2) | **PARTIAL** — چهار مورد Pass؛ «Entry Snapshot» موجودیت فیزیکی مستقل نیست (مفهوم = رکوردهای PO) |
| 4 | اثبات Shift=0/1 با نمونهٔ واقعی رکورد | **PASS** (+ یافتهٔ ناسازگاری مستنداتی حل‌شده «PROVEN», ثبت‌شده در §۵) |
| 5 | Join/یکتایی/parity/NA | **PASS** (§۶) |
| 6 | آشتی 366/360/358 | **366 PASS، 358 PASS، Δ(8) mechanism PASS — per-ID cause NOT PROVEN ، 360 NOT PROVEN** (§۷) |
| 7 | Provenance per file (hash/commit/date/build) | **PARTIAL** — همه‌چیز به‌جز محتوای LFS اثبات‌شده؛ LFSها **BLOCKED** (§۱۰) |
| 8 | پوشش Step2→Step5 | **PASS** (§۱۱.۱) |
| 9 | ماتریس Canonical Entry Snapshot | **PASS** — با دو پرچم ارتقا (MAE/MFE، DeepStop) در انتظار تأیید مالک (§۱۱.۲/§۱۲) |

---

## پیوست — نمونهٔ رکوردهای واقعی (verbatim)

**۱) Population (R3) — header و ردیف SetupID=4:**
```
Event;SetupID;StepFrom;StepTo;T_decision;DecisionServerTime;FeatureShift;BarOpenTime_Shift1;BarCloseTime_Shift1;ATR_Shift1;ATRPct_Shift1;ADX_Shift1;DIPlus_Shift1;DIMinus_Shift1;PriceRef_Shift1;strict_before_check;C2M2_Decision;LatchStatus;Direction;Mode;Rule;SourceCommit;SpecVersion;Build
A;4;3;4;2014.01.08 00:14:04;2014.01.08 00:14:04;1;2014.01.08 00:13:00;2014.01.08 00:14:00;0.39;0.031846;19.2933;24.0442;28.7849;1228.58;true;ALLOW;FIRST;BUY;0;M0_COLLECT;061a1bb47c5bc7768f66cc9dec291d5c5e8b7135;C2_M2_M0_POPULATION_COLLECTION_V1;C2M2_M0COLLECT_V1
```

**۲) RX Features (R2) — مقادیر ستون‌های shift (نمونه):** `M1_Shift=1; M5_Shift=1; M15_Shift=1; M30_Shift=1; H1_Shift=1; H4_Shift=1` (برقرار در هر 4,864 ردیف).

**۳) MarketEdge (R2) — نمونهٔ ردیف اجراشده:** `EventRowID=2 … FeatureShift_M1=1; WasExecuted=1; ExecutedSetupID=1; StrategyStopFlag=0`.

**۴) legacy CF (R1) — نمونهٔ زنجیرهٔ SetupID=169 (A3→4 در DEV با `:00`):**
```
169 A 2014.07.06 22:15:00 3->4 Mode=0 Rule=NONE
169 P 2014.07.06 22:15:00 3->4 Mode=0 Rule=M0
169 F 2014.07.06 22:15:00 3->4 Mode=0 Rule=M0
(علاوه: SetupIDs 470: 2015.05.06 03:20:00 ; 1072: 2017.06.20 11:42:00 — هر دو :00)
```

**۵) Stats_XAUUSD.txt (byte-identical در R1/R2/R3-Summary):**
```
RowCount=0 / HTMLRowCount=0 / NextSetupID=4865 / Stat0=4864 / Stat1=4863 / Stat2=1894 / Stat3=1821 / Stat4=717 / Stat5=240 / Stat6=191
```
(Stat2=1,894 == S1C SingleStepN ✓؛ Stat6=191 == تعداد legهای StepNo=5 ✓)

---

*پایان گزارش Audit & Reconciliation V1 — read-only؛ بدون تغییر کد/threshold/backtest/استراتژی.*
