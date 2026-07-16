# Internal Password Spraying

## Overview

Password spraying is an authentication attack in which a single password is tested against multiple user accounts.

Unlike traditional brute-force attacks, which try many passwords against one account, password spraying distributes authentication attempts across many accounts to reduce the likelihood of triggering account lockout policies.

---

# Objectives

After completing this lab, you should understand:

- How password spraying differs from brute forcing
- Why organizations are vulnerable to password spraying
- Authentication protocols commonly involved
- Indicators defenders can monitor
- Security controls that mitigate password spraying

---

# Password Spraying vs Brute Force

| Brute Force | Password Spraying |
|-------------|------------------|
| Many passwords against one account | One password against many accounts |
| High lockout risk | Lower lockout risk |
| Very noisy | Lower volume per account |
| Easier to detect | Often more difficult to identify |

---

# Why Password Spraying Works

Common contributing factors include:

- Weak passwords
- Password reuse
- Predictable seasonal passwords
- Default passwords
- Missing MFA
- Weak password policies

Examples of predictable passwords include:

- Welcome1
- Summer2026
- Winter2026
- CompanyName1

---

# Common Authentication Services

Password spraying may target authentication services such as:

- Active Directory
- Kerberos
- SMB
- LDAP
- VPN authentication
- Web applications using AD authentication
- Microsoft 365
- Outlook Web Access
- Citrix
- Remote Desktop gateways

---

# General Assessment Workflow

```text
Collect Usernames
        │
        ▼
Understand Password Policy
        │
        ▼
Assess Authentication Controls
        │
        ▼
Perform Authorized Testing
        │
        ▼
Identify Valid Credentials
        │
        ▼
Validate Findings
        │
        ▼
Document Results
```

---

# Common Tools

Security professionals may use assessment tools capable of evaluating authentication controls during authorized engagements.

Examples include:

- Kerberos assessment tools
- PowerShell-based Active Directory assessment scripts
- SMB assessment frameworks

These tools should only be used in environments where explicit authorization has been granted.

---

# Detection Opportunities

Blue teams should monitor for:

- Multiple failed authentication attempts
- Authentication attempts against many users from one source
- Kerberos pre-authentication failures
- SMB authentication failures
- Authentication attempts occurring over extended periods
- Unusual login locations
- Unusual login timing

---

# Windows Security Events

| Event ID | Description |
|-----------|-------------|
| 4625 | Failed logon |
| 4624 | Successful logon |
| 4771 | Kerberos pre-authentication failure |
| 4768 | Kerberos TGT request |
| 4769 | Kerberos service ticket request |

---

# Indicators of Password Spraying

Possible indicators include:

- Many users targeted with the same authentication pattern
- Numerous failed logins from one IP
- Authentication attempts outside business hours
- Repeated failures followed by a successful login
- Distributed authentication failures across many accounts

---

# Mitigations

## Multi-Factor Authentication

MFA significantly reduces the effectiveness of password-only attacks.

---

## Strong Password Policy

Implement:

- Long passphrases
- Password history
- Complexity requirements
- Common password blocklists

---

## Account Lockout

Configure appropriate lockout thresholds while monitoring for distributed authentication attempts.

---

## Microsoft LAPS

Assign unique local administrator passwords to every workstation to prevent password reuse.

---

## Least Privilege

Provide separate privileged accounts for administrators and restrict administrative access.

---

## Continuous Monitoring

Monitor:

- Authentication logs
- Kerberos events
- SMB authentication
- VPN authentication
- Microsoft 365 sign-ins
- Identity protection alerts

---

# Password Spraying Lifecycle

```text
Reconnaissance
      │
      ▼
User Enumeration
      │
      ▼
Password Policy Review
      │
      ▼
Authentication Assessment
      │
      ▼
Credential Validation
      │
      ▼
Post-Assessment Documentation
```

---

# Defensive Best Practices

- Require MFA for all users
- Disable legacy authentication
- Enforce long passphrases
- Monitor failed logins
- Alert on authentication anomalies
- Deploy Microsoft LAPS
- Educate users about password hygiene
- Audit password policies regularly

---

# Key Takeaways

- Password spraying tests one password against many accounts.
- It attempts to avoid account lockout policies by limiting attempts per user.
- Weak passwords and password reuse increase organizational risk.
- Monitoring authentication events and enforcing MFA are among the most effective defenses.
- Password spraying is a technique that security teams should understand for both offensive testing (with authorization) and defensive detection.
