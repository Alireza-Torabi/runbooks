# راهنمای نصب کامل FreePBX 14.0.16.9 و Asterisk 13.20.0 روی CentOS 7.9

## ۱. مشخصات و پیش‌نیازهای نصب

**پیش‌نیازهای ماشین مجازی:** حداقل ۲ پردازنده مجازی، ۴ گیگابایت حافظه، ۳۰ گیگابایت دیسک، ISO نسخه CentOS 7.9، دسترسی root، نام میزبان قابل‌حل و مخازن نصب در دسترس یا نسخه آفلاین کامل بسته‌ها.

تمام فرمان‌ها برای **نصب تازه CentOS** نوشته شده‌اند. منابع سازگار آرشیوی CentOS 7 و PHP 5.6 باید از قبل در دسترس باشند. در نصب آفلاین، بسته‌ها و سورس‌ها را از رایانه دیگری تهیه و به مخزن محلی انتقال دهید. برای نسخه‌های دقیق از بسته‌های `latest` استفاده نکنید.

| مؤلفه | نسخه |
|---|---|
| سیستم‌عامل | CentOS 7.9 x86_64 (Minimal) |
| Asterisk | **13.20.0**، نصب از سورس |
| FreePBX | **14.0.16.9** |
| PHP | **5.6.40** (Webtatic php56w) |
| MariaDB | 5.5.x (نسخه آزمایش‌شده: 5.5.68) |
| Apache | 2.4.x (نسخه آزمایش‌شده: 2.4.6) |
| Node.js | **8.17.0** |



```bash
cat /etc/centos-release
uname -m
hostnamectl set-hostname pbx-lab.example.test
timedatectl set-timezone UTC
yum repolist
mkdir -p /usr/src /root/freepbx-packages
```

## ۲. نصب بسته‌های سیستم‌عامل

بسته‌های پایه، ابزارهای کامپایل و سرویس‌های اصلی را نصب کنید. فرمان‌های این بخش کاربر `asterisk` را می‌سازند و سرویس‌های پایگاه داده و وب را فعال می‌کنند. در صورت غیرفعال بودن ماژول‌های موردنیاز هنگام کامپایل، کتابخانه‌های `-devel` را متناسب با خطای `configure` نصب کنید.

```bash
yum install -y \
  gcc gcc-c++ make automake autoconf libtool patch bison \
  wget curl tar gzip git subversion perl vim \
  ncurses-devel openssl-devel libxml2-devel sqlite-devel \
  libuuid-devel uuid-devel jansson-devel libcurl-devel \
  newt-devel libedit-devel \
  httpd mariadb mariadb-server \
  sox cronie crontabs net-tools \
  unixODBC unixODBC-devel libtiff-devel \
  libxslt-devel gnutls-devel

id asterisk >/dev/null 2>&1 || useradd -m -d /home/asterisk -s /bin/bash asterisk
systemctl enable mariadb httpd
systemctl start mariadb httpd
systemctl is-active mariadb httpd
```

## ۳. نصب PHP 5.6.40 و وابستگی‌ها

برای PHP 5.6.40 مخزن سازگار Webtatic EL7 یا آینه محلی همان بسته‌ها را آماده کنید. بسته `webtatic-release.rpm` و تمام وابستگی‌های آن باید در نصب آفلاین موجود باشند. بسته‌های `php56w` را با PHP پیش‌فرض CentOS مخلوط نکنید. مقادیر فایل `php.ini` که در ادامه آمده‌اند فقط بخش‌هایی از تنظیمات هستند و نباید کل فایل با آنها جایگزین شود.

```bash
# RPM local
yum localinstall -y /root/freepbx-packages/webtatic-release.rpm

# RPM repository
yum install -y \
  php56w php56w-cli php56w-common \
  php56w-devel php56w-gd php56w-mbstring \
  php56w-mcrypt php56w-mysqlnd php56w-pdo \
  php56w-pear php56w-bcmath php56w-xml \
  php56w-xmlrpc php56w-soap php56w-process \
  php56w-opcache php56w-intl php56w-ldap

php -v
php -m | grep -Ei 'bcmath|gd|intl|mbstring|mcrypt|mysqli|pdo|soap|xml'
```


```ini
memory_limit = 256M
upload_max_filesize = 120M
post_max_size = 120M
date.timezone = UTC
```


```bash
pear list | grep Console_Getopt || pear install Console_Getopt
```

## ۴. نصب Node.js 8.17.0

نسخه دقیق Node.js 8.17.0 را از آرشیو از پیش تهیه‌شده یا بسته RPM محلی نصب کنید. پیش از ادامه، خروجی `node -v` باید دقیقاً `v8.17.0` باشد.

```bash
cd /usr/local
# Pre-stage the exact archive.
tar -xJf /usr/src/node-v8.17.0-linux-x64.tar.xz
ln -sfn /usr/local/node-v8.17.0-linux-x64 /usr/local/node8
ln -sfn /usr/local/node8/bin/node /usr/local/bin/node
ln -sfn /usr/local/node8/bin/npm /usr/local/bin/npm
node -v
npm -v
```

پیش از نصب ماژول‌های وب، خروجی `node -v` باید `v8.17.0` باشد.

## ۵. کامپایل و نصب Asterisk 13.20.0 از سورس

فایل رسمی `asterisk-13.20.0.tar.gz` را در `/usr/src` قرار دهید. برای ماژول PJSIP کتابخانه PJPROJECT سازگار لازم است. در `make menuselect` از فعال بودن `chan_sip`، `chan_pjsip` و `app_macro` اطمینان حاصل کنید. اگر PJSIP قابل انتخاب نیست، ابتدا PJPROJECT سازگار و قابل لینک به shared library را نصب و تنظیمات را دوباره اجرا کنید. دستور `make samples` فقط در نصب اولیه کاربرد دارد.

```bash
cd /usr/src
test -f asterisk-13.20.0.tar.gz
tar -xzf asterisk-13.20.0.tar.gz
cd /usr/src/asterisk-13.20.0

./configure --libdir=/usr/lib64
make menuselect
```


```bash
make -j2
make install
make samples
ldconfig
asterisk -V
```



```ini
[directories]
astetcdir => /etc/asterisk
astmoddir => /usr/lib64/asterisk/modules
astvarlibdir => /var/lib/asterisk
astdbdir => /var/lib/asterisk
astkeydir => /var/lib/asterisk
astdatadir => /var/lib/asterisk
astagidir => /var/lib/asterisk/agi-bin
astspooldir => /var/spool/asterisk
astrundir => /var/run/asterisk
astlogdir => /var/log/asterisk

[options]
runuser = asterisk
rungroup = asterisk
```

```bash
mkdir -p /var/run/asterisk /var/log/asterisk /var/spool/asterisk /var/lib/asterisk
chown -R asterisk:asterisk /etc/asterisk /var/run/asterisk \
  /var/log/asterisk /var/spool/asterisk /var/lib/asterisk /usr/lib64/asterisk
```

## ۶. آماده‌سازی Apache و پایگاه داده

Apache را طوری تنظیم کنید که Workerها با کاربر `asterisk` اجرا شوند. در `/etc/httpd/conf/httpd.conf` مقدارهای User و Group و دسترسی `.htaccess` را تعیین کنید. MariaDB را راه‌اندازی و با `mysql_secure_installation` پیکربندی کنید؛ نصب‌کننده FreePBX باید بتواند با دسترسی مدیریتی محلی دیتابیس‌های موردنیاز را ایجاد کند.

```bash
sed -i -E 's/^(User|Group) .*/\1 asterisk/' /etc/httpd/conf/httpd.conf
sed -i 's/AllowOverride None/AllowOverride All/g' /etc/httpd/conf/httpd.conf
httpd -t
systemctl restart httpd
ps -eo user,comm | grep httpd | head
```


```bash
systemctl enable mariadb
systemctl start mariadb
mysql_secure_installation
mysql -e 'SELECT VERSION();'
```

## ۷. نصب نسخه دقیق FreePBX 14.0.16.9

بسته نصب **نسخه دقیق 14.0.16.9** را از قبل تهیه و در `/usr/src` با نام مشخص‌شده قرار دهید. بسته `freepbx-14.0-latest.tgz` برای نصب بازتولیدپذیر مناسب نیست، چون ممکن است نسخه متفاوتی داشته باشد. پس از استخراج، وجود دو فایل `start_asterisk` و `install` را بررسی کنید، مقدار `FREEPBX_SRC` را با پوشه واقعی تطبیق دهید و نصب را اجرا کنید. سپس نسخه نصب‌شده را کنترل و حساب مدیر وب را ایجاد کنید.

```bash
cd /usr/src
test -f freepbx-14.0.16.9.tgz
mkdir -p /usr/src/freepbx-install
tar -xzf freepbx-14.0.16.9.tgz -C /usr/src/freepbx-install
find /usr/src/freepbx-install -maxdepth 3 -name start_asterisk -o -name install
```


```bash
FREEPBX_SRC=/usr/src/freepbx-install/freepbx
test -x "$FREEPBX_SRC/start_asterisk"
test -x "$FREEPBX_SRC/install"
cd "$FREEPBX_SRC"
./start_asterisk start
./install -n
fwconsole -V
```


```bash
fwconsole chown
fwconsole reload
asterisk -rx 'module show like chan_sip'
asterisk -rx 'module show like chan_pjsip'
asterisk -rx 'module show like app_macro'
```

## ۸. ساخت سرویس FreePBX و راه‌اندازی خودکار

فایل واحد `freepbx.service` را مطابق محتوای زیر در `/etc/systemd/system/` ایجاد کنید. این سرویس از نوع `oneshot` است؛ وضعیت `active (exited)` می‌تواند طبیعی باشد و سلامت Asterisk باید جدا بررسی شود.

```ini
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

```bash
systemctl daemon-reload
systemctl enable httpd mariadb freepbx
systemctl start freepbx
systemctl is-active httpd mariadb freepbx
asterisk -rx 'core show uptime'
```

## ۹. پیکربندی AMI و آزمون دسترسی

کاربران سفارشی AMI را در `/etc/asterisk/manager_custom.conf` تعریف کنید. نمونه شامل کاربر فقط‌خواندنی و کاربر با سطح Read/Write است. رمزهای متفاوت و محدوده‌های IP مجاز را مطابق مقصد تنظیم کنید. پس از `manager reload` مجوزهای هر حساب و باز بودن TCP 5038 را بررسی کنید. برای تست مستقل پاسخ Action، حالت `Events: off` در کلاینت AMI قابل استفاده است.

```ini
[monitor_ro]
secret = REPLACE_WITH_SECRET_1
deny = 0.0.0.0/0.0.0.0
permit = 198.51.100.0/255.255.255.0
read = system,call,log,verbose,agent,user,config,dtmf,reporting,cdr,dialplan
write =

[control_rw]
secret = REPLACE_WITH_SECRET_2
deny = 0.0.0.0/0.0.0.0
permit = 198.51.100.0/255.255.255.0
read = all
write = all
```

```bash
chown asterisk:asterisk /etc/asterisk/manager_custom.conf
chmod 640 /etc/asterisk/manager_custom.conf
asterisk -rx 'manager reload'
asterisk -rx 'manager show users'
asterisk -rx 'manager show user monitor_ro'
asterisk -rx 'manager show user control_rw'
ss -lntp | grep ':5038'
```

## ۱۰. ساخت داخلی SIP و آزمون Softphone

در مسیر Applications > Extensions دو داخلی از نوع Chan_SIP مانند `2001` و `2002` بسازید. برای هرکدام Secret جداگانه تعیین و تنظیمات را Apply کنید. پورت معمول Chan_SIP برابر UDP 5160 و پورت PJSIP برابر UDP 5060 است؛ مقدار واقعی را از SIP Settings بخوانید. در MicroSIP، دامنه فقط IP سرور و SIP Server شامل IP و پورت است. پس از Register، تماس دوطرفه، RTP و DTMF را بررسی کنید.

```bash
asterisk -rx 'sip show peers'
asterisk -rx 'sip show peer 2001'
asterisk -rx 'pjsip show endpoints'
ss -lunp | grep -E ':(5060|5160)\b'
asterisk -rx 'core show channels'
```

## ۱۱. بررسی لاگ‌ها و صحت عملکرد

فرمان‌های زیر وضعیت نسخه‌ها، سرویس‌ها، رابط وب، اعلان‌ها و لاگ‌ها را نشان می‌دهند. پاسخ HTTP 200 به‌تنهایی برای تأیید موفقیت وب کافی نیست؛ داشبورد باید واقعاً بدون خطا نمایش داده شود. برای مشکلات Register نوع داخلی، پورت، Login و Secret را بررسی کنید. پس از Reboot تست تماس را تکرار کنید.

```bash
asterisk -V
fwconsole -V
php -v
node -v
systemctl is-active mariadb httpd freepbx
curl -sS -L -o /dev/null -w 'HTTP=%{http_code}\n' http://127.0.0.1/admin/config.php
asterisk -rx 'core show uptime'
tail -n 50 /var/log/asterisk/freepbx.log
tail -n 50 /var/log/httpd/error_log
fwconsole notification --list
```

## ۱۲. الزامات امنیتی محیط عملیاتی

برای استقرار در محیط عملیاتی، SELinux و Firewall باید به‌درستی پیکربندی و فعال باشند؛ تنظیم این دو در دامنه این راهنمای نصب قرار ندارد.



