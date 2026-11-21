<p align="center"><a href="README.md">English</a> | <b>فارسی</b></p>

# http/1.1 در پایتون

<p align="center">
  <a href="https://app.codecrafters.io/users/EhsanIC">
    <img src="https://backend.codecrafters.io/progress/http-server/f9ed6ef7-6647-42c3-8f7d-071db4eb3c41" alt="CodeCrafters progress" width="100%"/>
  </a>
</p>

یک سرور HTTP/1.1 که از صفر با پایتون نوشته شده است؛ فقط با سوکت‌های کتابخانهٔ استاندارد و بدون هیچ فریم‌ورک وب.

## امکانات

- سرور خام TCP روی `localhost:4221`
- مسیرها: `GET /`، `GET /echo/{str}`، `GET /user-agent`، `GET /files/{filename}`، `POST /files/{filename}`؛ هر مسیر دیگری `404` برمی‌گرداند
- فلگ اختیاری `--directory <path>` پوشه‌ای را تعیین می‌کند که `/files` از آن سرو می‌شود
- فشرده‌سازی gzip همراه با هدر `Content-Encoding: gzip` وقتی درخواست `Accept-Encoding: gzip` داشته باشد
- یک ترد برای هر اتصال، هر کدام با وضعیت خصوصی خودش
- اتصال‌های پایدار: هر سوکت در یک حلقه چندین درخواست را مدیریت می‌کند
- فریم‌بندی درخواست بر اساس `Content-Length`؛ بایت‌های باقی‌مانده در بافر می‌مانند، بنابراین درخواست‌های pipelined درست کار می‌کنند
- هدر `Connection: close` در پاسخ تکرار می‌شود و بعد از آن سوکت بسته می‌شود

## ساختار پروژه

```
http/
├── app/
│   ├── main.py                   # نقطهٔ ورود: حلقهٔ accept، حلقهٔ خواندن هر اتصال، بستن اتصال
│   ├── request.py                # پارس درخواست و فریم‌بندی با Content-Length
│   ├── handlers.py               # جدول مسیرها و منطق هر مسیر
│   └── response.py               # خط وضعیت، هدرها، بدنه، gzip، Connection: close
├── test/
│   ├── helpers.py                # خواندن دقیقاً یک پاسخ، فریم‌شده با Content-Length
│   ├── test_client.py            # مسیرها، echo و پاسخ User-Agent
│   ├── test_files.py             # GET /files/{filename}
│   ├── test_post_files.py        # POST /files/{filename}
│   ├── test_concurrent.py        # چند اتصال هم‌زمان وقتی یکی بی‌کار مانده است
│   ├── test_persistent.py        # keep-alive، pipelining، کدگذاری جدا برای هر درخواست
│   └── test_connection_close.py  # هدر Connection: close و بسته‌شدن سوکت
├── client_test.ps1               # سرور را اجرا می‌کند، همهٔ تست‌ها را می‌زند و خلاصه چاپ می‌کند
├── your_program.sh               # اسکریپت اجرا، هم محلی و هم روی CodeCrafters
└── docs/                         # تصاویری که در این README استفاده شده‌اند
```

## نحوهٔ کار

**حلقهٔ accept.** تابع `main()` روی `localhost:4221` گوش می‌دهد و برای هر اتصال یک ترد daemon اجرا می‌کند تا یک کلاینت کند بقیه را بلاک نکند.

**بافر و فریم‌بندی.** هر اتصال بافر بایتی مخصوص خودش را دارد. `parse_request()` دنبال `\r\n\r\n` می‌گردد، `Content-Length` را می‌خواند و فقط وقتی کل بدنه رسیده باشد `(raw, leftover)` برمی‌گرداند. در غیر این صورت `(None, buffer)` برمی‌گرداند و حلقه بایت بیشتری می‌خواند. باقی‌مانده در بافر می‌ماند و همین باعث می‌شود pipelining کار کند: دو درخواست در یک بسته، به‌صورت دو درخواست جداگانه پردازش می‌شوند.

**هندلر.** درخواست کامل به یک `Request` تبدیل می‌شود (متد، مسیر، route، هدرها، بدنه)، در `ROUTES` جست‌وجو می‌شود و به هندلر مربوطه می‌رسد. هندلر بایت‌های پاسخ را برمی‌گرداند. اگر درخواست `Connection: close` داشته باشد، هدر به پاسخ اضافه می‌شود و حلقهٔ اتصال برمی‌گردد و سوکت بسته می‌شود.

## پروتکل

HTTP/1.1 ساده روی TCP. یک تبادل نمونه:

```http
GET /echo/banana HTTP/1.1
Host: localhost:4221

HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 6

banana
```

- درخواست‌ها با `Content-Length` فریم‌بندی می‌شوند؛ اگر این هدر نباشد بدنه خالی است.
- پاسخ‌ها همیشه `Content-Type` و `Content-Length` دارند.
- کدهای وضعیت: `200`، `201`، `404`.
- `Accept-Encoding: gzip` بدنه را فشرده می‌کند و `Content-Encoding: gzip` اضافه می‌شود.
- `Connection: close` در پاسخ تکرار می‌شود و سپس سوکت بسته می‌شود.

## تست‌ها

### تست‌های مرحله‌ای CodeCrafters

![CodeCrafters tests](docs/codecrafters-tests.png)

همهٔ مراحل رسمی پاس می‌شوند.

### کلاینت تست سفارشی

`client_test.ps1` سرور را اجرا می‌کند و مجموعه‌های تست پایتونی را در سطح سوکت اجرا می‌کند: مسیرها، echo، User-Agent، GET و POST فایل، gzip، keep-alive، pipelining، اتصال‌های هم‌زمان و `Connection: close`. پاسخ‌ها را بر اساس `Content-Length` می‌خواند، نه با انتظار برای بسته‌شدن سوکت، و در پایان یک خلاصهٔ یکجا چاپ می‌کند.

```powershell
pwsh ./client_test.ps1
```

![client test](docs/client-test.png)

این کلاینت تست با هوش مصنوعی ساخته شده است.

## نویسندگی

سرور HTTP داخل پوشهٔ `app/` را خودم نوشته‌ام. از هوش مصنوعی برای توضیح مفاهیم و بازبینی کد استفاده کرده‌ام. کلاینت تست (`test/` و `client_test.ps1`) با هوش مصنوعی ساخته شده است.

## چیزهایی که یاد گرفتم

- فریم‌بندی HTTP/1.1 روی سیم: TCP یک جریان بایت است و `Content-Length` — نه مرز بسته‌ها — مشخص می‌کند یک درخواست کجا تمام می‌شود.
- چرا بافر مخصوص هر اتصال برای keep-alive و pipelining لازم است؛ خواندن یک‌باره برای هر درخواست به محض اینکه دو درخواست در یک بسته باشند می‌شکند.
- چرخهٔ عمر اتصال: به‌صورت پیش‌فرض keep-alive، بستن با `Connection: close`، و تصمیم‌گیری برای هر درخواست روی سوکتی که باز می‌ماند.
- تست سرور در سطح سوکت، از جمله تشخیص «بسته‌شدن واقعی» از «معلق ماندن».
