# Internal Password Spraying (Linux)

> **Objective:** Understand how password spraying works in Active Directory environments, recognize the attack workflow, and learn defensive considerations. This documentation is intended for authorized lab environments (HTB, labs, and approved internal penetration tests).

---

# Overview

Password spraying is an authentication attack where **one password** is tested against **many user accounts**.

Unlike brute force attacks, which test **many passwords against one account**, password spraying reduces the likelihood of triggering account lockout policies.

---

## Brute Force vs Password Spraying

| Brute Force | Password Spraying |
|-------------|------------------|
| Many passwords → One account | One password → Many accounts |
| High lockout risk | Lower lockout risk |
| Very noisy | Quieter |
| Easy to detect | Harder to detect |

---

# Why Password Spraying Works

## Human Nature

Many users choose passwords that are easy to remember.

Common examples include:

- Welcome1
- Password123
- CompanyName1
- Summer2026!
- Winter2026!

If password policies are weak, one common password may work for multiple accounts.

---

## Organizational Weaknesses

Password spraying succeeds because of:

- Weak password policies
- Password reuse
- Default passwords never changed
- Predictable seasonal passwords
- Lack of Multi-Factor Authentication (MFA)
- Poor password auditing

---

# Prerequisites

Before performing password spraying, you typically need:

- A list of valid usernames
- Knowledge of the password policy
- Domain Controller IP
- One or more common password candidates

Possible username sources include:

- OSINT
- LDAP enumeration
- Kerberos username enumeration
- Internal documentation
- Email formats

---

# Password Spraying Workflow

```text
Username Enumeration
        │
        ▼
Password Policy Discovery
        │
        ▼
Choose One Password
        │
        ▼
Authenticate Against Many Users
        │
        ▼
Successful Login?
        │
  ┌─────┴─────┐
  │           │
 No          Yes
  │           │
  ▼           ▼
Continue   Validate Credentials
                │
                ▼
      Internal Enumeration
```

---

# Method 1 — RPC Authentication

## Concept

RPC authentication uses Microsoft's Remote Procedure Call service.

Each username is authenticated individually.

If authentication succeeds, the server returns the authenticated account information.

---

## Protocol

| Service | Port |
|---------|------|
| RPC | TCP 135 |

---

## Authentication Flow

```text
Username
    │
Password
    │
    ▼
RPC Authentication
    │
    ▼
Authentication Result
    │
 ┌──┴──┐
 │     │
Fail  Success
 │      │
 ▼      ▼
Next  Record Credential
```

---

## Advantages

- Uses native Samba utilities
- No additional tools required
- Available on most Linux distributions

---

## Disadvantages

- One connection per user
- Slower than Kerberos
- More network traffic

---

# Method 2 — Kerberos Password Spraying

## Concept

Kerberos authentication requests are sent directly to the Domain Controller's Key Distribution Center (KDC).

If pre-authentication succeeds, the credentials are valid.

---

## Protocol

| Service | Port |
|---------|------|
| Kerberos | TCP/UDP 88 |

---

## Authentication Flow

```text
Username
     │
Password
     │
     ▼
KDC (Domain Controller)
     │
     ▼
Pre-Authentication
     │
 ┌───┴────┐
 │        │
Fail    Success
 │        │
 ▼        ▼
Stop   TGT Issued
            │
            ▼
    Valid Credentials
```

---

## Advantages

- Very fast
- Uses native Active Directory authentication
- Less SMB traffic
- Lower network noise

---

## Disadvantages

- Generates Kerberos authentication events
- Requires correct domain information
- Easier to detect through authentication logs

---

# Method 3 — SMB Authentication

## Concept

SMB authentication attempts to log into the Domain Controller using SMB.

Successful authentication confirms valid credentials.

---

## Protocol

| Service | Port |
|---------|------|
| SMB | TCP 445 |

---

## Authentication Flow

```text
Username List
      │
      ▼
SMB Authentication
      │
      ▼
Domain Controller
      │
      ▼
Authentication Result
```

---

## Advantages

- Fast
- Easy to validate credentials
- Useful for additional post-authentication enumeration

---

## Disadvantages

- Generates SMB authentication logs
- More visible than Kerberos
- Easier for defenders to monitor

---

# Credential Validation

After identifying a successful login, validate it before proceeding.

Validation confirms:

- Username is correct
- Password is correct
- Domain connectivity
- Accessible services
- Host information
- Authentication functionality

---

# Local Administrator Password Spraying

## Concept

Instead of authenticating against **domain accounts**, authentication is attempted against **local administrator accounts** across multiple systems.

This often occurs after compromising one machine and obtaining the local administrator credential.

---

## Attack Flow

```text
Compromise Host A
        │
Dump Local Administrator Credential
        │
        ▼
Authenticate to Host B
        │
        ▼
Authenticate to Host C
        │
        ▼
Authenticate to Host D
```

---

## Why It Works

Organizations sometimes deploy identical local administrator passwords using:

- Golden images
- Manual deployments
- Poor password management

---

## Mitigation

Microsoft LAPS assigns a **unique local administrator password** to every computer.

This prevents password reuse across systems.

---

# Complete Attack Lifecycle

```text
Username Enumeration
        │
        ▼
Password Policy Discovery
        │
        ▼
Password Selection
        │
        ▼
Password Spraying
        │
        ▼
Successful Credentials
        │
        ▼
Credential Validation
        │
        ▼
Internal Enumeration
        │
        ▼
Privilege Escalation
        │
        ▼
Lateral Movement
```

---

# Detection Opportunities

Security teams can detect password spraying by monitoring:

- Multiple failed logins
- One password attempted against many accounts
- Kerberos pre-authentication failures
- SMB authentication failures
- RPC authentication failures
- Authentication bursts from one source
- Logon failures across many users within a short period

---

# Mitigation

## Strong Password Policy

- Minimum 14 characters
- Complexity requirements
- Password history
- Block common passwords

---

## Multi-Factor Authentication (MFA)

MFA greatly reduces the effectiveness of stolen or guessed passwords.

---

## Account Lockout Policy

Implement reasonable lockout thresholds while monitoring for password spraying behavior.

---

## Microsoft LAPS

Randomize local administrator passwords on every workstation.

---

## Security Monitoring

Monitor:

- Kerberos failures
- SMB failures
- RPC authentication attempts
- Authentication anomalies
- Password spraying patterns

---

## User Awareness

Educate users to avoid:

- Seasonal passwords
- Company-name passwords
- Default passwords
- Password reuse

---

# Key Takeaways

- Password spraying tests **one password against many accounts**.
- It is designed to avoid triggering account lockout policies.
- Common authentication protocols include:
  - Kerberos (TCP/UDP 88)
  - SMB (TCP 445)
  - RPC (TCP 135)
- Successful credentials should always be validated before further assessment.
- Strong password policies, MFA, LAPS, monitoring, and user awareness significantly reduce the effectiveness of password spraying attacks.

---
