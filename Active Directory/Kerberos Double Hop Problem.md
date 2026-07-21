# Kerberos Double Hop Problem - Active Directory Lab Documentation

> **Educational Purpose:**  
> This documentation explains the Kerberos Double Hop issue during authorized Active Directory security assessments and how authentication delegation affects remote enumeration.

---

# Lab Objective

The goal of this lab is to understand why Active Directory enumeration tools fail when executed through a remote WinRM session and how to properly authenticate when accessing additional network resources.

Scenario:

```
Linux Attack Host
        |
        | Evil-WinRM
        ↓
Windows Target
        |
        | PowerView needs DC access
        ↓
Domain Controller
```

The problem occurs because the user's credentials are not automatically forwarded to the second system.

---

# Lab Environment

| Host | Purpose | Credentials |
|---|---|---|
| ATTACK01 | Linux attack machine | htb-student:HTB_@cademy_stdnt! |
| MS01 | Windows target | htb-student:Academy_student_AD! |
| Domain Controller | Active Directory services | Domain user |

---

# What is the Kerberos Double Hop Problem?

The Double Hop problem occurs when:

1. A user authenticates to a remote machine.
2. That machine attempts to access another resource.
3. The original credentials are not available for the second connection.

Example:

```
User
 |
 | Authentication Hop 1
 ↓
Windows Server
 |
 | Authentication Hop 2
 ↓
Domain Controller
```

The first authentication works.

The second authentication fails because the Kerberos ticket is not forwarded.

---

# Why Does This Happen?

When connecting using:

```
evil-winrm
```

the remote session receives only a ticket for the target machine.

The credentials are not cached for additional authentication.

Checking tickets:

```powershell
klist
```

Output:

```
Only one ticket exists
Target: HOST/MS01
```

There is no Domain Controller ticket.

---

# Step 1 - Connect Using Evil-WinRM

From the Linux attack host:

```bash
evil-winrm -i 172.16.5.X -u forend -p Klmcargo2
```

or:

```bash
evil-winrm -i 172.16.5.X -u bdavis -p [PASSWORD]
```

You now have a remote PowerShell session.

---

# Step 2 - Check Kerberos Tickets

Inside the Evil-WinRM session:

```powershell
klist
```

Example:

```
Cached Tickets: (1)

Server: HOST/MS01
```

The session only contains a ticket for the current host.

---

# Step 3 - Attempt Active Directory Enumeration

Load PowerView:

```powershell
Import-Module .\PowerView.ps1
```

Attempt enumeration:

```powershell
Get-DomainUser -SPN
```

Result:

```
Exception calling "FindAll"
An operations error occurred.
```

The command fails because PowerView needs to communicate with Active Directory services but cannot authenticate to the Domain Controller.

---

# Solution 1 - Use PSCredential

Create a credential object:

```powershell
$SecPassword = ConvertTo-SecureString 'Klmcargo2' -AsPlainText -Force

$Cred = New-Object System.Management.Automation.PSCredential(
'INLANEFREIGHT\forend',
$SecPassword
)
```

Pass credentials to PowerView:

```powershell
Get-DomainUser -SPN -Credential $Cred | Select samaccountname
```

Result:

```
samaccountname
--------------
sqlservice
backupsvc
websvc
```

The command works because PowerView now has credentials to authenticate to Active Directory.

---

# Solution 2 - Use RDP Instead of WinRM

When connecting through RDP:

```
Linux
 |
 | RDP
 ↓
Windows Host
```

Windows caches the user's credentials.

PowerView commands can run normally:

```powershell
Get-DomainUser -SPN
```

No additional credential parameter is required.

---

# Command Comparison

## Evil-WinRM

Fails:

```powershell
Get-DomainUser -SPN
```

Reason:

```
No forwarded Kerberos credentials
```

---

## Evil-WinRM With Credentials

Works:

```powershell
Get-DomainUser -SPN -Credential $Cred
```

Reason:

```
Explicit authentication provided
```

---

## RDP Session

Works:

```powershell
Get-DomainUser -SPN
```

Reason:

```
Credentials are available in the session
```

---

# Understanding the Authentication Flow

## Evil-WinRM

```
User
 |
 | Kerberos Authentication
 ↓
Windows Host
 |
 | No credential delegation
 ↓
Domain Controller
 |
 X Authentication fails
```

---

## RDP

```
User
 |
 | Interactive Login
 ↓
Windows Host
 |
 | Cached Credentials Available
 ↓
Domain Controller
 |
 ✓ Authentication succeeds
```

---

# Detection Opportunities

Security teams can monitor:

- Remote PowerShell sessions
- WinRM authentication activity
- Unusual Active Directory queries
- Excessive LDAP enumeration
- Suspicious remote administration behavior

Important events:

| Event ID | Description |
|---|---|
| 4624 | Successful logon |
| 4648 | Explicit credential usage |
| 4768 | Kerberos authentication request |
| 4769 | Kerberos service ticket request |

---

# Defensive Recommendations

Organizations should:

- Limit unnecessary WinRM access
- Monitor remote PowerShell usage
- Use least privilege permissions
- Restrict administrative accounts
- Configure Kerberos delegation securely
- Monitor abnormal Active Directory enumeration

---

# Key Takeaways

- WinRM sessions do not automatically forward Kerberos credentials.
- The Double Hop problem occurs when accessing a second resource.
- `klist` shows available Kerberos tickets.
- PowerView requires valid authentication to query Active Directory.
- Passing credentials with `-Credential` solves many remote enumeration issues.
- RDP behaves differently because it creates an interactive logon session.

---

# References

- Microsoft Kerberos Authentication Documentation
- Microsoft PowerShell Remoting Documentation
- PowerView Documentation
- Active Directory Security Documentation
