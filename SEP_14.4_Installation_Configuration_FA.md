# <div dir="rtl" align="right">راهنمای نصب و پیکربندی Symantec Endpoint Protection 14.4</div>

## <div dir="rtl">1. هدف و معماری</div>

<div dir="rtl" align="right">این راهنما برای پیاده‌سازی **Symantec Endpoint Protection Manager (SEPM)</div>
<div dir="rtl" align="right">14.4** و کلاینت‌های **SEP 14.4** در یک شبکه سازمانی تهیه شده است. موضوعات</div>
<div dir="rtl" align="right">اصلی شامل نصب Windows و Linux، مهاجرت از Kaspersky، توزیع متمرکز،</div>
<div dir="rtl" align="right">LiveUpdate، Policyهای Workstation/Server/Domain Controller و عیب‌یابی</div>
<div dir="rtl" align="right">Remote Deployment است.</div>

> <span dir="rtl">**نکته امنیتی:** رمزهای واقعی، نام‌های کاربری مدیریتی و اطلاعات حساس را</span>
> <span dir="rtl">داخل اسکریپت‌ها، GPOها یا مستندات Git ذخیره نکنید. در این سند از</span>
> <span dir="rtl">Placeholder استفاده شده است.</span>

## <span dir="rtl">2. ساختار پیشنهادی Groupها</span>

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

<div dir="rtl" align="right">برای Domain Controller، Database Server، File Server و Application</div>
<div dir="rtl" align="right">Server از Policy یکسان استفاده نکنید.</div>

## <span dir="rtl">3. Packageهای نصب</span>

<div dir="rtl" align="right">در SEPM:</div>

``` text
Admin
→ Install Packages
→ Client Install Packages
```

<div dir="rtl" align="right">Package ویندوز 64 بیتی SEP برای Windows Workstation و Windows Server</div>
<div dir="rtl" align="right">مشترک است. تفاوت اصلی با **Client Install Feature Set**، **Client</div>
<div dir="rtl" align="right">Install Settings** و Policyهای Group ایجاد می‌شود.</div>

<div dir="rtl" align="right">نمونه Package پایه:</div>

``` text
Symantec Endpoint Protection 14.4 for WIN64BIT
```

<div dir="rtl" align="right">برای Linux از Package مربوط به:</div>

``` text
Symantec Agent for Linux 14.4
```

<div dir="rtl" align="right">استفاده می‌شود.</div>

## <span dir="rtl">4. Feature Set پیشنهادی</span>

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

<div dir="rtl" align="right">برای Serverها Feature Set کامل قابل استفاده است، اما Firewall و</div>
<div dir="rtl" align="right">Application Control باید با Policy مخصوص Role و Pilot اجرا شوند.</div>

## <span dir="rtl">5. مهاجرت از Kaspersky Endpoint Security</span>

<div dir="rtl" align="right">سناریوی مورد بررسی:</div>

``` text
Kaspersky Endpoint Security for Windows 12.4.x
+ Kaspersky Network Agent
```

<div dir="rtl" align="right">اگر Password Protection برای Uninstall فعال است، بهترین روش این است که</div>
<div dir="rtl" align="right">ابتدا از **Kaspersky Security Center** عملیات حذف انجام شود.</div>

<div dir="rtl" align="right">ترتیب:</div>

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

<div dir="rtl" align="right">Network Agent را قبل از KES حذف نکنید، زیرا برای اجرای Remote Task به آن</div>
<div dir="rtl" align="right">نیاز دارید.</div>

<div dir="rtl" align="right">روی قابلیت Third-Party Removal در SEPM برای عبور از Password Protection</div>
<div dir="rtl" align="right">حساب نکنید.</div>

## <span dir="rtl">6. Remote Deployment ویندوز</span>

<div dir="rtl" align="right">از:</div>

``` text
Client Deployment Wizard
```

<div dir="rtl" align="right">استفاده کنید. در شبکه Domain بهتر است به جای وابستگی به `Browse Network`</div>
<div dir="rtl" align="right">از `Search Network`، IP، Range یا Computer Name استفاده شود.</div>

### <span dir="rtl">پیش‌نیازهای Remote Push</span>

<div dir="rtl" align="right">از SEPM Server:</div>

``` powershell
Test-NetConnection <CLIENT-IP> -Port 445
Test-NetConnection <CLIENT-IP> -Port 135
```

<div dir="rtl" align="right">تست Administrative Share:</div>

``` cmd
net use \\<CLIENT-IP>\admin$ /user:<DOMAIN>\<DEPLOY-ACCOUNT> *
```

<div dir="rtl" align="right">تست Remote Service Control:</div>

``` cmd
sc.exe \\<CLIENT-IP> query RemoteRegistry
```

<div dir="rtl" align="right">تست Remote Registry:</div>

``` cmd
reg query \\<CLIENT-IP>\HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion
```

### <span dir="rtl">پورت‌های مهم</span>

``` text
TCP 445              SMB / ADMIN$
TCP 135              RPC Endpoint Mapper
TCP 49152-65535      Dynamic RPC (Windows modern)
```

<div dir="rtl" align="right">Dynamic RPC را برای کل شبکه باز نکنید؛ Source را تا حد امکان به IP سرور</div>
<div dir="rtl" align="right">مدیریت/SEPM محدود کنید.</div>

## <span dir="rtl">7. فعال کردن Remote Registry با GPO</span>

<div dir="rtl" align="right">مسیر:</div>

``` text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ System Services
→ Remote Registry
```

<div dir="rtl" align="right">تنظیم:</div>

``` text
Define this policy setting: Enabled
Startup mode: Automatic
```

<div dir="rtl" align="right">سپس:</div>

``` cmd
gpupdate /force
```

<div dir="rtl" align="right">و بررسی:</div>

``` powershell
Get-Service RemoteRegistry
```

## <span dir="rtl">8. Windows Firewall و GPO</span>

<div dir="rtl" align="right">برای خاموش کردن Windows Firewall از GPO:</div>

``` text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Windows Defender Firewall with Advanced Security
→ Windows Defender Firewall Properties
```

<div dir="rtl" align="right">برای Profile موردنظر:</div>

``` text
Firewall state: Off
```

<div dir="rtl" align="right">بررسی:</div>

``` powershell
Get-NetFirewallProfile | Select-Object Name,Enabled
```

> <span dir="rtl">روی Production، مخصوصاً Domain Controller، Windows Firewall را فقط</span>
> <span dir="rtl">زمانی Disable کنید که SEP Firewall نصب، فعال و Ruleهای آن تست شده</span>
> <span dir="rtl">باشند. خاموش بودن همزمان هر دو Firewall قابل قبول نیست.</span>

## <span dir="rtl">9. LiveUpdate و دریافت Update توسط SEPM</span>

<div dir="rtl" align="right">در SEPM:</div>

``` text
Admin
→ Servers
→ Local Site
```

<div dir="rtl" align="right">وضعیت Downloadهای LiveUpdate و آخرین Content Revision را بررسی کنید.</div>

<div dir="rtl" align="right">برای اجرای Download دستی:</div>

``` text
Admin
→ Servers
→ Local Site
→ Download LiveUpdate Content
```

<div dir="rtl" align="right">Interval نمونه برای SEPM:</div>

``` text
Every 4 hours
```

## <span dir="rtl">10. Update گرفتن Clientها از SEPM</span>

Policy:

``` text
Policies
→ LiveUpdate
→ LiveUpdate Settings
```

<div dir="rtl" align="right">برای Clientهای داخلی:</div>

``` text
Use the default management server    Enabled
Direct Internet LiveUpdate           Disabled (در صورت عدم نیاز)
```

<div dir="rtl" align="right">ارتباط Client با SEPM را در Client بررسی کنید:</div>

``` text
SEP
→ Help
→ Troubleshooting
```

<div dir="rtl" align="right">Server و Group باید SEPM داخلی را نشان دهند.</div>

<div dir="rtl" align="right">تست شبکه‌ای رایج:</div>

``` powershell
Test-NetConnection <SEPM-IP> -Port 8014
```

<div dir="rtl" align="right">برای اثبات Update شدن، Revision تعریف‌های SEPM و Client را مقایسه کنید.</div>

## 11. Communication Settings

<div dir="rtl" align="right">برای شبکه متوسط:</div>

``` text
Mode: Pull
Heartbeat: 30 minutes
Download Randomization: Enabled
```

<div dir="rtl" align="right">برای Serverها می‌توان Interval کوتاه‌تری انتخاب کرد، ولی بدون نیاز واقعی</div>
<div dir="rtl" align="right">Load مدیریتی را افزایش ندهید.</div>

## <span dir="rtl">12. Password Protection کلاینت</span>

<div dir="rtl" align="right">در Group موردنظر:</div>

``` text
Clients
→ <Group>
→ Policies
→ Password Settings
```

<div dir="rtl" align="right">گزینه‌های مهم:</div>

``` text
Require a password to uninstall the client
Require a password to run CleanWipe
Require a password to stop the client service
Require a password to import/export policy
```

<div dir="rtl" align="right">اگر `Use the default client password` انتخاب شده باشد، رمز Site-level</div>
<div dir="rtl" align="right">استفاده می‌شود و معمولاً به صورت Plain Text نمایش داده نمی‌شود. برای کنترل</div>
<div dir="rtl" align="right">بهتر می‌توان:</div>

``` text
Use a group client password
```

<div dir="rtl" align="right">را انتخاب و Password جدید تعریف کرد.</div>

## <span dir="rtl">13. Windows Server قدیمی و Azure Code Signing</span>

<div dir="rtl" align="right">اگر Installer خطای زیر داد:</div>

``` text
Symantec Endpoint Protection can only be installed on systems with Azure Code Signing support
```

<div dir="rtl" align="right">Windows را Patch کنید. روی Windows Server 2022 با Build بسیار قدیمی،</div>
<div dir="rtl" align="right">آخرین Cumulative Update را نصب و سیستم را Restart کنید.</div>

<div dir="rtl" align="right">همچنین وجود Root Certificate موردنیاز Microsoft را بررسی کنید:</div>

``` powershell
Get-ChildItem Cert:\LocalMachine\Root |
Where-Object {
    $_.Subject -like "*Microsoft Identity Verification Root Certificate Authority 2020*"
}
```

## <span dir="rtl">14. نصب SEP روی Ubuntu 22.04</span>

<div dir="rtl" align="right">برای Package کامل Offline:</div>

``` bash
chmod +x LinuxInstaller.ubuntu22
sudo ./LinuxInstaller.ubuntu22 -- -g
```

<div dir="rtl" align="right">اگر Dependencyها ناقص باشند:</div>

``` bash
sudo apt update
sudo apt install -y at auditd libelf-dev
```

<div dir="rtl" align="right">و دوباره:</div>

``` bash
sudo ./LinuxInstaller.ubuntu22 -- -g
```

### <span dir="rtl">Secure Boot و MOK</span>

<div dir="rtl" align="right">اگر Secure Boot فعال باشد و Moduleهای SEP لود نشوند:</div>

``` text
sisevt    not loaded
sisap     not loaded
```

<div dir="rtl" align="right">Public Key محصول را به MOK اضافه کنید:</div>

``` bash
sudo mokutil --import /usr/lib/symantec/sdcssagent/driver/sis-key.der
```

<div dir="rtl" align="right">قبل از Import می‌توانید Certificate را بررسی کنید:</div>

``` bash
openssl x509 \
  -inform DER \
  -in /usr/lib/symantec/sdcssagent/driver/sis-key.der \
  -noout -subject -issuer -serial -fingerprint -sha256
```

<div dir="rtl" align="right">سپس Reboot کنید و در کنسول UEFI:</div>

``` text
Enroll MOK
→ Confirm
→ Enter temporary MOK password
→ Reboot
```

<div dir="rtl" align="right">بعد از Boot:</div>

``` bash
sudo /usr/lib/symantec/status.sh
lsmod | grep -E 'sisevt|sisap'
```

<div dir="rtl" align="right">Enroll کردن Public Signing Key معتبر Symantec/Broadcom به MOK، Secure</div>
<div dir="rtl" align="right">Boot را Disable نمی‌کند.</div>

## <span dir="rtl">15. Group و Policy مخصوص Domain Controller</span>

<div dir="rtl" align="right">ساختار:</div>

``` text
Servers
└── Domain-Controllers
```

<div dir="rtl" align="right">Policyهای پیشنهادی:</div>

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

<div dir="rtl" align="right">از Exceptionهای گسترده مثل `C:\Windows\` یا `*.db` خودداری کنید.</div>
<div dir="rtl" align="right">Exclusion باید Role-specific و حداقلی باشد.</div>

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

<div dir="rtl" align="right">برای DC به صورت Pilot:</div>

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

<div dir="rtl" align="right">اگر SEP Firewall جایگزین Windows Firewall شده و Ruleها تست شده‌اند:</div>

``` text
Disable Windows Firewall: Disable Once Only
```

<div dir="rtl" align="right">اگر Windows Firewall با Domain GPO مدیریت می‌شود و Ruleهای مهم دارد،</div>
<div dir="rtl" align="right">ابتدا:</div>

``` text
No Action
```

<div dir="rtl" align="right">را نگه دارید تا Migration کنترل‌شده انجام شود.</div>

## <span dir="rtl">16. Firewall Rule Set پیشنهادی برای DC</span>

<div dir="rtl" align="right">اصل مهم: Rule عمومی `Allow all applications / Any / Any` در نهایت باید</div>
<div dir="rtl" align="right">حذف یا Disable شود، اما فقط بعد از ساخت و تست Allow Ruleهای لازم.</div>

<div dir="rtl" align="right">نمونه:</div>

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

<div dir="rtl" align="right">برای DC-to-DC می‌توان در فاز اول `Service: Any` را فقط بین IPهای مشخص</div>
<div dir="rtl" align="right">DCها مجاز کرد و بعداً محدودتر نمود.</div>

<div dir="rtl" align="right">مواردی مثل UPnP، SSDP، Bonjour، LLMNR، Wireless EAPOL و VPN روی DC در</div>
<div dir="rtl" align="right">صورت عدم نیاز باید Disable شوند.</div>

## <span dir="rtl">17. Rollout امن Firewall روی DC</span>

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

<div dir="rtl" align="right">SEP 14.4 به تنهایی **DLP کامل و Content-aware** نیست.</div>

<div dir="rtl" align="right">`Application and Device Control` می‌تواند برای کنترل USB، Device و</div>
<div dir="rtl" align="right">Application استفاده شود، اما برای Policyهایی مانند تشخیص اطلاعات حساس</div>
<div dir="rtl" align="right">داخل فایل و جلوگیری از خروج محتوا از USB/Email/Web به محصول **Symantec</div>
<div dir="rtl" align="right">Data Loss Prevention** یا DLP معادل نیاز است.</div>

## <span dir="rtl">19. عیب‌یابی RPC Error 1722</span>

<div dir="rtl" align="right">خطا:</div>

``` text
[SC] OpenSCManager FAILED 1722:
The RPC server is unavailable.
```

<div dir="rtl" align="right">اگر TCP 135، TCP 445 و `ADMIN$` سالم هستند، Dynamic RPC و Windows</div>
<div dir="rtl" align="right">Firewall را بررسی کنید.</div>

<div dir="rtl" align="right">روی Client:</div>

``` powershell
Get-Service RpcSs,RpcEptMapper,DcomLaunch,RemoteRegistry
```

<div dir="rtl" align="right">تست از Management Server:</div>

``` cmd
sc.exe \\<CLIENT-IP> query RemoteRegistry
```

<div dir="rtl" align="right">پورت‌های Dynamic RPC:</div>

``` text
TCP 49152-65535
```

<div dir="rtl" align="right">Firewall Rule را فقط از Management/SEPM Server به Client Network محدود</div>
<div dir="rtl" align="right">کنید.</div>

## <span dir="rtl">20. ترتیب کلی اجرای پروژه</span>

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

## <span dir="rtl">21. اصول امنیتی نهایی</span>

-   <span dir="rtl">SEP را جایگزین Patch Management، Backup و Privileged Access</span>
<div dir="rtl" align="right">    Management ندانید.</div>
-   <span dir="rtl">برای Domain Controllerها Backup قابل بازیابی و ترجیحاً</span>
<div dir="rtl" align="right">    Immutable/Offline داشته باشید.</div>
-   <span dir="rtl">دسترسی RDP/WinRM به DC را به Jump Server یا Management Network محدود</span>
<div dir="rtl" align="right">    کنید.</div>
-   <span dir="rtl">از ذخیره Passwordهای مدیریتی در Script و GPO به صورت Plain Text</span>
<div dir="rtl" align="right">    خودداری کنید.</div>
-   <span dir="rtl">Policyهای Firewall و Application Control را ابتدا Pilot کنید.</span>
-   <span dir="rtl">Definition Update و Client Health را از SEPM مانیتور کنید.</span>
-   <span dir="rtl">برای مقابله با ransomware، محدودسازی lateral movement به اندازه خود</span>
<div dir="rtl" align="right">    Antivirus اهمیت دارد.</div>
:::
