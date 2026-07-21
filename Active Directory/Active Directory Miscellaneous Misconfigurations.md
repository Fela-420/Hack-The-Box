# Active Directory Miscellaneous Misconfigurations

> Educational Purpose: This guide explains common Active Directory security misconfigurations found during authorized security assessments and penetration testing labs.

---

# Overview

This section covers common Active Directory weaknesses that are often overlooked and can lead to:

- Credential exposure
- Account compromise
- Privilege escalation
- Domain compromise

---

# Common Misconfigurations

| Misconfiguration | Location | Security Impact |
|---|---|---|
| Passwords in Description Fields | User account descriptions | Credentials exposed |
| PASSWD_NOTREQD Flag | UserAccountControl attribute | Weak or empty passwords |
| ASREPRoasting | Accounts without Kerberos pre-authentication | Offline password cracking |
| Group Policy Preferences (GPP) Passwords | SYSVOL Policies | Password recovery |
| SYSVOL Scripts | Login scripts and automation files | Hardcoded credentials |
| Exchange Permissions | Exchange security groups | Possible DCSync abuse |
| Printer Bug | MS-RPRN protocol | Authentication coercion |
| AD DNS Records | Internal DNS database | Hidden host discovery |

---

# 1. Passwords Stored in Description Fields

Administrators sometimes store sensitive information inside Active Directory description attributes.

## Enumeration

```powershell
Get-DomainUser * | Where-Object {$_.Description -ne $null} | Select samaccountname,description

Why This Matters

Description fields may contain:

Temporary passwords
Admin notes
Service account information
Sensitive operational details

Example:

Username: backupadmin

Description:
Password changed to Summer2025!
Risk

Sensitive information stored in AD attributes can often be read by normal domain users.

2. PASSWD_NOTREQD Account Discovery

The PASSWD_NOTREQD flag means an account may not require a password.

Find Accounts
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select samaccountname,useraccountcontrol
Why This Matters

Accounts configured with this flag may allow:

Weak passwords
Empty passwords
Easier account compromise
Security Recommendations
Remove PASSWD_NOTREQD where unnecessary
Enforce strong password policies
Audit userAccountControl settings
3. ASREPRoasting
Concept

ASREPRoasting targets accounts where Kerberos pre-authentication is disabled.

Normal authentication:

User Login Request
        |
        ↓
Kerberos Pre-Authentication
        |
        ↓
Authentication

Without pre-authentication:

Request AS-REP Ticket
        |
        ↓
Encrypted Response Returned
        |
        ↓
Offline Password Audit
Find ASREPRoastable Users
Get-DomainUser -PreauthNotRequired | Select samaccountname
Request AS-REP Hash

Using Rubeus:

.\Rubeus.exe asreproast /user:USERNAME /nowrap /format:hashcat
Crack AS-REP Hash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt --show
4. Group Policy Preferences (GPP) Passwords
Overview

Older Group Policy Preferences allowed administrators to store credentials inside XML files.

Location:

SYSVOL

Example:

\\DOMAIN\SYSVOL\DOMAIN\Policies\
Why It Is Dangerous

GPP passwords were encrypted using a publicly known AES key.

Users with SYSVOL access could recover these credentials.

Decrypt Password
gpp-decrypt [cpassword_value]
Find GPP Passwords
crackmapexec smb DC_IP -u user -p password -M gpp_password

Check autologin files:

crackmapexec smb DC_IP -u user -p password -M gpp_autologin
5. SYSVOL Scripts
Overview

SYSVOL commonly contains login scripts and automation scripts.

Common files:

*.bat
*.cmd
*.vbs
*.ps1
Search For
password
username
admin
credential
net use

Example:

type \\DOMAIN\SYSVOL\DOMAIN\scripts\login.bat
Risk

Hardcoded credentials may lead to:

Account compromise
Lateral movement
Privilege escalation
6. Exchange Permissions Abuse
Overview

Exchange security groups may have powerful Active Directory permissions.

Example:

Exchange Windows Permissions
Risk

Misconfigured Exchange permissions can lead to:

ACL abuse
Replication privilege abuse
DCSync attacks
Audit

Review:

Exchange security groups
Delegated permissions
Replication privileges
7. Printer Bug (MS-RPRN)
Overview

The Printer Bug abuses the Windows Print Spooler service.

It allows forcing a machine to authenticate to another host.

Why It Matters

Authentication coercion can lead to:

NTLM relay attacks
Credential exposure
Domain compromise
Check Printer Spooler
Get-SpoolStatus -ComputerName TARGET
Recommendations
Disable Print Spooler where unnecessary
Enable SMB signing
Monitor authentication events
8. Active Directory DNS Enumeration
Overview

Active Directory DNS records can reveal internal infrastructure.

Possible discoveries:

Hidden servers
Development systems
Internal applications
Management hosts
Enumerate DNS Records
adidnsdump -u domain\\user ldap://DC_IP -r
9. Finding GPO Misconfigurations

Group Policy Objects may contain dangerous permissions.

Enumeration
Get-DomainGPO | Get-ObjectAcl | Where-Object {$_.SecurityIdentifier -eq $sid}
Quick Reference
Attack	Hashcat Mode	Tool
ASREPRoast	18200	Rubeus, GetNPUsers.py
Kerberoast	13100	Rubeus, GetUserSPNs.py
GPP Password Recovery	-	gpp-decrypt
Lab Findings
Question	Answer
PASSWD_NOTREQD User	ygroce
ASREPRoast Password	Pass@word
Detection Opportunities

Monitor:

Unusual Kerberos AS-REQ requests
Suspicious account changes
SYSVOL access patterns
Group Policy modifications
Authentication coercion attempts
Abnormal DNS queries
Defensive Recommendations
Never store passwords in descriptions
Remove PASSWD_NOTREQD flags
Enforce Kerberos pre-authentication
Remove old GPP password files
Rotate exposed credentials
Audit SYSVOL permissions
Disable unnecessary Print Spooler services
Monitor privileged group changes
Review Exchange permissions
Perform regular Active Directory audits
Key Takeaways
Small Active Directory mistakes can create major security risks.
User descriptions may expose credentials.
Kerberos weaknesses allow offline password auditing.
SYSVOL should never contain plaintext credentials.
Group permissions must be reviewed regularly.
DNS enumeration can reveal hidden infrastructure.
Proper auditing prevents privilege escalation paths.
References
MITRE ATT&CK: T1558.004 - AS-REP Roasting
MITRE ATT&CK: T1552 - Unsecured Credentials
Microsoft Active Directory Security Documentation
BloodHound Documentation
PowerView Documentation
