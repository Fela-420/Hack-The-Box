# Kerberoasting Explained

## Overview

Kerberoasting is an Active Directory attack technique where an authenticated domain user requests Kerberos service tickets (TGS tickets) for service accounts and attempts to recover weak service account passwords through offline password cracking.

Because the attack only requires a standard domain account and the password cracking happens offline, Kerberoasting remains one of the most common privilege escalation techniques in Active Directory environments.

> **Note:** Kerberoasting targets **service accounts**, not normal user accounts.

---

# How Kerberos Authentication Works

Before understanding Kerberoasting, it's helpful to know how Kerberos works.

```
User
 │
 │ Request Service Ticket
 ▼
Domain Controller (KDC)
 │
 │ Creates TGS Ticket
 │
 │ Encrypts ticket using the
 │ Service Account's password
 ▼
User receives TGS
 │
 ▼
Service
```

The important detail:

> **The service ticket is encrypted using the service account's password-derived key.**

If the password is weak, the ticket can be cracked offline.

---

# What is a Service Principal Name (SPN)?

A **Service Principal Name (SPN)** uniquely identifies a service running under a domain account.

Examples include:

```
HTTP/webserver
MSSQLSvc/SQL01
HOST/DC01
CIFS/FileServer
LDAP/DC01
```

Whenever a user accesses one of these services, Kerberos issues a service ticket.

---

# Why Kerberoasting Works

| Reason | Explanation |
|---------|-------------|
| Any authenticated user can request service tickets | No administrative privileges are required. |
| Tickets are encrypted using the service account's password | Weak passwords make offline cracking feasible. |
| Password cracking occurs offline | No further interaction with the domain controller is needed after obtaining the ticket. |
| Service accounts often have elevated privileges | Compromising one may provide access to sensitive resources. |

---

# Attack Workflow

```
Authenticate as Domain User
            │
            ▼
Enumerate Service Accounts (SPNs)
            │
            ▼
Request Kerberos Service Ticket (TGS)
            │
            ▼
Save Ticket
            │
            ▼
Offline Password Cracking
            │
            ▼
Recover Service Account Password
            │
            ▼
Authenticate Using Service Account
```

---

# Step 1 – Enumerate Service Accounts

The first step is identifying accounts with registered SPNs.

Example output:

```
SQLService
SAPService
BackupSvc
IISService
ExchangeSvc
```

These accounts become potential Kerberoasting targets.

---

# Step 2 – Request a Service Ticket

A Kerberos Ticket Granting Service (TGS) ticket is requested for the target service account.

The Domain Controller returns the encrypted service ticket.

```
User
   │
   │ Request TGS
   ▼
Domain Controller
   │
   │ Encrypt Ticket
   ▼
TGS Ticket
```

---

# Step 3 – Save the Ticket

The returned ticket is stored locally.

Example:

```
sqlservice_tgs
```

No further communication with the domain controller is required.

---

# Step 4 – Crack the Ticket Offline

Password cracking tools compare candidate passwords against the encrypted ticket until the correct password is found.

Because this occurs offline:

- No account lockouts
- No repeated authentication attempts
- Minimal network activity

---

# Step 5 – Recover the Password

Example:

```
Service Account:
SQLService

Recovered Password:
Winter2024!
```

The recovered credentials can then be used according to the permissions assigned to that service account.

---

# Common SPNs

| Service | Example SPN |
|----------|-------------|
| Microsoft SQL Server | MSSQLSvc/SQL01 |
| IIS Web Server | HTTP/Web01 |
| SMB File Sharing | CIFS/File01 |
| LDAP | LDAP/DC01 |
| Remote Host | HOST/SERVER01 |

---

# Common Targets

Service accounts frequently run:

- SQL Servers
- IIS Web Servers
- Backup Software
- Monitoring Applications
- Enterprise Applications
- Scheduled Tasks

---

# Why Service Accounts Matter

Service accounts often:

- Have passwords that change infrequently.
- Are used by critical applications.
- Possess elevated permissions.
- Access sensitive systems and databases.

These characteristics make them valuable targets if weak passwords are used.

---

# Defensive Recommendations

Organizations can reduce Kerberoasting risk by:

- Using long, randomly generated passwords for service accounts.
- Using Group Managed Service Accounts (gMSAs).
- Rotating service account passwords regularly.
- Monitoring for unusual volumes of Kerberos service ticket requests.
- Applying the principle of least privilege to service accounts.

---

# Key Takeaways

- Kerberoasting targets **service accounts with SPNs**.
- Any authenticated domain user can request Kerberos service tickets.
- The service ticket is encrypted using the service account's password-derived key.
- Offline password cracking avoids account lockouts and minimizes detection opportunities.
- Strong service account passwords and managed service accounts significantly reduce the effectiveness of Kerberoasting attacks.

---

## References

- Microsoft Kerberos Authentication
- MIT Kerberos Documentation
- Active Directory Security Best Practices
- Group Managed Service Accounts (gMSA)
