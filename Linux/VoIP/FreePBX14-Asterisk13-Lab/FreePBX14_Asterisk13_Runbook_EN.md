# Lab Runbook — FreePBX 14 and Asterisk 13

**Document version:** 1.2  
**Scope:** Offline, isolated virtual-machine lab  
## 1. Purpose and reference environment

Set up and validate a PBX laboratory with the specified software versions, web access, AMI, SIP registration, and internal-call testing.

| Component | Version / setting |
|---|---|
| OS | CentOS 7.9, kernel `3.10.0-1160.el7.x86_64` |
| Example hostname | `pbx-lab.example.test` |
| Example IP | `192.0.2.10/24` |
| Asterisk | **Exactly `13.20.0`**, source-built |
| FreePBX | **Exactly `14.0.16.9`** |
| Apache | `httpd 2.4.6` |
| PHP | `5.6.40` |
| MariaDB | `5.5.68` |
| Node.js | `8.17.0` |
| SIP drivers | `chan_pjsip` UDP `5060`; `chan_sip` UDP `5160` |
| AMI | TCP `5038` |

## 2. Prerequisites

- Isolated VM, administrative console access, stable storage and internal IP networking; take a snapshot.
- Obtain and verify offline installation sources for the **exact** releases: Asterisk `13.20.0` source, FreePBX `14.0.16.9`, development dependencies, PHP 5.6, Apache, MariaDB and Node.js.
- Verify hostname, host resolution, clocks and time zone.
- Use an `asterisk` OS account for the running PBX processes and appropriate file ownership.

```bash
hostnamectl
ip -4 addr
uname -r
id asterisk
rpm -q httpd mariadb-server php
php -v
node -v
```

Paths:

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

## 3. Base installation: implementation checkpoints

Installation sequence:

1. Provision CentOS 7.9 and pinned, offline dependencies.
2. Build and install Asterisk **exactly `13.20.0`** from its corresponding source; confirm `chan_sip`, `chan_pjsip`, and `app_macro` are included.
3. Install MariaDB, Apache and PHP 5.6. Set up the database and application credentials using the version-specific FreePBX installer procedure.
4. Install FreePBX **exactly `14.0.16.9`** from its matching installer. Apache workers run under the `asterisk` user.
5. Check PHP settings: `memory_limit=256M` and a configured time zone in `/etc/php.ini`.
6. Verify `fwconsole` and web access.

```bash
asterisk -V
asterisk -rx 'module show like chan_sip'
asterisk -rx 'module show like chan_pjsip'
asterisk -rx 'module show like app_macro'
php -r 'echo ini_get("memory_limit"), PHP_EOL, date_default_timezone_get(), PHP_EOL;'
```

## 4. FreePBX systemd service

Create the following `oneshot` service unit:

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

After creating the unit:

```bash
systemctl daemon-reload
systemctl enable freepbx
systemctl start mariadb httpd
systemctl start freepbx
systemctl is-active mariadb httpd freepbx
asterisk -rx 'core show uptime'
```

`active (exited)` is expected for this `oneshot` unit. Confirm that the Asterisk process is running separately.

## 5. Web UI and common initial issues

Web URL (substitute the target server address):

```text
http://192.0.2.10/admin/config.php
```

```bash
systemctl is-active httpd mariadb freepbx
curl -sS -L -o /dev/null -w 'HTTP=%{http_code}\n' http://127.0.0.1/admin/config.php
```

**HTTP 200 is not sufficient proof of success:** FreePBX exceptions can be rendered with status 200. Visually verify that the dashboard works. Verify the FreePBX dashboard and underlying services. Mail Queue status depends on mail configuration.

### Cache-path alert

If `Cache path is not writable!` appears, check ownership and write access on `/var/spool/asterisk/cache`. Once the underlying problem is resolved, remove a stale notification if needed:

```bash
fwconsole notification --list
fwconsole notification --delete framework CACHEPATCH
```

FreePBX modules also maintain per-module `assets/less/cache` directories; a LESS exception identifies the path that needs inspection.

```bash
ls -ld /var/spool/asterisk/cache
ls -ld /var/www/html/admin/modules/dashboard/assets/less/cache
namei -l /var/spool/asterisk/cache
fwconsole notification --list
```

### GPG / signature validation

If GPG signature validation fails due to access to `/home/asterisk/.gnupg`, verify `asterisk:asterisk` ownership and the following permissions:

```bash
chmod 700 /home/asterisk/.gnupg
find /home/asterisk/.gnupg -type d -exec chmod 700 {} \;
find /home/asterisk/.gnupg -type f -exec chmod 600 {} \;
```

The GPG keyring is stored in the application user home directory.

## 6. AMI configuration

Asterisk Manager Interface commonly uses TCP `5038`. Check it with:

```bash
ss -lntp | grep ':5038'
asterisk -rx 'manager show users'
```

Place additional accounts in **`/etc/asterisk/manager_custom.conf`** so normal FreePBX configuration regeneration does not overwrite them. Account definitions:

- `monitor_ro`: monitoring/event access without AMI write permission.
- `control_rw`: AMI read/write access.

AMI ACLs are configured through `permit` and `deny` entries.

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

AMI read/write classes govern event access and actions. Authorization is verified using permitted and denied AMI actions.

```bash
cp -a /etc/asterisk/manager_custom.conf /root/manager_custom.conf.bak
# Securely edit the file, e.g. vi /etc/asterisk/manager_custom.conf
asterisk -rx 'manager reload'
asterisk -rx 'manager show users'
asterisk -rx 'manager show user monitor_ro'
asterisk -rx 'manager show user control_rw'
```

Check the AMI listener address and port in `manager.conf`.

## 7. SIP and extension configuration

Check SIP channel drivers and listener ports:

```bash
asterisk -rx 'module show like chan_sip'
asterisk -rx 'module show like chan_pjsip'
ss -lunp | grep -E ':(5060|5160)\b'
grep -E '^[[:space:]]*(rtpstart|rtpend)[[:space:]]*=' /etc/asterisk/rtp.conf
```

In **Applications → Extensions**, create test extensions such as `2001` and `2002`; set/check a distinct SIP secret, submit, and select **Apply Config**. Choose the extension driver before configuring the softphone.

In this configuration, **Chan_SIP uses UDP 5160** and **PJSIP uses UDP 5060**. The secret field location in FreePBX 14 can vary by extension type and form.

## 8. MicroSIP registration example

Example MicroSIP (open-source, Windows) settings for **Chan_SIP extension 2001**:

| MicroSIP field | Value |
|---|---|
| Account Name | `2001` |
| SIP Server | `192.0.2.10:5160` |
| SIP Proxy | Empty |
| Username | `2001` |
| Domain | `192.0.2.10` |
| Login | `2001` |
| Password | Extension 2001 SIP secret |
| Display Name | `2001` |
| Transport | `UDP` |
| Media Encryption | `Disabled` for basic lab test |

Set `Domain` to the SIP host address only, without the extension prefix.

```bash
asterisk -rx 'sip show peer 2001'
asterisk -rx 'sip show peers'
```

Verify the registered softphone address, not just the app's `Online` indicator. A displayed `OK` status depends on Qualify configuration. For `Offline` status, inspect MicroSIP logs and Asterisk output.

## 9. Acceptance test matrix

| ID | Test | Expected outcome |
|---|---|---|
| T01 | Versions | Asterisk exactly `13.20.0`, FreePBX exactly `14.0.16.9` |
| T02 | Boot | `httpd`, `mariadb`, `freepbx` active; Asterisk responsive |
| T03 | UI | Dashboard displays without an exception |
| T04 | SIP | Both channel drivers and assigned UDP listeners active |
| T05 | Registration | Chan_SIP `2001` registers successfully |
| T06 | Two extensions | `2001` and `2002` ring; two-way media works |
| T07 | AMI | Both roles connect; reader passes negative privilege tests |
| T08 | Reboot | UI, registration, AMI and calls continue to function |

Record each validation result in the target installation; this checklist does not imply that every test has already passed.

## 10. Troubleshooting reference

```bash
# Health
asterisk -V
asterisk -rx 'core show uptime'
systemctl is-active httpd mariadb freepbx

# SIP
asterisk -rx 'sip show peers'
asterisk -rx 'sip show peer 2001'
asterisk -rx 'pjsip show endpoints'
ss -lunp | grep -E ':(5060|5160)\b'

# AMI
asterisk -rx 'manager show users'
ss -lntp | grep ':5038'

# Logs and notifications
tail -n 50 /var/log/asterisk/freepbx.log
tail -n 50 /var/log/httpd/error_log
fwconsole notification --list
```

- **HTTP 403:** Check Apache authorization configuration and `/var/log/httpd/error_log`.
- **HTTP 200 with exception:** Inspect the actual body and browser page; the code alone is inconclusive.
- **LESS cache exception:** Check permissions on the exact module's cache directory.
- **GPG exception:** Verify `.gnupg` owner, permissions and ability of the web process to access signatures.
- **CACHEPATCH:** Prove actual cache writeability before dismissing notifications.
- **MicroSIP Offline:** Check secret, Login, Domain, extension type, and UDP 5160 versus 5060.

## 11. Final checks

- Review FreePBX notices using `fwconsole notification --list`.
- After each reboot, test service health, SIP registration, AMI connectivity, and two-way audio.
- Check any optional modules and their dependent services separately.
