<div dir="rtl" align="right">

# راهنمای عملیاتی (Runbook) نصب و راه‌اندازی آزمایشگاهی FreePBX 14 و Asterisk 13

**نسخه سند:** 1.0 — ۱۴۰۵/۰۷/۱۸ (2026-10-10)  
**دامنه:** ماشین مجازی آزمایشگاهی، بدون اینترنت  
**استثناء:** مراحل نصب VMware Tools عمداً در این سند نیامده‌اند.

> **نکته درباره دقت سند:** این Runbook بر مبنای وضعیت تأییدشده و فرمان‌های ثبت‌شده در جلسه تهیه شده است. فرمان‌های اولیه نصب CentOS، کامپایل Asterisk از سورس و نصب اولیه FreePBX به‌طور کامل در اطلاعات در دسترس این سند وجود ندارند؛ بنابراین این مراحل به‌عنوان پیش‌نیاز و نقاط کنترل نوشته شده‌اند، نه یک اسکریپت نصب بازسازی‌شده و تأییدنشده.

## ۱. هدف و محدوده

راه‌اندازی یک PBX آزمایشگاهی با نسخه‌های **دقیق** زیر، تأیید عملکرد وب، AMI و SIP، و فراهم‌کردن امکان رجیستر و تماس داخلی بدون دست‌زدن به PBX عملیاتی.

| مؤلفه | نسخه / مقدار مشاهده‌شده |
|---|---|
| سیستم‌عامل | CentOS 7.9، کرنل `3.10.0-1160.el7.x86_64` |
| نام میزبان (نمونه) | `pbx-lab.example.test` |
| IP نمونه (ویژه مستندات) | `192.0.2.10/24` |
| Asterisk | **`13.20.0`**، کامپایل‌شده از سورس |
| FreePBX | **`14.0.16.9`** |
| Apache | `httpd 2.4.6` |
| PHP | `5.6.40` |
| MariaDB | `5.5.68` |
| Node.js | `8.17.0` |
| SIP | `chan_pjsip` پورت UDP `5060`؛ `chan_sip` پورت UDP `5160` |
| AMI | TCP `5038` |

**هشدار:** CentOS 7، PHP 5.6، FreePBX 14 و Asterisk 13 قدیمی و خارج از پشتیبانی هستند. از این پیکربندی صرفاً برای بازتولید و تست استفاده کنید. برای محیط عملیاتی حتماً **SELinux و Firewall باید به‌درستی پیکربندی و فعال باشند**؛ جزئیات آن طبق درخواست خارج از محدوده این Runbook است. عدم اتصال به اینترنت، ایمنی شبکه داخلی را تضمین نمی‌کند.

## ۲. پیش‌نیازها و آماده‌سازی

- VM ایزوله با دسترسی مدیریتی کنسول، دیسک و شبکه پایدار؛ از VM Snapshot بگیرید.
- رسانه‌ها و بسته‌های آفلاین **سازگار با نسخه‌های دقیق** بالا را از قبل تأمین و صحت‌سنجی کنید: سورس رسمی Asterisk `13.20.0`، بسته نصب FreePBX `14.0.16.9`، وابستگی‌های build، PHP 5.6، MariaDB، Apache و Node.js.
- نام میزبان، DNS/hosts، مسیرها، زمان سیستم و دسترسی وب کلاینت آزمایشگاهی را کنترل کنید.
- حساب اجرای سرویس‌های FreePBX/Asterisk باید `asterisk` باشد؛ هنگام اجرا و مالکیت فایل‌ها آن را بررسی کنید.

```bash
hostnamectl
ip -4 addr
uname -r
id asterisk
rpm -q httpd mariadb-server php
php -v
node -v
```

**مسیرهای اصلی تأییدشده:**

```text
/usr/sbin/asterisk
/usr/sbin/fwconsole
/etc/asterisk/
/etc/amportal.conf
/var/www/html/admin/
/var/log/asterisk/freepbx.log
/var/spool/asterisk/cache
/home/asterisk/.gnupg
```

## ۳. نصب پایه — راهنمای کنترل، نه بازسازی دستورهای قبلی

مراحل اولیه در جلسه انجام شده‌اند، اما فرمان‌های کامل آنها قابل بازیابی از اطلاعات موجود نیستند؛ برای بازسازی ماشین، از سورس/بسته‌های آرشیوشده همان نسخه استفاده و این ترتیب را رعایت کنید:

1. CentOS 7.9 و وابستگی‌های توسعه را از منابع **آفلاینِ معتبر** نصب و نسخه‌ها را تثبیت کنید.
2. Asterisk **دقیقاً `13.20.0`** را از سورس همان نسخه build/install کنید؛ در پیکربندی build وجود `chan_sip`، `chan_pjsip` و `app_macro` را کنترل کنید. **دستورهای configure/menuselect و dependencyهای دقیق نصب نخست در سوابق موجود نیستند؛ حدس نزنید.**
3. MariaDB، Apache و PHP 5.6 را نصب و فعال کنید؛ پایگاه داده و حساب‌های اختصاصی FreePBX را مطابق بسته نصب تنظیم کنید. رمزها را در سند قرار ندهید.
4. FreePBX **دقیقاً `14.0.16.9`** را با روش رسمی سازگار با این نسخه نصب کنید. تنظیم Apache به اجرای workerهای وب با کاربر `asterisk` در این محیط انجام شده بود.
5. `memory_limit` در `/etc/php.ini` به `256M` و timezone به `Asia/Tehran` تنظیم شده بود؛ در بازسازی بررسی کنید.
6. پس از نصب، `fwconsole`، ورود به پنل و فایل‌های پیکربندی Asterisk را بررسی کنید؛ نسخه‌ها را خودکار به‌روزرسانی نکنید.

```bash
asterisk -V
asterisk -rx 'module show like chan_sip'
asterisk -rx 'module show like chan_pjsip'
asterisk -rx 'module show like app_macro'
php -r 'echo ini_get("memory_limit"), PHP_EOL, date_default_timezone_get(), PHP_EOL;'
```

## ۴. سرویس FreePBX در systemd

در این Lab سرویس سفارشی `freepbx.service` به‌شکل `oneshot` تنظیم شده است. نمونه ثبت‌شده:

</div>

```ini
# /etc/systemd/system/freepbx.service
[Unit]
Description=FreePBX VoIP Server
Wants=network-online.target
After=network-online.target mariadb.service
Requires=mariadb.service

[Service]
Type=oneshot
User=root
RemainAfterExit=yes
ExecStart=/usr/sbin/fwconsole start
ExecStop=/usr/sbin/fwconsole stop
TimeoutStartSec=180
TimeoutStopSec=120

[Install]
WantedBy=multi-user.target
```

<div dir="rtl" align="right">

در ماشین بازسازی‌شده، پس از ایجاد فایل سرویس:

</div>

```bash
systemctl daemon-reload
systemctl enable freepbx
systemctl start mariadb httpd
systemctl start freepbx
systemctl is-active mariadb httpd freepbx
asterisk -rx 'core show uptime'
```

<div dir="rtl" align="right">

`active (exited)` برای **همین سرویس سفارشی oneshot** طبیعی است؛ باید جداگانه زنده بودن پردازش Asterisk را تأیید کنید. `fwconsole status` در نسخه مورد استفاده دستور معتبری نبود.

## ۵. دسترسی وب و بررسی اولیه

آدرس پنل در Lab:

</div>

```text
http://192.0.2.10/admin/config.php
```

<div dir="rtl" align="right">

</div>

```bash
systemctl is-active httpd mariadb freepbx
curl -sS -L -o /dev/null -w 'HTTP=%{http_code}\n' http://127.0.0.1/admin/config.php
```

<div dir="rtl" align="right">

**مهم:** دریافت `HTTP=200` اثبات نمی‌کند Dashboard بدون Exception بارگذاری شده است؛ صفحه را واقعاً در مرورگر باز و مشاهده کنید. در Lab پس از رفع اشکال، وضعیت Asterisk، MySQL و Web Server در داشبورد سبز دیده شد. مقدار Mail Queue ممکن است به پیکربندی ایمیل بستگی داشته باشد.

### هشدار Cache

هشدار `Cache path is not writable!` به مسیر `/var/spool/asterisk/cache` اشاره داشت؛ مالکیت و امکان نوشتن بررسی شدند. بعد از رفع مانع، اعلان قدیمی با این دستور حذف شد و در بررسی تصویری برنگشت:

</div>

```bash
fwconsole notification --list
fwconsole notification --delete framework CACHEPATCH
```

<div dir="rtl" align="right">

**قبل از حذف اعلان، علت واقعی را برطرف کنید**؛ پاک کردن اعلان، اصلاح دسترسی نیست. پوشه‌های `assets/less/cache` ماژول‌های FreePBX نیز باید قابل‌نوشتن برای فرایند وب باشند. در صورت مشاهده Exception مربوط به LESS، مسیر دقیق ذکرشده را بررسی کنید. ابزار عیب‌یابی:

</div>

```bash
ls -ld /var/spool/asterisk/cache
ls -ld /var/www/html/admin/modules/dashboard/assets/less/cache
namei -l /var/spool/asterisk/cache
fwconsole notification --list
```

<div dir="rtl" align="right">

### GPG و امضاهای ماژول

در نصب آزمایشگاهی، خطای عدم دسترسی به `/home/asterisk/.gnupg` دیده شد و رفع شد. مالک فایل‌ها `asterisk` بود و در پایان، مجوزهای محدود زیر برای آن تأیید شدند:

</div>

```bash
chmod 700 /home/asterisk/.gnupg
find /home/asterisk/.gnupg -type d -exec chmod 700 {} \;
find /home/asterisk/.gnupg -type f -exec chmod 600 {} \;
```

<div dir="rtl" align="right">

برای جلوگیری از افشای کلیدها، **محتویات `secring.gpg` و سایر کلیدها را چاپ یا ارسال نکنید**. جزئیات سیاست امنیتی به درخواست کاربر از این سند کنار گذاشته شده است.

## ۶. تنظیم AMI

AMI در این Lab روی TCP `5038` فعال بود. برای مشاهده وضعیت:

</div>

```bash
ss -lntp | grep ':5038'
asterisk -rx 'manager show users'
```

<div dir="rtl" align="right">

برای تعریف کاربران، از فایل **`/etc/asterisk/manager_custom.conf`** استفاده کنید؛ تغییر مستقیم فایل‌های تولیدشده FreePBX ممکن است با Apply Config بازنویسی شود. پیش از ویرایش فایل، پشتیبان تهیه کنید. دو پروفایل موردنظر:

- `monitor_ro`: صرفاً رویدادها/دسترسی‌های خواندنی AMI، بدون اختیار write.
- `control_rw`: دسترسی خواندن و نوشتن کامل **در حد نیاز تست**؛ این حساب پرقدرت است.

دو شبکه `198.51.100.0/24` و `203.0.113.0/24` صرفاً **نمونه مستندسازی** هستند و قبل از اجرا باید با IPهای مورداعتماد جایگزین شوند؛ ترجیحاً هر حساب فقط از IP میزبان موردنیاز مجاز باشد.

</div>

```ini
; Add to /etc/asterisk/manager_custom.conf
; Replace secrets locally. Do not commit credentials.

[monitor_ro]
secret = REPLACE_WITH_UNIQUE_STRONG_READER_SECRET
deny = 0.0.0.0/0.0.0.0
permit = 198.51.100.0/255.255.255.0
permit = 203.0.113.0/255.255.255.0
read = system,call,log,verbose,agent,user,config,dtmf,reporting,cdr,dialplan
write =

[control_rw]
secret = REPLACE_WITH_UNIQUE_STRONG_ADMIN_SECRET
deny = 0.0.0.0/0.0.0.0
permit = 198.51.100.0/255.255.255.0
permit = 203.0.113.0/255.255.255.0
read = all
write = all
```

<div dir="rtl" align="right">

**هشدار فنی:** AMI `read` و `write` مدل مجوز Action/رویدادهای Asterisk است؛ برای اثبات اینکه `monitor_ro` هیچ Action تغییردهنده‌ای را اجرا نمی‌کند، تست منفی جداگانه انجام دهید. نام‌ها را دوبار در فایل تعریف نکنید و رمز واقعی را در لاگ/مخزن ثبت نکنید.

</div>

```bash
cp -a /etc/asterisk/manager_custom.conf /root/manager_custom.conf.bak
# Edit file securely, e.g. vi /etc/asterisk/manager_custom.conf
asterisk -rx 'manager reload'
asterisk -rx 'manager show users'
asterisk -rx 'manager show user monitor_ro'
asterisk -rx 'manager show user control_rw'
```

<div dir="rtl" align="right">

در این نمونه، `manager.conf` قبلاً به‌گونه‌ای تنظیم شده بود که AMI روی همه رابط‌ها (`0.0.0.0:5038`) گوش دهد. این انتخاب را برای محیط عملیاتی تکرار نکنید مگر با کنترل دقیق دسترسی. پس از تست اتصال، حتماً محدوده مجاز را بررسی کنید.

## ۷. SIP و ساخت Extension

در Lab هر دو درایور فعال و این پورت‌ها شنود شدند:

</div>

```bash
asterisk -rx 'module show like chan_sip'
asterisk -rx 'module show like chan_pjsip'
ss -lunp | grep -E ':(5060|5160)\b'
grep -E '^[[:space:]]*(rtpstart|rtpend)[[:space:]]*=' /etc/asterisk/rtp.conf
```

<div dir="rtl" align="right">

در پنل `Applications → Extensions` داخلی آزمایشی بسازید؛ نمونه‌های سناریو `2001` و `2002` هستند. **داخلی 2001 در پایان جلسه از نوع Chan_SIP اعلام شد.** برای هر داخلی Secret اختصاصی و قوی تعیین/بازبینی کنید، `Submit` و سپس `Apply Config` بزنید.

**نکته:** پورت SIP برای `Chan_SIP` این Lab برابر `5160/UDP` است؛ برای `PJSIP` برابر `5060/UDP`. این دو را اشتباه نگیرید. محل نمایش Secret در UI ممکن است بسته به نوع داخلی و فرم FreePBX متفاوت باشد.

## ۸. تنظیم MicroSIP و رجیستر

MicroSIP متن‌باز برای تست روی Windows انتخاب شد. نمونه تنظیم داخلی **Chan_SIP 2001**:

| فیلد | مقدار |
|---|---|
| Account Name | `2001` |
| SIP Server | `192.0.2.10:5160` |
| SIP Proxy | خالی |
| Username | `2001` |
| Domain | `192.0.2.10` |
| Login | `2001` |
| Password | Secret داخلی در FreePBX |
| Display Name | `2001` |
| Transport | `UDP` |
| Media Encryption | `Disabled` در تست پایه |

**اشتباه مشاهده‌شده:** مقدار `Domain` نباید `2001@192.0.2.10` باشد؛ فقط IP/دامنه سرور را وارد کنید.

</div>

```bash
asterisk -rx 'sip show peer 2001'
asterisk -rx 'sip show peers'
```

<div dir="rtl" align="right">

برای تشخیص رجیستر، علاوه بر وضعیت `Online` در Softphone، آدرس IP ثبت‌شده در `sip show peer 2001` را ببینید. وضعیت `OK` به تنظیم Qualify وابسته است. برای خطاهای `Offline`، تب Logs نرم‌افزار را همراه با خروجی امن CLI بررسی کنید؛ رمز را منتشر نکنید.

## ۹. سناریوی آزمون پذیرش

| شناسه | آزمون | معیار قبولی |
|---|---|---|
| T01 | نسخه‌ها | Asterisk دقیقاً `13.20.0` و FreePBX دقیقاً `14.0.16.9` |
| T02 | Boot | سرویس‌های `httpd`، `mariadb`، `freepbx` فعال؛ Asterisk پاسخگو |
| T03 | Web | Dashboard بدون Exception باز شود |
| T04 | SIP | هر دو درایور و پورت‌های تعیین‌شده فعال باشند |
| T05 | رجیستر | داخلی `2001` روی Chan_SIP ثبت شود |
| T06 | دو داخلی | `2001` و `2002` زنگ بخورند؛ صدای دوطرفه برقرار باشد |
| T07 | AMI | حساب Read-only متصل شود و تست مجوز منفی را بگذراند؛ حساب مدیریتی طبق نیاز تست شود |
| T08 | بعد از Reboot | رجیستر، وب، AMI و تماس دوباره کار کنند |

**مرز شواهد:** باز شدن Dashboard، وجود سرویس‌ها، پورت‌های SIP و نسخه Asterisk تأیید شده‌اند. **تماس صوتی موفق بین 2001 و 2002 و رجیستر نهایی MicroSIP در مکالمه تأیید نشده‌اند**؛ موارد T05–T08 باید در اجرای بعدی ثبت شوند.

## ۱۰. عیب‌یابی سریع

</div>

```bash
# Component health
asterisk -V
asterisk -rx 'core show uptime'
systemctl is-active httpd mariadb freepbx

# SIP diagnostics
asterisk -rx 'sip show peers'
asterisk -rx 'sip show peer 2001'
asterisk -rx 'pjsip show endpoints'
ss -lunp | grep -E ':(5060|5160)\b'

# AMI
asterisk -rx 'manager show users'
ss -lntp | grep ':5038'

# Logs and notices
tail -n 50 /var/log/asterisk/freepbx.log
tail -n 50 /var/log/httpd/error_log
fwconsole notification --list
```

<div dir="rtl" align="right">

- **وب 403:** لاگ `httpd/error_log` و قوانین دسترسی وب را بررسی کنید؛ در یکی از تست‌های موقت، پاسخ `403` از `AH01630: client denied by server configuration` بود.
- **وب 200 ولی Exception:** صرفاً روی کد HTTP اتکا نکنید؛ متن صفحه را بررسی کنید.
- **خطای LESS:** مالکیت و دسترسی مسیر Cache همان ماژول را بررسی کنید.
- **خطای GPG:** مالکیت و مجوز `.gnupg` و امکان استفاده فرایند وب از GPG را بررسی کنید.
- **CachePATCH:** قبل از حذف هشدار، صحت نوشتن در `/var/spool/asterisk/cache` را ثابت کنید.
- **Softphone Offline:** Secret، `Login`، `Domain`، پورت 5160/5060 و نوع داخلی را بررسی کنید.

## ۱۱. وضعیت شناخته‌شده و موارد باز

- داشبورد **دو اعلان آسیب‌پذیری ماژول** نشان می‌داد: `core 14.0.28.91` و `voicemail 14.0.6.23`؛ اعلان ارتقا به `core 14.0.29` و `voicemail 14.0.7` اشاره می‌کرد. **به‌روزرسانی بدون آزمون سازگاری و منابع آفلاین انجام نشود.**
- در تصویر پس از یکی از Rebootها، وضعیت Asterisk در Dashboard یک علامت هشدار زرد داشت؛ با وجود پاسخگو بودن CLI، علت این علامت هنوز تعیین تکلیف نشده است.
- فرآیند `pm2` در برخی لاگ‌ها پیام Deprecated تولید می‌کرد؛ عملکرد PM2 پس از Boot باید در صورت نیاز ماژول‌ها بررسی شود.
- وضعیت واقعی ثبت‌نام دو Softphone، تماس دوطرفه، RTP و تست مجوزهای AMI هنوز باید با شواهد ثبت شود.
- این سند **شامل نصب VMware Tools نیست**.

## ۱۲. شرط خروج از Lab به محیط عملیاتی

**این نصب را بدون بازطراحی مستقیماً به Production منتقل نکنید.** پیش از هر استقرار عملیاتی لازم است نسخه‌های پشتیبانی‌شده انتخاب شوند، آسیب‌پذیری‌ها برطرف شوند، اعتبارنامه‌ها تغییر کنند، TLS/HTTPS و مدیریت دسترسی برقرار شود و **SELinux و Firewall حتماً به‌درستی پیکربندی و فعال شوند**. همچنین سیاست‌های بکاپ/بازیابی، ثبت رخداد، محدودسازی AMI/SIP/RTP، مانیتورینگ و آزمون تماس کامل نیاز است.

</div>
