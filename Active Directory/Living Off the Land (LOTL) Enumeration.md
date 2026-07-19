# Living Off the Land (LOTL) Enumeration

## Overview

Living Off the Land (LOTL) is the practice of using **native Windows utilities** already installed on a system instead of uploading third-party tools.

This approach is useful when security controls such as antivirus, EDR, application whitelisting, or restricted internet access prevent external tools from being executed.

> **Goal:** Collect as much information as possible using built-in Windows and Active Directory commands.

---

# Why Use LOTL?

| Challenge | Why It Happens | Native Alternative |
|------------|----------------|--------------------|
| External tools blocked | Antivirus or EDR | Built-in Windows commands |
| Internet access restricted | Firewall or proxy | Use installed utilities |
| Executables quarantined | Security policies | Native binaries (LOLBins) |
| Uploads monitored | Endpoint monitoring | Existing system tools |

---

# Enumeration Workflow

```
Host Enumeration
        │
        ▼
PowerShell Inspection
        │
        ▼
Security Controls
        │
        ▼
Logged-in Users
        │
        ▼
Network Enumeration
        │
        ▼
WMI Enumeration
        │
        ▼
Active Directory Enumeration
        │
        ▼
Shares & Resources
```

---

# Part 1 — Basic Host Enumeration

Gather basic information about the host.

## Computer Name

```cmd
hostname
```

Displays the hostname.

---

## Current User

```cmd
whoami
```

Shows the current security context.

---

## Operating System

```cmd
systeminfo
```

Useful information includes:

- Windows Version
- Build Number
- Installed Hotfixes
- Boot Time
- Domain
- Memory

---

## Network Configuration

```cmd
ipconfig /all
```

Look for:

- IPv4 Address
- DNS Servers
- DHCP Server
- Domain Name
- Default Gateway

---

## Environment Variables

```cmd
set
```

or

```powershell
Get-ChildItem Env:
```

Useful variables include:

```
USERNAME
USERDOMAIN
COMPUTERNAME
APPDATA
TEMP
```

---

## Domain Information

```cmd
echo %USERDOMAIN%
```

Returns:

```
INLANEFREIGHT
```

---

## Domain Controller

```cmd
echo %LOGONSERVER%
```

Example:

```
\\DC01
```

---

# Part 2 — PowerShell Enumeration

## Installed Modules

```powershell
Get-Module -ListAvailable
```

---

## PowerShell Version

```powershell
Get-Host
```

---

## Execution Policies

```powershell
Get-ExecutionPolicy -List
```

---

## Bypass Execution Policy (Current Session)

```powershell
Set-ExecutionPolicy Bypass -Scope Process
```

Only affects the current PowerShell session.

---

## Environment Variables

```powershell
Get-ChildItem Env:
```

---

## PowerShell History

```powershell
Get-Content (Get-PSReadLineOption).HistorySavePath
```

May reveal:

- Credentials
- Administrative commands
- Previous scripts

---

# Part 3 — Security Controls

## Windows Firewall

```cmd
netsh advfirewall show allprofiles
```

Check whether profiles are:

- ON
- OFF

---

## Windows Defender Service

```cmd
sc query windefend
```

---

## Defender Status

```powershell
Get-MpComputerStatus
```

Interesting fields include:

- AntivirusEnabled
- RealTimeProtectionEnabled
- AMRunningMode
- AntivirusSignatureVersion

---

# Part 4 — Logged-In Users

## Interactive Sessions

```cmd
qwinsta
```

Example:

```
console
Administrator
Active
```

Useful for identifying active interactive sessions.

---

# Part 5 — Network Enumeration

## ARP Cache

```cmd
arp -a
```

Shows recently contacted systems.

---

## Routing Table

```cmd
route print
```

Useful for identifying reachable networks.

---

## IP Configuration

```cmd
ipconfig /all
```

Review:

- DNS
- Gateway
- DHCP
- Interface information

---

# Part 6 — Windows Management Instrumentation (WMI)

## Installed Hotfixes

```cmd
wmic qfe get Caption,Description,HotFixID
```

---

## Computer Information

```cmd
wmic computersystem get Name,Domain,Manufacturer,Model
```

---

## Running Processes

```cmd
wmic process list /format:list
```

---

## Domain Information

```cmd
wmic ntdomain list /format:list
```

---

## User Accounts

```cmd
wmic useraccount list /format:list
```

---

## Local Groups

```cmd
wmic group list /format:list
```

---

# Part 7 — Net Commands

## Password Policy

```cmd
net accounts
```

---

## Domain Password Policy

```cmd
net accounts /domain
```

---

## Domain Users

```cmd
net user /domain
```

---

## Specific User

```cmd
net user username /domain
```

---

## Domain Groups

```cmd
net group /domain
```

---

## Domain Admins

```cmd
net group "Domain Admins" /domain
```

---

## Local Administrators

```cmd
net localgroup Administrators
```

---

## Network Shares

```cmd
net share
```

---

## Computers

```cmd
net view
```

---

## Shares on Remote Computer

```cmd
net view \\HOSTNAME
```

---

## Alternative Binary

```cmd
net1 user
```

`net1.exe` performs the same functions as `net.exe`.

---

# Part 8 — Active Directory Enumeration with Dsquery

> Available on systems with RSAT or Domain Controller tools installed.

---

## List Users

```cmd
dsquery user
```

---

## List Computers

```cmd
dsquery computer
```

---

## List Groups

```cmd
dsquery group
```

---

## Search an OU

```cmd
dsquery * "OU=Employees,DC=corp,DC=local"
```

---

## Disabled Users

```cmd
dsquery user -disabled
```

---

## Display Disabled User Information

```cmd
dsquery user -disabled | dsget user -samid -display -desc
```

Descriptions occasionally contain operational notes or other information that administrators intended for internal use.

---

# Part 9 — LDAP Filters

Search for disabled accounts.

```cmd
dsquery * -filter "(&(objectCategory=person)(userAccountControl:1.2.840.113556.1.4.803:=2))"
```

---

## Common UserAccountControl Values

| Value | Meaning |
|--------|----------|
| 512 | Normal User |
| 514 | Disabled User |
| 4096 | Workstation Trust Account |
| 8192 | Domain Controller |

---

# Part 10 — Enumeration Checklist

## Host

```cmd
hostname

whoami

systeminfo
```

---

## Network

```cmd
ipconfig /all

arp -a

route print
```

---

## Security Controls

```cmd
netsh advfirewall show allprofiles

sc query windefend
```

```powershell
Get-MpComputerStatus
```

---

## Active Directory

```cmd
net user /domain

net group /domain

net group "Domain Admins" /domain

net localgroup Administrators

dsquery user

dsquery computer

dsquery user -disabled | dsget user -samid -display -desc
```

---

## Network Resources

```cmd
net share

net view

net view \\HOSTNAME
```

---

# Key Takeaways

- Living Off the Land relies entirely on native Windows tools.
- Native commands often blend with legitimate administrative activity.
- Valuable information can be gathered without installing external software.
- Understanding Windows command-line utilities is a fundamental skill for both penetration testers and defenders.

---

## References

- Microsoft Windows Command Reference
- Microsoft PowerShell Documentation
- Active Directory Administration
- Windows Management Instrumentation (WMI)
- LDAP Query Syntax
