# Matched M0 vs M2 — Endpoint Extraction (setup-level, V1)

تاریخ: 2026-09-10 — وضعیت: **Descriptive-Only**؛ فقط از داده‌های موجود M0/M2.
محرمات رعایت‌شده: NO NEW BACKTEST / NO CODE CHANGE / NO THRESHOLD CHANGE / NO STRATEGY CHANGE / NO FEATURE SELECTION / NO OPTIMIZATION؛ بدون p-value/regression/bootstrap/hypothesis test؛ بدون هیچ ادعای causal (بخش VETO/ALLOW صرفاً reporting توصیفی است).

منابع: PO dataset دو run (M0 @`3352b18:M0/`، M2 @`0e91848:cfv2/`) + C2M2 decisions (M2) + Lifecycle (فقط برای تعیین EOT) + دو ReportTester.xlsx.
روش تطبیق: content-key = (Step1 OpenTime + Direction + FillPrice) — یکتا و بدون برخورد (تأییدشده در گزارش Reconciliation). جدول setup-level: **`C2M2_MATCHED_ENDPOINTS_SETUPLEVEL_V1.csv`** (4,778 ردیف × 41 ستون).

---

## 1) Endpoint Availability Matrix (صادقانه)

| Endpoint | وضعیت در دادهٔ موجود |
|---|---|
| M0/M2 MaxStep | ✅ موجود (PO: `SetupMaxStep`) |
| FinalProfit (هر setup) | ✅ (`SetupFinalProfit`, ردیف StepNo=1) — مجموع‌پذیر، لایهٔ dataset |
| D_PnL = M2 − M0 | ✅ محاسبه‌شده‌در-جدول |
| DollarSL | ✅ دو فرم: hit-flag (`FinalEvent ∋ 'SL$'` ⇔ `SetupFinalStop`) + مقدار برنامه‌ریزی‌شدهٔ Step1 (`DollarStopUSD`) |
| TP-family | ✅ پرچم‌های setup: `SetupHitTP1`, `SetupHitTP2`, `SetupHitEMA200`, `SetupFinalTarget`, توکن‌های TP در `SetupFinalEvent`, سطحوح planned `TP1..TP5`. HitTP3 به‌صورت Boolean در PO **نیست** (توکن TP3 در FinalEvent + رویداد LC: TP3_TOUCH وجود دارد) |
| BE (Basket BE) | ✅ پرچم hit از `FinalEvent ∋ 'NET_BASKET_BE'` (Lifecycle رویداد BE جداگانه‌ای ندارد) |
| setup-level DeepStop flag | ❌ **NOT AVAILABLE** — هیچ فیلد setup-level در PO/LC؛ فقط تجمیع leg-level در POSummary داریم: STOP/DEEPSTOP/LOSS = M0: 383/332/386، M2: 365/257/368 (قاعدهٔ طبقه‌بندیِ آن تجمیع‌ها در دادهٔ صادرشده موجود نیست ⇒ join یا بازسازی **NOT PROVEN**) |
| setup-level MAE / MFE | ❌ **NOT AVAILABLE** — در PO (370 ستون) فیلد MAE/MFE نیست؛ نزدیک‌ترین خروجی موجود: `SetupMaxDDMoney/Pct` (در جدول درج شده). MarketEdge دارای `FutureMFE/FutureMAE` است اما آن متریکِ horizonِ پروتکلِ پژوهشِ سیگنال است (نه MAE/MFE درون‌معامله) ⇒ برای یکسانی استخراج، از endpoints کنار گذاشته شد |
| هر خروجیِ جمع‌آوری‌شدهٔ دیگر | ✅ Duration, MaxDD, PlannedDollarStop, OutcomeComplete, EOT |

## 2) جدول Setup-Level (N=4,778)

- فایل CSV: کلید (`MatchKey_Step1Time`), `Dir`, `FillPrice`, `M0_ID`, `M2_ID`, `M2_Decision` ∈ {VETO/ALLOW/INVALID/NONE}، و سپس endpointهای دو طرف + `D_PnL`.
- PARTITION (قطعی، از C2M2 decisions): VETO=49 + ALLOW=367 + INVALID=9 + NONE(بدون تصمیم Step3→4 در M2)=4,353 ⇒ مجموع = **4,778** ✅
- M0-only=86 و M2-only=4 **خارج** از این جدول/تحلیل‌اند (گزارش مجزا در بخش ۱۰) — مطابق دستور.

## 3) Aggregate — Profit / D_PnL (descriptive)

| Group | N | Σ M0 FinalProfit | Σ M2 FinalProfit | Σ D_PnL | mean D | median D | min D | max D | D>0 | D=0 | D<0 |
|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|
| ALL matched (4,778) | 4778 | -743.07 | -1456.25 | -713.18 | -0.1493 | 0.0 | -324.7 | 366.26 | 25 | 4729 | 24 |
| VETO subset (49) | 49 | -4854.96 | -5568.14 | -713.18 | -14.5547 | 0.25 | -324.7 | 366.26 | 25 | 0 | 24 |
| ALLOW subset (367) | 367 | -14687.47 | -14687.47 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0 | 367 | 0 |
| INVALID subset (9) — اطلاعاتی | 9 | 68.23 | 68.23 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0 | 9 | 0 |
| NO_DECISION (4,353) — اطلاعاتی | 4353 | 18731.13 | 18731.13 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0 | 4353 | 0 |

**Identity (عددی):** 49+367+9+4,353 = 4,778 ✅؛ ΣD(ALL) = **−713.18** = ΣD(VETO) = **−713.18**؛ D_PnL در ALLOW/INVALID/NONE برای **تمام** ردیف‌ها صفر است (descriptive arithmetic fact).

## 4) Aggregate — Endpoint counters (M0 / M2، در هر group)

| Group | MaxStep برابر | M0>M2 | M0<M2 | FinalStop$ (M0/M2) | BasketBE | TargetHit | ReachedStep5 | HitTP1 | HitTP2 | HitEMA200 |
|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|
| ALL matched | 4729 | 49 | 0 | 87/90 | 356/323 | 4334/4364 | 187/169 | 2152/2155 | 1992/1999 | 4062/4073 |
| VETO | 0 | 49 | 0 | 16/19 | 33/0 | 0/30 | 18/0 | 3/6 | 2/9 | 37/48 |
| ALLOW | 367 | 0 | 0 | 53/53 | 314/314 | 0/0 | 167/167 | 36/36 | 77/77 | 277/277 |
| INVALID — اطلاعاتی | 9 | 0 | 0 | 0/0 | 9/9 | 0/0 | 2/2 | 2/2 | 4/4 | 9/9 |
| NO_DECISION — اطلاعاتی | 4353 | 0 | 0 | 18/18 | 0/0 | 4334/4334 | 0/0 | 2111/2111 | 1909/1909 | 3739/3739 |

## 5) VETO subset = 49 (descriptive-only — بدون هیچ ادعای علی نسبت به ALLOW)

- MaxStep pair: **31 × (M0 4 → M2 3)** و **18 × (M0 5 → M2 3)** ⇒ در هر 49، MaxStep در M0 بزرگ‌تر از M2 است (ساختار گیت Step3→4 بودنِ تصمیم با داده سازگار).
- Σ FinalProfit به‌ازای این 49: M0 = **−4,854.96**، M2 = **−5,568.14** ⇒ Σ D_PnL = **−713.18** (25 ردیف D>0 / 0 ردیف D=0 / 24 ردیف D<0؛ بازهٔ [−324.70, +366.26]).
- FinalStop($) در این group: M0 = 16، M2 = 19؛ BasketBE: M0 = 33، M2 = 0؛ TargetHit: M0 = 0، M2 = 30؛ ReachedStep5: M0 = 18، M2 = 0.
- ردیف‌به‌ردیفِ همین 49 (با T_decision و پارامترهای ویژگی) در گزارش قبلی `C2M2_MATCHED_M0_VS_M2_SETUPID_RECONCILIATION_V1.md` جدول‌بندی شده است.

> قید الزامی: این اعداد status توصیفیِ journeyِ بدون‌تغییر (M0) همان‌ستاپ‌هاست؛ «اگر VETO نمی‌شد چه» (counterfactual) **بخشی از این گزارش نیست** و هیچ استنباط causal/loss-prevention/DeepStop-avoidance صادر نمی‌شود.

## 6) ALLOW subset = 367 (descriptive-only)

- در هر 367: D_PnL = **0 دقیقاً**؛ MaxStep (M0/M2) به‌ازای همهٔ موارد برابر (4→4 یا 5→5).
- Σ FinalProfit هر دو طرف = **−14,687.47** یکسان. FinalStop: M0 = 53، M2 = 53؛ BasketBE: M0/M2 = 314/314؛ ReachedStep5: 167/167.
- اطلاعاتی (غیرازدرخواست ولی برای کامل‌بودن partition): INVALID=9 همه D=0؛ NO_DECISION=4,353 همه D=0.

## 7) Total Policy Outcome (کل runها — هرکدام جداگانه و جدا از لایه‌ها)

| منبع | M0 (4,864) | M2 (4,782) | یادداشت |
|---|--:|--:|---|
| Σ SetupFinalProfit (dataset) | **−626.69** | **−1,441.12** | layer-separated؛ index ها با فایل‌ها verify |
| MT5 Total Net Profit (xlsx) | **−800.19** | **−1,614.62** | لایهٔ رسمی MT5 — هرگز با لایهٔ بالا مخلوط نشده |
| MT5 Profit Factor | 0.979276 | 0.957556 | توصیفی |
| gap (MT5 − dataset) | −173.50 | −173.50 | احتمالاً swap/commission؛ تطبیق دقیق NOT PROVEN |

Identity (reconciliation قابل‌بازآزمایی): matched + only = کل: M0: −743.07 + (+116.38) = −626.69 ✅؛ M2: −1,456.25 + (+15.13) = −1,441.12 ✅. و داخل matched: policy-difference‌الجهتِ dataset (M2−M0 = −713.18) زیرمجموعه‌ای از gap کل runهاست: −1,441.12 − (−626.69) = −814.43 = ΣD(VETO) −713.18 + (M2-only − M0-only مجموع = 15.13 − 116.38 = −101.25) ✅ ⇒ **بسته با باقی‌ماندهٔ صفر**.

## 8) Limitations

1. MAE/MFE و DeepStop در سطح setup در اکسپورت‌ها موجود **نیست** (بخش ۱). بازسازی آنها از tick/leg-level از حوزهٔ این مأموریت خارج است (would-be new feature engineering ⇒ ممنوع).
2. `SetupMaxDDMoney/Pct` موجود و در جدول درج شده، اما تعریف متفاوتی با MAE/MFE کلاسیک دارد — جایگزین نخوانده شود.
3. MarketEdge `FutureMFE/FutureMAE` به تعمد کنار گذاشته شد (متریک horizon سیگنال، نه معاملهٔ اجراشده؛ join با SetupID تنها برای زیرمجموعهٔ executed ممکن است).
4. قاعدهٔ exact طبقه‌بندی «STOP/DEEPSTOP» در POSummary در دادهٔ خام موجود نیست ⇒ هرگونه valid back-mapping روی آن **NOT PROVEN** است.

## 9) وضعیت بسته‌بندی (Status)

استخراج endpoint یکسان و قابل‌reconciliation است: تمام مقادیر Setup-level و Aggregateها با identityهای صریح بسته شدند (عدم‌باقیماندهٔ عددی = 0). هیچ تحلیل علی، آمار استنباطی، feature selection یا optimization انجام نشده است. Production = M0؛ Freeze = NOT READY تغییر نکرده است.
