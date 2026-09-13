# M0 GATE REPORT — بررسی مستقل M0 Gate روی artifact های واقعی run

- تاریخ بررسی: 2026-09-04 · منبع: شاخهٔ `agent-regression-data` @ `29dfb15` (پوشهٔ `v2_research_export_v2_cf` = خروجی‌های M0 که کاربر روی MT5 خود اجرا کرده) + تاریخچهٔ git (`241d4b7`) برای baseline واقعی NEW.
- ابزار: نسخهٔ فریزشدهٔ `check_m0_gate.py` (sha `7a9af51a…`) — اما **اجرای کامل آن در این سندباکس ممکن نشد** (دلایل در بخش ۲). این گزارش = بررسی مستقل بخش‌به‌بخش با همان معیارهای A–G فریز.
- هیچ threshold ای استخراج نشد؛ هیچ rule ای تغییر نکرد؛ هیچ M1/M2/M3 اجرا نشد.

---

## 1. موجودیت و یکپارچگی artifact ها (SHA-256 واقعی = LFS oid)

فایل‌های M0 در درخت git فقط اشاره‌گر LFS هستند (سیاست `*.csv/*.xlsx/*.log` = LFS در `.gitattributes`). sha256 محتوای واقعی هر فایل = `oid` ثبت‌شده در اشاره‌گر (خود LFS این تضمین را می‌دهد و `git lfs ls-files -l` سمت شما همان را نشان می‌دهد):

| فایل M0 (پوشهٔ v2_research_export_v2_cf) | sha256 کامل (content، راستی‌آزمایی‌شده از oid) | اندازه |
|---|---|---|
| RX_…_Labels.csv | `c41112a1f7e8e165a83b3c9baace0597cec0d00495206bb5f497a088b0c4b909` | 1,010,876 |
| PositionOpenDataset_XAUUSD.csv | `e5da577a641eac9904e31de09921e977a98bac87eca6ce1d2b2d216fd4994f2e` | 27,241,059 |
| RX_…_SetupLifecycleEvents.csv | `f0cf38e479f7d50833530236d658a15db712ab16d0cbf4e4d15cb8c4284d6638` | 2,720,230 |
| RX_…_Features.csv | `3f98180fc9a37748f81fb547db8f204e128a66e609daa22be9899f80dba0888b` | (همان NEW) |
| MarketEdge_MARKET_EDGE_MFE_ATR_12_V1_XAUUSD.csv | `ff4a64268721c2eca865eb31f344c114d4194fa8763935cfcb618e2e8f70999d` | (همان NEW) |
| 20260904.log | `5d5a8cd55f49457472824dd16324b7e9eb9eeab296cc2a32425bd2a4eba70e73` | LFS |
| ReportTester-91306235.xlsx | `e730bb416e5cec97f489a4af9c33f4dfcab9d9f083a0591d7cd3b4ae104679a5` | LFS |
| Step1_OutcomeComparison_XAUUSD.csv | `0445f66c32b382e812feebc3dc70ac095e7ba47856b6a86e8e056a9d3a3decaf` | LFS |
| Step1_OutcomeComparison_Summary_XAUUSD.csv | `77319958979a2358d52f34b1e8a9aea8e9243beba7c07a5f9040f4bc956b8c60` | LFS |
| PositionOpenIndicatorStats_XAUUSD.csv | `e24358fa021d38de16ea2e35f29134271faa3cdb4b45e252473ccc95dfaa05f7` | LFS |
| ProfileStats_XAUUSD_D_FULL.csv | `903e6d9712dcb619a2a0354cb8dcfa6aa19149a4e5faccf728f9843ea9bde954` | LFS |
| PositionOpenSummary_XAUUSD.txt | (واقعی، UTF-16) blob `75869e86…` | 5,876 |
| Stats_XAUUSD.txt | (واقعی) | 256 |
| v2_research_export_v2_cf.mq5 | **`567b94a2bb15fbd5f41c4d9fd4b601b0f8a5370e99b17e00e95b492965e91e3b`** ✅ = فایل فریز | 2,184,911 |
| v2_research_export_v2_cf.ex5 | `3669e4839d8d901e556349cf986c107bd1097f0d53ed9909d4309a7c860ef65f` | 1,737,222 (شاهد Compile) |

Baseline NEW (واقعی، از تاریخچهٔ git `241d4b7`، sha256 با اشاره‌گرهای LFS همان شاخه تطبیق داده شد):
- NEW Labels = `88ee205c2cc9f94798a9bed0f6ad7ef6b8cca2d7c5cd305c873e90e8ccb524dc` (1,010,537 بایت) · NEW PO = `175f3d86434f31a6d655012449eacfd9611e52cff4bfa51cec5ac8882cefeee2` (27,240,289) · NEW Lifecycle = `f0cf38e479f7d50833530236d658a15db712ab16d0cbf4e4d15cb8c4284d6638` (2,720,230) · NEW Features = `3f98180fc9a37748f81fb547db8f204e128a66e609daa22be9899f80dba0888b` · NEW MarketEdge = `ff4a64268721c2eca865eb31f344c114d4194fa8763935cfcb618e2e8f70999d`

**یافته‌های کلیدی یکپارچگی:**
- `M0 Features == NEW Features` (sha یکسان) → جمعیت استپ‌۱ یکسان.
- **`M0 SetupLifecycleEvents == NEW SetupLifecycleEvents` (sha یکسان، بایت‌به‌بایت)** → کل جریان رویدادهای اجرا (STEP1..5_OPEN، TP1/2/3_TOUCH، EMA200_TOUCH، EXIT_REQUESTED/COMPLETED با زمان‌ها) **در M0 و NEW یکسان است**.
- `M0 MarketEdge == NEW MarketEdge` (sha یکسان) → snapshot های اندیکاتور ورود یکسان.
- `M0 PositionOpenIndicatorStats == NEW PositionOpenIndicatorStats` و `M0 ProfileStats_D_FULL == NEW ProfileStats_D_FULL` (sha یکسان) → snapshot های لحظهٔ باز شدن پوزیشن و توزیع سود/ضرر نهایی پوزیشن‌ها یکسان.
- `M0 Stats == NEW Stats`: NextSetupID=4866؛ توزیع MaxStep 1895/1821/718/240/191 — **برابر**.
- M0 Labels و M0 PO با NEW **تفاوت جزئی اندازه** دارند (+339 و +770 بایت) — کاملاً سازگار با «فقط تغییر مقادیر ردیابی MAE/MFE/MaxDD استپ≥۴» (اعداد عمیق‌تر/پهن‌تر می‌شوند)، اما **اثبات ردیف‌به‌ردیف بدون خواندن خود فایل‌ها ممکن نیست**.

---

## 2. چرا اجرای کامل `check_m0_gate.py` در این سندباکس ممکن نشد (blocker محیطی)

سه فایل لازم برای گیت از دسترس سندباکس خارج‌اند:
1. **M0 Labels.csv و M0 PositionOpenDataset.csv** — محتوای واقعی فقط در LFS است (این دو فایل در تاریخچهٔ git قبل از LFS وجود ندارند). دانلود LFS نیازمند هاست‌های `media.githubusercontent.com` / `github-cloud.githubusercontent.com` است که **egress سندباکس آن‌ها را قطع می‌کند** (تست: TCP وصل می‌شود، TLS با `SSL_ERROR_SYSCALL` خاتمه می‌یابد؛ `api.github.com` و `github.com` در دسترس‌اند ولی API محتوای LFS را smudge نمی‌کند و فقط pointer برمی‌گرداند — آزمایش شد). git-lfs هم در سندباکس نیست و مسیر دانلود آن نیز به همان هاست‌های مسدود ختم می‌شود.
2. **`CF_events_m0.csv` در artifact ها وجود ندارد** (نه در پوشهٔ کامیت‌شده، نه در فهرست پوشهٔ محلی شما) — این فایل برای بخش G گیت **و** برای کل Phase-3 (جمعیت Q75 = ردیف‌های Event=A با DecisionTime<2023) الزامی است.
3. **20260904.log و ReportTester xlsx** (برای راستی‌آزمایی کامل config: start date، tick mode، inp_cf_mode، broker settings) — LFS و مسدود.

طبق قانون تصمیم فریز: بدون امکان اجرای کامل گیت → **نمی‌توان PASS اعلام کرد** → STOP (بخش ۶).

---

## 3. بررسی بخش‌به‌بخش A–G (با شواهد در دسترس)

| بخش | وضعیت | شواهد / دلیل |
|---|---|---|
| **A. شرایط run** | ⚠️ PARTIAL | تأییدشده: Symbol=XAUUSD (Manifest/Summary)؛ RunID=`RX_1388707200_XAUUSD` (یکسان با NEW)؛ انتهای run 2026.08.19 23:59 (Summary == NEW). تأییدنشده: start date، tick mode، `inp_cf_mode`، broker settings (نیازمند .log/.xlsx/runinfo — LFS/در دسترس نیست). |
| **B. شمارش‌ها** | ✅ EQ (شواهد قوی) | Setup=4,865 (Manifest rows=4865، DQR «Step1 rows: 4865»، Stats 4865، Features==NEW)؛ PO rows=9,606 (Summary == NEW)؛ ردیف‌های استپ ۱..۵: Lifecycle یکسان + DQR (Step4Plus=431، Step5=191) + Stats (MaxStep 1895/1821/718/240/191) → معادل 4,865/2,970/1,149/431/191. |
| **C. هم‌ارزی ردیف‌به‌ردیف Labels/PO** | ❌ UNVERIFIED (مسدود) | فایل‌های M0 Labels/PO از LFS قابل واکشی نیستند. شواهد غیرمستقیم (Lifecycle/Features/Summary/Stats یکسان) قویاً حاکی از اجرای یکسان است؛ اندازه‌ها (+339/+770) با اثر fix سازگار. مستقیم قابل اثبات نیست. |
| **D. شمارش FinalEvent/Lifecycle** | ✅ Lifecycle EQ / ⚠️ FinalEvent غیرمستقیم | جریان lifecycle بایت‌به‌بایت یکسان (sha یکسان) → STEP1..5_OPEN (4,865/2,970/1,149/431/191)، TP1/2/3_TOUCH (2,194/2,027/763)، EMA200_TOUCH (4,134)، EXIT_REQ/COMP (4,865/4,865) همگی یکسان. شمارش FinalEvent (4416/361/88) از Labels م0 خوانده نشد؛ چون خروج/علت‌ها یکسان‌اند، انتظار یکسانی می‌رود (غیرمستقیم). |
| **E. Net Profit −562.90** | ❌ UNVERIFIED (مسدود) | از Labels م0 مستقیم خوانده نشد. با fills/exits/لات یکسان (Lifecycle یکسان) انتظار −562.90 می‌رود — غیرمستقیم. |
| **F. استثنای MAE/MFE (فقط استپ≥۴)** | ❌ UNVERIFIED (مسدود) | نیازمند مقایسهٔ ردیف‌به‌ردیف MAEMoney/MFEMoney/MaxDDMoney/MaxDDPct بین M0 و NEW. نکتهٔ حمایتی: افزایش ۳۳۹ بایتی Labels م0 با تغییر مقادیر ردیابی استپ≥۴ (اثر fix) سازگار است؛ تأیید قطعی ممکن نیست. |
| **G. CF_events_m0.csv** | ❌ **MISSING (hard fail)** | فایل در artifact ها وجود ندارد. (جزئیات/تشخیص در بخش ۴.) |

---

## 4. تشخیص فقدان `CF_events_m0.csv`

- انتظار: build فریز در M0 (mode=0) برای **هر تلاش ورود** یک ردیف `A` می‌نویسد (شامل ۴,۸۶۵ تلاش استپ‌۱) → فایل باید چند هزار ردیف و چند صد کیلوبایت باشد و کنار فایل‌های `RX_*` (که از همان run جمع‌آوری شده‌اند) ظاهر شود.
- واقعیت: فایل غایب است.
- علت‌های محتمل (غیرقابل تفکیک با artifact های موجود):
  1. فایل در Tester sandbox نوشته شده و هنگام جمع‌آوری خروجی‌ها کپی نشده — **محل جستجو: `<MT5 Data Folder>\Tester\Files\CF_events_m0.csv`** (و پوشه‌های `Tester\Agent-*\MQL5\Files\`)؛
  2. اجرا با ex5 ای انجام شده که کد CF logging را اجرا نکرده (هرچند ex5 موجود در پوشه متعلق به cf.mq5 فریز است)؛
  3. FileOpen در محیط Tester ناموفق بوده (بدون log خطا چون ما روی بازشدن فایل Print نداریم — نقص ثبت‌گر).
- پیامد: بدون این فایل، **هم بخش G گیت و هم Phase-3 (Q75 از ردیف‌های Event=A) غیرممکن‌اند** — صرف‌نظر از بقیهٔ نتایج.

---

## 5. Compile

- شواهد Compile سمت شما: **PASS** — فایل `v2_research_export_v2_cf.ex5` (1,737,222 بایت، sha `3669e483…`) در کنار mq5 فریز (sha `567b94a2…` ✅) تولید و کامیت شده است.

---

## 6. Verdict نهایی (طبق قانون تصمیم فریز)

> **M0 GATE = NOT PASS (INCONCLUSIVE — اجرای کامل گیت در این سندباکس مسدود است؛ نه PASS اعلام می‌شود و نه FAIL عددی)**
> **STOP — هیچ threshold ای استخراج نشد؛ هیچ M1/M2/M3 اجرا نمی‌شود تا زمانی که گیت روی artifact های کامل و در دسترس PASS شود.**

مهم: این «ناتوانی در PASS» به‌معنای رد عددی M0 نیست — همهٔ شواهد در دسترس (Lifecycle، Features، Summary، Stats — همه بایت/فیلد یکسان با NEW) به نفع بازتولید موفق است. اما طبق فریز، بدون اجرای کامل گیت (A–G روی فایل‌های واقعی) نمی‌توان جلو رفت.

### برای ادامه (هر سه مورد لازم است)
1. **M0 Labels.csv و M0 PositionOpenDataset.csv** را از راهی در دسترسِ سندباکس ارائه کنید: مثلاً push به ریپو در مسیری که از LFS مستثنا شده (الگوی جدید در `.gitattributes` شبیه `v2_research_export_v1_CONTROL_REGEN_A1/*.csv`)، یا تغییر پسوند/zip (فقط csv/xlsx/log تحت LFS‌اند)، یا ضمیمهٔ مستقیم.
2. **`CF_events_m0.csv`** را بیابید (`<MT5 Data>\Tester\Files\…` یا پوشه‌های Agent) و همان‌جا قرار دهید؛ اگر واقعاً ساخته نشده → علت را با یک M0 کوتاه (مثلاً ۱–۲ هفته) با ثبت‌گر بررسی و سپس M0 کامل را دوباره اجرا کنید.
3. **Config snapshot**: `20260904.log` و `ReportTester…xlsx` (یا خلاصهٔ متنی tester settings: start/end، مدل/حساب، اسپرد، کمیسیون، execution + مقدار `inp_cf_mode`) + فایل `runinfo.txt`.

پس از فراهم‌شدن هر سه، `check_m0_gate.py` (نسخهٔ فریز، بدون تغییر) روی artifact های واقعی اجرا می‌شود و نتیجهٔ قطعی PASS/FAIL اعلام می‌گردد. هیچ تغییری در Production یا ابزارهای فریز رخ نداده است.
