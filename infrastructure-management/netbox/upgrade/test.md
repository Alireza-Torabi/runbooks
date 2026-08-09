<div dir="rtl" align="right">

# Runbook پیاده‌سازی Private IP Wildcard DNS با CoreDNS

## هدف

هدف ایجاد یک سرویس DNS داخلی مشابه `sslip.io` و `nip.io` است که IP خصوصی را از داخل نام DNS استخراج کرده و بدون نیاز به ساخت DNS Record، همان IP را برگرداند.

دامنه مورد استفاده:

</div>

```text
dev.internal
```

<div dir="rtl" align="right">

فرمت استاندارد پیشنهادی:

</div>

```text
<service>.<private-ip-with-dashes>.dev.internal
```

<div dir="rtl" align="right">

مثال:

</div>

```text
192-168-10-25.dev.internal
grafana.192-168-10-25.dev.internal
api.10-20-30-40.dev.internal
gitlab.172-16-50-20.dev.internal
```

<div dir="rtl" align="right">

که به‌ترتیب به این IPها Resolve می‌شوند:

</div>

```text
192.168.10.25
192.168.10.25
10.20.30.40
172.16.50.20
```

<div dir="rtl" align="right">

هیچ DNS Record ثابتی برای این نام‌ها ساخته نمی‌شود.

---

# 1. معماری

معماری پیشنهادی در شبکه سازمان:

</div>

```text
Developers / DevOps / Servers
             |
             v
    Corporate DNS Servers
    Windows DNS / BIND
             |
             | Conditional Forwarder
             | dev.internal
             v
       +-------------+
       |   CoreDNS   |
       |192.168.10.53|
       +-------------+
             |
             | Dynamic DNS Response
             v

grafana.192-168-50-20.dev.internal
                  |
                  v
             192.168.50.20
```

<div dir="rtl" align="right">

در این معماری CoreDNS قرار نیست DNS اصلی کاربران باشد.

DNS فعلی سازمان همچنان DNS اصلی باقی می‌ماند و فقط Queryهای مربوط به:

</div>

```text
*.dev.internal
```

<div dir="rtl" align="right">

به CoreDNS ارسال می‌شوند.

این طراحی در شبکه‌هایی که Active Directory و Windows DNS دارند، گزینه مناسبی است.

---

# 2. محدوده IPهای مجاز

این سرویس فقط باید Private IPv4 Addressهای RFC1918 را قبول کند:

</div>

```text
10.0.0.0/8

172.16.0.0/12

192.168.0.0/16
```

<div dir="rtl" align="right">

در نتیجه مثلاً موارد زیر معتبر هستند:

</div>

```text
app.10-10-20-30.dev.internal

app.172-16-20-30.dev.internal

app.172-31-20-30.dev.internal

app.192-168-100-50.dev.internal
```

<div dir="rtl" align="right">

اما این موارد نباید Resolve شوند:

</div>

```text
app.8-8-8-8.dev.internal

app.172-40-10-20.dev.internal

app.203-0-113-20.dev.internal

app.192-168-999-20.dev.internal
```

<div dir="rtl" align="right">

این محدودیت عمداً اعمال می‌شود تا سرویس فقط برای شبکه داخلی قابل استفاده باشد.

---

# 3. مشخصات نمونه سرور

در این Runbook فرض می‌کنیم:

</div>

```text
CoreDNS Server IP : 192.168.10.53
DNS Zone          : dev.internal
DNS Port          : TCP/UDP 53
Health Port       : 127.0.0.1:8080
Ready Port        : 127.0.0.1:8181
Metrics Port      : 127.0.0.1:9153
```

<div dir="rtl" align="right">

آدرس `192.168.10.53` نمونه است و باید با IP واقعی سرور جایگزین شود.

---

# 4. بررسی پورت 53

قبل از نصب بررسی کنید سرویس دیگری روی IP موردنظر پورت 53 را اشغال نکرده باشد.

</div>

```bash
sudo ss -lntup | grep ':53'
```

<div dir="rtl" align="right">

در Ubuntu ممکن است `systemd-resolved` روی آدرس Loopback در حال Listen باشد.

در این طراحی CoreDNS را فقط روی IP اصلی سرور Bind می‌کنیم:

</div>

```text
192.168.10.53
```

<div dir="rtl" align="right">

بنابراین نیازی نیست بدون دلیل `systemd-resolved` را غیرفعال کنیم.

CoreDNS دارای پلاگین `bind` برای Listen کردن روی IP مشخص است.

---

# 5. نصب CoreDNS

در زمان تهیه این Runbook نسخه مورد استفاده:

</div>

```text
CoreDNS 1.14.6
```

<div dir="rtl" align="right">

## تشخیص معماری CPU

</div>

```bash
uname -m
```

<div dir="rtl" align="right">

برای `x86_64`:

</div>

```bash
export COREDNS_VERSION="1.14.6"
export COREDNS_ARCH="amd64"
```

<div dir="rtl" align="right">

برای ARM64:

</div>

```bash
export COREDNS_VERSION="1.14.6"
export COREDNS_ARCH="arm64"
```

<div dir="rtl" align="right">

دانلود:

</div>

```bash
cd /tmp

curl -fL \
  -o coredns.tgz \
  "https://github.com/coredns/coredns/releases/download/v${COREDNS_VERSION}/coredns_${COREDNS_VERSION}_linux_${COREDNS_ARCH}.tgz"
```

<div dir="rtl" align="right">

Extract:

</div>

```bash
tar -xzf coredns.tgz
```

<div dir="rtl" align="right">

نصب Binary:

</div>

```bash
sudo install -m 0755 coredns /usr/local/bin/coredns
```

<div dir="rtl" align="right">

بررسی نسخه:

</div>

```bash
/usr/local/bin/coredns -version
```

---

<div dir="rtl" align="right">

# 6. ایجاد User سرویس

CoreDNS را با کاربر `root` اجرا نمی‌کنیم.

</div>

```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /bin/false \
  coredns
```

<div dir="rtl" align="right">

ایجاد مسیر تنظیمات:

</div>

```bash
sudo mkdir -p /etc/coredns
sudo chown root:coredns /etc/coredns
sudo chmod 0750 /etc/coredns
```

---

<div dir="rtl" align="right">

# 7. ساخت Corefile

فایل زیر را ایجاد کنید:

</div>

```bash
sudo nano /etc/coredns/Corefile
```

<div dir="rtl" align="right">

کانفیگ Production پیشنهادی:

</div>

```caddy
dev.internal:53 {

    bind 192.168.10.53

    errors
    log

    health 127.0.0.1:8080
    ready 127.0.0.1:8181
    prometheus 127.0.0.1:9153

    reload

    #
    # Private IPv4 Dynamic DNS
    #
    # Supported networks:
    #
    #   10.0.0.0/8
    #   172.16.0.0/12
    #   192.168.0.0/16
    #

    template IN A dev.internal {

        #
        # 10.0.0.0/8
        #

        match "(^|[.])(?P<a>10)-(?P<b>(25[0-5]|2[0-4][0-9]|1[0-9]{2}|[1-9]?[0-9]))-(?P<c>(25[0-5]|2[0-4][0-9]|1[0-9]{2}|[1-9]?[0-9]))-(?P<d>(25[0-5]|2[0-4][0-9]|1[0-9]{2}|[1-9]?[0-9]))[.]dev[.]internal[.]$"

        #
        # 172.16.0.0/12
        #

        match "(^|[.])(?P<a>172)-(?P<b>(1[6-9]|2[0-9]|3[01]))-(?P<c>(25[0-5]|2[0-4][0-9]|1[0-9]{2}|[1-9]?[0-9]))-(?P<d>(25[0-5]|2[0-4][0-9]|1[0-9]{2}|[1-9]?[0-9]))[.]dev[.]internal[.]$"

        #
        # 192.168.0.0/16
        #

        match "(^|[.])(?P<a>192)-(?P<b>168)-(?P<c>(25[0-5]|2[0-4][0-9]|1[0-9]{2}|[1-9]?[0-9]))-(?P<d>(25[0-5]|2[0-4][0-9]|1[0-9]{2}|[1-9]?[0-9]))[.]dev[.]internal[.]$"

        answer "{{ .Name }} 30 IN A {{ .Group.a }}.{{ .Group.b }}.{{ .Group.c }}.{{ .Group.d }}"

        fallthrough
    }

    #
    # Reject everything else inside dev.internal
    #

    template IN ANY dev.internal {
        rcode NXDOMAIN
    }
}
```

<div dir="rtl" align="right">

پلاگین `template` در CoreDNS دقیقاً برای تولید Dynamic DNS Response بر اساس Query طراحی شده و امکان استفاده از Regex و Named Capture Group را فراهم می‌کند. مستندات رسمی حتی نمونه‌ای مشابه استخراج IP از hostname ارائه می‌کنند.

---

# 8. Permission فایل تنظیمات

</div>

```bash
sudo chown root:coredns /etc/coredns/Corefile

sudo chmod 0640 /etc/coredns/Corefile
```

---

<div dir="rtl" align="right">

# 9. ایجاد Systemd Service

فایل زیر را ایجاد کنید:

</div>

```bash
sudo nano /etc/systemd/system/coredns.service
```

<div dir="rtl" align="right">

محتوا:

</div>

```ini
[Unit]
Description=CoreDNS Internal Private-IP DNS
Documentation=https://coredns.io/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple

User=coredns
Group=coredns

WorkingDirectory=/etc/coredns

ExecStart=/usr/local/bin/coredns \
    -conf /etc/coredns/Corefile

Restart=on-failure
RestartSec=3

AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE

NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true
ProtectSystem=strict

LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

<div dir="rtl" align="right">

به دلیل استفاده از:

</div>

```ini
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

<div dir="rtl" align="right">

CoreDNS می‌تواند با User غیر Root روی پورت 53 Listen کند.

---

# 10. فعال‌سازی سرویس

</div>

```bash
sudo systemctl daemon-reload

sudo systemctl enable --now coredns
```

<div dir="rtl" align="right">

بررسی وضعیت:

</div>

```bash
systemctl status coredns
```

<div dir="rtl" align="right">

بررسی Log:

</div>

```bash
journalctl -u coredns -n 100 --no-pager
```

<div dir="rtl" align="right">

مشاهده Live Log:

</div>

```bash
journalctl -u coredns -f
```

---

<div dir="rtl" align="right">

# 11. بررسی Listen Port

</div>

```bash
sudo ss -lntup | grep ':53'
```

<div dir="rtl" align="right">

باید CoreDNS را روی:

</div>

```text
192.168.10.53:53
```

<div dir="rtl" align="right">

برای TCP و UDP مشاهده کنید.

---

# 12. Health Check

</div>

```bash
curl http://127.0.0.1:8080/health
```

<div dir="rtl" align="right">

خروجی مورد انتظار:

</div>

```text
OK
```

<div dir="rtl" align="right">

Readiness:

</div>

```bash
curl http://127.0.0.1:8181/ready
```

<div dir="rtl" align="right">

خروجی:

</div>

```text
OK
```

<div dir="rtl" align="right">

CoreDNS به‌صورت رسمی Health endpoint را روی `/health` و Ready endpoint را روی `/ready` پشتیبانی می‌کند.

---

# 13. تست مستقیم DNS

ابتدا مستقیماً CoreDNS را Query کنید.

## تست شبکه 192.168/16

</div>

```bash
dig @192.168.10.53 grafana.192-168-50-20.dev.internal A
```

<div dir="rtl" align="right">

خروجی مورد انتظار:

</div>

```text
grafana.192-168-50-20.dev.internal. 30 IN A 192.168.50.20
```

<div dir="rtl" align="right">

تست بدون Service Name:

</div>

```bash
dig @192.168.10.53 192-168-100-50.dev.internal A
```

<div dir="rtl" align="right">

باید برگرداند:

</div>

```text
192.168.100.50
```

---

<div dir="rtl" align="right">

## تست شبکه 10/8

</div>

```bash
dig @192.168.10.53 api.10-20-30-40.dev.internal A
```

<div dir="rtl" align="right">

خروجی:

</div>

```text
10.20.30.40
```

---

<div dir="rtl" align="right">

## تست شبکه 172.16/12

</div>

```bash
dig @192.168.10.53 gitlab.172-20-50-60.dev.internal A
```

<div dir="rtl" align="right">

خروجی:

</div>

```text
172.20.50.60
```

---

<div dir="rtl" align="right">

# 14. تست Security Boundary

Public IP نباید Resolve شود:

</div>

```bash
dig @192.168.10.53 test.8-8-8-8.dev.internal A
```

<div dir="rtl" align="right">

انتظار:

</div>

```text
NXDOMAIN
```

<div dir="rtl" align="right">

همچنین:

</div>

```bash
dig @192.168.10.53 test.217-218-100-10.dev.internal A
```

<div dir="rtl" align="right">

باید:

</div>

```text
NXDOMAIN
```

<div dir="rtl" align="right">

باشد.

---

# 15. تست Invalid IPv4

</div>

```bash
dig @192.168.10.53 test.192-168-999-10.dev.internal A
```

<div dir="rtl" align="right">

انتظار:

</div>

```text
NXDOMAIN
```

<div dir="rtl" align="right">

همین‌طور:

</div>

```bash
dig @192.168.10.53 test.10-300-20-30.dev.internal A
```

<div dir="rtl" align="right">

باید رد شود.

---

# 16. اتصال به Windows DNS

اگر کلاینت‌های شبکه از Active Directory DNS استفاده می‌کنند، توصیه می‌شود DNS کلاینت‌ها تغییر نکند.

مثلاً:

</div>

```text
Client
   |
   v
Windows DNS
192.168.10.10
   |
   | Conditional Forwarder
   | dev.internal
   v
CoreDNS
192.168.10.53
```

<div dir="rtl" align="right">

روی Windows DNS Server با PowerShell:

</div>

```powershell
Add-DnsServerConditionalForwarderZone `
    -Name "dev.internal" `
    -MasterServers 192.168.10.53 `
    -ReplicationScope "Forest"
```

<div dir="rtl" align="right">

Microsoft این Cmdlet را برای ایجاد Conditional Forwarder و تعیین Master DNS Server ارائه می‌کند.

بررسی:

</div>

```powershell
Get-DnsServerZone -Name "dev.internal"
```

<div dir="rtl" align="right">

اکنون از Client معمولی:

</div>

```powershell
Resolve-DnsName grafana.192-168-50-20.dev.internal
```

<div dir="rtl" align="right">

باید:

</div>

```text
192.168.50.20
```

<div dir="rtl" align="right">

برگردد.

---

# 17. اتصال به BIND

اگر DNS اصلی سازمان BIND است:

</div>

```conf
zone "dev.internal" {
    type forward;
    forward only;

    forwarders {
        192.168.10.53;
    };
};
```

<div dir="rtl" align="right">

سپس:

</div>

```bash
named-checkconf
```

<div dir="rtl" align="right">

و:

</div>

```bash
sudo systemctl reload named
```

<div dir="rtl" align="right">

یا در Debian/Ubuntu:

</div>

```bash
sudo systemctl reload bind9
```

---

<div dir="rtl" align="right">

# 18. Firewall

CoreDNS نباید از اینترنت قابل دسترس باشد.

حداقل دسترسی لازم:

</div>

```text
UDP/53
TCP/53
```

<div dir="rtl" align="right">

اگر فقط Windows DNS به CoreDNS Query می‌زند، بهترین حالت این است که فقط DNS Serverهای سازمان اجازه دسترسی داشته باشند.

مثلاً:

</div>

```text
192.168.10.10  -> CoreDNS UDP/53
192.168.10.10  -> CoreDNS TCP/53

192.168.10.11  -> CoreDNS UDP/53
192.168.10.11  -> CoreDNS TCP/53
```

<div dir="rtl" align="right">

و سایر Sourceها Block شوند.

---

# 19. دلیل عدم استفاده از Forwarder عمومی در CoreDNS

عمداً در Corefile چیزی مانند این قرار نداده‌ایم:

</div>

```caddy
forward . 8.8.8.8
```

<div dir="rtl" align="right">

یا:

</div>

```caddy
forward . 1.1.1.1
```

<div dir="rtl" align="right">

این CoreDNS فقط یک سرویس تخصصی برای:

</div>

```text
dev.internal
```

<div dir="rtl" align="right">

است.

Recursive DNS عمومی سازمان همچنان توسط DNS Serverهای اصلی انجام می‌شود.

این جداسازی باعث ساده‌تر شدن Scope، امنیت و Troubleshooting می‌شود.

---

# 20. نحوه استفاده تیم DevOps

فرض کنیم Developer یک VM با IP زیر ایجاد می‌کند:

</div>

```text
192.168.80.37
```

<div dir="rtl" align="right">

نیازی به درخواست DNS Record ندارد.

می‌تواند مستقیم استفاده کند:

</div>

```text
web.192-168-80-37.dev.internal
```

<div dir="rtl" align="right">

یا:

</div>

```text
api.192-168-80-37.dev.internal
```

<div dir="rtl" align="right">

یا:

</div>

```text
anything.192-168-80-37.dev.internal
```

<div dir="rtl" align="right">

همه این نام‌ها به:

</div>

```text
192.168.80.37
```

<div dir="rtl" align="right">

Resolve می‌شوند.

---

# 21. مثال‌های کاربردی

</div>

```text
grafana.192-168-10-20.dev.internal
argocd.192-168-10-21.dev.internal
jenkins.192-168-10-22.dev.internal
nexus.192-168-10-23.dev.internal

api.192-168-50-101.dev.internal
frontend.192-168-50-102.dev.internal
backend.192-168-50-103.dev.internal

test.10-10-20-30.dev.internal
demo.10-20-30-40.dev.internal

gitlab.172-20-10-50.dev.internal
runner.172-20-10-51.dev.internal
```

---

<div dir="rtl" align="right">

# 22. استفاده با Nginx

مثلاً سروری با IP:

</div>

```text
192.168.50.20
```

<div dir="rtl" align="right">

می‌تواند hostname زیر داشته باشد:

</div>

```text
myapp.192-168-50-20.dev.internal
```

<div dir="rtl" align="right">

و در Nginx:

</div>

```nginx
server {
    listen 80;

    server_name myapp.192-168-50-20.dev.internal;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

<div dir="rtl" align="right">

# 23. استفاده با Docker

Developer می‌تواند یک Container روی:

</div>

```text
192.168.50.30
```

<div dir="rtl" align="right">

داشته باشد و بدون ساخت DNS Record از:

</div>

```text
mycontainer.192-168-50-30.dev.internal
```

<div dir="rtl" align="right">

استفاده کند.

---

# 24. استفاده در Kubernetes

این روش برای Node IP، LoadBalancer IP یا Ingress VIP هم قابل استفاده است.

مثلاً:

</div>

```text
Ingress IP:

192.168.100.50
```

<div dir="rtl" align="right">

آدرس سرویس:

</div>

```text
project1.192-168-100-50.dev.internal
```

<div dir="rtl" align="right">

Ingress:

</div>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: project1

spec:

  rules:

    - host: project1.192-168-100-50.dev.internal

      http:

        paths:

          - path: /
            pathType: Prefix

            backend:

              service:
                name: project1
                port:
                  number: 80
```

---

<div dir="rtl" align="right">

# 25. Monitoring

Metrics به‌صورت Local روی:

</div>

```text
http://127.0.0.1:9153/metrics
```

<div dir="rtl" align="right">

در دسترس هستند.

تست:

</div>

```bash
curl http://127.0.0.1:9153/metrics
```

<div dir="rtl" align="right">

CoreDNS پلاگین رسمی Prometheus دارد و Metrics DNS، تعداد Queryها و خطاهای Template را Export می‌کند.

در صورت نیاز می‌توان این Endpoint را فقط برای Prometheus Server باز کرد.

---

# 26. High Availability

برای محیط سازمانی توصیه می‌شود حداقل دو CoreDNS داشته باشید:

</div>

```text
CoreDNS-01
192.168.10.53

CoreDNS-02
192.168.10.54
```

<div dir="rtl" align="right">

Corefile هر دو یکسان باشد.

در Windows DNS:

</div>

```powershell
Add-DnsServerConditionalForwarderZone `
    -Name "dev.internal" `
    -MasterServers 192.168.10.53,192.168.10.54 `
    -ReplicationScope "Forest"
```

<div dir="rtl" align="right">

معماری:

</div>

```text
                    +--> CoreDNS-01
                    |    192.168.10.53
Windows DNS --------+
                    |
                    +--> CoreDNS-02
                         192.168.10.54
```

---

<div dir="rtl" align="right">

# 27. Convention پیشنهادی

فرمت رسمی تیم DevOps بهتر است این باشد:

</div>

```text
<service>.<ip>.dev.internal
```

<div dir="rtl" align="right">

مثلاً:

</div>

```text
grafana.192-168-10-20.dev.internal
argocd.192-168-10-30.dev.internal
api.10-10-20-50.dev.internal
```

<div dir="rtl" align="right">

و IP همیشه با `-` نوشته شود:

</div>

```text
192-168-10-20
```

<div dir="rtl" align="right">

نه:

</div>

```text
192.168.10.20
```

<div dir="rtl" align="right">

این Convention خواناتر است و تشخیص IP داخل hostname را ساده می‌کند.

---

# 28. Checklist نهایی

* CoreDNS نصب شده باشد.
* CoreDNS با User غیر Root اجرا شود.
* فقط روی IP داخلی سرور Listen کند.
* TCP/53 فعال باشد.
* UDP/53 فعال باشد.
* Public IPها Resolve نشوند.
* فقط RFC1918 مجاز باشد.
* `dev.internal` روی DNS اصلی Conditional Forward شود.
* CoreDNS به DNS عمومی Forward نکند.
* Health Check فعال باشد.
* Firewall فقط Sourceهای موردنیاز را قبول کند.
* ترجیحاً دو CoreDNS برای HA وجود داشته باشد.
* Monitoring روی Prometheus فعال شود.
* TTL کوتاه، مثلاً 30 ثانیه، حفظ شود.

---

# 29. تست پذیرش نهایی

این دستورات باید موفق باشند:

</div>

```bash
dig grafana.192-168-10-20.dev.internal

dig app.10-20-30-40.dev.internal

dig gitlab.172-20-30-40.dev.internal
```

<div dir="rtl" align="right">

و به‌ترتیب:

</div>

```text
192.168.10.20

10.20.30.40

172.20.30.40
```

<div dir="rtl" align="right">

برگردانند.

اما:

</div>

```bash
dig app.8-8-8-8.dev.internal

dig app.217-218-10-20.dev.internal

dig app.192-168-999-20.dev.internal
```

<div dir="rtl" align="right">

باید:

</div>

```text
NXDOMAIN
```

<div dir="rtl" align="right">

برگردانند.

---

# 30. Rollback

ابتدا Conditional Forwarder مربوط به `dev.internal` را از DNS اصلی حذف کنید.

در Windows DNS:

</div>

```powershell
Remove-DnsServerZone \
    -Name "dev.internal" \
    -Force
```

<div dir="rtl" align="right">

سپس CoreDNS:

</div>

```bash
sudo systemctl disable --now coredns
```

<div dir="rtl" align="right">

در صورت حذف کامل:

</div>

```bash
sudo rm -f /etc/systemd/system/coredns.service
sudo rm -rf /etc/coredns
sudo rm -f /usr/local/bin/coredns

sudo systemctl daemon-reload
```

---

<div dir="rtl" align="right">

# نتیجه نهایی

پس از پیاده‌سازی، شبکه داخلی عملاً سرویس اختصاصی مشابه `sslip.io` / `nip.io` خواهد داشت:

</div>

```text
grafana.192-168-10-20.dev.internal
                   |
                   v
             192.168.10.20


api.10-20-30-40.dev.internal
             |
             v
         10.20.30.40


gitlab.172-20-50-60.dev.internal
                  |
                  v
            172.20.50.60
```

<div dir="rtl" align="right">

بدون:

* ساخت دستی DNS Record
* تغییر DNS Client
* وابستگی به سرویس اینترنتی
* استفاده از Public IP
* نیاز به دامنه Public

و با کنترل کامل DNS در داخل شبکه سازمان.

</div>
