::: {dir="rtl" align="right"}
# راهنمای نصب و پیکربندی Symantec Endpoint Protection 14.4

## 1. هدف و معماری

این راهنما برای پیاده‌سازی **Symantec Endpoint Protection Manager (SEPM)
14.4** و کلاینت‌های **SEP 14.4** در یک شبکه سازمانی تهیه شده است. موضوعات
اصلی شامل نصب Windows و Linux، مهاجرت از Kaspersky، توزیع متمرکز،
LiveUpdate، Policyهای Workstation/Server/Domain Controller و عیب‌یابی
Remote Deployment است.

> **نکته امنیتی:** رمزهای واقعی، نام‌های کاربری مدیریتی و اطلاعات حساس را
> داخل اسکریپت‌ها، GPOها یا مستندات Git ذخیره نکنید. در این سند از
> Placeholder استفاده شده است.

## 2. ساختار پیشنهادی Groupها

``` text
My Company
├── Workstations
│   └── Windows-Clients
├── Servers
│   ├── Windows-Servers
│   ├── Domain-Controllers
│   └── Linux-Servers
└── Test
    └── Pilot-Clients
```

برای Domain Controller، Database Server، File Server و Application
Server از Policy یکسان استفاده نکنید.

## 3. Packageهای نصب

در SEPM:

``` text
Admin
→ Install Packages
→ Client Install Packages
```

Package ویندوز 64 بیتی SEP برای Windows Workstation و Windows Server
مشترک است. تفاوت اصلی با **Client Install Feature Set**، **Client
Install Settings** و Policyهای Group ایجاد می‌شود.

نمونه Package پایه:

``` text
Symantec Endpoint Protection 14.4 for WIN64BIT
```

برای Linux از Package مربوط به:

``` text
Symantec Agent for Linux 14.4
```

استفاده می‌شود.

## 4. Feature Set پیشنهادی

### Workstation

``` text
Virus and Spyware Protection
SONAR
Download Insight
Intrusion Prevention
Firewall
Application and Device Control
```

### Windows Server

``` text
Virus and Spyware Protection       Enable
SONAR                              Enable
Intrusion Prevention              Enable / Pilot
Firewall                          Enable / Pilot
Application and Device Control    Role dependent / Pilot
```

برای Serverها Feature Set کامل قابل استفاده است، اما Firewall و
Application Control باید با Policy مخصوص Role و Pilot اجرا شوند.

## 5. مهاجرت از Kaspersky Endpoint Security

سناریوی مورد بررسی:

``` text
Kaspersky Endpoint Security for Windows 12.4.x
+ Kaspersky Network Agent
```

اگر Password Protection برای Uninstall فعال است، بهترین روش این است که
ابتدا از **Kaspersky Security Center** عملیات حذف انجام شود.

ترتیب:

``` text
Disable/authorize Kaspersky uninstall
        ↓
Uninstall Kaspersky Endpoint Security
        ↓
Reboot if required
        ↓
Verify KES removal
        ↓
Uninstall Kaspersky Network Agent
        ↓
Deploy SEP
```

Network Agent را قبل از KES حذف نکنید، زیرا برای اجرای Remote Task به آن
نیاز دارید.

روی قابلیت Third-Party Removal در SEPM برای عبور از Password Protection
حساب نکنید.

## 6. Remote Deployment ویندوز

از:

``` text
Client Deployment Wizard
```

استفاده کنید. در شبکه Domain بهتر است به جای وابستگی به `Browse Network`
از `Search Network`، IP، Range یا Computer Name استفاده شود.

### پیش‌نیازهای Remote Push

از SEPM Server:

``` powershell
Test-NetConnection <CLIENT-IP> -Port 445
Test-NetConnection <CLIENT-IP> -Port 135
```

تست Administrative Share:

``` cmd
net use \\<CLIENT-IP>\admin$ /user:<DOMAIN>\<DEPLOY-ACCOUNT> *
```

تست Remote Service Control:

``` cmd
sc.exe \\<CLIENT-IP> query RemoteRegistry
```

تست Remote Registry:

``` cmd
reg query \\<CLIENT-IP>\HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion
```

### پورت‌های مهم

``` text
TCP 445              SMB / ADMIN$
TCP 135              RPC Endpoint Mapper
TCP 49152-65535      Dynamic RPC (Windows modern)
```

Dynamic RPC را برای کل شبکه باز نکنید؛ Source را تا حد امکان به IP سرور
مدیریت/SEPM محدود کنید.

## 7. فعال کردن Remote Registry با GPO

مسیر:

``` text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ System Services
→ Remote Registry
```

تنظیم:

``` text
Define this policy setting: Enabled
Startup mode: Automatic
```

سپس:

``` cmd
gpupdate /force
```

و بررسی:

``` powershell
Get-Service RemoteRegistry
```

## 8. Windows Firewall و GPO

برای خاموش کردن Windows Firewall از GPO:

``` text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Windows Defender Firewall with Advanced Security
→ Windows Defender Firewall Properties
```

برای Profile موردنظر:

``` text
Firewall state: Off
```

بررسی:

``` powershell
Get-NetFirewallProfile | Select-Object Name,Enabled
```

> روی Production، مخصوصاً Domain Controller، Windows Firewall را فقط
> زمانی Disable کنید که SEP Firewall نصب، فعال و Ruleهای آن تست شده
> باشند. خاموش بودن همزمان هر دو Firewall قابل قبول نیست.

## 9. LiveUpdate و دریافت Update توسط SEPM

در SEPM:

``` text
Admin
→ Servers
→ Local Site
```

وضعیت Downloadهای LiveUpdate و آخرین Content Revision را بررسی کنید.

برای اجرای Download دستی:

``` text
Admin
→ Servers
→ Local Site
→ Download LiveUpdate Content
```

Interval نمونه برای SEPM:

``` text
Every 4 hours
```

## 10. Update گرفتن Clientها از SEPM

Policy:

``` text
Policies
→ LiveUpdate
→ LiveUpdate Settings
```

برای Clientهای داخلی:

``` text
Use the default management server    Enabled
Direct Internet LiveUpdate           Disabled (در صورت عدم نیاز)
```

ارتباط Client با SEPM را در Client بررسی کنید:

``` text
SEP
→ Help
→ Troubleshooting
```

Server و Group باید SEPM داخلی را نشان دهند.

تست شبکه‌ای رایج:

``` powershell
Test-NetConnection <SEPM-IP> -Port 8014
```

برای اثبات Update شدن، Revision تعریف‌های SEPM و Client را مقایسه کنید.

## 11. Communication Settings

برای شبکه متوسط:

``` text
Mode: Pull
Heartbeat: 30 minutes
Download Randomization: Enabled
```

برای Serverها می‌توان Interval کوتاه‌تری انتخاب کرد، ولی بدون نیاز واقعی
Load مدیریتی را افزایش ندهید.

## 12. Password Protection کلاینت

در Group موردنظر:

``` text
Clients
→ <Group>
→ Policies
→ Password Settings
```

گزینه‌های مهم:

``` text
Require a password to uninstall the client
Require a password to run CleanWipe
Require a password to stop the client service
Require a password to import/export policy
```

اگر `Use the default client password` انتخاب شده باشد، رمز Site-level
استفاده می‌شود و معمولاً به صورت Plain Text نمایش داده نمی‌شود. برای کنترل
بهتر می‌توان:

``` text
Use a group client password
```

را انتخاب و Password جدید تعریف کرد.

## 13. Windows Server قدیمی و Azure Code Signing

اگر Installer خطای زیر داد:

``` text
Symantec Endpoint Protection can only be installed on systems with Azure Code Signing support
```

Windows را Patch کنید. روی Windows Server 2022 با Build بسیار قدیمی،
آخرین Cumulative Update را نصب و سیستم را Restart کنید.

همچنین وجود Root Certificate موردنیاز Microsoft را بررسی کنید:

``` powershell
Get-ChildItem Cert:\LocalMachine\Root |
Where-Object {
    $_.Subject -like "*Microsoft Identity Verification Root Certificate Authority 2020*"
}
```

## 14. نصب SEP روی Ubuntu 22.04

برای Package کامل Offline:

``` bash
chmod +x LinuxInstaller.ubuntu22
sudo ./LinuxInstaller.ubuntu22 -- -g
```

اگر Dependencyها ناقص باشند:

``` bash
sudo apt update
sudo apt install -y at auditd libelf-dev
```

و دوباره:

``` bash
sudo ./LinuxInstaller.ubuntu22 -- -g
```

### Secure Boot و MOK

اگر Secure Boot فعال باشد و Moduleهای SEP لود نشوند:

``` text
sisevt    not loaded
sisap     not loaded
```

Public Key محصول را به MOK اضافه کنید:

``` bash
sudo mokutil --import /usr/lib/symantec/sdcssagent/driver/sis-key.der
```

قبل از Import می‌توانید Certificate را بررسی کنید:

``` bash
openssl x509 \
  -inform DER \
  -in /usr/lib/symantec/sdcssagent/driver/sis-key.der \
  -noout -subject -issuer -serial -fingerprint -sha256
```

سپس Reboot کنید و در کنسول UEFI:

``` text
Enroll MOK
→ Confirm
→ Enter temporary MOK password
→ Reboot
```

بعد از Boot:

``` bash
sudo /usr/lib/symantec/status.sh
lsmod | grep -E 'sisevt|sisap'
```

Enroll کردن Public Signing Key معتبر Symantec/Broadcom به MOK، Secure
Boot را Disable نمی‌کند.

## 15. Group و Policy مخصوص Domain Controller

ساختار:

``` text
Servers
└── Domain-Controllers
```

Policyهای پیشنهادی:

``` text
DC-Malware-Protection
DC-IPS
DC-MEM
DC-Firewall
DC-Exceptions
DC-LiveUpdate
DC-AppDevice-Control
```

### Malware Protection

``` text
Auto-Protect              ON
SONAR                     ON
Download Insight          ON
Tamper Protection         ON
Daily Active Scan         Off-hours
Weekly Full Scan          Off-hours
```

از Exceptionهای گسترده مثل `C:\Windows\` یا `*.db` خودداری کنید.
Exclusion باید Role-specific و حداقلی باشد.

### Intrusion Prevention

``` text
Enable Network Intrusion Prevention             ON
Enable excluded hosts                           OFF
Enable Browser Intrusion Prevention for Windows ON
Log detections but do not block                 OFF
Enable third-party management of extensions     OFF
Enable URL reputation                           ON
Out-of-band scanning                            OFF initially
Use signature subset for servers                ON
```

### Firewall - Protection and Stealth

برای DC به صورت Pilot:

``` text
Enable port scan detection                      ON
Enable denial of service detection              ON
Enable anti-MAC spoofing                        OFF initially
Automatically block attacker's IP address       ON / Pilot
Block duration                                  600 seconds

Stealth mode Web browsing                       OFF
TCP resequencing                                OFF
OS fingerprint masquerading                     OFF
Peer-to-peer authentication                     OFF
```

### Windows Integration

اگر SEP Firewall جایگزین Windows Firewall شده و Ruleها تست شده‌اند:

``` text
Disable Windows Firewall: Disable Once Only
```

اگر Windows Firewall با Domain GPO مدیریت می‌شود و Ruleهای مهم دارد،
ابتدا:

``` text
No Action
```

را نگه دارید تا Migration کنترل‌شده انجام شود.

## 16. Firewall Rule Set پیشنهادی برای DC

اصل مهم: Rule عمومی `Allow all applications / Any / Any` در نهایت باید
حذف یا Disable شود، اما فقط بعد از ساخت و تست Allow Ruleهای لازم.

نمونه:

  -----------------------------------------------------------------------------
  Priority       Rule           Source            Service        Action
  -------------- -------------- ----------------- -------------- --------------
  1              DC-to-DC       DC IPs            Required AD    Allow
                                                  traffic        

  2              DNS            Internal Networks TCP/UDP 53     Allow

  3              Kerberos       Internal Networks TCP/UDP 88     Allow

  4              NTP            Internal Networks UDP 123        Allow

  5              LDAP           Internal Networks TCP/UDP 389    Allow

  6              Kerberos       Internal Networks TCP/UDP 464    Allow
                 Password                                        

  7              SMB            Internal Networks TCP 445        Allow

  8              RPC Endpoint   Internal Networks TCP 135        Allow
                 Mapper                                          

  9              Dynamic RPC    Internal Networks TCP            Allow
                                                  49152-65535    

  10             Global Catalog Internal Networks TCP 3268/3269  Allow

  11             LDAPS          Internal Networks TCP 636        Allow

  12             SEPM           SEPM IP           Required SEP   Allow
                 Management                       traffic        

  13             Admin          Jump/Management   RDP/WinRM as   Allow
                 Management     hosts             required       

  90             Block Other    Any               Any            Block + Log
                 Inbound                                         
  -----------------------------------------------------------------------------

برای DC-to-DC می‌توان در فاز اول `Service: Any` را فقط بین IPهای مشخص
DCها مجاز کرد و بعداً محدودتر نمود.

مواردی مثل UPnP، SSDP، Bonjour، LLMNR، Wireless EAPOL و VPN روی DC در
صورت عدم نیاز باید Disable شوند.

## 17. Rollout امن Firewall روی DC

``` text
Create explicit Allow rules
        ↓
Keep existing Allow-All temporarily
        ↓
Apply only to Pilot DC
        ↓
Monitor Traffic Log
        ↓
Validate DNS/Kerberos/LDAP/GPO/SYSVOL/Replication
        ↓
Fix missing rules
        ↓
Disable Allow-All
        ↓
Enable final Block + Logging
        ↓
Deploy to remaining DCs
```

## 18. DLP

SEP 14.4 به تنهایی **DLP کامل و Content-aware** نیست.

`Application and Device Control` می‌تواند برای کنترل USB، Device و
Application استفاده شود، اما برای Policyهایی مانند تشخیص اطلاعات حساس
داخل فایل و جلوگیری از خروج محتوا از USB/Email/Web به محصول **Symantec
Data Loss Prevention** یا DLP معادل نیاز است.

## 19. عیب‌یابی RPC Error 1722

خطا:

``` text
[SC] OpenSCManager FAILED 1722:
The RPC server is unavailable.
```

اگر TCP 135، TCP 445 و `ADMIN$` سالم هستند، Dynamic RPC و Windows
Firewall را بررسی کنید.

روی Client:

``` powershell
Get-Service RpcSs,RpcEptMapper,DcomLaunch,RemoteRegistry
```

تست از Management Server:

``` cmd
sc.exe \\<CLIENT-IP> query RemoteRegistry
```

پورت‌های Dynamic RPC:

``` text
TCP 49152-65535
```

Firewall Rule را فقط از Management/SEPM Server به Client Network محدود
کنید.

## 20. ترتیب کلی اجرای پروژه

``` text
1. Upgrade/Patch SEPM and operating systems
2. Create groups
3. Create Pilot group
4. Create Windows/Linux packages
5. Configure LiveUpdate
6. Configure client communication
7. Configure password protection
8. Remove previous antivirus
9. Deploy SEP to Pilot clients
10. Validate connectivity and definitions
11. Deploy Workstations
12. Deploy Windows Servers
13. Deploy Linux Servers
14. Build dedicated Domain Controller policies
15. Pilot DC Firewall/IPS
16. Monitor logs and tune policies
17. Roll out to remaining systems
18. Configure reporting and alerts
```

## 21. اصول امنیتی نهایی

-   SEP را جایگزین Patch Management، Backup و Privileged Access
    Management ندانید.
-   برای Domain Controllerها Backup قابل بازیابی و ترجیحاً
    Immutable/Offline داشته باشید.
-   دسترسی RDP/WinRM به DC را به Jump Server یا Management Network محدود
    کنید.
-   از ذخیره Passwordهای مدیریتی در Script و GPO به صورت Plain Text
    خودداری کنید.
-   Policyهای Firewall و Application Control را ابتدا Pilot کنید.
-   Definition Update و Client Health را از SEPM مانیتور کنید.
-   برای مقابله با ransomware، محدودسازی lateral movement به اندازه خود
    Antivirus اهمیت دارد.
:::
