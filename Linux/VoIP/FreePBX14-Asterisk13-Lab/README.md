# FreePBX 14.0.16.9 + Asterisk 13.20.0 — CentOS 7.9

Installation runbooks for reproducing a CentOS 7.9 PBX with source-built **Asterisk 13.20.0** and **FreePBX 14.0.16.9**.

## Installation documentation

- [English — Complete Installation Runbook](FreePBX14_Asterisk13_Runbook_EN.md)
- [فارسی — راهنمای کامل نصب](FreePBX14_Asterisk13_Runbook_FA.md)

## Stack

| Component | Version |
|---|---|
| Operating system | CentOS 7.9 x86_64 |
| Asterisk | 13.20.0 |
| FreePBX | 14.0.16.9 |
| PHP | 5.6.40 |
| MariaDB | 5.5.x |
| Apache | 2.4.x |
| Node.js | 8.17.0 |

## Installation sequence

1. Provision CentOS and required repositories or offline packages.
2. Install build tools, libraries, Apache, MariaDB and PHP 5.6.
3. Install Node.js 8.17.0.
4. Compile Asterisk 13.20.0 and enable the required SIP modules.
5. Install the pinned FreePBX 14.0.16.9 package.
6. Configure systemd startup, AMI accounts and SIP extensions.
7. Verify registration, internal calls and startup after reboot.

The runbooks include commands, configuration examples, version checks and troubleshooting procedures.

## Quick verification

```bash
asterisk -V
fwconsole -V
php -v
node -v
systemctl is-active httpd mariadb freepbx
asterisk -rx 'core show uptime'
```

## فارسی

این مخزن شامل دو راهنمای اجرایی برای نصب مجدد CentOS 7.9، کامپایل **Asterisk 13.20.0**، نصب **FreePBX 14.0.16.9**، نصب سرویس‌ها و وابستگی‌ها و تست داخلی‌ها و AMI است.

برای مراحل اجرایی به [نسخه فارسی](FreePBX14_Asterisk13_Runbook_FA.md) یا [نسخه انگلیسی](FreePBX14_Asterisk13_Runbook_EN.md) مراجعه کنید.
