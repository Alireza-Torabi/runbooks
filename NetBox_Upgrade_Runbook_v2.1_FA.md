# Runbook ارتقای مستقیم NetBox در محیط Production

**نسخه سند:** 2.1  
**تاریخ:** 2026-08-08  
**نوع تغییر:** ارتقای مستقیم Production با Downtime برنامه‌ریزی‌شده  
**روش Rollback:** بازگردانی Snapshotهای VMware  
**Backup دیتابیس:** طبق تصمیم Change Plan انجام نمی‌شود  
**محیط آزمایشی:** طبق تصمیم Change Plan ایجاد نمی‌شود  

---

## 1. هدف و محدوده

این Runbook برای ارتقای مستقیم محیط زیر تهیه شده است:

| مؤلفه | وضعیت فعلی | وضعیت هدف |
|---|---:|---:|
| NetBox Community | 4.4.0 | 4.6.7 |
| Python مورد استفاده NetBox | 3.10.12 | 3.12.x |
| PostgreSQL | 15.15 | بدون تغییر |
| Redis Server | 6.0.16 | بدون تغییر |
| `netbox-contract` | 2.4.2 | 2.4.6 |
| `netbox-floorplan-plugin` | 0.8.0 | 0.9.2 |
| سیستم‌عامل | Ubuntu 22.04.5 LTS | بدون تغییر |

سرورها:

| نقش | آدرس |
|---|---|
| NetBox Application Server | `192.168.169.170` |
| PostgreSQL Database Server | `192.168.169.171` |

ساختار فعلی نصب:

```text
/opt/netbox -> /opt/netbox-4.4.0/
```

ساختار نهایی:

```text
/opt/netbox-4.4.0/
/opt/netbox-4.6.7/
/opt/netbox -> /opt/netbox-4.6.7/
```

---

## 2. قانون حیاتی Rollback

اسکریپت `upgrade.sh` روی دیتابیس PostgreSQL، Migration اجرا می‌کند.

بنابراین اگر بعد از شروع `upgrade.sh` نیاز به Rollback داشتید، باید **هر دو VM** را به Snapshot قبل از ارتقا برگردانید:

1. VM سرور NetBox Application
2. VM سرور PostgreSQL

بعد از اجرای Migration، انجام یکی از کارهای زیر به‌تنهایی Rollback معتبر محسوب نمی‌شود:

```text
برگرداندن فقط Snapshot سرور Application
تغییر دادن Symlink به /opt/netbox-4.4.0
نصب مجدد نسخه‌های قدیمی Python Packageها
```

Snapshot هر دو VM را تا زمان تأیید کامل UI، API، RQ Worker و هر دو Plugin نگه دارید.

---

## 3. علائم اجرای دستورات

- `[APP]` یعنی دستور روی `192.168.169.170`
- `[DB]` یعنی دستور روی `192.168.169.171`
- `[VMWARE]` یعنی عملیات در vCenter یا ESXi

تمام دستورات لینوکسی با کاربری اجرا می‌شوند که دسترسی `sudo` دارد.

---

# فاز A — بررسی‌های قبل از تغییر

## 4. بررسی Snapshotها

### [VMWARE]

وجود Snapshot قبل از ارتقا را برای هر دو VM تأیید کنید:

```text
Netbox-APP.mvmco.ir
Netbox-DB-pgsql.mvmco.ir
```

هر دو Snapshot باید قبل از موارد زیر گرفته شده باشند:

```text
نصب Python 3.12
تغییر فایل‌های NetBox
تغییر Symlink
اجرای Migration
```

نام دقیق Snapshotها را در Change Record ثبت کنید.

---

## 5. بررسی سلامت PostgreSQL

### [DB]

```bash
sudo -u postgres pg_isready \
  -h 127.0.0.1 \
  -p 5432

sudo -u postgres psql \
  -X \
  -d netbox \
  -c "
SELECT
    current_database() AS database,
    current_user AS connected_as,
    current_setting('server_version') AS server_version;
"
```

خروجی مورد انتظار:

```text
127.0.0.1:5432 - accepting connections
database = netbox
server_version = 15.15
```

روی سرور PostgreSQL هیچ Upgrade یا تغییر Package انجام نمی‌شود.

---

## 6. بررسی سلامت NetBox فعلی

### [APP]

```bash
readlink -f /opt/netbox
```

خروجی مورد انتظار:

```text
/opt/netbox-4.4.0
```

وضعیت سرویس‌ها:

```bash
sudo systemctl is-active \
  netbox \
  netbox-rq \
  netbox-housekeeping.timer \
  redis-server \
  nginx
```

تمام سرویس‌ها باید `active` باشند.

بررسی Redis:

```bash
redis-cli ping
```

خروجی:

```text
PONG
```

بررسی Django:

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py check
```

خروجی مورد انتظار:

```text
System check identified no issues
```

بررسی Pluginهای فعال بدون نمایش Secret:

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py shell -c \
  "from django.conf import settings; print(settings.PLUGINS)"
```

خروجی مورد انتظار:

```python
['netbox_contract', 'netbox_floorplan']
```

اگر نسخه فعلی NetBox قبل از شروع Change سالم نیست، Upgrade را آغاز نکنید.

---

# فاز B — نصب Python 3.12

## 7. نصب Side-by-Side روی Ubuntu 22.04

### [APP]

Python پیش‌فرض Ubuntu 22.04 را تغییر ندهید.

به‌طور مشخص، هیچ‌کدام از دستورات زیر نباید اجرا شوند:

```text
update-alternatives برای python یا python3
حذف python3.10
تغییر دستی /usr/bin/python3
```

Python 3.12 از PPA مربوط به Deadsnakes نصب می‌شود و فقط برای Virtual Environment جدید NetBox استفاده خواهد شد.

```bash
sudo apt update

sudo apt install -y \
  software-properties-common \
  ca-certificates \
  wget \
  build-essential \
  libxml2-dev \
  libxslt1-dev \
  libffi-dev \
  libpq-dev \
  libssl-dev \
  zlib1g-dev

sudo add-apt-repository -y \
  ppa:deadsnakes/ppa

sudo apt update

sudo apt install -y \
  python3.12 \
  python3.12-venv \
  python3.12-dev
```

بررسی:

```bash
/usr/bin/python3.12 --version
/usr/bin/python3 --version
```

خروجی صحیح:

```text
Python 3.12.x
Python 3.10.12
```

این وضعیت کاملاً درست است:

```text
Python سیستم‌عامل = 3.10
Python محیط NetBox جدید = 3.12
```

---

# فاز C — آماده‌سازی نسخه جدید قبل از Downtime

## 8. دانلود NetBox 4.6.7

### [APP]

```bash
cd /tmp

sudo rm -rf \
  /opt/netbox-4.6.7 \
  /tmp/netbox-v4.6.7.tar.gz

wget \
  -O /tmp/netbox-v4.6.7.tar.gz \
  https://github.com/netbox-community/netbox/archive/refs/tags/v4.6.7.tar.gz
```

بررسی سالم بودن Archive:

```bash
tar -tzf \
  /tmp/netbox-v4.6.7.tar.gz \
  >/dev/null
```

Extract:

```bash
sudo tar -xzf \
  /tmp/netbox-v4.6.7.tar.gz \
  -C /opt
```

بررسی نسخه:

```bash
test -f \
  /opt/netbox-4.6.7/netbox/release.yaml

grep '^version:' \
  /opt/netbox-4.6.7/netbox/release.yaml
```

خروجی:

```text
version: "4.6.7"
```

در این مرحله Symlink فعلی هنوز نباید تغییر کرده باشد:

```bash
readlink -f /opt/netbox
```

خروجی:

```text
/opt/netbox-4.4.0
```

---

## 9. انتقال Configuration و فایل‌های محلی

### [APP]

کپی فایل اصلی Configuration:

```bash
sudo cp -p \
  /opt/netbox-4.4.0/netbox/netbox/configuration.py \
  /opt/netbox-4.6.7/netbox/netbox/configuration.py
```

اگر LDAP Configuration وجود دارد:

```bash
if sudo test -f \
  /opt/netbox-4.4.0/netbox/netbox/ldap_config.py
then
  sudo cp -p \
    /opt/netbox-4.4.0/netbox/netbox/ldap_config.py \
    /opt/netbox-4.6.7/netbox/netbox/ldap_config.py
fi
```

کپی Gunicorn Configuration:

```bash
sudo cp -p \
  /opt/netbox-4.4.0/gunicorn.py \
  /opt/netbox-4.6.7/gunicorn.py
```

ایجاد مسیرهای موردنیاز:

```bash
sudo mkdir -p \
  /opt/netbox-4.6.7/netbox/media \
  /opt/netbox-4.6.7/netbox/scripts \
  /opt/netbox-4.6.7/netbox/reports
```

کپی Media:

```bash
sudo cp -a \
  /opt/netbox-4.4.0/netbox/media/. \
  /opt/netbox-4.6.7/netbox/media/
```

کپی Custom Scripts:

```bash
sudo cp -a \
  /opt/netbox-4.4.0/netbox/scripts/. \
  /opt/netbox-4.6.7/netbox/scripts/
```

کپی Reports:

```bash
sudo cp -a \
  /opt/netbox-4.4.0/netbox/reports/. \
  /opt/netbox-4.6.7/netbox/reports/
```

اصلاح مالکیت:

```bash
sudo chown -R \
  netbox:netbox \
  /opt/netbox-4.6.7/netbox/media \
  /opt/netbox-4.6.7/netbox/scripts \
  /opt/netbox-4.6.7/netbox/reports
```

---

## 10. ساخت `local_requirements.txt` تمیز

### [APP]

فایل قدیمی را کورکورانه کپی نکنید؛ به‌خصوص اگر قبلاً خروجی `pip freeze` داخل آن قرار گرفته باشد.

بر اساس Inventory فعلی، فقط دو Plugin سفارشی لازم است:

```bash
sudo tee \
  /opt/netbox-4.6.7/local_requirements.txt \
  >/dev/null <<'EOF'
netbox-contract==2.4.6
netbox-floorplan-plugin==0.9.2
EOF
```

بررسی:

```bash
sudo cat \
  /opt/netbox-4.6.7/local_requirements.txt
```

خروجی دقیق باید این باشد:

```text
netbox-contract==2.4.6
netbox-floorplan-plugin==0.9.2
```

Packageهای Core مانند موارد زیر نباید در `local_requirements.txt` Pin شوند:

```text
Django
django-rq
rq
redis
gunicorn
django-redis
django-storages
```

نسخه صحیح این Packageها توسط خود NetBox 4.6.7 نصب خواهد شد.

---

## 11. افزودن `API_TOKEN_PEPPERS`

### [APP]

NetBox 4.5 توکن‌های API نسخه دوم را معرفی کرده است. Configuration کپی‌شده از NetBox 4.4 ممکن است `API_TOKEN_PEPPERS` نداشته باشد.

دستور زیر فقط در صورت نبودن این Parameter، یک Pepper تصادفی و امن ایجاد می‌کند. مقدار Secret چاپ نمی‌شود.

```bash
sudo /usr/bin/python3.12 - <<'PY'
from pathlib import Path
import re
import secrets

path = Path(
    "/opt/netbox-4.6.7/netbox/netbox/configuration.py"
)

text = path.read_text()

if not re.search(
    r"(?m)^\s*API_TOKEN_PEPPERS\s*=",
    text,
):
    pepper = secrets.token_urlsafe(48)

    with path.open("a") as handle:
        handle.write("\n")
        handle.write(
            "# Added for NetBox v4.5+ API token v2 support\n"
        )
        handle.write(
            f"API_TOKEN_PEPPERS = {{1: {pepper!r}}}\n"
        )

    print("API_TOKEN_PEPPERS added.")
else:
    print(
        "API_TOKEN_PEPPERS already exists; no change made."
    )
PY
```

بدون نمایش مقدار Pepper بررسی کنید که Parameter تعریف شده است:

```bash
sudo /usr/bin/python3.12 - <<'PY'
from pathlib import Path
import re

text = Path(
    "/opt/netbox-4.6.7/netbox/netbox/configuration.py"
).read_text()

found = bool(
    re.search(
        r"(?m)^\s*API_TOKEN_PEPPERS\s*=",
        text,
    )
)

print(
    f"API_TOKEN_PEPPERS defined: {found}"
)
PY
```

خروجی:

```text
API_TOKEN_PEPPERS defined: True
```

بررسی Syntax فایل:

```bash
sudo /usr/bin/python3.12 \
  -m py_compile \
  /opt/netbox-4.6.7/netbox/netbox/configuration.py
```

نبود خروجی یعنی Syntax فایل معتبر است.

---

# فاز D — شروع Downtime

## 12. توقف سرویس‌هایی که روی دیتابیس Write انجام می‌دهند

### [APP]

ابتدا Web Application، سپس Worker و Housekeeping را متوقف کنید:

```bash
sudo systemctl stop netbox

sudo systemctl stop netbox-rq

sudo systemctl stop \
  netbox-housekeeping.timer

sudo systemctl stop \
  netbox-housekeeping.service \
  2>/dev/null || true
```

بررسی:

```bash
sudo systemctl is-active \
  netbox \
  netbox-rq \
  netbox-housekeeping.timer
```

خروجی مورد انتظار:

```text
inactive
inactive
inactive
```

سرویس‌های زیر باید روشن بمانند:

```text
PostgreSQL
Redis
Nginx
```

Redis را متوقف نکنید، زیرا `upgrade.sh` در انتهای کار Reindex را در Queue قرار می‌دهد.

---

# فاز E — Cutover و اجرای Upgrade

## 13. تغییر Symlink

### [APP]

```bash
sudo ln -sfn \
  /opt/netbox-4.6.7/ \
  /opt/netbox
```

بررسی:

```bash
readlink -f /opt/netbox
```

خروجی:

```text
/opt/netbox-4.6.7
```

---

## 14. اجرای `upgrade.sh`

### [APP]

دستور زیر Upgrade را با Python 3.12 اجرا می‌کند و خروجی کامل را در Log ذخیره می‌کند:

```bash
sudo bash \
  -o pipefail \
  -c '
cd /opt/netbox &&
PYTHON=/usr/bin/python3.12 ./upgrade.sh 2>&1 |
tee /var/log/netbox-upgrade-4.6.7.log
'
```

اسکریپت رسمی عملیات زیر را انجام می‌دهد:

```text
ساخت Virtual Environment با Python 3.12
نصب Dependencyهای NetBox 4.6.7
نصب netbox-contract 2.4.6
نصب netbox-floorplan-plugin 0.9.2
اجرای Migrationهای NetBox
اجرای Migrationهای Pluginها
بررسی Cable Pathها
ساخت Documentation محلی
اجرای collectstatic
حذف Content Typeهای منسوخ
Queue کردن Search Reindex
حذف Sessionهای منقضی
```

خروجی موفق باید با عبارت زیر تمام شود:

```text
Upgrade complete!
```

### Stop Condition

اگر `upgrade.sh` Error داد:

```text
سرویس netbox را Start نکنید
سرویس netbox-rq را Start نکنید
دستور makemigrations اجرا نکنید
Dependencyها را دستی Downgrade نکنید
فقط طبق بخش Rollback هر دو VM را Revert کنید
```

مشاهده Log:

```bash
sudo less \
  /var/log/netbox-upgrade-4.6.7.log
```

---

# فاز F — بررسی قبل از Start سرویس‌ها

## 15. بررسی نسخه‌ها و Dependencyها

### [APP]

نسخه Python داخل Virtual Environment:

```bash
/opt/netbox/venv/bin/python \
  --version
```

نسخه NetBox:

```bash
grep '^version:' \
  /opt/netbox/netbox/release.yaml
```

بررسی Dependency Conflict:

```bash
/opt/netbox/venv/bin/pip \
  check
```

خروجی مورد انتظار:

```text
Python 3.12.x
version: "4.6.7"
No broken requirements found.
```

نمایش نسخه Packageهای مهم:

```bash
/opt/netbox/venv/bin/pip show \
  Django \
  gunicorn \
  redis \
  rq \
  django-rq \
  netbox-contract \
  netbox-floorplan-plugin \
  | grep -E '^(Name|Version):'
```

نسخه‌های کلیدی مورد انتظار:

```text
Django 6.0.7
gunicorn 26.0.0
redis 7.4.1
rq 2.10.0
django-rq 4.1.1
netbox-contract 2.4.6
netbox-floorplan-plugin 0.9.2
```

توجه:

```text
redis 7.4.1 = Python Client داخل Virtual Environment
Redis Server 6.0.16 = سرویس Redis سیستم‌عامل
```

این دو مورد مستقل از هم هستند.

---

## 16. بررسی Django

### [APP]

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py check
```

خروجی:

```text
System check identified no issues
```

---

## 17. بررسی Migrationها

### [APP]

```bash
MIGRATION_PLAN="$(
  sudo -u netbox \
    /opt/netbox/venv/bin/python \
    /opt/netbox/netbox/manage.py \
    showmigrations --plan
)" || {
  echo "ERROR: Could not read migration plan."
  exit 1
}

if grep -qE \
  '^\[ \]' \
  <<<"$MIGRATION_PLAN"
then
  echo "ERROR: Pending migrations detected."
  printf '%s\n' "$MIGRATION_PLAN" \
    | grep -E '^\[ \]'
  exit 1
else
  echo "No pending migrations detected."
fi
```

خروجی صحیح:

```text
No pending migrations detected.
```

---

## 18. بررسی Pluginها

### [APP]

بررسی Pluginهای Loaded:

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py shell -c \
  "from django.conf import settings; print(settings.PLUGINS)"
```

خروجی:

```python
['netbox_contract', 'netbox_floorplan']
```

بررسی اینکه Modelهای Pluginها توسط Django Load می‌شوند:

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py shell <<'PY'
from django.apps import apps

for app_label in (
    "netbox_contract",
    "netbox_floorplan",
):
    app = apps.get_app_config(app_label)

    print(
        app_label,
        "loaded:",
        bool(app),
    )
PY
```

خروجی:

```text
netbox_contract loaded: True
netbox_floorplan loaded: True
```

---

## 19. بررسی اتصال به دیتابیس و Redis

### [APP]

این دستور Password یا Secret را نمایش نمی‌دهد:

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py shell <<'PY'
from django.conf import settings
from django.db import connection

db = settings.DATABASES["default"]

with connection.cursor() as cursor:
    cursor.execute("SHOW server_version")
    pg_version = cursor.fetchone()[0]

print(
    "Database host:",
    db.get("HOST"),
)

print(
    "Database name:",
    db.get("NAME"),
)

print(
    "PostgreSQL version:",
    pg_version,
)

print(
    "Redis tasks DB:",
    settings.REDIS["tasks"]["DATABASE"],
)

print(
    "Redis caching DB:",
    settings.REDIS["caching"]["DATABASE"],
)
PY
```

خروجی مورد انتظار:

```text
Database host: 192.168.169.171
Database name: netbox
PostgreSQL version: 15.15
Redis tasks DB: 0
Redis caching DB: 1
```

پاک کردن فقط Cache خود NetBox:

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py shell -c \
  "from django.core.cache import cache; cache.clear(); print('NetBox cache cleared.')"
```

دیتابیس Queue یعنی Redis DB شماره صفر را Flush نکنید.

---

# فاز G — Start سرویس‌ها

## 20. راه‌اندازی سرویس‌ها

### [APP]

```bash
sudo systemctl daemon-reload

sudo systemctl start netbox

sudo systemctl start netbox-rq

sudo systemctl start \
  netbox-housekeeping.timer
```

بررسی:

```bash
sudo systemctl is-active \
  netbox \
  netbox-rq \
  netbox-housekeeping.timer \
  redis-server \
  nginx
```

خروجی صحیح:

```text
active
active
active
active
active
```

نمایش Status:

```bash
sudo systemctl \
  --no-pager \
  --full \
  status \
  netbox \
  netbox-rq
```

بررسی Logهای اخیر:

```bash
sudo journalctl \
  -u netbox \
  -u netbox-rq \
  --since "-10 minutes" \
  --no-pager
```

موارد زیر نباید به‌صورت تکرارشونده دیده شوند:

```text
Python traceback
ImportError
ModuleNotFoundError
Plugin compatibility error
Pending migration error
Database authentication error
Redis connection error
HTTP 500 traceback
```

---

## 21. بررسی HTTP و API

### [APP]

اگر URL واقعی متفاوت است، مقدار URL را تغییر دهید:

```bash
curl \
  -k \
  -sS \
  -o /dev/null \
  -w 'HTTP status: %{http_code}\n' \
  https://netbox.mvmco.ir/
```

کدهای زیر قابل قبول‌اند:

```text
200
302
```

بررسی Status API:

```bash
curl \
  -k \
  -sS \
  https://netbox.mvmco.ir/api/status/
```

پاسخ باید شامل وضعیت NetBox و اتصال دیتابیس باشد.

اگر DNS روی سرور Application قابل Resolve نیست، تست را از Browser انجام دهید یا با `Host Header` صحیح به Nginx محلی متصل شوید.

---

## 22. Functional Acceptance

از طریق Browser وارد NetBox شوید و موارد زیر را بررسی کنید:

- [ ] Login بدون Error انجام می‌شود
- [ ] System Status نسخه 4.6.7 را نشان می‌دهد
- [ ] Python نسخه 3.12.x است
- [ ] PostgreSQL نسخه 15.15 است
- [ ] Dashboard بدون HTTP 500 باز می‌شود
- [ ] Devices List باز می‌شود
- [ ] Racks و Rack Elevation باز می‌شوند
- [ ] IPAM Prefixes باز می‌شود
- [ ] IP Addresses باز می‌شوند
- [ ] Circuits باز می‌شوند
- [ ] Contracts Plugin باز می‌شود
- [ ] رکوردهای قبلی Contracts قابل مشاهده هستند
- [ ] Floorplan Plugin باز می‌شود
- [ ] Floorplanهای قبلی قابل مشاهده هستند
- [ ] Background Queues یک Worker فعال نشان می‌دهد
- [ ] REST API Status پاسخ می‌دهد
- [ ] Logهای NetBox و RQ فاقد Error تکرارشونده هستند

برای Integrationهای API به تغییرات زیر توجه کنید:

```text
توکن‌های قدیمی v1 در NetBox 4.6 هنوز کار می‌کنند ولی Deprecated هستند
توکن‌های جدید v2 به API_TOKEN_PEPPERS نیاز دارند
GraphQL Filterهای ID و Enum ممکن است به Lookup صریح نیاز داشته باشند
/api/extras/object-types/ حذف شده است
/api/core/object-types/ جایگزین آن است
/api/dcim/cable-terminations/ در NetBox 4.5+ فقط Read-only است
```

---

# فاز H — ثبت نتیجه

## 23. ثبت وضعیت نهایی

### [APP]

```bash
printf '%s\n' \
  "NetBox path: $(readlink -f /opt/netbox)" \
  "NetBox release: $(grep '^version:' /opt/netbox/netbox/release.yaml)" \
  "Python: $(/opt/netbox/venv/bin/python --version 2>&1)" \
  "Redis server: $(redis-server --version)" \
  "Upgrade log: /var/log/netbox-upgrade-4.6.7.log"
```

تا پایان دوره پذیرش موارد زیر را حذف نکنید:

```text
/opt/netbox-4.4.0
Snapshot سرور Application
Snapshot سرور PostgreSQL
/var/log/netbox-upgrade-4.6.7.log
```

---

# Rollback

## 24. شرایط اجرای Rollback

در صورت مشاهده هرکدام از موارد زیر Rollback کنید:

```text
شکست upgrade.sh بعد از شروع Migration
Start نشدن سرویس netbox
Start نشدن netbox-rq
Load نشدن netbox-contract
Load نشدن netbox-floorplan
از دسترس خارج بودن رکوردهای قبلی Pluginها
HTTP 500 تکرارشونده
Database Migration Error
ناسازگاری غیرقابل رفع Dependency
```

---

## 25. توقف سرویس‌ها قبل از Revert

### [APP]

اگر VM هنوز در دسترس است:

```bash
sudo systemctl stop \
  netbox \
  2>/dev/null || true

sudo systemctl stop \
  netbox-rq \
  2>/dev/null || true

sudo systemctl stop \
  netbox-housekeeping.timer \
  2>/dev/null || true
```

---

## 26. بازگردانی Snapshotها

### [VMWARE]

ترتیب عملیات:

```text
1. Power Off سرور NetBox Application
2. Power Off سرور PostgreSQL
3. Revert کردن Snapshot قبل از Upgrade سرور PostgreSQL
4. Revert کردن Snapshot قبل از Upgrade سرور Application
5. Power On سرور PostgreSQL
6. انتظار تا PostgreSQL آماده اتصال شود
7. Power On سرور Application
8. بررسی سرویس‌های NetBox 4.4.0
```

نکته مهم:

```text
ترتیب Start باید Database و سپس Application باشد
```

---

## 27. بررسی بعد از Rollback

### [DB]

```bash
sudo -u postgres pg_isready \
  -h 127.0.0.1 \
  -p 5432
```

### [APP]

```bash
readlink -f /opt/netbox

grep '^version:' \
  /opt/netbox/netbox/release.yaml

sudo systemctl is-active \
  netbox \
  netbox-rq \
  netbox-housekeeping.timer \
  redis-server \
  nginx

sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py check
```

وضعیت مورد انتظار:

```text
/opt/netbox-4.4.0
NetBox 4.4.0
Virtual Environment مبتنی بر Python 3.10.12
تمام سرویس‌ها active
Django check بدون Error
```

---

# Troubleshooting

## 28. Python 3.12 در APT پیدا نمی‌شود

### [APP]

```bash
apt-cache policy \
  python3.12

grep -R \
  deadsnakes \
  /etc/apt/sources.list \
  /etc/apt/sources.list.d/ \
  2>/dev/null

sudo apt update
```

`/usr/bin/python3` را تغییر ندهید.

---

## 29. خطای Unsupported Python Version

`PYTHON` باید بعد از `sudo` مشخص شود:

```bash
cd /opt/netbox

sudo PYTHON=/usr/bin/python3.12 \
  ./upgrade.sh
```

---

## 30. خطای Plugin Compatibility

بررسی فایل:

```bash
cat \
  /opt/netbox/local_requirements.txt
```

محتوای صحیح:

```text
netbox-contract==2.4.6
netbox-floorplan-plugin==0.9.2
```

بررسی نسخه‌های نصب‌شده:

```bash
/opt/netbox/venv/bin/pip show \
  netbox-contract \
  netbox-floorplan-plugin
```

اگر Plugin مانع Migration یا Start سرویس شد، از Snapshot Rollback استفاده کنید.

---

## 31. خطای `502 Bad Gateway`

### [APP]

```bash
sudo systemctl status \
  netbox \
  --no-pager \
  -l

sudo journalctl \
  -u netbox \
  -n 200 \
  --no-pager

sudo ss \
  -lntp \
  | grep -E \
    'gunicorn|8001'
```

مسیرهای `ExecStart` باید همچنان از Symlink استفاده کنند:

```text
/opt/netbox/venv/bin/gunicorn
/opt/netbox/venv/bin/python3
```

---

## 32. Migration معلق باقی مانده است

دستور زیر را اجرا نکنید:

```text
python manage.py makemigrations
```

اسکریپت رسمی Upgrade را مجدداً اجرا کنید:

```bash
cd /opt/netbox

sudo PYTHON=/usr/bin/python3.12 \
  ./upgrade.sh
```

بررسی انتهای Log:

```bash
sudo tail \
  -n 200 \
  /var/log/netbox-upgrade-4.6.7.log
```

اگر اجرای مجدد نیز Fail شد، Rollback کامل VMware انجام دهید.

---

## 33. خطای `API_TOKEN_PEPPERS`

بدون نمایش مقدار Secret بررسی کنید:

```bash
sudo /usr/bin/python3.12 - <<'PY'
from pathlib import Path
import re

text = Path(
    "/opt/netbox/netbox/netbox/configuration.py"
).read_text()

found = bool(
    re.search(
        r"(?m)^\s*API_TOKEN_PEPPERS\s*=",
        text,
    )
)

print(found)
PY
```

خروجی صحیح:

```text
True
```

---

# معیار نهایی موفقیت

Change فقط زمانی موفق است که تمام موارد زیر برقرار باشند:

```text
/opt/netbox به /opt/netbox-4.6.7 Resolve شود
NetBox نسخه 4.6.7 را گزارش کند
Virtual Environment از Python 3.12.x استفاده کند
PostgreSQL روی نسخه 15.15 باقی بماند
Redis Server روی نسخه 6.0.16 باقی بماند
netbox-contract نسخه 2.4.6 باشد
netbox-floorplan-plugin نسخه 0.9.2 باشد
Django system check موفق باشد
هیچ Migration معلقی وجود نداشته باشد
سرویس netbox فعال باشد
سرویس netbox-rq فعال باشد
netbox-housekeeping.timer فعال باشد
Contracts Plugin کار کند
Floorplan Plugin کار کند
REST API Status پاسخ دهد
در Journal خطای تکرارشونده وجود نداشته باشد
```
