# Final Endpoint Reconciliation — Matrix (V1)

تاریخ: 2026-09-10 — وضعیت: **Reconciliation-Only**؛ فقط از داده‌ها و فایل‌های موجود.
محرمات رعایت‌شده: NO NEW BACKTEST / NO CODE CHANGE / NO THRESHOLD CHANGE / NO STRATEGY CHANGE / NO FEATURE SELECTION / NO OPTIMIZATION / NO P-VALUE / NO BOOTSTRAP / NO REGRESSION / NO STATISTICAL INFERENCE.

## 0) شناسایی artifactِ تحت بازرسی (شفافیت نام)

- فایل مورد استناد مأموریت (`C2_M2_ENDPOINT_MATCHED_SETUPS.csv`) **به‌همین نام در مخزن وجود ندارد**. artifactِ جدول setup-level موجود در مخزن = **`C2M2_MATCHED_ENDPOINTS_SETUPLEVEL_V1.csv`** (خروجی فاز Endpoint Extraction همین pipeline است). این گزارش آن artifact موجود را با sourceهای اصلی reconcile می‌کند. اگر فایل دیگری با نام مذکور در بیرون از مخزن ساخته شده، در حوزهٔ این مخزن **NOT VERIFIABLE** است.

## 1) Reconciliationِ artifact در برابر sourceهای اصلی (اثبات‌پذیر و عددی)

| کنترل | نتیجه |
|---|---|
| روش بازتولید | parse مستقیم `PositionOpenDataset_XAUUSD.csv` دو run + `C2M2_Step34_Decisions_m2.csv` + content-key تطبیق (Step1 OpenTime+Dir+FillPrice) — روش ثبت‌شده در گزارش‌های قبلی |
| تعداد ردیف artifact  artifact خوانده‌شده | 4,778 = N matched ✅ |
| Header (41 ستون) | byte-identical با بازتولید ✅ |
| **SHA-256 کل محتوا** | **`0804eecaea369020…` — بازتولید ≡ artifact کامیت‌شده (کاملاً یکسان)** ✅ |
| Partition از خودِ CSV | VETO=49, ALLOW=367, INVALID=9, NO_DECISION=4,353 ⇒ جمع 4,778 ✅ |
| Σ از خودِ CSV | Σ M0 FinalProfit = −743.07؛ Σ M2 = −1,456.25؛ Σ D_PnL = −713.18 ✅ (برابر با aggregateهای گزارش‌شده قبلی) |
| M0-only=86 / M2-only=4 | جدا (در artifact حضور ندارند) ✅ |

نتیجه: setup-level table **fully reconciled — reproducible, role-verified against primary sources.**

## 2) Endpoint Reconciliation Matrix

قواعد وضعیت:
- **PASS** = Source + Unit + Definition همگی داخل داده‌های موجود مستندند، mapping قابل‌بازتولید است ⇒ آمادهٔ inference.
- **PARTIAL** = field موجود و mapping قابل‌بازتولید است اما Definitionِ دقیقِ محاسباتی داخل داده/اسناد اثبات نیست ⇒ قبل از inference به مستندسازی Definition نیاز دارد.
- **NOT RECONCILED / NOT PROVEN** = mapping مستقلِ قابل‌بازتولید به setup-level وجود ندارد.

| Endpoint | Source دقیق | Unit of observation | Definition (provenance) | M0 | M2 | Reproducible mapping | Inference-ready | وضعیت |
|---|---|---|---|---|---|---|---|---|
| **MaxStep** | PO `SetupMaxStep` (ردیف StepNo=1) | setup | بیشینهٔ پلهٔ رسیده (فیلد بومی EA) | ✅ | ✅ | ✅ (content-join، byte-verified) | ✅ | **PASS** |
| **FinalProfit** | PO `SetupFinalProfit` (ردیف StepNo=1) | setup (دلار، لایهٔ dataset) | سود حاصل final ستاپ در لایهٔ dataset (لایه از MT5 Net جدا باقی می‌ماند) | ✅ | ✅ | ✅ | ✅ | **PASS** |
| **D_PnL** | مشتق مستقیم: `M2_FinalProfit − M0_FinalProfit` | setup | تعریف ثبت‌شده به‌دستور مالک در همین مأموریت؛ گرد کردن به ۲ رقم | ✅ | ✅ | ✅ (deterministic) | ✅ | **PASS** |
| **DollarSL (planned)** | PO `DollarStopUSD` per leg (setup-level: مقدار Step1 در artifact) | leg/planned | فاصلهٔ stop برنامه‌ریزی‌شده به دلار؛ تعریف داخل POSummary: «Strategy has no fixed broker SL price; DollarStopUSD is exported instead» | ✅ | ✅ | ✅ | ✅ | **PASS** |
| **DollarSL (hit-event)** | PO `SetupFinalStop` (بولین بومی) + توکن `SL$` در `SetupFinalEvent` | setup | نیل به stop دلاری؛ cross-exact: ۲ تعریف مستقل دولایه در هر دو run یکسان: 88 (M0) / 90 (M2) | ✅ | ✅ | ✅ | ✅ | **PASS** |
| **TP-family**: HitTP1 / HitTP2 / HitEMA200 / FinalTarget | PO `SetupHitTP1`, `SetupHitTP2`, `SetupHitEMA200`, `SetupFinalTarget` (بولین‌های بومی) | setup | رسیدن/هیت هر target در سفر ستاپ (فیلد بومی) — cross-check داخلی: `SetupHitEMA200` = شمارش setup-های دارای LC `EMA200_TOUCH` در هر دو run به‌طاق دقیق (4133/4077) | ✅ | ✅ | ✅ | ✅ | **PASS** |
| **TP-family**: TP3-hit | رویداد LC `TP3_TOUCH` per SetupID + توکن `TP3` در `SetupFinalEvent` | setup | سومین پلهٔ target لمس‌شد (رویداد جدول LC بومی است؛ artifact بولین جدا ندارد) | ✅ | ✅ | ✅ | ✅ | **PASS** (macro-family؛ فقط روش اتکاء به LC لحاظ شود) |
| **TP-family**: سطح planned TP1..TP5 | PO `TP1..TP5` numeric per leg | leg/planned | سطوح قیمت برنامه‌ریزی‌شده | ✅ | ✅ | ✅ | ✅ | **PASS** |
| **BasketBE** | توکن بومی `NET_BASKET_BE` در PO `SetupFinalEvent` | setup | حضور token واژگان‌بومیِ خروج در نقطهٔ basket-BE؛ تعریف در self-documenting data+code vocabulary موجود است (اسناد نگارشی جدا ندارد) | ✅ | ✅ | ✅ | ✅ | **PASS** |
| **MaxDD (Money/Pct)** | PO `SetupMaxDDMoney` / `SetupMaxDDPct` (فیلد بومی) | setup | نام فیلد موجود و فراگرفته است اما **مسیر دقیق محاسبه (basis/فرمول/leg- vs basket-level) در داده/اسناد مستند نیست** | ✅ | ✅ | ✅ | ⛔ تا مستندشدن definition | **PARTIAL** |
| **MAE** | ندارد | — | هیچ فیلد setup-level؛ MarketEdge `FutureMAE` متریکِ horizonِ سیگنالِ پژوهشی با معنای متفاوت است و join صرفاً برای زیرمجموعهٔ executed ممکن است ⇒ به‌عنوان setup-trade-MAE تعریف‌پذیر نیست | ❌ | ❌ | ❌ | ⛔ | **NOT RECONCILED / NOT PROVEN** |
| **MFE** | ندارد | — | همانند MAE (`FutureMFE`) | ❌ | ❌ | ❌ | ⛔ | **NOT RECONCILED / NOT PROVEN** |
| **DeepStop (setup-level)** | ندارد | — | فقط تجمیع leg-level در POSummary هست (countهای STOP/DEEPSTOP) و قاعدهٔ طبقه‌بندی آن در دادهٔ خام موجود نیست؛ back-map به SetupID طبق قاعدهٔ مالک مجاز نیست | ❌ | ❌ | ❌ | ⛔ | **NOT RECONCILED / NOT PROVEN** |
| **Dataset-vs-MT5 PnL gap** | دو لایهٔ جداگانه (PO را در برابر xlsx MT5) | run-level | عدد gap محاسبه شد (−173.50 هر دو run)؛ **منشأ: NOT DETERMINED — به swap/commission یا هیچ عامل مشخصی نسبت داده نمی‌شود** | ✅ | ✅ | ✅ (تحت همان قید) | ⛔ | **NOT DETERMINED** (نه یک endpoint با وضعیت PASS/PARTIAL) |

## 3) کنترل Accounting/Identity (فقط توصیفی)

| کنترل | مقدار | وضعیت |
|---|---|---|
| Partition | 49 + 367 + 9 + 4,353 = 4,778 | ✅ |
| Σ D_PnL (ALL) | −713.18؛ کاملاً در زیرمجموعهٔ VETO متمرکز (ALLOW/INVALID/NONE: همه D=0) | ✅ accounting identity — هیچ استنباط causal نیست |
| VETO MaxStep جفت‌ها | 31 × (4→3) + 18 × (5→3) | ✅ |
| ALLOW MaxStep | هر 367 جفت برابر (D=0) | ✅ |
| شناسایی فرمال only-groups | M0-only=86 ⇒ Σ=+116.38؛ M2-only=4 ⇒ Σ=+15.13 (خارج از matched، بدون blend) | ✅ |
| Policy totals جدا | dataset: M0 −626.69 / M2 −1,441.12؛ MT5 Net: −800.19 / −1,614.62؛ gap −173.50/−173.50 = **NOT DETERMINED** | ✅ لایه‌ها مخلوط نشده‌اند |
| Identity زنجیره | −743.07+116.38=−626.69 / −1,456.25+15.13=−1,441.12 / −814.43=−713.18−101.25 | ✅ باقی‌ماندهٔ صفر |

## 4) آماده برای Astra (فعلی)

1. جدول setup-level تأییدشدهٔ 4,778×41 + روش بازتولید کامل (content-key join)؛ **byte-for-byte با sourceها توأم (SHA ثابت)**.
2. endpointهای **PASS** آمادهٔ استفاده به‌عنوان متغیرهای آمادهٔ inference (در آینده، با شرط Reconciliation-final): MaxStep, FinalProfit, D_PnL, DollarSL (planned + hit-event), TP-family (HitTP1/HitTP2/HitEMA200/FinalTarget/TP3-LC/سطح planned TP1..TP5), BasketBE.
3. بستهٔ اثبات‌های accounting partition (49/367/9/4,353) + جداگانهٔ only-groups (86/+116.38 و 4/+15.13) + policy totals جدا در هر دو لایه.
4. قواعد حاکم گزارش‌بند است (association-only؛ بی‌هیچ inference/ca).

## 5) موارد باز (قبل از inference)

1. **MaxDD: PARTIAL** — نیاز به مستندسازی Definition دقیق (مسیر محاسبهٔ `SetupMaxDDMoney/Pct` در داده/اسناد اثبات نیست). تا آن زمان در inference به‌کار نرود.
2. **MAE/MFE: NOT RECONCILED / NOT PROVEN** — نیاز به mapping مستقل و قابل‌بازتولید (هرگز از PO/LC تنها)؛ تصمیم‌گیری بعدی مالک دربارهٔ ایجاد pipeline صادرکنندهٔ MAE/MFE در run آینده (در حوزهٔ این مأموریت نیست؛ هر بک‌تست جدید فعلاً ممنوع).
3. **DeepStop setup-level: NOT RECONCILED / NOT PROVEN** — نیاز به قاعدهٔ طبقه‌بندی قابل‌بازتولید (تجمیع POSummary کافی نیست).
4. **Dataset-vs-MT5 PnL gap: NOT DETERMINED** — نیاز به مصالحهٔ deal-level رسمی MT5 (خارج از حوزهٔ داده‌های موجود).
5. **Statistical Inference هنوز انجام نشده** — گیت‌های پیش‌نیاز: پذیرشِ export-level matrix این گزارش (قسمت ۲) و حل قسمت‌های بازِ بالا.

*وضعیت پایدار*: Production = M0؛ Freeze = NOT READY؛ هیچ code/threshold/strategy/backtest/statistical-inference در این فاز انجام نشده است.
