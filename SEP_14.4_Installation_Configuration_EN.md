# Symantec Endpoint Protection 14.4 Installation and Configuration Guide

## 1. Purpose and Architecture

This guide covers deployment of **Symantec Endpoint Protection Manager
(SEPM) 14.4** and **SEP 14.4** clients in an enterprise network. It
includes Windows and Linux installation, Kaspersky migration,
centralized deployment, LiveUpdate, workstation/server/domain-controller
policies, and remote-deployment troubleshooting.

> **Security note:** Do not store real administrative passwords,
> credentials, or sensitive infrastructure details in scripts, GPOs, or
> Git documentation. This guide uses placeholders.

## 2. Recommended Group Structure

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

Do not use one generic server policy for Domain Controllers, database
servers, file servers, and application servers.

## 3. Installation Packages

In SEPM:

``` text
Admin
→ Install Packages
→ Client Install Packages
```

The Windows x64 SEP package is used for both Windows workstations and
Windows Servers. Server/workstation differences are implemented through
**Client Install Feature Sets**, **Client Install Settings**, and group
policies.

Typical base package:

``` text
Symantec Endpoint Protection 14.4 for WIN64BIT
```

For Linux use:

``` text
Symantec Agent for Linux 14.4
```

## 4. Recommended Feature Sets

### Workstations

``` text
Virus and Spyware Protection
SONAR
Download Insight
Intrusion Prevention
Firewall
Application and Device Control
```

### Windows Servers

``` text
Virus and Spyware Protection       Enable
SONAR                              Enable
Intrusion Prevention              Enable / Pilot
Firewall                          Enable / Pilot
Application and Device Control    Role dependent / Pilot
```

Full server protection is reasonable, but Firewall and Application
Control must be tested with role-specific policies.

## 5. Migrating from Kaspersky Endpoint Security

Scenario:

``` text
Kaspersky Endpoint Security for Windows 12.4.x
+ Kaspersky Network Agent
```

If uninstall password protection is enabled, the preferred method is to
remove KES from **Kaspersky Security Center** first.

Sequence:

``` text
Authorize/disable Kaspersky uninstall protection
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

Do not remove Network Agent before KES if it is required for the remote
uninstall task.

Do not rely on SEPM third-party removal to bypass Kaspersky password
protection.

## 6. Windows Remote Deployment

Use:

``` text
Client Deployment Wizard
```

In an Active Directory environment, prefer `Search Network`, IP ranges,
or computer names rather than depending on legacy `Browse Network`.

### Remote Push Prerequisites

From the SEPM server:

``` powershell
Test-NetConnection <CLIENT-IP> -Port 445
Test-NetConnection <CLIENT-IP> -Port 135
```

Administrative share test:

``` cmd
net use \\<CLIENT-IP>\admin$ /user:<DOMAIN>\<DEPLOY-ACCOUNT> *
```

Remote Service Control test:

``` cmd
sc.exe \\<CLIENT-IP> query RemoteRegistry
```

Remote Registry test:

``` cmd
reg query \\<CLIENT-IP>\HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion
```

### Important Ports

``` text
TCP 445              SMB / ADMIN$
TCP 135              RPC Endpoint Mapper
TCP 49152-65535      Dynamic RPC on modern Windows
```

Do not expose Dynamic RPC broadly. Restrict the source to the
management/SEPM server wherever possible.

## 7. Enable Remote Registry with GPO

Path:

``` text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ System Services
→ Remote Registry
```

Configure:

``` text
Define this policy setting: Enabled
Startup mode: Automatic
```

Then:

``` cmd
gpupdate /force
```

Verify:

``` powershell
Get-Service RemoteRegistry
```

## 8. Windows Firewall through GPO

To disable Windows Firewall via GPO:

``` text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Windows Defender Firewall with Advanced Security
→ Windows Defender Firewall Properties
```

For the required profile:

``` text
Firewall state: Off
```

Verify:

``` powershell
Get-NetFirewallProfile | Select-Object Name,Enabled
```

> On production systems, especially Domain Controllers, disable Windows
> Firewall only after SEP Firewall is installed, enabled, and validated.
> Never leave both host firewalls disabled.

## 9. SEPM LiveUpdate

In SEPM:

``` text
Admin
→ Servers
→ Local Site
```

Review LiveUpdate downloads and the latest content revision.

Manual content download:

``` text
Admin
→ Servers
→ Local Site
→ Download LiveUpdate Content
```

Example SEPM interval:

``` text
Every 4 hours
```

## 10. Client Updates from SEPM

Policy:

``` text
Policies
→ LiveUpdate
→ LiveUpdate Settings
```

For internal clients:

``` text
Use the default management server    Enabled
Direct Internet LiveUpdate           Disabled when not required
```

On a client:

``` text
SEP
→ Help
→ Troubleshooting
```

The Server and Group fields should identify the internal SEPM.

Typical connectivity test:

``` powershell
Test-NetConnection <SEPM-IP> -Port 8014
```

Compare SEPM and client definition revisions to verify actual content
delivery.

## 11. Communication Settings

Typical medium-sized environment:

``` text
Mode: Pull
Heartbeat: 30 minutes
Download Randomization: Enabled
```

Shorter server heartbeat intervals are possible, but avoid unnecessary
management load.

## 12. Client Password Protection

For the target group:

``` text
Clients
→ <Group>
→ Policies
→ Password Settings
```

Important options:

``` text
Require a password to uninstall the client
Require a password to run CleanWipe
Require a password to stop the client service
Require a password to import/export policy
```

When `Use the default client password` is selected, the site-level
default password is used and is normally not displayed in plaintext. To
take explicit control, select:

``` text
Use a group client password
```

and define a new group password.

## 13. Azure Code Signing Prerequisites on Windows

If installation fails with:

``` text
Symantec Endpoint Protection can only be installed on systems with Azure Code Signing support
```

patch Windows first. On an old Windows Server 2022 build, install a
current cumulative update and reboot.

Also verify the required Microsoft root certificate:

``` powershell
Get-ChildItem Cert:\LocalMachine\Root |
Where-Object {
    $_.Subject -like "*Microsoft Identity Verification Root Certificate Authority 2020*"
}
```

## 14. SEP Installation on Ubuntu 22.04

For a full offline package:

``` bash
chmod +x LinuxInstaller.ubuntu22
sudo ./LinuxInstaller.ubuntu22 -- -g
```

If dependencies are missing:

``` bash
sudo apt update
sudo apt install -y at auditd libelf-dev
```

Retry:

``` bash
sudo ./LinuxInstaller.ubuntu22 -- -g
```

### Secure Boot and MOK

If Secure Boot is enabled and SEP kernel modules remain unloaded:

``` text
sisevt    not loaded
sisap     not loaded
```

Import the product public signing certificate into MOK:

``` bash
sudo mokutil --import /usr/lib/symantec/sdcssagent/driver/sis-key.der
```

Optionally verify the certificate first:

``` bash
openssl x509 \
  -inform DER \
  -in /usr/lib/symantec/sdcssagent/driver/sis-key.der \
  -noout -subject -issuer -serial -fingerprint -sha256
```

Reboot and use the UEFI console:

``` text
Enroll MOK
→ Confirm
→ Enter the temporary MOK password
→ Reboot
```

Verify after boot:

``` bash
sudo /usr/lib/symantec/status.sh
lsmod | grep -E 'sisevt|sisap'
```

Enrolling a verified Symantec/Broadcom public signing key does not
disable Secure Boot.

## 15. Dedicated Domain Controller Policies

Recommended group:

``` text
Servers
└── Domain-Controllers
```

Recommended policies:

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

Avoid broad exclusions such as `C:\Windows\` or `*.db`. Keep exclusions
minimal and role-specific.

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

Pilot these settings on a DC:

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

If SEP Firewall has been validated as the host firewall:

``` text
Disable Windows Firewall: Disable Once Only
```

If Windows Firewall is still centrally managed by Domain GPO and
contains important rules, keep:

``` text
No Action
```

until a controlled migration is completed.

## 16. Suggested DC Firewall Rule Set

The generic `Allow all applications / Any / Any` rule should eventually
be disabled, but only after explicit AD rules are created and tested.

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

For initial DC-to-DC validation, `Service: Any` can be allowed only
between explicitly listed DC IPs, then tightened later.

Disable unnecessary DC services/rules such as UPnP, SSDP, Bonjour,
LLMNR, wireless EAPOL, and VPN when they are not required.

## 17. Safe DC Firewall Rollout

``` text
Create explicit Allow rules
        ↓
Keep existing Allow-All temporarily
        ↓
Apply only to a Pilot DC
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

## 18. DLP Capability

SEP 14.4 alone is **not a full content-aware DLP product**.

`Application and Device Control` can restrict USB devices, applications,
and device classes. For content-aware policies such as identifying
sensitive data inside files and blocking exfiltration through USB,
email, or web channels, use **Symantec Data Loss Prevention** or an
equivalent DLP platform.

## 19. Troubleshooting RPC Error 1722

Error:

``` text
[SC] OpenSCManager FAILED 1722:
The RPC server is unavailable.
```

If TCP 135, TCP 445, and `ADMIN$` already work, check Dynamic RPC and
the Windows host firewall.

On the client:

``` powershell
Get-Service RpcSs,RpcEptMapper,DcomLaunch,RemoteRegistry
```

From the management server:

``` cmd
sc.exe \\<CLIENT-IP> query RemoteRegistry
```

Modern Windows Dynamic RPC range:

``` text
TCP 49152-65535
```

Restrict firewall access to the SEPM/management server rather than
opening this range broadly.

## 20. Recommended Project Sequence

``` text
1. Upgrade/patch SEPM and operating systems
2. Create groups
3. Create a Pilot group
4. Create Windows/Linux packages
5. Configure LiveUpdate
6. Configure client communication
7. Configure password protection
8. Remove the previous antivirus
9. Deploy SEP to Pilot clients
10. Validate connectivity and definitions
11. Deploy workstations
12. Deploy Windows Servers
13. Deploy Linux Servers
14. Build dedicated Domain Controller policies
15. Pilot DC Firewall/IPS
16. Monitor logs and tune policies
17. Roll out to remaining systems
18. Configure reporting and alerts
```

## 21. Final Security Principles

-   SEP does not replace patch management, backup, or privileged-access
    controls.
-   Maintain tested, recoverable, preferably immutable/offline backups
    for Domain Controllers.
-   Restrict RDP/WinRM access to DCs to jump servers or management
    networks.
-   Never store administrative passwords in plaintext scripts or GPOs.
-   Pilot Firewall and Application Control policies before broad
    deployment.
-   Monitor definition status and endpoint health from SEPM.
-   Ransomware defense requires lateral-movement containment in addition
    to endpoint antivirus.
