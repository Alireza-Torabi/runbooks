<h1 dir="rtl" align="right">راهنمای راه‌اندازی Private CA و ACME داخلی با Smallstep</h1>

<p dir="rtl" align="right">
این Runbook برای انتشار عمومی در GitHub آماده شده و عمداً هیچ نام سازمان، دامنه واقعی، IP واقعی، ایمیل واقعی یا اطلاعات محیط عملیاتی در آن وجود ندارد.
تمام نام‌ها و آدرس‌ها نمونه هستند و باید قبل از استفاده با مقادیر محیط خودتان جایگزین شوند.
</p>

<h2 dir="rtl" align="right">۱. هدف</h2>

<p dir="rtl" align="right">
هدف، ساخت یک Certificate Authority داخلی است که بتواند برای سرویس‌های شبکه داخلی Certificate صادر کند و با استفاده از ACME، فرایند صدور و تمدید Certificate تا حد ممکن شبیه Let's Encrypt باشد.
</p>

<p dir="rtl" align="right">در این راهنما از نام‌های نمونه زیر استفاده می‌شود:</p>

```text
Internal DNS domain: corp.example.com
CA hostname:         ca.corp.example.com
Example service:     grafana.dev.corp.example.com
```

<p dir="rtl" align="right">
دامنه <code>example.com</code> برای مستندات رزرو شده است. در محیط واقعی، آن را با دامنه داخلی یا Subdomain تحت کنترل خودتان جایگزین کنید.
</p>

<h2 dir="rtl" align="right">۲. معماری پیشنهادی</h2>

```text
                     Offline Root CA
                           |
                           | signs
                           v
                  Online Issuing CA
                       step-ca
                           |
                          ACME
          +----------------+----------------+
          |                |                |
          v                v                v
       Certbot          win-acme        cert-manager
      Linux/Nginx      Windows/IIS      Kubernetes
```

<p dir="rtl" align="right">
Root CA باید آفلاین نگهداری شود و مستقیماً Certificate سرویس‌های روزمره را صادر نکند.
یک Intermediate/Issuing CA آنلاین مسئول صدور Certificateها خواهد بود.
</p>

<h2 dir="rtl" align="right">۳. سرورها و منابع پیشنهادی</h2>

<h3 dir="rtl" align="right">Offline Root CA</h3>

| Item | Recommendation |
|---|---|
| OS | Ubuntu Server 24.04 LTS Minimal |
| CPU | 1 vCPU |
| RAM | 2 GB |
| Disk | 10 GB |
| Network | Disconnected after setup |
| Software | `step-cli` |
| Backup | At least two encrypted offline copies |

<h3 dir="rtl" align="right">Online Issuing CA</h3>

| Item | Recommendation |
|---|---|
| OS | Ubuntu Server 24.04 LTS |
| CPU | 2 vCPU |
| RAM | 4 GB |
| Disk | 20–40 GB |
| Network | Internal server VLAN |
| IP | Static |
| DNS | `ca.corp.example.com` |
| Service | `step-ca` |
| Time | Reliable NTP required |

<h2 dir="rtl" align="right">۴. پیش‌نیاز DNS و Firewall</h2>

<p dir="rtl" align="right">در DNS داخلی باید رکورد CA وجود داشته باشد:</p>

```text
ca.corp.example.com    A    <CA_SERVER_IP>
```

<p dir="rtl" align="right">دسترسی‌های معمول:</p>

```text
Clients -> CA Server        TCP/443

For ACME HTTP-01:
CA Server -> Web Server     TCP/80

For ACME TLS-ALPN-01:
CA Server -> Target Server  TCP/443
```

<p dir="rtl" align="right">
برای Wildcard Certificate معمولاً از DNS-01 استفاده می‌شود و ACME Client باید امکان ایجاد TXT Record در DNS را داشته باشد.
</p>

<h2 dir="rtl" align="right">۵. طول عمر پیشنهادی Certificateها</h2>

| Certificate | Suggested Lifetime |
|---|---:|
| Root CA | 10 years |
| Intermediate CA | 5 years |
| Service/Leaf Certificate | 90 days |

<h2 dir="rtl" align="right">۶. ساخت Offline Root CA</h2>

<p dir="rtl" align="right">ابتدا روی ماشین Offline Root، ابزار <code>step-cli</code> را نصب کنید:</p>

```bash
sudo -i

apt-get update
apt-get install -y curl gpg ca-certificates

curl -fsSL \
  https://packages.smallstep.com/keys/apt/repo-signing-key.gpg \
  -o /etc/apt/keyrings/smallstep.asc

cat > /etc/apt/sources.list.d/smallstep.sources <<'EOF'
Types: deb
URIs: https://packages.smallstep.com/stable/debian
Suites: debs
Components: main
Signed-By: /etc/apt/keyrings/smallstep.asc
EOF

apt-get update
apt-get install -y step-cli
```

<p dir="rtl" align="right">ساخت Root CA:</p>

```bash
sudo -i
umask 077

mkdir -p /root/private-root-ca
cd /root/private-root-ca

step certificate create \
  --profile root-ca \
  --not-after 87600h \
  "Example Organization Root CA" \
  root_ca.crt \
  root_ca.key
```

<p dir="rtl" align="right">بررسی Certificate:</p>

```bash
step certificate inspect root_ca.crt
step certificate fingerprint root_ca.crt
```

<p dir="rtl" align="right">
فایل <code>root_ca.key</code> بسیار حساس است و نباید روی Online CA، Git repository، Ansible repository یا هیچ سرور عادی دیگری کپی شود.
</p>

<h2 dir="rtl" align="right">۷. Backup مربوط به Root CA</h2>

<p dir="rtl" align="right">حداقل موارد زیر را به‌صورت آفلاین نگهداری کنید:</p>

```text
root_ca.crt
root_ca.key
Root CA private-key password
Root CA fingerprint
```

<p dir="rtl" align="right">
Password را ترجیحاً جدا از Backup حاوی Private Key نگهداری کنید.
</p>

<h2 dir="rtl" align="right">۸. نصب Online CA Server</h2>

```bash
sudo -i

apt-get update
apt-get install -y curl gpg ca-certificates jq libcap2-bin

curl -fsSL \
  https://packages.smallstep.com/keys/apt/repo-signing-key.gpg \
  -o /etc/apt/keyrings/smallstep.asc

cat > /etc/apt/sources.list.d/smallstep.sources <<'EOF'
Types: deb
URIs: https://packages.smallstep.com/stable/debian
Suites: debs
Components: main
Signed-By: /etc/apt/keyrings/smallstep.asc
EOF

apt-get update
apt-get install -y step-cli step-ca

step version
step-ca version
```

<h2 dir="rtl" align="right">۹. Initialize اولیه step-ca</h2>

```bash
sudo -i
umask 077

openssl rand -base64 48 > /root/step-ca-password.txt
openssl rand -base64 48 > /root/step-ca-provisioner-password.txt

chmod 600 /root/step-ca-*.txt

export STEPPATH=/root/.step

step ca init \
  --name "Example Internal PKI" \
  --dns "ca.corp.example.com" \
  --address ":443" \
  --provisioner "ca-admin@example.com" \
  --password-file /root/step-ca-password.txt \
  --provisioner-password-file /root/step-ca-provisioner-password.txt
```

<p dir="rtl" align="right">
این مرحله یک Root و Intermediate موقت می‌سازد. در طراحی این Runbook، Root واقعی همان Root آفلاین است؛ بنابراین کلیدهای موقت CA جایگزین می‌شوند.
</p>

<h2 dir="rtl" align="right">۱۰. حذف CA موقت از Online Server</h2>

```bash
shred -u "$STEPPATH/secrets/root_ca_key"
shred -u "$STEPPATH/secrets/intermediate_ca_key"

rm -f \
  "$STEPPATH/certs/root_ca.crt" \
  "$STEPPATH/certs/intermediate_ca.crt"
```

<h2 dir="rtl" align="right">۱۱. انتقال Public Root Certificate</h2>

<p dir="rtl" align="right">
فقط <code>root_ca.crt</code> را به Online CA منتقل کنید. کلید خصوصی Root نباید منتقل شود.
</p>

```bash
install -m 0644 \
  /root/root_ca.crt \
  "$STEPPATH/certs/root_ca.crt"
```

<h2 dir="rtl" align="right">۱۲. ساخت Intermediate CSR روی Online CA</h2>

```bash
step certificate create \
  "Example Internal Issuing CA" \
  /root/intermediate_ca.csr \
  "$STEPPATH/secrets/intermediate_ca_key" \
  --csr \
  --password-file /root/step-ca-password.txt
```

<p dir="rtl" align="right">
Private Key مربوط به Intermediate باید روی Online CA باقی بماند. فقط CSR برای امضا به Offline Root منتقل می‌شود.
</p>

<h2 dir="rtl" align="right">۱۳. امضای Intermediate روی Offline Root</h2>

```bash
step certificate sign \
  --profile intermediate-ca \
  --path-len 0 \
  --not-after 43800h \
  /path/to/intermediate_ca.csr \
  root_ca.crt \
  root_ca.key \
  > intermediate_ca.crt
```

<p dir="rtl" align="right">بررسی:</p>

```bash
step certificate verify \
  intermediate_ca.crt \
  --roots root_ca.crt
```

<p dir="rtl" align="right">
سپس فقط فایل <code>intermediate_ca.crt</code> را به Online CA بازگردانید.
</p>

<h2 dir="rtl" align="right">۱۴. نصب Intermediate Certificate روی Online CA</h2>

```bash
install -m 0644 \
  /root/intermediate_ca.crt \
  "$STEPPATH/certs/intermediate_ca.crt"

step certificate verify \
  "$STEPPATH/certs/intermediate_ca.crt" \
  --roots "$STEPPATH/certs/root_ca.crt"
```

<h2 dir="rtl" align="right">۱۵. فعال‌سازی ACME</h2>

```bash
step ca provisioner add internal-acme --type ACME
```

<p dir="rtl" align="right">ACME Directory:</p>

```text
https://ca.corp.example.com/acme/internal-acme/directory
```

<p dir="rtl" align="right">تنظیم Certificateهای ۹۰ روزه:</p>

```bash
step ca provisioner update \
  internal-acme \
  --x509-min-dur=5m \
  --x509-default-dur=2160h \
  --x509-max-dur=2160h
```

<h2 dir="rtl" align="right">۱۶. اجرای step-ca به‌صورت System Service</h2>

```bash
useradd \
  --user-group \
  --system \
  --create-home \
  --home /etc/step-ca \
  --shell /bin/false \
  step

OLD_STEPPATH="$(step path)"

cp -a "$OLD_STEPPATH/." /etc/step-ca/

install \
  -o step \
  -g step \
  -m 600 \
  /root/step-ca-password.txt \
  /etc/step-ca/password.txt

sed -i \
  "s#${OLD_STEPPATH}#/etc/step-ca#g" \
  /etc/step-ca/config/ca.json \
  /etc/step-ca/config/defaults.json

chown -R step:step /etc/step-ca
chmod 700 /etc/step-ca/secrets
chmod 600 /etc/step-ca/secrets/*
chmod 600 /etc/step-ca/password.txt
```

<p dir="rtl" align="right">اجازه Bind روی TCP/443:</p>

```bash
setcap CAP_NET_BIND_SERVICE=+eip "$(command -v step-ca)"
```

<p dir="rtl" align="right">ساخت Service:</p>

```bash
cat > /etc/systemd/system/step-ca.service <<'EOF'
[Unit]
Description=Private Certificate Authority
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=step
Group=step
Environment=STEPPATH=/etc/step-ca
WorkingDirectory=/etc/step-ca
ExecStart=/usr/bin/step-ca \
  /etc/step-ca/config/ca.json \
  --password-file /etc/step-ca/password.txt
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now step-ca
```

<p dir="rtl" align="right">بررسی:</p>

```bash
systemctl status step-ca
journalctl -u step-ca -f
```

<h2 dir="rtl" align="right">۱۷. تست CA و ACME</h2>

```bash
step ca health \
  --ca-url https://ca.corp.example.com \
  --root /etc/step-ca/certs/root_ca.crt
```

```bash
curl \
  --cacert /etc/step-ca/certs/root_ca.crt \
  https://ca.corp.example.com/acme/internal-acme/directory
```

<h2 dir="rtl" align="right">۱۸. توزیع Root CA روی Clientها</h2>

<p dir="rtl" align="right">
فقط Public Root Certificate باید روی Clientها Trust شود:
</p>

```text
root_ca.crt
```

<h3 dir="rtl" align="right">Ubuntu / Debian</h3>

```bash
sudo cp Example-Root-CA.crt \
  /usr/local/share/ca-certificates/Example-Root-CA.crt

sudo update-ca-certificates
```

<h3 dir="rtl" align="right">RHEL / Rocky / AlmaLinux / CentOS</h3>

```bash
sudo cp Example-Root-CA.crt \
  /etc/pki/ca-trust/source/anchors/Example-Root-CA.crt

sudo update-ca-trust extract
```

<h3 dir="rtl" align="right">Windows PowerShell</h3>

```powershell
Import-Certificate `
  -FilePath "C:\PKI\Example-Root-CA.crt" `
  -CertStoreLocation "Cert:\LocalMachine\Root"
```

<h3 dir="rtl" align="right">Windows Command Prompt</h3>

```cmd
certutil -addstore -f Root C:\PKI\Example-Root-CA.crt
```

<h3 dir="rtl" align="right">Windows Domain با Group Policy</h3>

```text
Computer Configuration
  -> Policies
  -> Windows Settings
  -> Security Settings
  -> Public Key Policies
  -> Trusted Root Certification Authorities
```

<p dir="rtl" align="right">
برای محیط Domain، GPO معمولاً بهترین روش توزیع Root CA است.
</p>

<h2 dir="rtl" align="right">۱۹. روش اول: Nginx + Certbot</h2>

<p dir="rtl" align="right">نصب:</p>

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

<p dir="rtl" align="right">فرض کنید Nginx دارای این Virtual Host است:</p>

```nginx
server {
    listen 80;
    server_name grafana.dev.corp.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
```

<p dir="rtl" align="right">دریافت Certificate:</p>

```bash
sudo env \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
  certbot --nginx \
  --server https://ca.corp.example.com/acme/internal-acme/directory
```

<h3 dir="rtl" align="right">ساخت Wrapper ساده برای تیم DevOps</h3>

```bash
sudo tee /usr/local/sbin/internal-certbot >/dev/null <<'EOF'
#!/bin/sh

export REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt

exec certbot \
  --server https://ca.corp.example.com/acme/internal-acme/directory \
  "$@"
EOF

sudo chmod 0755 /usr/local/sbin/internal-certbot
```

<p dir="rtl" align="right">از این به بعد:</p>

```bash
sudo internal-certbot --nginx
```

<h2 dir="rtl" align="right">۲۰. Certbot Standalone</h2>

```bash
sudo env \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
  certbot certonly \
  --standalone \
  -d app.dev.corp.example.com \
  --server https://ca.corp.example.com/acme/internal-acme/directory
```

<h2 dir="rtl" align="right">۲۱. Certbot Webroot</h2>

```bash
sudo env \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
  certbot certonly \
  --webroot \
  -w /var/www/html \
  -d app.dev.corp.example.com \
  --server https://ca.corp.example.com/acme/internal-acme/directory
```

<h2 dir="rtl" align="right">۲۲. Apache + Certbot</h2>

```bash
sudo apt install -y certbot python3-certbot-apache
```

```bash
sudo env \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
  certbot --apache \
  --server https://ca.corp.example.com/acme/internal-acme/directory
```

<h2 dir="rtl" align="right">۲۳. Renewal خودکار Certbot</h2>

<p dir="rtl" align="right">بررسی Timer موجود:</p>

```bash
systemctl list-timers | grep -i certbot
```

<p dir="rtl" align="right">تست Renewal:</p>

```bash
sudo env \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
  certbot renew --dry-run
```

<p dir="rtl" align="right">
اگر Certbot شما برای دسترسی به Private ACME CA به Environment Variable نیاز دارد، همان متغیر باید در systemd service/timer مربوط به Renewal نیز در دسترس باشد.
</p>

<h2 dir="rtl" align="right">۲۴. روش دوم: step CLI</h2>

<p dir="rtl" align="right">Bootstrap:</p>

```bash
step ca bootstrap \
  --ca-url https://ca.corp.example.com \
  --fingerprint <ROOT_CA_FINGERPRINT>
```

<p dir="rtl" align="right">دریافت Certificate:</p>

```bash
step ca certificate \
  app.dev.corp.example.com \
  app.crt \
  app.key
```

<p dir="rtl" align="right">Renew:</p>

```bash
step ca renew app.crt app.key
```

<h2 dir="rtl" align="right">۲۵. روش سوم: Windows + IIS + win-acme</h2>

<p dir="rtl" align="right">
برای IIS، یکی از روش‌های بسیار رایج استفاده از win-acme است.
ابتدا Root CA باید در Local Machine Trusted Root Store قرار گرفته باشد.
</p>

```cmd
wacs.exe --baseuri https://ca.corp.example.com/acme/internal-acme/directory
```

<p dir="rtl" align="right">فرایند معمول:</p>

```text
Detect IIS site/binding
        |
        v
ACME validation
        |
        v
Issue certificate
        |
        v
Install in Windows Certificate Store
        |
        v
Update IIS HTTPS binding
        |
        v
Scheduled renewal
```

<h2 dir="rtl" align="right">۲۶. روش چهارم: Windows Native CSR</h2>

<p dir="rtl" align="right">نمونه فایل:</p>

```text
C:\PKI\request.inf
```

```ini
[Version]
Signature="$Windows NT$"

[NewRequest]
Subject = "CN=app01.dev.corp.example.com"
KeyLength = 3072
Exportable = FALSE
MachineKeySet = TRUE
RequestType = PKCS10
KeyUsage = 0xa0

[Extensions]
2.5.29.17 = "{text}"
_continue_ = "dns=app01.dev.corp.example.com"
```

<p dir="rtl" align="right">ساخت CSR:</p>

```cmd
certreq -new C:\PKI\request.inf C:\PKI\app01.csr
```

<p dir="rtl" align="right">پس از دریافت Certificate:</p>

```cmd
certreq -accept C:\PKI\app01.cer
```

<h2 dir="rtl" align="right">۲۷. روش پنجم: OpenSSL</h2>

<p dir="rtl" align="right">
OpenSSL به‌تنهایی ACME Client نیست، اما برای ساخت Private Key، CSR، Inspect، Verify و تبدیل فرمت Certificate بسیار مفید است.
</p>

<h3 dir="rtl" align="right">ساخت Private Key و CSR</h3>

```bash
openssl req \
  -new \
  -newkey rsa:3072 \
  -nodes \
  -keyout app.key \
  -out app.csr \
  -subj "/CN=app.dev.corp.example.com" \
  -addext "subjectAltName=DNS:app.dev.corp.example.com"
```

<h3 dir="rtl" align="right">Sign کردن CSR با step-ca</h3>

```bash
TOKEN="$(step ca token app.dev.corp.example.com)"

step ca sign \
  --token "$TOKEN" \
  app.csr \
  app.crt
```

<h3 dir="rtl" align="right">مشاهده Certificate</h3>

```bash
openssl x509 \
  -in app.crt \
  -text \
  -noout
```

<h3 dir="rtl" align="right">بررسی SAN</h3>

```bash
openssl x509 \
  -in app.crt \
  -noout \
  -ext subjectAltName
```

<h3 dir="rtl" align="right">بررسی تاریخ اعتبار</h3>

```bash
openssl x509 \
  -in app.crt \
  -noout \
  -dates
```

<h3 dir="rtl" align="right">Verify Chain</h3>

```bash
openssl verify \
  -CAfile Example-Root-CA.crt \
  app.crt
```

<h3 dir="rtl" align="right">تست TLS Server</h3>

```bash
openssl s_client \
  -connect app.dev.corp.example.com:443 \
  -servername app.dev.corp.example.com \
  -showcerts
```

<h2 dir="rtl" align="right">۲۸. تبدیل PEM به PFX برای Windows</h2>

```bash
openssl pkcs12 \
  -export \
  -out app.pfx \
  -inkey app.key \
  -in app.crt \
  -certfile intermediate_ca.crt
```

<p dir="rtl" align="right">Import در Windows:</p>

```powershell
Import-PfxCertificate `
  -FilePath "C:\PKI\app.pfx" `
  -CertStoreLocation "Cert:\LocalMachine\My"
```

<h2 dir="rtl" align="right">۲۹. روش ششم: Kubernetes + cert-manager</h2>

<p dir="rtl" align="right">
برای Kubernetes، cert-manager معمولاً بهترین انتخاب است.
</p>

<p dir="rtl" align="right">نمونه ClusterIssuer:</p>

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-acme
spec:
  acme:
    email: pki-admin@example.com
    server: https://ca.corp.example.com/acme/internal-acme/directory

    # Add the private CA root bundle using the method supported
    # by the cert-manager version deployed in your cluster.

    privateKeySecretRef:
      name: internal-acme-account

    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

<p dir="rtl" align="right">نمونه Certificate:</p>

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: application-tls
  namespace: application
spec:
  secretName: application-tls
  dnsNames:
    - app.dev.corp.example.com
  issuerRef:
    name: internal-acme
    kind: ClusterIssuer
```

<h2 dir="rtl" align="right">۳۰. روش هفتم: Caddy و Traefik</h2>

<p dir="rtl" align="right">
Caddy و Traefik هر دو ACME Client داخلی دارند و می‌توانند مستقیماً از Private ACME Server Certificate بگیرند.
</p>

<p dir="rtl" align="right">مفهوم اصلی در هر دو محصول:</p>

```text
ACME CA URL:
https://ca.corp.example.com/acme/internal-acme/directory
```

<p dir="rtl" align="right">
Root CA داخلی نیز باید برای Process یا Container مربوطه Trusted باشد.
</p>

<h2 dir="rtl" align="right">۳۱. Wildcard Certificate</h2>

<p dir="rtl" align="right">برای Certificateهایی مانند:</p>

```text
*.dev.corp.example.com
```

<p dir="rtl" align="right">
از DNS-01 Challenge استفاده کنید. برای سرویس‌های عادی، صدور Certificate اختصاصی برای هر سرویس معمولاً انتخاب بهتری است.
</p>

<h2 dir="rtl" align="right">۳۲. روش پیشنهادی بر اساس محیط</h2>

| Environment | Recommended Method |
|---|---|
| Nginx on Linux | Certbot |
| Apache on Linux | Certbot |
| Linux without web server | Certbot Standalone |
| Minimal Linux | `step-cli` or an ACME client |
| Windows IIS | win-acme |
| Windows manual request | `certreq` |
| Kubernetes | cert-manager |
| Caddy | Native ACME |
| Traefik | Native ACME |
| Legacy applications | OpenSSL CSR + CA signing |
| Windows PFX requirement | OpenSSL PKCS#12 |

<h2 dir="rtl" align="right">۳۳. نکات امنیتی مهم</h2>

<ul dir="rtl" align="right">
<li>Root Private Key باید آفلاین باشد.</li>
<li>Root Private Key هرگز در Git ذخیره نشود.</li>
<li>Intermediate Private Key فقط روی Online CA نگهداری شود.</li>
<li>Root Certificate عمومی است و می‌تواند روی Clientها توزیع شود.</li>
<li>Private Key هر سرویس تا جای ممکن روی همان سرویس تولید شود.</li>
<li>برای CA Server از NTP مطمئن استفاده شود.</li>
<li>صدور Certificate فقط برای Namespaceهای مجاز محدود شود.</li>
<li>Backupهای CA رمزنگاری و دوره‌ای Restore-Test شوند.</li>
<li>دسترسی مدیریتی Online CA محدود و Audit شود.</li>
</ul>

<h2 dir="rtl" align="right">۳۴. Monitoring</h2>

<p dir="rtl" align="right">حداقل موارد زیر Monitor شوند:</p>

```text
step-ca service status
TCP/443 availability
ACME directory availability
Disk usage
NTP synchronization
Certificate expiration
Renewal failures
CA backup status
```

<h2 dir="rtl" align="right">۳۵. Backup Online CA</h2>

<p dir="rtl" align="right">حداقل مسیرهای زیر را در Backup Policy قرار دهید:</p>

```text
/etc/step-ca/config/
/etc/step-ca/certs/
/etc/step-ca/secrets/
/etc/step-ca/db/
/etc/step-ca/password.txt
```

<p dir="rtl" align="right">
Backup حاوی Private Key و Password باید رمزنگاری شده و دسترسی به آن محدود باشد.
</p>

<h2 dir="rtl" align="right">۳۶. Acceptance Criteria</h2>

```text
[ ] Root CA is offline
[ ] Root private key is not present on the online CA
[ ] Intermediate certificate is signed by the offline Root CA
[ ] CA DNS name resolves internally
[ ] step-ca is available on TCP/443
[ ] ACME directory is reachable
[ ] Linux clients trust the internal Root CA
[ ] Windows clients trust the internal Root CA
[ ] Certbot can issue and renew certificates
[ ] win-acme can issue and renew IIS certificates
[ ] Kubernetes can issue certificates through cert-manager
[ ] Private keys are excluded from source-control repositories
[ ] Backup and restore procedures are tested
```

<h2 dir="rtl" align="right">۳۷. نکات قبل از انتشار این فایل در GitHub</h2>

<p dir="rtl" align="right">
قبل از Commit نهایی، مطمئن شوید هیچ‌کدام از موارد زیر وارد Repository نشده باشند:
</p>

```text
Real organization names
Real internal domains
Real public or private IP mappings
Real usernames or administrator emails
Root/Intermediate private keys
Certificate passwords
ACME account credentials
DNS API tokens
Production hostnames
VPN or firewall information
Backup locations
```

<p dir="rtl" align="right">
برای نمونه‌ها از دامنه‌های رزروشده مانند <code>example.com</code> و Placeholderهایی مانند <code>&lt;CA_SERVER_IP&gt;</code> استفاده کنید.
</p>

<h2 dir="rtl" align="right">۳۸. خلاصه معماری</h2>

```text
Offline Root CA
      |
      v
Online Intermediate / Issuing CA
      |
    step-ca
      |
     ACME
      |
 +----+----------+-------------+
 |               |             |
Certbot       win-acme     cert-manager
Linux         Windows      Kubernetes
```

<p dir="rtl" align="right">
این طراحی یک Private PKI قابل اتوماسیون ایجاد می‌کند و تجربه صدور Certificate را برای سرویس‌های داخلی بسیار نزدیک به Let's Encrypt می‌کند، بدون اینکه Root Private Key در شبکه آنلاین قرار گیرد.
</p>
