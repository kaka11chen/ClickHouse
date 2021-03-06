---
machine_translated: true
machine_translated_rev: 72537a2d527c63c07aa5d2361a8829f3895cf2bd
---

### در اپراتورها {#select-in-operators}

این `IN`, `NOT IN`, `GLOBAL IN` و `GLOBAL NOT IN` اپراتورها به طور جداگانه تحت پوشش, از قابلیت های خود را کاملا غنی است.

سمت چپ اپراتور یا یک ستون یا یک تاپل است.

مثالها:

``` sql
SELECT UserID IN (123, 456) FROM ...
SELECT (CounterID, UserID) IN ((34, 123), (101500, 456)) FROM ...
```

اگر سمت چپ یک ستون است که در شاخص است, و در سمت راست مجموعه ای از ثابت است, سیستم با استفاده از شاخص برای پردازش پرس و جو.

Don't list too many values explicitly (i.e. millions). If a data set is large, put it in a temporary table (for example, see the section “External data for query processing”), سپس با استفاده از یک خرده فروشی.

سمت راست اپراتور می تواند مجموعه ای از عبارات ثابت, مجموعه ای از تاپل با عبارات ثابت (نشان داده شده در نمونه های بالا), و یا به نام یک جدول پایگاه داده و یا زیرخاکی در داخل پرانتز را انتخاب کنید.

اگر سمت راست اپراتور نام یک جدول است (مثلا, `UserID IN users`), این معادل به خرده فروشی است `UserID IN (SELECT * FROM users)`. با استفاده از این هنگام کار با داده های خارجی است که همراه با پرس و جو ارسال می شود. مثلا, پرس و جو را می توان همراه با مجموعه ای از شناسه کاربر لود شده به ارسال ‘users’ جدول موقت, که باید فیلتر شود.

اگر در سمت راست اپراتور یک نام جدول است که موتور مجموعه ای است (مجموعه داده های تهیه شده است که همیشه در رم), مجموعه داده خواهد شد بیش از دوباره برای هر پرس و جو ایجاد نمی.

زیرخاکری ممکن است بیش از یک ستون برای فیلتر کردن تاپل را مشخص کنید.
مثال:

``` sql
SELECT (CounterID, UserID) IN (SELECT CounterID, UserID FROM ...) FROM ...
```

ستون به سمت چپ و راست اپراتور در باید همان نوع.

اپراتور در و زیرخاکی ممکن است در هر بخشی از پرس و جو رخ می دهد, از جمله در توابع کل و توابع لامبدا.
مثال:

``` sql
SELECT
    EventDate,
    avg(UserID IN
    (
        SELECT UserID
        FROM test.hits
        WHERE EventDate = toDate('2014-03-17')
    )) AS ratio
FROM test.hits
GROUP BY EventDate
ORDER BY EventDate ASC
```

``` text
┌──EventDate─┬────ratio─┐
│ 2014-03-17 │        1 │
│ 2014-03-18 │ 0.807696 │
│ 2014-03-19 │ 0.755406 │
│ 2014-03-20 │ 0.723218 │
│ 2014-03-21 │ 0.697021 │
│ 2014-03-22 │ 0.647851 │
│ 2014-03-23 │ 0.648416 │
└────────────┴──────────┘
```

برای هر روز پس از مارس 17, تعداد درصد تعداد بازدید از صفحات ساخته شده توسط کاربران که سایت در مارس 17 بازدید.
یک خرده فروشی در بند در همیشه اجرا فقط یک بار بر روی یک سرور. هیچ زیرمجموعه وابسته وجود دارد.

## پردازش پوچ {#null-processing-1}

در طول پردازش درخواست, در اپراتور فرض می شود که در نتیجه یک عملیات با [NULL](../syntax.md#null-literal) همیشه برابر است با `0`, صرف نظر از اینکه `NULL` است در سمت راست یا چپ اپراتور. `NULL` ارزش ها در هر مجموعه داده شامل نمی شود, به یکدیگر مربوط نیست و نمی توان در مقایسه.

در اینجا یک مثال با است `t_null` جدول:

``` text
┌─x─┬────y─┐
│ 1 │ ᴺᵁᴸᴸ │
│ 2 │    3 │
└───┴──────┘
```

در حال اجرا پرس و جو `SELECT x FROM t_null WHERE y IN (NULL,3)` به شما نتیجه زیر را می دهد:

``` text
┌─x─┐
│ 2 │
└───┘
```

شما می توانید ببینید که ردیف که `y = NULL` از نتایج پرس و جو پرتاب می شود. دلیل این است که تاتر نمی توانید تصمیم بگیرید که چه `NULL` در `(NULL,3)` تنظیم, بازگشت `0` به عنوان نتیجه عملیات و `SELECT` این سطر را از خروجی نهایی حذف می کند.

``` sql
SELECT y IN (NULL, 3)
FROM t_null
```

``` text
┌─in(y, tuple(NULL, 3))─┐
│                     0 │
│                     1 │
└───────────────────────┘
```

## توزیع Subqueries {#select-distributed-subqueries}

دو گزینه برای در بازدید کنندگان با کارخانه های فرعی وجود دارد (شبیه به می پیوندد): طبیعی `IN` / `JOIN` و `GLOBAL IN` / `GLOBAL JOIN`. در نحوه اجرا برای پردازش پرس و جو توزیع شده متفاوت است.

!!! attention "توجه"
    به یاد داشته باشید که الگوریتم های زیر توضیح داده شده ممکن است متفاوت بسته به کار [تنظیمات](../../operations/settings/settings.md) `distributed_product_mode` تنظیمات.

هنگام استفاده از به طور منظم در, پرس و جو به سرور از راه دور ارسال, و هر یک از اجرا می شود کارخانه های فرعی در `IN` یا `JOIN` بند بند.

هنگام استفاده از `GLOBAL IN` / `GLOBAL JOINs`, اول همه زیرمجموعه ها برای اجرا `GLOBAL IN` / `GLOBAL JOINs` و نتایج در جداول موقت جمع می شوند. سپس جداول موقت به هر سرور از راه دور ارسال, جایی که نمایش داده شد با استفاده از این داده های موقت اجرا.

برای پرس و جو غیر توزیع, استفاده از به طور منظم `IN` / `JOIN`.

مراقب باشید در هنگام استفاده از کارخانه های فرعی در `IN` / `JOIN` بند برای پردازش پرس و جو توزیع.

بیایید نگاهی به برخی از نمونه. فرض کنید که هر سرور در خوشه طبیعی است **_تمل**. هر سرور همچنین دارای یک **توزیع _تماس** جدول با **توزیع شده** نوع, که به نظر می رسد در تمام سرور در خوشه.

برای پرس و جو به **توزیع _تماس** پرس و جو به تمام سرورهای راه دور ارسال می شود و با استفاده از **_تمل**.

برای مثال پرس و جو

``` sql
SELECT uniq(UserID) FROM distributed_table
```

به تمام سرورهای راه دور ارسال می شود

``` sql
SELECT uniq(UserID) FROM local_table
```

و به صورت موازی اجرا می شود تا زمانی که به مرحله ای برسد که نتایج متوسط می تواند ترکیب شود. سپس نتایج میانی به سرور درخواست کننده بازگردانده می شود و با هم ادغام می شوند و نتیجه نهایی به مشتری ارسال می شود.

حالا اجازه دهید به بررسی یک پرس و جو با در:

``` sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM local_table WHERE CounterID = 34)
```

-   محاسبه تقاطع مخاطبان از دو سایت.

این پرس و جو خواهد شد به تمام سرور از راه دور به عنوان ارسال می شود

``` sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM local_table WHERE CounterID = 34)
```

به عبارت دیگر, داده های تعیین شده در بند در خواهد شد بر روی هر سرور به طور مستقل جمع, تنها در سراسر داده است که به صورت محلی بر روی هر یک از سرور های ذخیره شده.

این به درستی و بهینه کار خواهد کرد اگر شما برای این مورد تهیه و داده ها در سراسر سرورهای خوشه گسترش یافته اند به طوری که داده ها را برای یک شناسه تنها ساکن به طور کامل بر روی یک سرور واحد. در این مورد, تمام اطلاعات لازم در دسترس خواهد بود به صورت محلی بر روی هر سرور. در غیر این صورت نتیجه نادرست خواهد بود. ما به این تنوع از پرس و جو به عنوان مراجعه کنید “local IN”.

برای اصلاح چگونه پرس و جو کار می کند زمانی که داده ها به طور تصادفی در سراسر سرور خوشه گسترش, شما می توانید مشخص **توزیع _تماس** در داخل یک خرده فروشی. پرس و جو شبیه به این خواهد بود:

``` sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

این پرس و جو خواهد شد به تمام سرور از راه دور به عنوان ارسال می شود

``` sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

خرده فروشی شروع خواهد شد در حال اجرا بر روی هر سرور از راه دور. پس از زیرخاکری با استفاده از یک جدول توزیع, خرده فروشی است که در هر سرور از راه دور خواهد شد به هر سرور از راه دور به عنوان خشمگین

``` sql
SELECT UserID FROM local_table WHERE CounterID = 34
```

مثلا, اگر شما یک خوشه از 100 سرور, اجرای کل پرس و جو نیاز 10,000 درخواست ابتدایی, که به طور کلی در نظر گرفته غیر قابل قبول.

در چنین مواردی, شما همیشه باید جهانی به جای در استفاده از. بیایید نگاه کنیم که چگونه برای پرس و جو کار می کند

``` sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID GLOBAL IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

سرور درخواست کننده خرده فروشی را اجرا خواهد کرد

``` sql
SELECT UserID FROM distributed_table WHERE CounterID = 34
```

و در نتیجه خواهد شد در یک جدول موقت در رم قرار داده است. سپس درخواست خواهد شد به هر سرور از راه دور به عنوان ارسال

``` sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID GLOBAL IN _data1
```

و جدول موقت `_data1` خواهد شد به هر سرور از راه دور با پرس و جو ارسال (نام جدول موقت پیاده سازی تعریف شده است).

این مطلوب تر از استفاده از نرمال در است. با این حال, نگه داشتن نکات زیر را در ذهن:

1.  هنگام ایجاد یک جدول موقت داده های منحصر به فرد ساخته شده است. برای کاهش حجم داده های منتقل شده بر روی شبکه مشخص متمایز در زیرخاکری. (شما لازم نیست برای انجام این کار برای عادی در.)
2.  جدول موقت خواهد شد به تمام سرور از راه دور ارسال. انتقال برای توپولوژی شبکه به حساب نمی. مثلا, اگر 10 سرور از راه دور در یک مرکز داده است که در رابطه با سرور درخواست بسیار از راه دور اقامت, داده ها ارسال خواهد شد 10 بار بیش از کانال به مرکز داده از راه دور. سعی کنید برای جلوگیری از مجموعه داده های بزرگ در هنگام استفاده از جهانی در.
3.  هنگام انتقال داده ها به سرور از راه دور, محدودیت در پهنای باند شبکه قابل تنظیم نیست. شما ممکن است شبکه بیش از حد.
4.  سعی کنید برای توزیع داده ها در سراسر سرور به طوری که شما لازم نیست که به استفاده از جهانی را به صورت منظم.
5.  اگر شما نیاز به استفاده از جهانی در اغلب, برنامه ریزی محل خوشه خانه کلیک به طوری که یک گروه واحد از کپی ساکن در بیش از یک مرکز داده با یک شبکه سریع بین, به طوری که یک پرس و جو را می توان به طور کامل در یک مرکز داده واحد پردازش.

همچنین حس می کند برای مشخص کردن یک جدول محلی در `GLOBAL IN` بند, در صورتی که این جدول محلی تنها بر روی سرور درخواست در دسترس است و شما می خواهید به استفاده از داده ها را از روی سرور از راه دور.
