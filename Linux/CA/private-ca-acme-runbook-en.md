# Private Internal CA and ACME Runbook with Smallstep

> **Public repository edition**
>
> This runbook is intentionally sanitized for publishing on GitHub. It contains no real organization names, internal domains, IP addresses, administrator accounts, or production identifiers. Replace all example values with those from your own environment.

## 1. Goal

Build an internal Certificate Authority that can issue TLS certificates for internal services and automate issuance and renewal through ACME, providing an operational experience similar to Let's Encrypt.

Example names used throughout this document:

```text
Internal DNS domain: corp.example.com
CA hostname:         ca.corp.example.com
Example service:     grafana.dev.corp.example.com
```

`example.com` is reserved for documentation. Replace it with a domain or subdomain that you control.

## 2. Recommended Architecture

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

The Root CA should remain offline and should not directly issue day-to-day service certificates. An online Intermediate/Issuing CA performs normal certificate issuance.

## 3. Suggested Server Resources

### Offline Root CA

| Item | Recommendation |
|---|---|
| OS | Ubuntu Server 24.04 LTS Minimal |
| CPU | 1 vCPU |
| RAM | 2 GB |
| Disk | 10 GB |
| Network | Disconnected after setup |
| Software | `step-cli` |
| Backup | At least two encrypted offline copies |

### Online Issuing CA

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

## 4. DNS and Firewall Prerequisites

Create an internal DNS record for the CA:

```text
ca.corp.example.com    A    <CA_SERVER_IP>
```

Typical network flows:

```text
Clients -> CA Server        TCP/443

For ACME HTTP-01:
CA Server -> Web Server     TCP/80

For ACME TLS-ALPN-01:
CA Server -> Target Server  TCP/443
```

Wildcard certificates normally use DNS-01, which requires the ACME client to create the required TXT record in DNS.

## 5. Suggested Certificate Lifetimes

| Certificate | Suggested Lifetime |
|---|---:|
| Root CA | 10 years |
| Intermediate CA | 5 years |
| Service/Leaf Certificate | 90 days |

## 6. Build the Offline Root CA

Install `step-cli` on the offline Root CA machine:

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

Create the Root CA:

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

Inspect and record the fingerprint:

```bash
step certificate inspect root_ca.crt
step certificate fingerprint root_ca.crt
```

`root_ca.key` is highly sensitive. Never place it on the online CA, in Git, in an Ansible repository, or on normal application servers.

## 7. Back Up the Root CA

Keep at least the following offline:

```text
root_ca.crt
root_ca.key
Root CA private-key password
Root CA fingerprint
```

Prefer storing the private-key password separately from the encrypted backup that contains the key.

## 8. Install the Online CA Server

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

## 9. Initialize `step-ca`

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

This initialization creates temporary Root and Intermediate CA material. In this design, the real Root CA is the offline Root created earlier, so the temporary CA key material is replaced.

## 10. Remove Temporary CA Material from the Online Server

```bash
shred -u "$STEPPATH/secrets/root_ca_key"
shred -u "$STEPPATH/secrets/intermediate_ca_key"

rm -f \
  "$STEPPATH/certs/root_ca.crt" \
  "$STEPPATH/certs/intermediate_ca.crt"
```

## 11. Copy Only the Public Root Certificate

Transfer only `root_ca.crt` to the online CA. Do **not** transfer the Root private key.

```bash
install -m 0644 \
  /root/root_ca.crt \
  "$STEPPATH/certs/root_ca.crt"
```

## 12. Generate the Intermediate CSR on the Online CA

```bash
step certificate create \
  "Example Internal Issuing CA" \
  /root/intermediate_ca.csr \
  "$STEPPATH/secrets/intermediate_ca_key" \
  --csr \
  --password-file /root/step-ca-password.txt
```

The Intermediate private key remains on the online CA. Transfer only the CSR to the offline Root CA.

## 13. Sign the Intermediate on the Offline Root CA

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

Verify it:

```bash
step certificate verify \
  intermediate_ca.crt \
  --roots root_ca.crt
```

Transfer only `intermediate_ca.crt` back to the online CA.

## 14. Install the Intermediate Certificate on the Online CA

```bash
install -m 0644 \
  /root/intermediate_ca.crt \
  "$STEPPATH/certs/intermediate_ca.crt"

step certificate verify \
  "$STEPPATH/certs/intermediate_ca.crt" \
  --roots "$STEPPATH/certs/root_ca.crt"
```

## 15. Enable ACME

Create an ACME provisioner:

```bash
step ca provisioner add internal-acme --type ACME
```

ACME directory:

```text
https://ca.corp.example.com/acme/internal-acme/directory
```

Set 90-day service certificates:

```bash
step ca provisioner update \
  internal-acme \
  --x509-min-dur=5m \
  --x509-default-dur=2160h \
  --x509-max-dur=2160h
```

## 16. Run `step-ca` as a System Service

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

Allow the unprivileged process to bind to TCP/443:

```bash
setcap CAP_NET_BIND_SERVICE=+eip "$(command -v step-ca)"
```

Create the systemd service:

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

Check status and logs:

```bash
systemctl status step-ca
journalctl -u step-ca -f
```

## 17. Test the CA and ACME Directory

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

## 18. Distribute the Root CA Certificate

Only the public Root certificate is trusted by clients:

```text
root_ca.crt
```

### Ubuntu / Debian

```bash
sudo cp Example-Root-CA.crt \
  /usr/local/share/ca-certificates/Example-Root-CA.crt

sudo update-ca-certificates
```

### RHEL / Rocky / AlmaLinux / CentOS

```bash
sudo cp Example-Root-CA.crt \
  /etc/pki/ca-trust/source/anchors/Example-Root-CA.crt

sudo update-ca-trust extract
```

### Windows PowerShell

```powershell
Import-Certificate `
  -FilePath "C:\PKI\Example-Root-CA.crt" `
  -CertStoreLocation "Cert:\LocalMachine\Root"
```

### Windows Command Prompt

```cmd
certutil -addstore -f Root C:\PKI\Example-Root-CA.crt
```

### Windows Domain via Group Policy

```text
Computer Configuration
  -> Policies
  -> Windows Settings
  -> Security Settings
  -> Public Key Policies
  -> Trusted Root Certification Authorities
```

For an Active Directory environment, Group Policy is usually the preferred Root CA distribution method.

## 19. Method 1: Nginx + Certbot

Install Certbot and the Nginx plugin:

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

Example Nginx virtual host:

```nginx
server {
    listen 80;
    server_name grafana.dev.corp.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
```

Request a certificate:

```bash
sudo env \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
  certbot --nginx \
  --server https://ca.corp.example.com/acme/internal-acme/directory
```

### Optional DevOps Wrapper

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

Usage:

```bash
sudo internal-certbot --nginx
```

## 20. Certbot Standalone

```bash
sudo env \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
  certbot certonly \
  --standalone \
  -d app.dev.corp.example.com \
  --server https://ca.corp.example.com/acme/internal-acme/directory
```

## 21. Certbot Webroot

```bash
sudo env \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
  certbot certonly \
  --webroot \
  -w /var/www/html \
  -d app.dev.corp.example.com \
  --server https://ca.corp.example.com/acme/internal-acme/directory
```

## 22. Apache + Certbot

```bash
sudo apt install -y certbot python3-certbot-apache
```

```bash
sudo env \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
  certbot --apache \
  --server https://ca.corp.example.com/acme/internal-acme/directory
```

## 23. Automatic Certbot Renewal

Inspect the existing Certbot timer:

```bash
systemctl list-timers | grep -i certbot
```

Test renewal:

```bash
sudo env \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
  certbot renew --dry-run
```

If your Certbot build requires `REQUESTS_CA_BUNDLE` to reach the private ACME server, ensure the same environment variable is available to the systemd renewal service/timer.

## 24. Method 2: `step` CLI

Bootstrap the client:

```bash
step ca bootstrap \
  --ca-url https://ca.corp.example.com \
  --fingerprint <ROOT_CA_FINGERPRINT>
```

Request a certificate:

```bash
step ca certificate \
  app.dev.corp.example.com \
  app.crt \
  app.key
```

Renew it:

```bash
step ca renew app.crt app.key
```

## 25. Method 3: Windows + IIS + win-acme

For IIS, `win-acme` is a common ACME client.

First ensure the internal Root CA is trusted in the Local Machine certificate store.

```cmd
wacs.exe --baseuri https://ca.corp.example.com/acme/internal-acme/directory
```

Typical flow:

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

## 26. Method 4: Native Windows CSR

Example request file:

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

Generate the CSR:

```cmd
certreq -new C:\PKI\request.inf C:\PKI\app01.csr
```

After receiving the signed certificate:

```cmd
certreq -accept C:\PKI\app01.cer
```

## 27. Method 5: OpenSSL

OpenSSL is not an ACME client by itself, but it is very useful for generating private keys and CSRs, inspecting certificates, verifying chains, and converting formats.

### Generate a Private Key and CSR

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

### Sign an Existing CSR with `step-ca`

```bash
TOKEN="$(step ca token app.dev.corp.example.com)"

step ca sign \
  --token "$TOKEN" \
  app.csr \
  app.crt
```

### Inspect the Certificate

```bash
openssl x509 \
  -in app.crt \
  -text \
  -noout
```

### Inspect SANs

```bash
openssl x509 \
  -in app.crt \
  -noout \
  -ext subjectAltName
```

### Check Validity

```bash
openssl x509 \
  -in app.crt \
  -noout \
  -dates
```

### Verify the Chain

```bash
openssl verify \
  -CAfile Example-Root-CA.crt \
  app.crt
```

### Test a TLS Server

```bash
openssl s_client \
  -connect app.dev.corp.example.com:443 \
  -servername app.dev.corp.example.com \
  -showcerts
```

## 28. Convert PEM to PFX for Windows

```bash
openssl pkcs12 \
  -export \
  -out app.pfx \
  -inkey app.key \
  -in app.crt \
  -certfile intermediate_ca.crt
```

Import the PFX:

```powershell
Import-PfxCertificate `
  -FilePath "C:\PKI\app.pfx" `
  -CertStoreLocation "Cert:\LocalMachine\My"
```

## 29. Method 6: Kubernetes + cert-manager

For Kubernetes, `cert-manager` is normally the preferred certificate automation component.

Example `ClusterIssuer`:

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

Example Certificate:

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

## 30. Method 7: Caddy and Traefik

Both Caddy and Traefik have native ACME support and can use a private ACME directory.

The important value is:

```text
https://ca.corp.example.com/acme/internal-acme/directory
```

The private Root CA must also be trusted by the corresponding process or container.

## 31. Wildcard Certificates

For names such as:

```text
*.dev.corp.example.com
```

use DNS-01 validation. For ordinary services, individual certificates are usually preferable to one large wildcard certificate.

## 32. Recommended Method by Environment

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

## 33. Security Guidelines

- Keep the Root private key offline.
- Never store the Root private key in Git.
- Keep the Intermediate private key only on the online CA.
- The Root certificate is public and should be distributed to clients.
- Generate service private keys on the service host whenever possible.
- Use reliable NTP on the CA.
- Restrict issuance to approved DNS namespaces.
- Encrypt CA backups and test restoration procedures.
- Limit and audit administrative access to the online CA.
- Protect DNS API credentials if DNS-01 is used.

## 34. Monitoring

At minimum, monitor:

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

## 35. Online CA Backup

Include at least:

```text
/etc/step-ca/config/
/etc/step-ca/certs/
/etc/step-ca/secrets/
/etc/step-ca/db/
/etc/step-ca/password.txt
```

Backups containing private keys or passwords must be encrypted and access-controlled.

## 36. Acceptance Criteria

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

## 37. Public Repository Sanitization Checklist

Before committing this runbook or derived configuration to a public repository, confirm that none of the following are present:

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

Use reserved documentation domains such as `example.com` and placeholders such as `<CA_SERVER_IP>` in public examples.

## 38. Final Architecture

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

This design provides an automatable private PKI with a certificate issuance workflow that is very similar to Let's Encrypt, while keeping the Root private key out of the online network.
