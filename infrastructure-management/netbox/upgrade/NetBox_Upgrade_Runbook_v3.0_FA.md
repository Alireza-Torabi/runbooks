<h1 dir="rtl" align="right">Runbook عمومی ارتقای NetBox</h1>

<p dir="rtl" align="right">
این سند یک Runbook عمومی برای ارتقای NetBox Community در نصب‌های Native Linux است. هیچ IP، hostname، رمز عبور، نام Plugin اختصاصی یا نسخهٔ ثابت مربوط به یک محیط خاص در آن وجود ندارد.
</p>

<p dir="rtl" align="right">
<strong>دامنه:</strong> نصب‌های سنتی NetBox که از Release Archive یا Git نصب شده‌اند، با systemd و PostgreSQL محلی یا Remote. برای Docker و Kubernetes باید Runbook جداگانه استفاده شود.
</p>

<h2 dir="rtl" align="right">۱. هدف</h2>

<p dir="rtl" align="right">
هدف این سند ارائهٔ یک روش تکرارپذیر برای Upgrade است. فرض شده Downtime قابل قبول است و یک روش Rollback معتبر وجود دارد. در مثال اصلی از VMware Snapshot استفاده می‌شود. اگر Snapshot در دسترس نیست، باید از روش Backup/Restore تست‌شدهٔ PostgreSQL استفاده شود.
</p>

<h2 dir="rtl" align="right">۲. قوانین مهم</h2>

<ol dir="rtl" align="right">
<li>قبل از Upgrade تمام Release Noteهای بین نسخهٔ فعلی و نسخهٔ مقصد را بررسی کنید.</li>
<li>سازگاری Python، PostgreSQL، Redis و تمام Pluginها را با نسخهٔ مقصد بررسی کنید.</li>
<li>روش نصب را تغییر ندهید؛ اگر NetBox با Release Archive نصب شده، Upgrade را با Release Archive انجام دهید و اگر Git-based است همان روش Git را ادامه دهید.</li>
<li>برای عبور از یک Major Version ابتدا به آخرین Minor Release از Major فعلی Upgrade کنید و سپس وارد Major بعدی شوید.</li>
<li>قبل از گرفتن Snapshot، Write activity مربوط به NetBox را متوقف کنید.</li>
<li>بعد از اجرای Migrationهای دیتابیس توسط <code>upgrade.sh</code>، Rollback باید شامل Database و Application با هم باشد.</li>
<li>Password، API Token، Redis Password، Private Key و <code>SECRET_KEY</code> را در Runbook یا Shell History قرار ندهید.</li>
</ol>

<p dir="rtl" align="right"><strong>مراجع رسمی:</strong></p>

- NetBox Upgrade Guide: https://netboxlabs.com/docs/netbox/installation/upgrading/
- NetBox Release Notes: https://netboxlabs.com/docs/netbox/release-notes/
- NetBox Releases: https://github.com/netbox-community/netbox/releases

---

<h2 dir="rtl" align="right">۳. متغیرهای Change</h2>

<p dir="rtl" align="right">قبل از شروع Maintenance Window مقادیر زیر را مشخص کنید:</p>

```bash
export CURRENT_VERSION='<CURRENT_VERSION>'
export TARGET_VERSION='<TARGET_VERSION>'
export NETBOX_LINK='/opt/netbox'
export NETBOX_ROOT='/opt'
export PYTHON_BIN='<SUPPORTED_PYTHON_BIN>'
```

<p dir="rtl" align="right">یک نمونه برای مسیر Python:</p>

```text
/usr/bin/python3.12
```

<p dir="rtl" align="right">
این مقدار فقط مثال است. قبل از Upgrade حتماً Compatibility Matrix رسمی نسخهٔ مقصد را بررسی کنید و نسخهٔ Python را بر اساس آن انتخاب کنید.
</p>

---

<h2 dir="rtl" align="right">۴. بررسی وضعیت فعلی</h2>

<h3 dir="rtl" align="right">۴.۱. تشخیص روش نصب</h3>

```bash
ls -ld /opt/netbox /opt/netbox/.git 2>/dev/null || true
readlink -f /opt/netbox 2>/dev/null || true
```

<ul dir="rtl" align="right">
<li>اگر <code>/opt/netbox</code> یک Symlink است و <code>/opt/netbox/.git</code> وجود ندارد، نصب از نوع Release Archive است.</li>
<li>اگر <code>/opt/netbox/.git</code> وجود دارد، نصب Git-based است.</li>
</ul>

<h3 dir="rtl" align="right">۴.۲. ثبت وضعیت Application</h3>

```bash
/opt/netbox/venv/bin/python --version
/opt/netbox/venv/bin/pip --version
/opt/netbox/venv/bin/pip check

systemctl status netbox --no-pager
systemctl status netbox-rq --no-pager
systemctl status netbox-housekeeping.timer --no-pager 2>/dev/null || true
```

<h3 dir="rtl" align="right">۴.۳. بررسی PostgreSQL و Redis</h3>

```bash
psql --version
redis-server --version
```

<p dir="rtl" align="right">برای مشاهدهٔ نسخهٔ PostgreSQL Server:</p>

```sql
SELECT version();
```

<p dir="rtl" align="right">اگر Redis دارای Authentication است از روش زیر استفاده کنید:</p>

```bash
redis-cli --askpass PING
```

<p dir="rtl" align="right">خروجی صحیح:</p>

```text
PONG
```

<h3 dir="rtl" align="right">۴.۴. بررسی سلامت NetBox قبل از Upgrade</h3>

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py check
```

<p dir="rtl" align="right">
اگر نصب فعلی از قبل System Error دارد، قبل از Upgrade مشکل را برطرف کنید.
</p>

---

<h2 dir="rtl" align="right">۵. بررسی Compatibility و Breaking Changeها</h2>

<table dir="rtl" align="right">
<tr><th>کامپوننت</th><th>موارد قابل بررسی</th></tr>
<tr><td>NetBox</td><td>مجاز بودن مسیر Upgrade مستقیم</td></tr>
<tr><td>Python</td><td>سازگاری نسخهٔ Python با نسخهٔ مقصد</td></tr>
<tr><td>PostgreSQL</td><td>حداقل نسخهٔ موردنیاز</td></tr>
<tr><td>Redis</td><td>حداقل نسخهٔ موردنیاز</td></tr>
<tr><td>Pluginها</td><td>سازگاری تک‌تک Pluginهای فعال</td></tr>
<tr><td>LDAP / SSO</td><td>سازگاری Libraryها با NetBox و Django جدید</td></tr>
<tr><td>REST API</td><td>Endpointها و Authenticationهای Deprecated/Removed</td></tr>
<tr><td>GraphQL</td><td>تغییرات Schema و Filtering</td></tr>
<tr><td>Webhook</td><td>تغییرات Payload</td></tr>
<tr><td>Custom Script/Report</td><td>تغییرات API و Model</td></tr>
</table>

<p dir="rtl" align="right">نمایش Pluginهای فعال:</p>

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py shell -c \
  'from django.conf import settings; print("\n".join(settings.PLUGINS))'
```

<p dir="rtl" align="right">بررسی Requirementهای محلی:</p>

```bash
if [ -f /opt/netbox/local_requirements.txt ]; then
  cat /opt/netbox/local_requirements.txt
fi
```

<p dir="rtl" align="right">
<strong>نکته:</strong> فایل <code>local_requirements.txt</code> بهتر است فقط شامل Packageهایی باشد که عمداً به NetBox اضافه شده‌اند؛ مانند Pluginها، LDAP Libraryها، Storage Backendها یا Dependencyهای اختصاصی سازمان.
</p>

<p dir="rtl" align="right">
Dependencyهای Core مانند Django، <code>redis</code>، <code>rq</code>، <code>django-rq</code> یا <code>gunicorn</code> را بدون الزام رسمی NetBox در این فایل Pin نکنید، چون <code>upgrade.sh</code> نسخه‌های موردنیاز Release مقصد را نصب می‌کند.
</p>

---

<h2 dir="rtl" align="right">۶. بررسی PyPI / Nexus / Repository داخلی</h2>

<p dir="rtl" align="right">
اگر Packageهای Python از Nexus، Artifactory، PyPI Mirror یا Repository داخلی دریافت می‌شوند، قبل از Upgrade بررسی کنید Python جدید نیز همان Repository را می‌بیند.
</p>

```bash
/opt/netbox/venv/bin/python -m pip config debug
/opt/netbox/venv/bin/python -m pip config list
env | grep '^PIP_' || true
```

<p dir="rtl" align="right">برای Upgrade بهتر است تنظیمات pip به‌صورت System-wide باشد:</p>

```text
/etc/pip.conf
```

<p dir="rtl" align="right">نمونهٔ عمومی:</p>

```ini
[global]
index-url = https://<PYPI_REPOSITORY_HOST>/repository/<PYPI_REPOSITORY>/simple
timeout = 60
```

<p dir="rtl" align="right">
اگر Repository از CA داخلی استفاده می‌کند، Root CA را در Trust Store سیستم‌عامل نصب کنید. تا حد امکان از <code>trusted-host</code> برای دور زدن TLS Validation استفاده نکنید.
</p>

<h3 dir="rtl" align="right">تست Repository با Python مقصد</h3>

```bash
"$PYTHON_BIN" -m venv /tmp/netbox-pip-test
/tmp/netbox-pip-test/bin/python -m pip config debug
/tmp/netbox-pip-test/bin/python -m pip index versions Django
rm -rf /tmp/netbox-pip-test
```

<p dir="rtl" align="right">برای هر Plugin نیز Version قابل نصب را بررسی کنید:</p>

```bash
python -m pip index versions <PLUGIN_PACKAGE>
```

---

<h2 dir="rtl" align="right">۷. آماده‌سازی Python سازگار</h2>

<p dir="rtl" align="right">
نسخهٔ Python پشتیبانی‌شده توسط Release مقصد را از Repository مورد تأیید سازمان یا Distribution نصب کنید.
</p>

<p dir="rtl" align="right">
بدون الزام رسمی سیستم‌عامل، Python پیش‌فرض <code>/usr/bin/python3</code> را Replace نکنید.
</p>

```bash
"$PYTHON_BIN" --version
"$PYTHON_BIN" -m venv --help >/dev/null
```

---

<h2 dir="rtl" align="right">۸. شروع Maintenance Window</h2>

<h3 dir="rtl" align="right">۸.۱. توقف Write Activity</h3>

```bash
sudo systemctl stop netbox
sudo systemctl stop netbox-rq
sudo systemctl stop netbox-housekeeping.timer 2>/dev/null || true
```

```bash
systemctl is-active netbox || true
systemctl is-active netbox-rq || true
```

<h3 dir="rtl" align="right">۸.۲. گرفتن Snapshot برای Rollback</h3>

<p dir="rtl" align="right">در صورت استفاده از VMware Snapshot، Snapshot هماهنگ از تمام Stateهای موردنیاز تهیه کنید:</p>

<ul dir="rtl" align="right">
<li>VM مربوط به NetBox Application</li>
<li>VM مربوط به PostgreSQL</li>
<li>Media یا Storage خارجی، اگر داخل Snapshot Application نیست</li>
<li>سایر Componentهای Stateful موردنیاز معماری</li>
</ul>

<p dir="rtl" align="right">الگوی پیشنهادی نام Snapshot:</p>

```text
pre-netbox-upgrade-YYYYMMDD-HHMM
```

<p dir="rtl" align="right">
بعد از اجرای Migration، Restore کردن فقط Application VM یک Rollback کامل محسوب نمی‌شود.
</p>

---

<h2 dir="rtl" align="right">۹. نصب Release مقصد</h2>

```bash
CURRENT_DIR="$(readlink -f /opt/netbox)"
echo "$CURRENT_DIR"
```

<h3 dir="rtl" align="right">روش A: Release Archive</h3>

```bash
cd /tmp
wget -O "netbox-v${TARGET_VERSION}.tar.gz" \
  "https://github.com/netbox-community/netbox/archive/v${TARGET_VERSION}.tar.gz"

sudo tar -xzf "netbox-v${TARGET_VERSION}.tar.gz" -C /opt

NEW_DIR="/opt/netbox-${TARGET_VERSION}"
sudo ln -sfn "${NEW_DIR}/" /opt/netbox
```

<p dir="rtl" align="right">کپی Configuration:</p>

```bash
sudo cp "${CURRENT_DIR}/netbox/netbox/configuration.py" \
  /opt/netbox/netbox/netbox/
```

<p dir="rtl" align="right">کپی LDAP Configuration در صورت وجود:</p>

```bash
if [ -f "${CURRENT_DIR}/netbox/netbox/ldap_config.py" ]; then
  sudo cp "${CURRENT_DIR}/netbox/netbox/ldap_config.py" \
    /opt/netbox/netbox/netbox/
fi
```

<p dir="rtl" align="right">کپی Requirementهای محلی:</p>

```bash
if [ -f "${CURRENT_DIR}/local_requirements.txt" ]; then
  sudo cp "${CURRENT_DIR}/local_requirements.txt" /opt/netbox/
fi
```

<p dir="rtl" align="right">کپی Media:</p>

```bash
if [ -d "${CURRENT_DIR}/netbox/media" ]; then
  sudo cp -pr "${CURRENT_DIR}/netbox/media/." /opt/netbox/netbox/media/
fi
```

<p dir="rtl" align="right">کپی Script و Report سفارشی:</p>

```bash
if [ -d "${CURRENT_DIR}/netbox/scripts" ]; then
  sudo cp -pr "${CURRENT_DIR}/netbox/scripts/." /opt/netbox/netbox/scripts/
fi

if [ -d "${CURRENT_DIR}/netbox/reports" ]; then
  sudo cp -pr "${CURRENT_DIR}/netbox/reports/." /opt/netbox/netbox/reports/
fi
```

<p dir="rtl" align="right">کپی Gunicorn Configuration در صورت استفاده:</p>

```bash
if [ -f "${CURRENT_DIR}/gunicorn.py" ]; then
  sudo cp "${CURRENT_DIR}/gunicorn.py" /opt/netbox/
fi
```

<h3 dir="rtl" align="right">روش B: Git</h3>

<p dir="rtl" align="right">فقط در صورتی استفاده شود که نصب فعلی Git-based است:</p>

```bash
cd /opt/netbox
sudo git fetch --tags
sudo git checkout "v${TARGET_VERSION}"
```

<p dir="rtl" align="right">
در همان Change Window روش نصب را از Release Archive به Git یا برعکس تغییر ندهید، مگر اینکه این Migration از قبل طراحی و تست شده باشد.
</p>

---

<h2 dir="rtl" align="right">۱۰. بروزرسانی Plugin Requirementها</h2>

<p dir="rtl" align="right">فایل زیر را بازبینی کنید:</p>

```text
/opt/netbox/local_requirements.txt
```

<p dir="rtl" align="right">مثال عمومی:</p>

```text
plugin-package-one==<COMPATIBLE_VERSION>
plugin-package-two==<COMPATIBLE_VERSION>
```

<p dir="rtl" align="right">Version قدیمی Pluginها را بدون بررسی Compatibility به Release جدید منتقل نکنید.</p>

---

<h2 dir="rtl" align="right">۱۱. اجرای Upgrade</h2>

```bash
cd /opt/netbox
sudo PYTHON="$PYTHON_BIN" ./upgrade.sh
```

<p dir="rtl" align="right">
اسکریپت رسمی Upgrade، Virtual Environment را بازسازی می‌کند، Dependencyهای NetBox و <code>local_requirements.txt</code> را نصب می‌کند، Migrationهای دیتابیس را اجرا می‌کند، Static Fileها را جمع‌آوری می‌کند و عملیات Housekeeping مربوط به Upgrade را انجام می‌دهد.
</p>

<h3 dir="rtl" align="right">شرط توقف</h3>

<ol dir="rtl" align="right">
<li>اگر <code>upgrade.sh</code> Fail شد سرویس‌های NetBox را Start نکنید.</li>
<li>خروجی کامل خطا را ذخیره کنید.</li>
<li>فقط خطای Dependency یا Configuration کاملاً مشخص را اصلاح کنید؛ در غیر این صورت Rollback انجام دهید.</li>
<li>بدون تغییر عمدی Schema، Migration جدید Django نسازید.</li>
</ol>

---

<h2 dir="rtl" align="right">۱۲. Validation قبل از Start سرویس‌ها</h2>

<h3 dir="rtl" align="right">۱۲.۱. بررسی Dependencyها</h3>

```bash
/opt/netbox/venv/bin/python --version
/opt/netbox/venv/bin/pip check
```

<p dir="rtl" align="right">خروجی مطلوب:</p>

```text
No broken requirements found.
```

<h3 dir="rtl" align="right">۱۲.۲. System Check</h3>

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py check
```

<p dir="rtl" align="right">خروجی مطلوب:</p>

```text
System check identified no issues (0 silenced).
```

<p dir="rtl" align="right">
پیغام‌های Deprecation یا <code>FutureWarning</code> لزوماً Failure نیستند، اما باید بررسی شوند و Configurationهای Deprecated در زمان مناسب حذف شوند.
</p>

<h3 dir="rtl" align="right">۱۲.۳. بررسی Migrationهای باقی‌مانده</h3>

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py showmigrations --plan \
  | grep '^\[ \]' \
  || echo 'No pending migrations detected.'
```

<h3 dir="rtl" align="right">۱۲.۴. بررسی Packageهای Plugin</h3>

```bash
/opt/netbox/venv/bin/pip list | sort
```

```bash
/opt/netbox/venv/bin/pip show <PLUGIN_PACKAGE>
```

---

<h2 dir="rtl" align="right">۱۳. Start سرویس‌ها</h2>

```bash
sudo systemctl restart netbox
sudo systemctl restart netbox-rq
sudo systemctl start netbox-housekeeping.timer 2>/dev/null || true
```

<p dir="rtl" align="right">بررسی وضعیت:</p>

```bash
systemctl --no-pager --full status netbox
systemctl --no-pager --full status netbox-rq
systemctl --no-pager --full status netbox-housekeeping.timer 2>/dev/null || true
```

<p dir="rtl" align="right">بررسی Logها:</p>

```bash
journalctl -u netbox -n 100 --no-pager
journalctl -u netbox-rq -n 100 --no-pager
```

---

<h2 dir="rtl" align="right">۱۴. Functional Validation</h2>

<ul dir="rtl" align="right">
<li>Web UI بدون خطا Load شود.</li>
<li>Login کار کند.</li>
<li>Version مقصد نمایش داده شود.</li>
<li>Search کار کند.</li>
<li>Create/Update/Delete روی یک Object تستی انجام شود.</li>
<li>Background Jobها پردازش شوند.</li>
<li>RQ Workerها سالم باشند.</li>
<li>Redis Connectivity سالم باشد.</li>
<li>REST API Authentication تست شود.</li>
<li>GraphQLهای مورد استفاده Automation تست شوند.</li>
<li>Webhookها Payload مورد انتظار را ارسال کنند.</li>
<li>LDAP/SSO در صورت استفاده تست شود.</li>
<li>تمام Pluginها Load شوند و صفحات اصلی آنها کار کند.</li>
<li>Custom Script و Reportها اجرا شوند.</li>
<li>Media و Imageها Load شوند.</li>
<li>Reverse Proxy و TLS سالم باشند.</li>
</ul>

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py check

/opt/netbox/venv/bin/pip check
```

<p dir="rtl" align="right">اگر Redis دارای Authentication است:</p>

```bash
redis-cli --askpass PING
```

---

<h2 dir="rtl" align="right">۱۵. Rollback با VMware Snapshot</h2>

<h3 dir="rtl" align="right">۱۵.۱. توقف NetBox</h3>

```bash
sudo systemctl stop netbox
sudo systemctl stop netbox-rq
sudo systemctl stop netbox-housekeeping.timer 2>/dev/null || true
```

<h3 dir="rtl" align="right">۱۵.۲. Revert کردن Snapshotهای هماهنگ</h3>

<p dir="rtl" align="right">تمام Snapshotهایی را که یک نقطهٔ زمانی Pre-upgrade مشترک را نشان می‌دهند Restore کنید:</p>

<ol dir="rtl" align="right">
<li>PostgreSQL VM</li>
<li>NetBox Application VM</li>
<li>Storage یا Componentهای Stateful دیگری که در Change حضور داشته‌اند</li>
</ol>

<p dir="rtl" align="right">
ترتیب دقیق Restore به طراحی Virtualization و Storage بستگی دارد، اما تا زمانی که تمام Stateهای لازم Restore نشده‌اند، سرویس‌ها را Start نکنید.
</p>

<h3 dir="rtl" align="right">۱۵.۳. Start محیط Restore‌شده</h3>

```bash
sudo systemctl start netbox
sudo systemctl start netbox-rq
sudo systemctl start netbox-housekeeping.timer 2>/dev/null || true
```

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py check
```

<p dir="rtl" align="right">
UI، Database Data، Workerها، Pluginها و Integrationها را بررسی کنید تا مطمئن شوید محیط به وضعیت قبل از Upgrade برگشته است.
</p>

<p dir="rtl" align="right">
<strong>هشدار مهم:</strong> بعد از اجرای Migrationهای دیتابیس توسط <code>upgrade.sh</code>، فقط تغییر Symlink به نسخهٔ قبلی Rollback کامل نیست. Database نیز باید به وضعیت سازگار قبل از Upgrade بازگردد.
</p>

---

<h2 dir="rtl" align="right">۱۶. Checklist پایان Change</h2>

<div dir="rtl" align="right">

- [ ] Release Noteها بررسی شدند.
- [ ] Upgrade Path تأیید شد.
- [ ] Python Compatibility تأیید شد.
- [ ] PostgreSQL Compatibility تأیید شد.
- [ ] Redis Compatibility تأیید شد.
- [ ] Plugin Compatibility تأیید شد.
- [ ] PyPI/Nexus با Python مقصد تست شد.
- [ ] قبل از Snapshot سرویس‌های NetBox/RQ متوقف شدند.
- [ ] Snapshotهای هماهنگ Rollback تهیه شدند.
- [ ] Release مقصد با همان روش نصب قبلی نصب شد.
- [ ] `local_requirements.txt` بازبینی و اصلاح شد.
- [ ] `upgrade.sh` با موفقیت تمام شد.
- [ ] `pip check` موفق بود.
- [ ] `manage.py check` موفق بود.
- [ ] Migration غیرمنتظره‌ای باقی نمانده است.
- [ ] سرویس NetBox سالم است.
- [ ] RQ Workerها سالم هستند.
- [ ] UI و Authentication تست شدند.
- [ ] REST API/GraphQL/Webhookها در صورت استفاده تست شدند.
- [ ] Pluginها تست شدند.
- [ ] Custom Script/Reportها تست شدند.
- [ ] Log و Monitoring بررسی شدند.
- [ ] نگهداری یا حذف Snapshotها طبق Change Policy انجام شد.

</div>

---

<h2 dir="rtl" align="right">۱۷. نکات محیط Production</h2>

<p dir="rtl" align="right">
برای سرویس‌های Business-critical بهتر است علاوه بر Snapshot، روش Backup/Restore دیتابیس نیز تست شده باشد. Consistency یک Snapshot به معماری Virtualization، Storage، PostgreSQL و وضعیت Application در زمان Snapshot وابسته است.
</p>

<p dir="rtl" align="right">
برای NetBoxهای Clustered، Containerized، Kubernetes، High Availability یا Horizontal Scale این Runbook را متناسب با معماری تغییر دهید و آن را بدون تطبیق مستقیم اجرا نکنید.
</p>

---

<h2 dir="rtl" align="right">۱۸. منابع</h2>

- NetBox Upgrade Guide: https://netboxlabs.com/docs/netbox/installation/upgrading/
- NetBox Release Notes: https://netboxlabs.com/docs/netbox/release-notes/
- NetBox GitHub Releases: https://github.com/netbox-community/netbox/releases
- NetBox Installation Guide: https://netboxlabs.com/docs/netbox/installation/

---

<p dir="rtl" align="right">
<strong>نسخهٔ سند:</strong> 3.0<br>
<strong>زبان:</strong> فارسی<br>
<strong>فرمت:</strong> GitHub Markdown با RTL-safe HTML<br>
<strong>مقادیر اختصاصی محیط:</strong> ندارد
</p>
