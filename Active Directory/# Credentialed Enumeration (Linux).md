# Credentialed Enumeration (Linux)

## Overview

Credentialed enumeration is the process of collecting information from an Active Directory environment **after successful authentication** using valid user credentials.

Compared to anonymous enumeration, authenticated users are typically allowed to view significantly more information about the domain.

---

# Learning Objectives

After completing this module, you should understand:

- The purpose of credentialed enumeration
- Information that becomes available after authentication
- Common categories of Active Directory objects
- Typical enumeration tools used during authorized security assessments
- How defenders can detect and limit excessive enumeration

---

# Authenticated vs Unauthenticated Access

| Unauthenticated | Authenticated |
|-----------------|---------------|
| Limited visibility | Increased visibility |
| Few accessible resources | Access to permitted AD objects |
| Minimal domain information | Users, groups, computers, shares, policies (subject to permissions) |

---

# Common Enumeration Categories

## User Enumeration

Collect information about:

- User accounts
- Account status
- Administrative accounts
- Service accounts
- Disabled accounts
- Password policy indicators

---

## Group Enumeration

Identify:

- Security groups
- Distribution groups
- Nested groups
- Privileged groups
- Group memberships

---

## Computer Enumeration

Gather information on:

- Domain-joined systems
- Servers
- Workstations
- Operating systems
- Hostnames

---

## Share Enumeration

Identify available network resources, including:

- Department shares
- Home directories
- Public shares
- Administrative shares
- Read and write permissions

---

# Common Assessment Tools

| Tool | Primary Purpose |
|------|-----------------|
| CrackMapExec (NetExec) | SMB assessment and enumeration |
| SMBMap | Network share discovery |
| rpcclient | RPC-based domain information |
| Impacket | Windows protocol interaction |
| Windapsearch | LDAP directory queries |
| BloodHound | Active Directory relationship analysis |

---

# Typical Assessment Workflow

```text
Authenticate
      │
      ▼
Enumerate Users
      │
      ▼
Enumerate Groups
      │
      ▼
Enumerate Computers
      │
      ▼
Enumerate Shares
      │
      ▼
Analyze Permissions
      │
      ▼
Document Findings
```

---

# Information Commonly Collected

During an authorized assessment, analysts may identify:

- User accounts
- Group memberships
- Domain structure
- Organizational Units (OUs)
- Computers
- Network shares
- File permissions
- Password policies
- Trust relationships

---

# Why Credentialed Enumeration Matters

Credentialed enumeration provides visibility into:

- Identity structure
- Access control
- Resource permissions
- Administrative delegation
- Potential security misconfigurations

This information helps defenders identify and remediate excessive permissions before they can be abused.

---

# Security Risks

Common risks include:

- Excessive permissions
- Overly permissive file shares
- Stale accounts
- Privileged service accounts
- Nested administrative groups
- Weak access control

---

# Defensive Monitoring

Security teams should monitor for:

- Large-scale LDAP queries
- Excessive SMB enumeration
- Repeated RPC requests
- Unusual authentication patterns
- Enumeration originating from unexpected hosts

---

# Mitigations

## Least Privilege

Grant users only the permissions required for their role.

---

## Access Reviews

Regularly audit:

- User permissions
- Group memberships
- Administrative privileges

---

## Network Segmentation

Restrict access to sensitive systems and management interfaces.

---

## Monitoring

Alert on:

- Unusual LDAP activity
- Large SMB share enumeration
- RPC enumeration
- Suspicious authentication behavior

---

## Administrative Tiering

Separate privileged accounts from standard user accounts.

---

# Key Takeaways

- Credentialed enumeration begins after successful authentication.
- Authenticated users typically have greater visibility into Active Directory.
- Enumeration focuses on users, groups, computers, shares, and permissions.
- The objective is to understand the environment and identify security weaknesses during an authorized assessment.
- Proper monitoring, least privilege, and regular permission reviews help reduce the risk posed by excessive enumeration.
