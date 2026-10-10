<div dir="rtl" align="right">

# راهنمای عملیاتی (Runbook) نصب و راه‌اندازی آزمایشگاهی FreePBX 14 و Asterisk 13

**نسخه سند:** 1.2  
**دامنه:** ماشین مجازی آزمایشگاهی آفلاین  
## ۱. هدف و محدوده

راه‌اندازی یک PBX آزمایشگاهی با نسخه‌های **دقیق** زیر، تأیید عملکرد وب، AMI و SIP و تست رجیستر و تماس داخلی.

| مؤلفه | نسخه / مقدار |
|---|---|
| سیستم‌عامل | CentOS 7.9، کرنل `3.10.0-1160.el7.x86_64` |
| نام میزبان | `pbx-lab.example.test` |
| آدرس IP نمونه | `192.0.2.10/24` |
| Asterisk | **`13.20.0`**، کامپایل‌شده از سورس |
| FreePBX | **`14.0.16.9`** |
| Apache | `httpd 2.4.6` |
| PHP | `5.6.40` |
| MariaDB | `5.5.68` |
| Node.js | `8.17.0` |
| SIP | `chan_pjsip` پورت UDP `5060`؛ `chan_sip` پورت UDP `5160` |
| AMI | TCP `5038` |

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

**مسیرهای اصلی:**

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

## ۳. نصب پایه

ترتیب مراحل نصب و آماده‌سازی:

1. CentOS 7.9 و وابستگی‌های توسعه را از منابع **آفلاینِ معتبر** نصب و نسخه‌ها را تثبیت کنید.
2. Asterisk **دقیقاً `13.20.0`** را از سورس همان نسخه build/install کنید؛ هنگام انتخاب ماژول‌ها `chan_sip`، `chan_pjsip` و `app_macro` را فعال کنید.
3. MariaDB، Apache و PHP 5.6 را نصب و فعال کنید؛ پایگاه داده و حساب‌های اختصاصی FreePBX را مطابق بسته نصب تنظیم کنید.
4. FreePBX **دقیقاً `14.0.16.9`** را با روش رسمی سازگار با این نسخه نصب کنید. فرایندهای وب را با کاربر `asterisk` اجرا کنید.
5. `memory_limit` در `/etc/php.ini` به `256M` و timezone به `Asia/Tehran` تنظیم کنید.
6. پس از نصب، `fwconsole`، پنل مدیریت و فایل‌های پیکربندی Asterisk را بررسی کنید.

```bash
asterisk -V
asterisk -rx 'module show like chan_sip'
asterisk -rx 'module show like chan_pjsip'
asterisk -rx 'module show like app_macro'
php -r 'echo ini_get("memory_limit"), PHP_EOL, date_default_timezone_get(), PHP_EOL;'
```

## ۴. سرویس FreePBX در systemd

سرویس `freepbx.service` را به‌شکل `oneshot` تعریف کنید:

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

پس از ایجاد فایل سرویس:

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

`active (exited)` برای سرویس `oneshot` طبیعی است؛ فعال بودن پردازش Asterisk را جداگانه بررسی کنید.

## ۵. دسترسی وب و بررسی اولیه

آدرس پنل (با جایگزینی آدرس سرور):

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

کد `HTTP=200` به‌تنهایی به معنای بارگذاری صحیح Dashboard نیست؛ وضعیت واقعی صفحه و سرویس‌های Asterisk، MySQL و Web Server بررسی می‌شود.

### هشدار Cache

اگر هشدار `Cache path is not writable!` ظاهر شد، مالکیت و دسترسی نوشتن مسیر `/var/spool/asterisk/cache` را بررسی کنید. پس از رفع علت، در صورت باقی‌ماندن اعلان می‌توان آن را حذف کرد:

</div>

```bash
fwconsole notification --list
fwconsole notification --delete framework CACHEPATCH
```

<div dir="rtl" align="right">

حذف اعلان، جایگزین رفع مشکل دسترسی نیست. پوشه‌های `assets/less/cache` ماژول‌های FreePBX نیز باید قابل‌نوشتن برای فرایند وب باشند. در صورت مشاهده Exception مربوط به LESS، مسیر دقیق ذکرشده را بررسی کنید. ابزار عیب‌یابی:

</div>

```bash
ls -ld /var/spool/asterisk/cache
ls -ld /var/www/html/admin/modules/dashboard/assets/less/cache
namei -l /var/spool/asterisk/cache
fwconsole notification --list
```

<div dir="rtl" align="right">

### GPG و امضاهای ماژول

اگر بررسی امضای ماژول‌ها با خطای دسترسی به `/home/asterisk/.gnupg` مواجه شد، مالکیت `asterisk:asterisk` و مجوزهای زیر را بررسی و تنظیم کنید:

</div>

```bash
chmod 700 /home/asterisk/.gnupg
find /home/asterisk/.gnupg -type d -exec chmod 700 {} \;
find /home/asterisk/.gnupg -type f -exec chmod 600 {} \;
```

<div dir="rtl" align="right">

فایل `secring.gpg` شامل اطلاعات حساس کلیدهاست.

## ۶. تنظیم AMI

AMI به‌صورت پیش‌فرض از TCP `5038` استفاده می‌کند. برای بررسی وضعیت:

</div>

```bash
ss -lntp | grep ':5038'
asterisk -rx 'manager show users'
```

<div dir="rtl" align="right">

برای تعریف کاربران، از فایل **`/etc/asterisk/manager_custom.conf`** استفاده کنید؛ تغییر مستقیم فایل‌های تولیدشده FreePBX ممکن است با Apply Config بازنویسی شود. پیش از ویرایش فایل، پشتیبان تهیه کنید. نمونه دو پروفایل:

- `monitor_ro`: صرفاً رویدادها/دسترسی‌های خواندنی AMI، بدون اختیار write.
- `control_rw`: دسترسی خواندن و نوشتن AMI.

محدوده‌های مجاز AMI در فایل پیکربندی هر حساب تعریف می‌شوند.

</div>

```ini
; Add to /etc/asterisk/manager_custom.conf
; Account credentials

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

مجوزهای AMI شامل دو دسته `read` و `write` هستند. صحت محدودیت حساب فقط‌خواندنی با تست Actionهای مجاز و غیرمجاز ارزیابی می‌شود.

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

وضعیت Listen شدن AMI و محدوده اتصال کاربران با دستورات بررسی AMI قابل مشاهده است.

## ۷. SIP و ساخت Extension

ماژول‌های SIP و پورت‌های شنود را بررسی کنید:

</div>

```bash
asterisk -rx 'module show like chan_sip'
asterisk -rx 'module show like chan_pjsip'
ss -lunp | grep -E ':(5060|5160)\b'
grep -E '^[[:space:]]*(rtpstart|rtpend)[[:space:]]*=' /etc/asterisk/rtp.conf
```

<div dir="rtl" align="right">

در پنل `Applications → Extensions` داخلی آزمایشی بسازید؛ نمونه‌های سناریو `2001` و `2002` هستند. برای هر داخلی Secret اختصاصی و قوی تعیین/بازبینی کنید، `Submit` و سپس `Apply Config` بزنید.

در این پیکربندی، پورت `Chan_SIP` برابر `5160/UDP` و پورت `PJSIP` برابر `5060/UDP` است. این دو را اشتباه نگیرید. محل نمایش Secret در UI ممکن است بسته به نوع داخلی و فرم FreePBX متفاوت باشد.

## ۸. تنظیم MicroSIP و رجیستر

برای رجیستر یک داخلی **Chan_SIP** در MicroSIP روی Windows، از تنظیمات نمونه زیر استفاده کنید:

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

`Domain` باید فقط IP یا دامنه سرور SIP باشد، نه شناسه کامل داخلی مانند `2001@192.0.2.10`.

</div>

```bash
asterisk -rx 'sip show peer 2001'
asterisk -rx 'sip show peers'
```

<div dir="rtl" align="right">

برای تشخیص رجیستر، علاوه بر وضعیت `Online` در Softphone، آدرس IP ثبت‌شده در `sip show peer 2001` را ببینید. وضعیت `OK` به تنظیم Qualify وابسته است. اگر وضعیت `Offline` بود، گزارش نرم‌افزار و خروجی Asterisk CLI را بررسی کنید.

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

نتیجه هر آزمون را در محیط مقصد ثبت کنید؛ تکمیل جدول به معنی اجرای موفق آزمون‌ها نیست.

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

- **وب 403:** قوانین دسترسی Apache و فایل `/var/log/httpd/error_log` را بررسی کنید.
- **وب 200 ولی Exception:** صرفاً روی کد HTTP اتکا نکنید؛ متن صفحه را بررسی کنید.
- **خطای LESS:** مالکیت و دسترسی مسیر Cache همان ماژول را بررسی کنید.
- **خطای GPG:** مالکیت و مجوز `.gnupg` و امکان استفاده فرایند وب از GPG را بررسی کنید.
- **CachePATCH:** قبل از حذف هشدار، صحت نوشتن در `/var/spool/asterisk/cache` را ثابت کنید.
- **Softphone Offline:** Secret، `Login`، `Domain`، پورت 5160/5060 و نوع داخلی را بررسی کنید.

## ۱۱. بررسی نهایی

- وضعیت اعلان‌های FreePBX را با `fwconsole notification --list` بررسی کنید.
- پس از هر راه‌اندازی مجدد، وضعیت سرویس‌ها، رجیستر داخلی‌ها، اتصال AMI و صدای دوطرفه را آزمایش کنید.
- در صورت وجود ماژول‌های جانبی، سلامت سرویس‌های وابسته به آنها را جداگانه کنترل کنید.

</div>
