# FreePBX 14.0.16.9 + Asterisk 13.20.0 — Installation Runbook (CentOS 7.9)

## 1. Reference installation and prerequisites

| Component | Pinned version |
|---|---|
| OS | CentOS 7.9 x86_64 (minimal) |
| Asterisk | **13.20.0**, source installation |
| FreePBX | **14.0.16.9**, version-pinned installation |
| PHP | **5.6.40** (Webtatic php56w) |
| MariaDB | 5.5.x (tested: 5.5.68) |
| Apache | 2.4.x (tested: 2.4.6) |
| Node.js | **8.17.0** |

**VM requirements:** 2 vCPU, 4 GB RAM, >= 30 GB disk, CentOS 7.9 x86_64 ISO, a console, root privileges, working host resolution and reachable installation repositories or a complete local/offline package repository. SIP-only tests do not require DAHDI hardware.

All commands assume a **fresh CentOS installation** and a root shell. Package repositories for CentOS 7 and PHP 5.6 must be supplied through compatible archived or locally mirrored sources; the RPM examples below assume repository names `base`, `updates`, `extras`, and `webtatic` have already been configured. Source archives must be fetched on a connected staging machine and transferred to `/usr/src` when installing offline. Never substitute a `latest` archive when an exact version is required.

```bash
cat /etc/centos-release
uname -m
hostnamectl set-hostname pbx-lab.example.test
timedatectl set-timezone UTC
yum repolist
mkdir -p /usr/src /root/freepbx-packages
```

## 2. Install operating-system packages

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

Additional `-devel` libraries are needed only for the Asterisk modules selected in `menuselect`. Install or stage missing packages reported by `./configure`; do not silently disable required `chan_sip`, `chan_pjsip`, or `app_macro` functionality.

## 3. Install PHP 5.6.40 and dependencies

Set up a compatible Webtatic EL7 repository (or equivalent local mirror). For an offline installation, transfer `webtatic-release-7-3.noarch.rpm`, all required PHP RPMs and their resolved dependencies to the local RPM repository. Do not mix CentOS PHP packages with `php56w`.

```bash
# When the release RPM is available locally:
yum localinstall -y /root/freepbx-packages/webtatic-release.rpm

# With a working Webtatic archive/local repository:
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

Adjust `/etc/php.ini`:

```ini
memory_limit = 256M
upload_max_filesize = 120M
post_max_size = 120M
date.timezone = UTC
```

The file above is a configuration **excerpt**, not a command to overwrite all of `/etc/php.ini`. To install PEAR console prerequisites using the local/available PEAR channel:

```bash
pear list | grep Console_Getopt || pear install Console_Getopt
```

## 4. Install Node.js 8.17.0

Install the **exact 8.17.0** binary package from a pre-staged Node.js archive compatible with the system, or from a pinned local RPM. Example using a staged upstream Linux x64 archive:

```bash
cd /usr/local
# Stage /usr/src/node-v8.17.0-linux-x64.tar.xz first.
tar -xJf /usr/src/node-v8.17.0-linux-x64.tar.xz
ln -sfn /usr/local/node-v8.17.0-linux-x64 /usr/local/node8
ln -sfn /usr/local/node8/bin/node /usr/local/bin/node
ln -sfn /usr/local/node8/bin/npm /usr/local/bin/npm
node -v
npm -v
```

`node -v` must output `v8.17.0` before installing the FreePBX UI modules.

## 5. Build Asterisk 13.20.0 from source

Stage the official **`asterisk-13.20.0.tar.gz`** archive in `/usr/src`. Stage any source dependency archives as required. For `chan_pjsip`, ensure a compatible PJPROJECT library is available; use the bundled option only if the checked-out Asterisk 13.20.0 configure script supports it. The following checks stop an incompatible build rather than accidentally dropping PJSIP.

```bash
cd /usr/src
test -f asterisk-13.20.0.tar.gz
tar -xzf asterisk-13.20.0.tar.gz
cd /usr/src/asterisk-13.20.0

./configure --libdir=/usr/lib64
make menuselect
```

In `make menuselect`, verify `chan_sip`, `chan_pjsip` and `app_macro` are enabled and dependencies are satisfied. If PJSIP is unavailable, build/install a compatible PIC-enabled PJPROJECT first, then rerun `./configure` and `make menuselect`.

```bash
make -j2
make install
make samples
ldconfig
asterisk -V
```

Expected: `Asterisk 13.20.0`. Do not run `make samples` over an already configured PBX.

Configure `/etc/asterisk/asterisk.conf` with these values under their **existing** sections:

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

## 6. Prepare Apache and database

In `/etc/httpd/conf/httpd.conf`, set the main Apache worker `User` and `Group` directives to `asterisk` and enable `.htaccess` overrides for the FreePBX document root. Verify the final `<Directory>` scope with `httpd -t`.

```bash
sed -i -E 's/^(User|Group) .*/\1 asterisk/' /etc/httpd/conf/httpd.conf
sed -i 's/AllowOverride None/AllowOverride All/g' /etc/httpd/conf/httpd.conf
httpd -t
systemctl restart httpd
ps -eo user,comm | grep httpd | head
```

Initialize MariaDB and set its root authentication using the standard `mysql_secure_installation` workflow. The FreePBX installer will use local database administrator access to create its schema and application credentials; keep those credentials out of documents.

```bash
systemctl enable mariadb
systemctl start mariadb
mysql_secure_installation
mysql -e 'SELECT VERSION();'
```

## 7. Install the exact FreePBX 14.0.16.9 release

Stage a verified FreePBX **14.0.16.9** installation artifact in `/usr/src` and rename that local file to `freepbx-14.0.16.9.tgz` if it has a different release-specific filename. **Do not use `freepbx-14.0-latest.tgz`** for version reproduction. Confirm the package includes `start_asterisk` and `install` after extraction.

```bash
cd /usr/src
test -f freepbx-14.0.16.9.tgz
mkdir -p /usr/src/freepbx-install
tar -xzf freepbx-14.0.16.9.tgz -C /usr/src/freepbx-install
find /usr/src/freepbx-install -maxdepth 3 -name start_asterisk -o -name install
```

Set `FREEPBX_SRC` to the directory containing both files (typically `/usr/src/freepbx-install/freepbx`):

```bash
FREEPBX_SRC=/usr/src/freepbx-install/freepbx
test -x "$FREEPBX_SRC/start_asterisk"
test -x "$FREEPBX_SRC/install"
cd "$FREEPBX_SRC"
./start_asterisk start
./install -n
fwconsole -V
```

Confirm the **reported installed FreePBX version** is `14.0.16.9`; an archive labeled “14.0” may contain a different patch release. Complete the initial admin-account setup via `http://<PBX-IP>/admin/`, then:

```bash
fwconsole chown
fwconsole reload
asterisk -rx 'module show like chan_sip'
asterisk -rx 'module show like chan_pjsip'
asterisk -rx 'module show like app_macro'
```

Install any additional **version-compatible, locally staged** FreePBX modules through Admin > Module Admin or `fwconsole ma install <module>`; inspect `fwconsole ma list` after installation. Do not automatically fetch latest module versions.

## 8. FreePBX service and boot startup

Create `/etc/systemd/system/freepbx.service`:

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

A `oneshot` unit can show `active (exited)` while Asterisk keeps running.

## 9. AMI configuration and access tests

Additional AMI accounts belong in `/etc/asterisk/manager_custom.conf`. Enable AMI in the manager's primary configuration using FreePBX-managed settings, then verify `5038/tcp` is listening. Example account definitions (replace IP ranges and secrets):

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

For AMI client connections, `Events: off` is useful when testing action responses independently of asynchronous events. Verify the read-only account is rejected for state-changing actions.

## 10. SIP, endpoints and softphone validation

Create two **Chan_SIP** extensions, for example `2001` and `2002`, in Applications > Extensions > Add Extension. Configure unique SIP secrets and save/apply. Typical lab mapping: PJSIP UDP `5060`, Chan_SIP UDP `5160` (read the actual FreePBX SIP Settings to confirm).

For MicroSIP, use `SIP Server=<PBX-IP>:5160`, `Domain=<PBX-IP>`, `Username=2001`, `Login=2001`, and the extension's SIP secret. Register a second client with `2002` and call between the two.

```bash
asterisk -rx 'sip show peers'
asterisk -rx 'sip show peer 2001'
asterisk -rx 'pjsip show endpoints'
ss -lunp | grep -E ':(5060|5160)\b'
asterisk -rx 'core show channels'
```

Verify successful registration, two-way audio, DTMF and hangup. Check the configured RTP port range in `/etc/asterisk/rtp.conf` when audio is absent.

## 11. Logs, verification and diagnosis

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

An HTTP 200 response does not guarantee that FreePBX rendered correctly: open the UI and confirm the dashboard. For permissions warnings, check the reported path, process owner and filesystem permissions before retrying. For SIP authentication errors, verify extension type, listener port, username, domain and secret. After a reboot, repeat all service and call tests.

## 12. Deployment security

Production deployments require properly configured and enabled **SELinux** and **Firewall**, plus appropriate network segmentation, secret management and access restrictions. These controls are outside the installation procedure in this runbook.
