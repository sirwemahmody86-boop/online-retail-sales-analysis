# نقشه‌راه پروژه Online Retail در Power BI

**وضعیت:**
- ✅ فاز پاکسازی (Power Query) — کامل
- ✅ فاز مدل‌سازی (Calendar Table + Relationship) — کامل
- ✅ فاز DAX Measures پایه — کامل
- ✅ فاز طراحی داشبورد (۴ صفحه: Overview, Products, Customers, Returns) — کامل
- 👉 مراحل اختیاری تکمیلی (Star Schema, RFM, پورتفولیو) — باقی‌مانده

**داده‌ی خام اولیه:** 541,909 ردیف | 8 ستون
**داده‌ی نهایی بعد از پاکسازی:** 536,641 ردیف

---

## ۱. اصلاح نوع داده‌ها (Data Type Audit)

**مشکل کشف‌شده:** تنظیمات پیش‌فرض Column Profiling روی "Top 1000 rows" بود، نه کل داده — باعث میشد آمار Empty/Error واقعی دیده نشه.

**اقدام:** تغییر به `View → Column profiling based on entire data set`

**اصلاح تایپ:**
- `CustomerID`: از Decimal Number (1.2) → **Text** تغییر داده شد (چون شناسه‌ست، نه مقدار قابل محاسبه)

---

## ۲. ستون `HasCustomerID` (Boolean)

**مسئله:** 135,080 ردیف (۲۵٪ از کل داده) مقدار `CustomerID` خالی دارن.

**تصمیم:** داده‌ی خام حذف نشد؛ به‌جاش یه پرچم ساخته شد تا در تحلیل‌های مشتری‌محور فیلتر بشه ولی در تحلیل‌های فروش کلی باقی بمونه.

```
= if [CustomerID] = null or Text.Trim(Text.From([CustomerID])) = "" then false else true
```

✅ اعتبارسنجی: `False` = 135,080

---

## ۳. ستون `TransactionType` (Categorical)

**مسئله:** سه الگوی متفاوت در `InvoiceNo` کشف شد:
| الگو | تعداد | معنی |
|---|---|---|
| شروع با عدد | 532,618 | فروش عادی |
| شروع با `C` | 9,288 | کنسل/برگشتی |
| شروع با `A` | 3 | اصلاح بدهی حسابداری (Bad Debt Adjustment) |

```
= if Text.StartsWith(Text.From([InvoiceNo]), "C") then "Cancelled"
  else if Text.StartsWith(Text.From([InvoiceNo]), "A") then "Adjustment"
  else "Sale"
```

⚠️ **نکته:** این ستون در لحظه طراحی شد ولی واقعاً در Power Query پیاده‌سازیش فراموش شد و بعداً — در فاز DAX Measures، وقتی Measure ها به این ستون نیاز پیدا کردن — ساخته شد. چون تا اون موقع رکوردهای تکراری (بخش ۵) از قبل حذف شده بودن، عدد اعتبارسنجی نهایی با عدد بالا (که روی داده‌ی خام محاسبه شده بود) فرق داره:

✅ اعتبارسنجی نهایی (روی 536,641 ردیف، بعد از حذف Duplicates):
| مقدار | تعداد |
|---|---|
| `Sale` | 527,387 |
| `Cancelled` | 9,251 |
| `Adjustment` | 3 |

---

## ۴. ستون `IsNonProduct` (Boolean)

**مسئله:** بعضی مقادیر `StockCode` محصول واقعی نیستن (پستی، کارمزد، تخفیف و...). همچنین یه باگ حساسیت به حروف کوچک/بزرگ کشف شد (`M` در برابر `m`، هر دو یعنی "Manual").

**کدهای غیرمحصول شناسایی‌شده:** POST, DOT, M, D, C2, BANK CHARGES, S, AMAZONFEE, CRUK, B

```
= List.Contains({"POST","DOT","M","D","C2","BANK CHARGES","S","AMAZONFEE","CRUK","B"},
    Text.Upper(Text.Trim([StockCode])))
```

⚠️ نکته: کدهایی مثل `DCGSSGIRL`, `DCGSSBOY`, `PADS` با اینکه فقط حرف هستن، محصول واقعی‌اند — پس فیلتر بر پایه‌ی لیست دقیق انجام شد، نه الگوی حدسی.

✅ اعتبارسنجی: `True` = 2,912 | `False` = 538,997 (جمعاً برابر ۵۴۱,۹۰۹)

---

## ۵. حذف رکوردهای تکراری (Duplicates)

**مسئله:** 5,268 ردیف کاملاً تکراری (تمام ۸ ستون یکسان) پیدا شد.

**اقدام:** انتخاب همه‌ی ستون‌ها → `Home → Remove Rows → Remove Duplicates`

✅ اعتبارسنجی: تعداد کل ردیف‌ها از 541,909 به **536,641** رسید.

---

## ۶. ستون `Sales`

```
= [Quantity] * [UnitPrice]
```

نکته: باید نوعش دستی به **Decimal Number** تغییر داده بشه (Custom Column به‌صورت پیش‌فرض نوع "Any" می‌گیره). مقادیر منفی (برای فاکتورهای کنسلی) کاملاً عادی و مورد انتظارن.

---

## ۷. تفکیک `Date` و `Time`

از ستون اصلی `InvoiceDate` (که دست‌نخورده نگه داشته شد)، دو ستون جدید با `Add Column` ساخته شد (نه `Split Column`، تا ریسک ابهام فرمت تاریخ Locale پیش نیاد):

- `InvoiceDate_Date` ← `Add Column → Date → Date Only`
- `InvoiceDate_Time` ← `Add Column → Time → Time Only`

---

## ۸. یکسان‌سازی `StockCode` (Upper + Trim)

**مسئله:** ۱۱۲ کد محصول به‌خاطر تفاوت حروف بزرگ/کوچک (مثل `15056BL` در برابر `15056bl`) به‌عنوان دو محصول جدا شناسایی می‌شدن — این روی ۲۲,۸۳۳ ردیف اثر داشت و باعث می‌شد فروش هر محصول بین دو ردیف تقسیم بشه.

**اقدام:** روی خود ستون `StockCode` (نه ستون جدید) اعمال شد، از طریق:
`Transform → Format → Trim`، سپس `Transform → Format → UPPERCASE`

معادل فرمولی:
```
= Table.TransformColumns(#"مرحله قبلی", {{"StockCode", each Text.Upper(Text.Trim(_)), type text}})
```

✅ اعتبارسنجی: تعداد Distinct مقادیر `StockCode` از **4,070** به **3,958** رسید (دقیقاً 4070 − 112 = 3958)

---

## ۹. Trim کردن `Description`

**مسئله:** ۱۱۳,۴۵۲ ردیف (بیش از ۲۰٪ داده) فاصله‌ی اضافه در ابتدا/انتهای متن داشتن (مثلاً `"WHITE MUG "` در برابر `"WHITE MUG"`).

**اقدام:** روی خود ستون `Description` اعمال شد (بدون Uppercase، چون این ستون فقط نمایشیه و در Relationship استفاده نمیشه):
`Transform → Format → Trim`

---

## ۱۰. ستون `IsZeroPrice` (Boolean)

**مسئله:** ۲,۵۱۵ ردیف (در داده‌ی خام) مقدار `UnitPrice = 0` داشتن. بررسی دقیق‌تر نشون داد این‌ها دو دسته‌ن:
- تراکنش‌های `TransactionType = "Adjustment"` (که قبلاً با `InvoiceNo` شروع‌شده با `A` شناسایی شدن)
- یادداشت‌های انبارداری روی `StockCode` عادی — مثل `damages`, `check`, `faulty`, `found`, `amazon`, `Dotcom sales` (۴۷۷ ردیف) — که همگی بدون استثنا `UnitPrice = 0` داشتن

**تصمیم:** متن `Description` این ردیف‌ها دست‌نخورده باقی موند (چون خودش اطلاعات معتبریه)؛ فقط یه پرچم برای فیلتر کردن ساخته شد.

```
= [UnitPrice] = 0
```

✅ اعتبارسنجی: روی داده‌ی خام (541,909 ردیف) مقدار `True` = 2,515 بود؛ روی داده‌ی بعد از حذف Duplicates (536,641 ردیف) مقدار `True` = **2,510** — چون ۵ ردیف از همون ۵,۲۶۸ ردیف تکراری‌ای که حذف شد، `UnitPrice=0` هم داشتن (2515 − 5 = 2510). این یه Cross-check خوب بود که نشون داد مراحل قبلی درست کار کردن.

---

## فاز مدل‌سازی (Data Modeling)

### ساخت `CalendarTable`

جدول تاریخ مستقل با DAX ساخته شد (لازم برای Time Intelligence، چون `InvoiceDate_Date` فقط تاریخ‌هایی رو داره که تراکنش واقعی توشون بوده، نه یه بازه‌ی پیوسته):

```dax
CalendarTable = CALENDARAUTO()
```

ستون‌های کمکی:
```dax
Year = YEAR(CalendarTable[Date])
MonthNumber = MONTH(CalendarTable[Date])
MonthName = FORMAT(CalendarTable[Date], "MMMM")
Quarter = "Q" & FORMAT(CalendarTable[Date], "Q")
WeekdayNumber = WEEKDAY(CalendarTable[Date], 2)
WeekdayName = FORMAT(CalendarTable[Date], "dddd")
YearMonth = FORMAT(CalendarTable[Date], "YYYY-MM")
YearMonthSort = YEAR(CalendarTable[Date]) * 100 + MONTH(CalendarTable[Date])
```

⚠️ نکته: `MonthName` و `WeekdayName` باید با **Sort by Column** به ترتیب `MonthNumber`/`WeekdayNumber` مرتب بشن، وگرنه در نمودارها به ترتیب الفبا مرتب میشن. همین کار برای `YearMonth` هم با ستون کمکی `YearMonthSort` لازمه (وگرنه نمودار روند زمانی بر اساس مقدار فروش مرتب میشه، نه تاریخ — یه بار همین اتفاق افتاد و نمودار خط کاملاً بهم‌ریخته نشون داد).

### علامت‌گذاری Date Table
`CalendarTable` → کلیک راست → **Mark as Date Table** → ستون `Date` به‌عنوان کلید.
⚠️ نکته: باید حتماً روی خود `CalendarTable` انجام بشه، نه روی جدول اصلی `Online Retail` (که تاریخ‌های تکراری داره و اصلاً اجازه‌ی Date Table شدن رو نمیده).

### Relationship
`CalendarTable[Date]` ↔ `Online Retail[InvoiceDate_Date]` — یک‌به‌چند (1:*)، فیلتر Single-direction.

### تصمیم مدل‌سازی
مدل فعلاً **تک‌جدولی (Flat Table)** باقی موند (نه Star Schema کامل با Dim جدا برای Customer/Product)، چون برای حجم داده‌ی فعلی (536K ردیف) مشکل کارایی ایجاد نمی‌کنه. تبدیل به Star Schema به فاز بعدی موکول شد.

### نکته‌ی جانبی مهم: مشکل Locale نمایش تاریخ
تنظیمات Regional سیستم روی فارسی/ایران بود، باعث می‌شد Power BI تاریخ‌ها رو به‌صورت شمسی نمایش بده (مثلاً `21/05/1390` به‌جای `2011-08-12`). با تبدیل دستی تاریخ شمسی به میلادی تایید شد که **خود داده درست پارس شده بود**، فقط نمایشش فارسی بود. اصلاح شد از طریق:
- `File → Options and settings → Options → Global/Current File → Regional Settings → English (United States)`
- برای اطمینان بیشتر: روی ستون `InvoiceDate` → `Change Type → Using Locale` → `English (United States)` صریحاً تنظیم شد.

### اصلاح بعدی: تعویض `CALENDARAUTO()` با `CALENDAR()`
در فاز طراحی داشبورد، نمودار روند فروش خالی/بهم‌ریخته نشون می‌داد. علتش این بود که `CALENDARAUTO()` کل مدل رو برای تشخیص بازه‌ی تاریخ می‌گرده و اشتباهاً ستون `InvoiceDate - time` (که داخلی از یه تاریخ پایه‌ی ثابت `1899/12/30` استفاده می‌کنه) رو هم لحاظ کرد، و بازه رو به سال ۱۸۹۹ کشوند. اصلاح شد با بازه‌ی صریح:
```dax
CalendarTable = 
CALENDAR(
    MIN('Online Retail'[InvoiceDate_Date]),
    MAX('Online Retail'[InvoiceDate_Date])
)
```

---

## فاز DAX Measures

یه جدول خالی و مستقل فقط برای نگهداری Measure ها ساخته شد (برای مرتب‌بودن، نه یه قانون اجباری):
```dax
_Measures = ROW("Info", "این جدول فقط برای Measure ها")
```
⚠️ نکته‌ی مهم: باید حتماً از دکمه‌ی **`New Table`** ساخته بشه (نه `New Measure`). یه‌بار به اشتباه با `New Measure` ساخته شد و چون `ROW(...)` یه جدول برمی‌گردونه نه یه مقدار Scalar، جدول به‌جای مستقل بودن، داخل `CalendarTable` تودرتو (Nested) شد — باید حذف و از نو با `New Table` ساخته می‌شد.

### Measure های ساخته‌شده

```dax
Total Revenue = 
CALCULATE(
    SUM('Online Retail'[Sales]),
    'Online Retail'[TransactionType] = "Sale"
)
```

```dax
Total Orders = 
CALCULATE(
    DISTINCTCOUNT('Online Retail'[InvoiceNo]),
    'Online Retail'[TransactionType] = "Sale"
)
```

```dax
Total Customers = 
CALCULATE(
    DISTINCTCOUNT('Online Retail'[CustomerID]),
    'Online Retail'[TransactionType] = "Sale",
    'Online Retail'[HasCustomerID] = TRUE
)
```

```dax
Average Order Value = DIVIDE([Total Revenue], [Total Orders])
```

```dax
Cancelled Value = 
CALCULATE(
    SUM('Online Retail'[Sales]),
    'Online Retail'[TransactionType] = "Cancelled"
)
```

```dax
Return Rate = 
DIVIDE(
    ABS([Cancelled Value]),
    [Total Revenue] + ABS([Cancelled Value])
)
```

✅ اعتبارسنجی نهایی (تمام اعداد با محاسبه‌ی مستقل پایتون Cross-check شدن):
| Measure | مقدار |
|---|---|
| `Total Revenue` | ≈ £10,631,049 |
| `Total Orders` | 22,061 |
| `Total Customers` | 4,339 |
| `Average Order Value` | ≈ £481.89 |
| `Cancelled Value` | ≈ -£893,980 |
| `Return Rate` | ≈ 7.76% |

---

## خلاصه‌ی جدول اعتبارسنجی نهایی

| مورد | عدد |
|---|---|
| ردیف خام اولیه | 541,909 |
| ردیف بعد از حذف تکراری‌ها | 536,641 |
| CustomerID خالی | 135,080 (۲۵٪) |
| فاکتور کنسلی | 9,288 |
| رکورد اصلاح حسابداری | 3 |
| ردیف غیرمحصول (پستی/کارمزد/...) | 2,912 |
| کد محصول تکراری (فقط اختلاف حروف بزرگ/کوچک) | 112 کد (۲۲,۸۳۳ ردیف) |
| ردیف با فاصله‌ی اضافه در Description | 113,452 |
| ردیف با UnitPrice=0 (بعد از حذف تکراری‌ها) | 2,510 |

---

## فاز طراحی داشبورد (در حال انجام)

### صفحه ۱: Overview ✅

چیدمان:
```
[KPI Cards - ردیف بالا: Total Revenue, Total Orders, Total Customers, Return Rate]
[نمودار روند فروش ماهانه]   |   [نقشه‌ی فروش بر اساس کشور]
```
- ۴ Card از `_Measures`
- نمودار روند فروش: Line Chart، X=`YearMonth` (با Sort by `YearMonthSort`)، Y=`Total Revenue`
- نقشه‌ی کشورها: Map، Location=`Country`, Size=`Total Revenue`

⚠️ مشکلات حل‌شده در این صفحه:
- Map خطای غیرفعال‌بودن می‌داد با اینکه از Security فعال شده بود → با **Sign in به اکانت Microsoft + Restart کامل Power BI Desktop** حل شد (Map/Filled Map به Azure Maps و ورود به اکانت نیاز داره).
- نمودار روند فروش خالی بود → به‌خاطر باگ `CALENDARAUTO()` بود (توضیح کامل در فاز مدل‌سازی، بخش «اصلاح بعدی»).

### صفحه ۲: Products ✅

- نمودار میله‌ای پرفروش‌ترین محصولات: Y=`Description`, X=`Total Revenue`, Top N=10
- جدول جزئیات محصولات

⚠️ مشکلات حل‌شده:
- فیلتر `Top N` کار نمی‌کرد چون فیلد **By value** خالی بود → باید `_Measures[Total Revenue]` توش گذاشته بشه.
- محصولات غیرواقعی (`POSTAGE`, `Manual`, `DOTCOM POSTAGE`) توی لیست پرفروش‌ترین‌ها ظاهر می‌شدن → با فیلتر سطح صفحه `IsNonProduct = False` رفع شد.

### صفحه ۳: Customers ✅

**Measures اضافه‌شده:**
| نام | فرمول DAX | هدف |
|---|---|---|
| `Revenue per Customer` | `DIVIDE([Total Revenue], [Total Customers])` | میانگین درآمد هر مشتری |
| `Purchase Frequency` | `DIVIDE(CALCULATE(DISTINCTCOUNT('Online Retail'[InvoiceNo]), 'Online Retail'[TransactionType]="Sale", 'Online Retail'[HasCustomerID]=TRUE), [Total Customers])` | میانگین تعداد سفارش هر مشتری (فقط سفارش‌های با مشتری شناخته‌شده) |
| `Revenue per Customer PM` | `CALCULATE([Revenue per Customer], DATEADD('CalendarTable'[Date], -1, MONTH))` | مقدار ماه قبل (برای Target) |
| `Revenue per Customer PY` | `CALCULATE([Revenue per Customer], SAMEPERIODLASTYEAR('CalendarTable'[Date]))` | مقدار سال قبل (ممکنه Blank باشه چون دیتاست یک‌ساله‌ست) |

**ستون‌های محاسباتی روی `Online Retail`:**
```dax
Customer Order Count = 
CALCULATE(
    DISTINCTCOUNT('Online Retail'[InvoiceNo]),
    ALLEXCEPT('Online Retail', 'Online Retail'[CustomerID])
)
```
```dax
Purchase Frequency Bucket = 
SWITCH(
    TRUE(),
    'Online Retail'[Customer Order Count] = 1, "1 time",
    'Online Retail'[Customer Order Count] = 2, "2 times",
    'Online Retail'[Customer Order Count] >= 3 && 'Online Retail'[Customer Order Count] <= 5, "3-5 times",
    'Online Retail'[Customer Order Count] > 5, "More than 5 times",
    "Unknown"
)
```
```dax
Bucket Sort Order = 
SWITCH(
    TRUE(),
    'Online Retail'[Customer Order Count] = 1, 1,
    'Online Retail'[Customer Order Count] = 2, 2,
    'Online Retail'[Customer Order Count] >= 3 && 'Online Retail'[Customer Order Count] <= 5, 3,
    'Online Retail'[Customer Order Count] > 5, 4,
    99
)
```
→ `Purchase Frequency Bucket` با `Sort by Column → Bucket Sort Order` مرتب شد.

**فیلتر سطح صفحه:** `HasCustomerID = True`

**ویژوال‌های ساخته‌شده:**
| ویژوال | نوع | فیلدها |
|---|---|---|
| Total Customers | Card | `Total Customers` |
| Revenue per Customer | KPI | Value: `Revenue per Customer`, Trend axis: `YearMonth` |
| Purchase Frequency | Card | `Purchase Frequency` |
| روند Revenue per Customer | Line chart | X: `YearMonth`, Y: `Revenue per Customer` |
| Top 10 Customers | Bar chart | Y: `CustomerID`, X: `Revenue per Customer`, Top N=10 by `Total Revenue` |
| توزیع فرکانس خرید | Bar chart | Y: `Purchase Frequency Bucket`, X: `% Customers in Bucket` |

⚠️ **مشکلات شناسایی‌شده که هنوز باز هستن (برای جلسه‌ی بعد):**
1. کارت چهارم تکراری بود (`Average Order Value` دوبار) — ✅ **حل شد**، با `AOV` جایگزین شد.
2. ~~عدد `Purchase Frequency`...~~ ✅ **حل شد** (توضیح در بالا).
3. ~~غلط املایی "Top 10 custometr"~~ ✅ **حل شد.**
4. ~~کارت Target روی `Revenue per Customer` هدف رو برابر یه عدد نامرتبط (903.22) نشون داد...~~ ✅ **حل شد.** علت: `Trend axis` روی `YearMonth` (متنی) یا بعداً `Date` (روزانه) تنظیم شده بود — KPI برای محاسبه‌ی درست Target نیاز به یه فیلد **Date واقعی با دانه‌بندی ماهانه** داره. راه‌حل:
   ```dax
   MonthStartDate = DATE(CalendarTable[Year], CalendarTable[MonthNumber], 1)
   ```
   و ست‌کردن `Trend axis = MonthStartDate` (به‌جای `Date` یا `YearMonth`). بعد از ساخت مجدد خود KPI Visual (چون یه‌بار به‌نظر «گیر کرده» بود و به تغییر فیلدها واکنش نشون نمی‌داد)، مشکل کاملاً برطرف شد.

### صفحه ۴: Returns ✅

**Measures اضافه‌شده:**
```dax
Cancelled Orders Count = 
CALCULATE(
    DISTINCTCOUNT('Online Retail'[InvoiceNo]),
    'Online Retail'[TransactionType] = "Cancelled"
)
```
```dax
Returned Quantity = 
CALCULATE(
    SUM('Online Retail'[Quantity]),
    'Online Retail'[TransactionType] = "Cancelled"
)
```
```dax
Returned Quantity (Abs) = ABS([Returned Quantity])
```

**ویژوال‌ها:**
| ویژوال | نوع | فیلدها |
|---|---|---|
| Cancelled Value | Card | `Cancelled Value` (≈ -£893.98K) |
| Return Rate | Card | `Return Rate` (≈ 7.76%) — از فیلتر صفحه مستثنی شد چون خودش هم Sale هم Cancelled لازم داره |
| روند برگشتی در طول زمان | Line chart | X: `MonthStartDate`, Y: `Cancelled Value` |
| پرتعدادترین محصولات برگشتی | Bar chart | Y: `Description`, X: `Returned Quantity (Abs)`, Top N=10 |

**فیلتر سطح صفحه:** `TransactionType = "Cancelled"` + `IsNonProduct = False`

⚠️ مشکلات حل‌شده:
- Top N روی نمودار محصولات کار نمی‌کرد چون دکمه‌ی **Apply filter** زده نشده بود (صرفاً پرکردن «By value» کافی نیست).
- چون `Quantity` برگشتی‌ها منفیه، Top N بدون `ABS()` برعکس (نزدیک‌ترین به صفر) مرتب می‌کرد → با `Returned Quantity (Abs)` درست شد.
- رکورد `Manual` (غیرمحصول) توی لیست پرتعدادترین‌ها ظاهر شد → با فیلتر `IsNonProduct=False` رفع شد.

---

## 🎉 وضعیت پروژه: هر چهار صفحه‌ی داشبورد تکمیل شد

```
✅ فاز پاکسازی
✅ فاز مدل‌سازی
✅ فاز DAX Measures
✅ صفحه Overview
✅ صفحه Products
✅ صفحه Customers
✅ صفحه Returns
```

---

## فاز بازبینی طراحی (Visual Design Polish)

### تم رنگی سفارشی
یه فایل تم Power BI (JSON) ساخته شد تا رنگ‌بندی هر ۴ صفحه یکدست بشه (پس‌زمینه‌ی Navy تیره `#0B1F44`، کارت‌ها `#132A55`، رنگ‌های Accent: سبز `#2ECC91`، صورتی‌مرجانی `#E8788B`، آبی روشن `#4FC3E8`). فایل: `OnlineRetail_NavyTheme.json` — از `View → Themes → Browse for themes` قابل Import‌ه.

### اصلاحات صفحه Overview
- نقشه‌ی کشورها (که به‌خاطر غلبه‌ی ۸۹٪ فروش بریتانیا عملاً بی‌فایده بود) با یه **Bar Chart افقی** (`Country ≠ United Kingdom`) جایگزین شد — خواناتر و مقایسه‌پذیرتر.
- کارت `Total Revenue` به‌اشتباه از نوع **KPI** (با Trend axis + Target) ساخته شده بود، نه Card ساده. چون این باعث می‌شد فقط عدد **یه ماه** (نه کل مجموع) رو با رنگ قرمز/درصد افت نشون بده — گمراه‌کننده برای یه کارت خلاصه‌ی «کل». با یه **Card ساده** جایگزین شد تا با سه کارت دیگه هماهنگ باشه.
- عنوان‌های پیش‌فرض (`Total Revenue by YearMonth` و مشابه) باید به متن انسانی‌تر تغییر کنن (هنوز کامل انجام نشده).

### اصلاحات صفحه Products
- فیلتر `IsNonProduct = False` که قبلاً ساخته بودیم، از این صفحه افتاده بود (`DOTCOM POSTAGE`, `AMAZON FEE` دوباره توی «پرفروش‌ترین‌ها» ظاهر شدن) → دوباره به‌عنوان فیلتر سطح صفحه اضافه شد.
- جدول کنار نمودار میله‌ای بدون قاب/پس‌زمینه‌ی هماهنگ با تم بود → با Background `#132A55` و Border `#1E3A6E` (Radius=8) قابدار شد، عنوان `Product Details` اضافه شد، و بر اساس `Total Revenue` نزولی Sort شد تا با نمودار میله‌ای هماهنگ باشه. ✅
- کارت‌های KPI صفحه Customers که بردر ناهماهنگ داشتن (یکی نارنجی، بقیه سبز/سفید) → یکدست شدن. ✅
- عنوان صفحات ناهماهنگ بودن (`Customer Analysis Dashboard`, `Retrun`) → به `Customers` و `Returns` تغییر کردن، هماهنگ با سبک یک‌کلمه‌ای بقیه‌ی صفحات (`Overview`, `Products`). ✅
- یه المان اضافه/خراب (متن ناقص "R RP") روی صفحه Returns پیدا و حذف شد. ✅
- فرمت عدد کارت `Total Revenue` در Overview (که به‌صورت خام `9.8095960M` نمایش داده می‌شد) با `Display units: Millions` و ۱ رقم اعشار به `£9.8M` اصلاح شد. ✅

### ناوبری سفارشی (Custom Navigation)
به‌جای تب‌های پیش‌فرض پایین صفحه (که ظاهر آماتور داشتن)، روی هر صفحه ۴ دکمه‌ی سفارشی (`Insert → Buttons → Blank` + Action = Page navigation) برای رفتن بین صفحات ساخته شد. بعد از تست دکمه‌ها (با `Ctrl+کلیک` در حالت Edit، چون Reading View فقط در Power BI Service وجود دارد نه Desktop)، صفحات `Products`, `Customers`, `Returns` با کلیک راست روی تب پایین → **Hide Page** مخفی شدن؛ فقط `Overview` به‌عنوان صفحه‌ی ورودی نمایان باقی موند.

⚠️ نکته: صفحات مخفی‌شده در حالت Edit (با آیکون چشم خط‌خورده) هنوز برای سازنده قابل‌دسترسی‌اند؛ فقط برای بیننده‌ی نهایی (بعد از Publish یا در Reading View) واقعاً مخفی می‌شوند.

### صفحه‌ی عنوان (Cover Page)
یه صفحه‌ی جدید به اسم `Cover` قبل از `Overview` اضافه شد، شامل: عنوان پروژه، زیرعنوان (بازه‌ی زمانی + تعداد رکورد)، نام تحلیلگر، توضیح کوتاه پروژه، و یک دکمه‌ی `View Dashboard →` که به صفحه‌ی `Overview` وصل می‌شود. این صفحه Hide نشد (باید اولین چیزی باشد که دیده می‌شود).

### Insight Callout ها
زیر مهم‌ترین نمودار هر صفحه، یک Text box با یک جمله‌ی بینش (نه فقط عدد خام) اضافه شد — فونت ۱۱-۱۲px Italic، رنگ `#C9D3E8`:

| صفحه | Insight Callout |
|---|---|
| Overview | 💡 85% of revenue comes from the UK alone — a significant geographic concentration risk |
| Products | 💡 REGENCY CAKESTAND 3 TIER is the top-selling product, generating £174K in revenue |
| Customers | 💡 Just 18% of customers ("Champions") drive 60% of total revenue |
| Returns | 💡 Returns are concentrated in a handful of specific products, not spread across the catalog |

### اصول کلی طراحی حرفه‌ای که در این بازبینی استفاده شد
1. **رنگ‌بندی محدود:** حداکثر ۲-۳ رنگ اصلی + خنثی، نه رنگین‌کمون
2. **هم‌ترازی دقیق:** همه‌ی KPI Card ها هم‌قد و هم‌فاصله (`Format → Align/Distribute`)
3. **فضای خالی (Whitespace):** ترجیح ۴-۵ Visual تمیز به‌جای ۸-۹ تای فشرده
4. **همخوانی عنوان با محتوا:** یه Visual نباید عنوانی داشته باشه که با نوع محاسبه‌اش (کل در برابر دوره‌ای) مغایرت داشته باشه
5. **راهنمایی مسیر کاربر:** ناوبری سفارشی + صفحه‌ی عنوان، تجربه‌ی کاربر رو از «یه‌مشت نمودار» به «یه گزارش هدایت‌شده» تبدیل می‌کنه
6. **داستان‌گویی با داده:** Insight Callout ها، عدد خام رو به یه پیام قابل‌فهم برای هر بیننده تبدیل می‌کنن

---

## راهنمای تحلیل با داشبورد (چطور از داشبورد استفاده کنیم)

داشبورد یه ابزار تصمیم‌گیریه، نه گزارش تزئینی. فرآیند پیشنهادی برای تحلیل:

1. **با یه سوال مشخص شروع کن** — نه «بذار ببینم چی هست»، بلکه چیزی مشخص مثل «چرا فروش آبان بالا رفت؟»
2. **با Overview شروع کن** — دنبال نقاط غیرعادی باش (جهش، افت، عدد غیرمنتظره)
3. **با Slicer فیلتر کن تا مقایسه کنی** — یه بازه یا کشور خاص رو جدا کن، ببین عدد چطور تغییر می‌کنه
4. **روی جزئیات کلیک کن (Drill down)** — برو Products/Customers، ببین کدوم مورد خاص باعث اون تغییر شده
5. **یه فرضیه بساز و با داده تستش کن** — مثلاً «فکر می‌کنم بیشتر برگشتی‌ها مال محصولات کریسمسی‌ان»؛ برو چک کن
6. **نتیجه رو به یه جمله‌ی قابل‌اقدام تبدیل کن** — نه فقط عدد، بلکه یه پیشنهاد عملی

**مثال واقعی از همین دیتاست:** افت شدید فروش در ۹ دسامبر ۲۰۱۱ (آخرین نقطه‌ی نمودار روند) یه سیگنال کسب‌وکاری واقعی نیست — چون خود دیتاست دقیقاً همون‌جا قطع می‌شه (داده‌ی ناقص برای اون ماه، نه افت واقعی فروش). این نکته باید صریح توی گزارش نهایی ذکر بشه تا کسی تصمیم اشتباه نگیره.

---
