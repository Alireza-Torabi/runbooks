::: {dir="rtl" align="right"}
```{=html}
<h1 dir="rtl" align="right">
```
راهنمای نصب و پیکربندی Symantec Endpoint Protection 14.4
```{=html}
</h1>
```
```{=html}
<h2 dir="rtl" align="right">
```
1.  هدف و معماری
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
این راهنما برای پیاده‌سازی \*\*Symantec Endpoint Protection Manager
(SEPM)
:::

::: {dir="rtl" align="right"}
14.4\*\* و کلاینت‌های **SEP 14.4** در یک شبکه سازمانی تهیه شده است.
موضوعات
:::

::: {dir="rtl" align="right"}
اصلی شامل نصب Windows و Linux، مهاجرت از Kaspersky، توزیع متمرکز،
:::

::: {dir="rtl" align="right"}
LiveUpdate، Policyهای Workstation/Server/Domain Controller و عیب‌یابی
:::

::: {dir="rtl" align="right"}
Remote Deployment است.
:::

```{=html}
<blockquote dir="rtl">
```
```{=html}
<p dir="rtl" align="right">
```
**نکته امنیتی:** رمزهای واقعی، نام‌های کاربری مدیریتی و اطلاعات حساس را
```{=html}
</p>
```
```{=html}
</blockquote>
```
```{=html}
<blockquote dir="rtl">
```
```{=html}
<p dir="rtl" align="right">
```
داخل اسکریپت‌ها، GPOها یا مستندات Git ذخیره نکنید. در این سند از
```{=html}
</p>
```
```{=html}
</blockquote>
```
```{=html}
<blockquote dir="rtl">
```
```{=html}
<p dir="rtl" align="right">
```
Placeholder استفاده شده است.
```{=html}
</p>
```
```{=html}
</blockquote>
```
```{=html}
<h2 dir="rtl" align="right">
```
2.  ساختار پیشنهادی Groupها
    ```{=html}
    </h2>
    ```

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

::: {dir="rtl" align="right"}
برای Domain Controller، Database Server، File Server و Application
:::

::: {dir="rtl" align="right"}
Server از Policy یکسان استفاده نکنید.
:::

```{=html}
<h2 dir="rtl" align="right">
```
3.  Packageهای نصب
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
در SEPM:
:::

``` text
Admin
→ Install Packages
→ Client Install Packages
```

::: {dir="rtl" align="right"}
Package ویندوز 64 بیتی SEP برای Windows Workstation و Windows Server
:::

::: {dir="rtl" align="right"}
مشترک است. تفاوت اصلی با **Client Install Feature Set**، \*\*Client
:::

::: {dir="rtl" align="right"}
Install Settings\*\* و Policyهای Group ایجاد می‌شود.
:::

::: {dir="rtl" align="right"}
نمونه Package پایه:
:::

``` text
Symantec Endpoint Protection 14.4 for WIN64BIT
```

::: {dir="rtl" align="right"}
برای Linux از Package مربوط به:
:::

``` text
Symantec Agent for Linux 14.4
```

::: {dir="rtl" align="right"}
استفاده می‌شود.
:::

```{=html}
<h2 dir="rtl" align="right">
```
4.  Feature Set پیشنهادی
    ```{=html}
    </h2>
    ```

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

::: {dir="rtl" align="right"}
برای Serverها Feature Set کامل قابل استفاده است، اما Firewall و
:::

::: {dir="rtl" align="right"}
Application Control باید با Policy مخصوص Role و Pilot اجرا شوند.
:::

```{=html}
<h2 dir="rtl" align="right">
```
5.  مهاجرت از Kaspersky Endpoint Security
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
سناریوی مورد بررسی:
:::

``` text
Kaspersky Endpoint Security for Windows 12.4.x
+ Kaspersky Network Agent
```

::: {dir="rtl" align="right"}
اگر Password Protection برای Uninstall فعال است، بهترین روش این است که
:::

::: {dir="rtl" align="right"}
ابتدا از **Kaspersky Security Center** عملیات حذف انجام شود.
:::

::: {dir="rtl" align="right"}
ترتیب:
:::

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

::: {dir="rtl" align="right"}
Network Agent را قبل از KES حذف نکنید، زیرا برای اجرای Remote Task به آن
:::

::: {dir="rtl" align="right"}
نیاز دارید.
:::

::: {dir="rtl" align="right"}
روی قابلیت Third-Party Removal در SEPM برای عبور از Password Protection
:::

::: {dir="rtl" align="right"}
حساب نکنید.
:::

```{=html}
<h2 dir="rtl" align="right">
```
6.  Remote Deployment ویندوز
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
از:
:::

``` text
Client Deployment Wizard
```

::: {dir="rtl" align="right"}
استفاده کنید. در شبکه Domain بهتر است به جای وابستگی به `Browse Network`
:::

::: {dir="rtl" align="right"}
از `Search Network`، IP، Range یا Computer Name استفاده شود.
:::

```{=html}
<h3 dir="rtl" align="right">
```
پیش‌نیازهای Remote Push
```{=html}
</h3>
```
::: {dir="rtl" align="right"}
از SEPM Server:
:::

``` powershell
Test-NetConnection <CLIENT-IP> -Port 445
Test-NetConnection <CLIENT-IP> -Port 135
```

::: {dir="rtl" align="right"}
تست Administrative Share:
:::

``` cmd
net use \\<CLIENT-IP>\admin$ /user:<DOMAIN>\<DEPLOY-ACCOUNT> *
```

::: {dir="rtl" align="right"}
تست Remote Service Control:
:::

``` cmd
sc.exe \\<CLIENT-IP> query RemoteRegistry
```

::: {dir="rtl" align="right"}
تست Remote Registry:
:::

``` cmd
reg query \\<CLIENT-IP>\HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion
```

```{=html}
<h3 dir="rtl" align="right">
```
پورت‌های مهم
```{=html}
</h3>
```
``` text
TCP 445              SMB / ADMIN$
TCP 135              RPC Endpoint Mapper
TCP 49152-65535      Dynamic RPC (Windows modern)
```

::: {dir="rtl" align="right"}
Dynamic RPC را برای کل شبکه باز نکنید؛ Source را تا حد امکان به IP سرور
:::

::: {dir="rtl" align="right"}
مدیریت/SEPM محدود کنید.
:::

```{=html}
<h2 dir="rtl" align="right">
```
7.  فعال کردن Remote Registry با GPO
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
مسیر:
:::

``` text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ System Services
→ Remote Registry
```

::: {dir="rtl" align="right"}
تنظیم:
:::

``` text
Define this policy setting: Enabled
Startup mode: Automatic
```

::: {dir="rtl" align="right"}
سپس:
:::

``` cmd
gpupdate /force
```

::: {dir="rtl" align="right"}
و بررسی:
:::

``` powershell
Get-Service RemoteRegistry
```

```{=html}
<h2 dir="rtl" align="right">
```
8.  Windows Firewall و GPO
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
برای خاموش کردن Windows Firewall از GPO:
:::

``` text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Windows Defender Firewall with Advanced Security
→ Windows Defender Firewall Properties
```

::: {dir="rtl" align="right"}
برای Profile موردنظر:
:::

``` text
Firewall state: Off
```

::: {dir="rtl" align="right"}
بررسی:
:::

``` powershell
Get-NetFirewallProfile | Select-Object Name,Enabled
```

```{=html}
<blockquote dir="rtl">
```
```{=html}
<p dir="rtl" align="right">
```
روی Production، مخصوصاً Domain Controller، Windows Firewall را فقط
```{=html}
</p>
```
```{=html}
</blockquote>
```
```{=html}
<blockquote dir="rtl">
```
```{=html}
<p dir="rtl" align="right">
```
زمانی Disable کنید که SEP Firewall نصب، فعال و Ruleهای آن تست شده
```{=html}
</p>
```
```{=html}
</blockquote>
```
```{=html}
<blockquote dir="rtl">
```
```{=html}
<p dir="rtl" align="right">
```
باشند. خاموش بودن همزمان هر دو Firewall قابل قبول نیست.
```{=html}
</p>
```
```{=html}
</blockquote>
```
```{=html}
<h2 dir="rtl" align="right">
```
9.  LiveUpdate و دریافت Update توسط SEPM
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
در SEPM:
:::

``` text
Admin
→ Servers
→ Local Site
```

::: {dir="rtl" align="right"}
وضعیت Downloadهای LiveUpdate و آخرین Content Revision را بررسی کنید.
:::

::: {dir="rtl" align="right"}
برای اجرای Download دستی:
:::

``` text
Admin
→ Servers
→ Local Site
→ Download LiveUpdate Content
```

::: {dir="rtl" align="right"}
Interval نمونه برای SEPM:
:::

``` text
Every 4 hours
```

```{=html}
<h2 dir="rtl" align="right">
```
10. Update گرفتن Clientها از SEPM
    ```{=html}
    </h2>
    ```

Policy:

``` text
Policies
→ LiveUpdate
→ LiveUpdate Settings
```

::: {dir="rtl" align="right"}
برای Clientهای داخلی:
:::

``` text
Use the default management server    Enabled
Direct Internet LiveUpdate           Disabled (در صورت عدم نیاز)
```

::: {dir="rtl" align="right"}
ارتباط Client با SEPM را در Client بررسی کنید:
:::

``` text
SEP
→ Help
→ Troubleshooting
```

::: {dir="rtl" align="right"}
Server و Group باید SEPM داخلی را نشان دهند.
:::

::: {dir="rtl" align="right"}
تست شبکه‌ای رایج:
:::

``` powershell
Test-NetConnection <SEPM-IP> -Port 8014
```

::: {dir="rtl" align="right"}
برای اثبات Update شدن، Revision تعریف‌های SEPM و Client را مقایسه کنید.
:::

## 11. Communication Settings

::: {dir="rtl" align="right"}
برای شبکه متوسط:
:::

``` text
Mode: Pull
Heartbeat: 30 minutes
Download Randomization: Enabled
```

::: {dir="rtl" align="right"}
برای Serverها می‌توان Interval کوتاه‌تری انتخاب کرد، ولی بدون نیاز واقعی
:::

::: {dir="rtl" align="right"}
Load مدیریتی را افزایش ندهید.
:::

```{=html}
<h2 dir="rtl" align="right">
```
12. Password Protection کلاینت
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
در Group موردنظر:
:::

``` text
Clients
→ <Group>
→ Policies
→ Password Settings
```

::: {dir="rtl" align="right"}
گزینه‌های مهم:
:::

``` text
Require a password to uninstall the client
Require a password to run CleanWipe
Require a password to stop the client service
Require a password to import/export policy
```

::: {dir="rtl" align="right"}
اگر `Use the default client password` انتخاب شده باشد، رمز Site-level
:::

::: {dir="rtl" align="right"}
استفاده می‌شود و معمولاً به صورت Plain Text نمایش داده نمی‌شود. برای کنترل
:::

::: {dir="rtl" align="right"}
بهتر می‌توان:
:::

``` text
Use a group client password
```

::: {dir="rtl" align="right"}
را انتخاب و Password جدید تعریف کرد.
:::

```{=html}
<h2 dir="rtl" align="right">
```
13. Windows Server قدیمی و Azure Code Signing
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
اگر Installer خطای زیر داد:
:::

``` text
Symantec Endpoint Protection can only be installed on systems with Azure Code Signing support
```

::: {dir="rtl" align="right"}
Windows را Patch کنید. روی Windows Server 2022 با Build بسیار قدیمی،
:::

::: {dir="rtl" align="right"}
آخرین Cumulative Update را نصب و سیستم را Restart کنید.
:::

::: {dir="rtl" align="right"}
همچنین وجود Root Certificate موردنیاز Microsoft را بررسی کنید:
:::

``` powershell
Get-ChildItem Cert:\LocalMachine\Root |
Where-Object {
    $_.Subject -like "*Microsoft Identity Verification Root Certificate Authority 2020*"
}
```

```{=html}
<h2 dir="rtl" align="right">
```
14. نصب SEP روی Ubuntu 22.04
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
برای Package کامل Offline:
:::

``` bash
chmod +x LinuxInstaller.ubuntu22
sudo ./LinuxInstaller.ubuntu22 -- -g
```

::: {dir="rtl" align="right"}
اگر Dependencyها ناقص باشند:
:::

``` bash
sudo apt update
sudo apt install -y at auditd libelf-dev
```

::: {dir="rtl" align="right"}
و دوباره:
:::

``` bash
sudo ./LinuxInstaller.ubuntu22 -- -g
```

```{=html}
<h3 dir="rtl" align="right">
```
Secure Boot و MOK
```{=html}
</h3>
```
::: {dir="rtl" align="right"}
اگر Secure Boot فعال باشد و Moduleهای SEP لود نشوند:
:::

``` text
sisevt    not loaded
sisap     not loaded
```

::: {dir="rtl" align="right"}
Public Key محصول را به MOK اضافه کنید:
:::

``` bash
sudo mokutil --import /usr/lib/symantec/sdcssagent/driver/sis-key.der
```

::: {dir="rtl" align="right"}
قبل از Import می‌توانید Certificate را بررسی کنید:
:::

``` bash
openssl x509 \
  -inform DER \
  -in /usr/lib/symantec/sdcssagent/driver/sis-key.der \
  -noout -subject -issuer -serial -fingerprint -sha256
```

::: {dir="rtl" align="right"}
سپس Reboot کنید و در کنسول UEFI:
:::

``` text
Enroll MOK
→ Confirm
→ Enter temporary MOK password
→ Reboot
```

::: {dir="rtl" align="right"}
بعد از Boot:
:::

``` bash
sudo /usr/lib/symantec/status.sh
lsmod | grep -E 'sisevt|sisap'
```

::: {dir="rtl" align="right"}
Enroll کردن Public Signing Key معتبر Symantec/Broadcom به MOK، Secure
:::

::: {dir="rtl" align="right"}
Boot را Disable نمی‌کند.
:::

```{=html}
<h2 dir="rtl" align="right">
```
15. Group و Policy مخصوص Domain Controller
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
ساختار:
:::

``` text
Servers
└── Domain-Controllers
```

::: {dir="rtl" align="right"}
Policyهای پیشنهادی:
:::

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

::: {dir="rtl" align="right"}
از Exceptionهای گسترده مثل `C:\Windows\` یا `*.db` خودداری کنید.
:::

::: {dir="rtl" align="right"}
Exclusion باید Role-specific و حداقلی باشد.
:::

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

::: {dir="rtl" align="right"}
برای DC به صورت Pilot:
:::

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

::: {dir="rtl" align="right"}
اگر SEP Firewall جایگزین Windows Firewall شده و Ruleها تست شده‌اند:
:::

``` text
Disable Windows Firewall: Disable Once Only
```

::: {dir="rtl" align="right"}
اگر Windows Firewall با Domain GPO مدیریت می‌شود و Ruleهای مهم دارد،
:::

::: {dir="rtl" align="right"}
ابتدا:
:::

``` text
No Action
```

::: {dir="rtl" align="right"}
را نگه دارید تا Migration کنترل‌شده انجام شود.
:::

```{=html}
<h2 dir="rtl" align="right">
```
16. Firewall Rule Set پیشنهادی برای DC
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
اصل مهم: Rule عمومی `Allow all applications / Any / Any` در نهایت باید
:::

::: {dir="rtl" align="right"}
حذف یا Disable شود، اما فقط بعد از ساخت و تست Allow Ruleهای لازم.
:::

::: {dir="rtl" align="right"}
نمونه:
:::

  -------------------------------------------------------------------------
  Priority      Rule          Source            Service       Action
  ------------- ------------- ----------------- ------------- -------------
  1             DC-to-DC      DC IPs            Required AD   Allow
                                                traffic       

  2             DNS           Internal Networks TCP/UDP 53    Allow

  3             Kerberos      Internal Networks TCP/UDP 88    Allow

  4             NTP           Internal Networks UDP 123       Allow

  5             LDAP          Internal Networks TCP/UDP 389   Allow

  6             Kerberos      Internal Networks TCP/UDP 464   Allow
                Password                                      

  7             SMB           Internal Networks TCP 445       Allow

  8             RPC Endpoint  Internal Networks TCP 135       Allow
                Mapper                                        

  9             Dynamic RPC   Internal Networks TCP           Allow
                                                49152-65535   

  10            Global        Internal Networks TCP 3268/3269 Allow
                Catalog                                       

  11            LDAPS         Internal Networks TCP 636       Allow

  12            SEPM          SEPM IP           Required SEP  Allow
                Management                      traffic       

  13            Admin         Jump/Management   RDP/WinRM as  Allow
                Management    hosts             required      

  90            Block Other   Any               Any           Block + Log
                Inbound                                       
  -------------------------------------------------------------------------

::: {dir="rtl" align="right"}
برای DC-to-DC می‌توان در فاز اول `Service: Any` را فقط بین IPهای مشخص
:::

::: {dir="rtl" align="right"}
DCها مجاز کرد و بعداً محدودتر نمود.
:::

::: {dir="rtl" align="right"}
مواردی مثل UPnP، SSDP، Bonjour، LLMNR، Wireless EAPOL و VPN روی DC در
:::

::: {dir="rtl" align="right"}
صورت عدم نیاز باید Disable شوند.
:::

```{=html}
<h2 dir="rtl" align="right">
```
17. Rollout امن Firewall روی DC
    ```{=html}
    </h2>
    ```

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

::: {dir="rtl" align="right"}
SEP 14.4 به تنهایی **DLP کامل و Content-aware** نیست.
:::

::: {dir="rtl" align="right"}
`Application and Device Control` می‌تواند برای کنترل USB، Device و
:::

::: {dir="rtl" align="right"}
Application استفاده شود، اما برای Policyهایی مانند تشخیص اطلاعات حساس
:::

::: {dir="rtl" align="right"}
داخل فایل و جلوگیری از خروج محتوا از USB/Email/Web به محصول \*\*Symantec
:::

::: {dir="rtl" align="right"}
Data Loss Prevention\*\* یا DLP معادل نیاز است.
:::

```{=html}
<h2 dir="rtl" align="right">
```
19. عیب‌یابی RPC Error 1722
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
خطا:
:::

``` text
[SC] OpenSCManager FAILED 1722:
The RPC server is unavailable.
```

::: {dir="rtl" align="right"}
اگر TCP 135، TCP 445 و `ADMIN$` سالم هستند، Dynamic RPC و Windows
:::

::: {dir="rtl" align="right"}
Firewall را بررسی کنید.
:::

::: {dir="rtl" align="right"}
روی Client:
:::

``` powershell
Get-Service RpcSs,RpcEptMapper,DcomLaunch,RemoteRegistry
```

::: {dir="rtl" align="right"}
تست از Management Server:
:::

``` cmd
sc.exe \\<CLIENT-IP> query RemoteRegistry
```

::: {dir="rtl" align="right"}
پورت‌های Dynamic RPC:
:::

``` text
TCP 49152-65535
```

::: {dir="rtl" align="right"}
Firewall Rule را فقط از Management/SEPM Server به Client Network محدود
:::

::: {dir="rtl" align="right"}
کنید.
:::

```{=html}
<h2 dir="rtl" align="right">
```
20. ترتیب کلی اجرای پروژه
    ```{=html}
    </h2>
    ```

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

```{=html}
<h2 dir="rtl" align="right">
```
21. اصول امنیتی نهایی
    ```{=html}
    </h2>
    ```

::: {dir="rtl" align="right"}
• SEP را جایگزین Patch Management، Backup و Privileged Access
:::

```{=html}
<div dir="rtl" align="right">
```
    Management ندانید.</div>

::: {dir="rtl" align="right"}
• برای Domain Controllerها Backup قابل بازیابی و ترجیحاً
:::

```{=html}
<div dir="rtl" align="right">
```
    Immutable/Offline داشته باشید.</div>

::: {dir="rtl" align="right"}
• دسترسی RDP/WinRM به DC را به Jump Server یا Management Network محدود
:::

```{=html}
<div dir="rtl" align="right">
```
    کنید.</div>

::: {dir="rtl" align="right"}
• از ذخیره Passwordهای مدیریتی در Script و GPO به صورت Plain Text
:::

```{=html}
<div dir="rtl" align="right">
```
    خودداری کنید.</div>

::: {dir="rtl" align="right"}
• Policyهای Firewall و Application Control را ابتدا Pilot کنید.
:::

::: {dir="rtl" align="right"}
• Definition Update و Client Health را از SEPM مانیتور کنید.
:::

::: {dir="rtl" align="right"}
• برای مقابله با ransomware، محدودسازی lateral movement به اندازه خود
:::

```{=html}
<div dir="rtl" align="right">
```
    Antivirus اهمیت دارد.</div>

:::
