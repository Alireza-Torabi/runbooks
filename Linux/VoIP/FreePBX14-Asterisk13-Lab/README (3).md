# FreePBX 14 + Asterisk 13.20.0 — Offline Lab Runbooks

[English](#english) | [فارسی](#فارسی)

## English

### Overview

This repository documents the setup, administration, and validation of a **FreePBX 14.0.16.9** laboratory instance running **exactly Asterisk 13.20.0** on **CentOS 7.9**. It is intended for offline compatibility experiments, SIP registration tests, internal calling, and Asterisk Manager Interface (AMI) integration checks.

### Reference stack

| Component | Version / role |
|---|---|
| Operating system | CentOS 7.9 |
| Asterisk | **13.20.0** (source-built) |
| FreePBX | **14.0.16.9** |
| PHP | 5.6.40 |
| Apache HTTP Server | 2.4.6 |
| MariaDB | 5.5.68 |
| Node.js | 8.17.0 |
| SIP | PJSIP (typically UDP 5060), Chan_SIP (typically UDP 5160) |
| AMI | TCP 5038 (restrict access to authorized clients) |

### Documentation

- **[English runbook](FreePBX14_Asterisk13_Runbook_EN.md)** — prerequisites, service management, FreePBX web UI, AMI accounts, SIP extensions, softphone registration, verification, and troubleshooting.
- **[راهنمای فارسی](FreePBX14_Asterisk13_Runbook_FA.md)** — equivalent Persian operational runbook.

### Quick verification (existing installation)

```bash
asterisk -V
asterisk -rx "core show uptime"
asterisk -rx "module show like chan_sip"
asterisk -rx "module show like chan_pjsip"
systemctl is-active httpd mariadb freepbx
ss -lunp | grep -E ':(5060|5160)\b'
```

The `freepbx` unit in this lab is a custom `oneshot` service and may report `active (exited)` when functioning normally. Verify real application health through the Asterisk CLI and web UI, not service state alone.

### Validation workflow

1. Verify software versions, processes, and FreePBX web access.
2. Configure example AMI read-only and read/write accounts with different credentials and restricted client addresses.
3. Create test SIP extensions in the FreePBX UI and apply configuration.
4. Register a SIP softphone such as MicroSIP using the **correct driver and port**.
5. Verify registration and complete an internal call, including two-way audio and RTP behavior.
6. Reboot and repeat service, registration, and call checks.

---

## فارسی

### معرفی

این مخزن شامل Runbookهای راه‌اندازی، مدیریت و تست یک محیط آزمایشگاهی **FreePBX 14.0.16.9** با نسخه دقیق **Asterisk 13.20.0** بر پایه **CentOS 7.9** است. هدف آن تست سازگاری، رجیستر داخلی‌های SIP، مکالمات داخلی و اتصال AMI در یک محیط آفلاین و ایزوله است.

### مستندات

- **[Runbook فارسی](FreePBX14_Asterisk13_Runbook_FA.md)** — پیش‌نیازها، مدیریت سرویس‌ها، پنل FreePBX، کاربران AMI، داخلی‌های SIP، تست Softphone، عیب‌یابی و چک‌لیست پذیرش.
- **[English Runbook](FreePBX14_Asterisk13_Runbook_EN.md)** — نسخه انگلیسی راهنمای عملیاتی.

### روند تست

1. بررسی نسخه‌های نرم‌افزار، سرویس‌ها و دسترسی به پنل وب.
2. تعریف حساب‌های AMI فقط‌خواندنی و مدیریتی با رمزهای مجزا و دسترسی شبکه محدود.
3. ایجاد داخلی‌های آزمایشی از طریق رابط FreePBX.
4. اتصال Softphone با انتخاب پورت متناسب با `Chan_SIP` یا `PJSIP`.
5. تست رجیستر، تماس داخلی، صدای دوطرفه و RTP.
6. راه‌اندازی مجدد ماشین و تکرار بررسی سلامت سرویس‌ها و تماس.

## Repository files

```text
README.md
FreePBX14_Asterisk13_Runbook_FA.md
FreePBX14_Asterisk13_Runbook_EN.md
```

