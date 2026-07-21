# Bleeding Edge Vulnerabilities - Active Directory Security Assessment Guide

> Educational Purpose:  
> This documentation explains three critical Active Directory vulnerabilities and how they can impact enterprise environments during authorized security assessments.

---

# Lab Objective

Understand how three major vulnerabilities can lead to Active Directory compromise:

- NoPac
- PrintNightmare
- PetitPotam

The goal is to understand:

- Vulnerability concepts
- Attack paths
- Tools involved
- Detection methods
- Defensive recommendations

---

# Vulnerability Overview

| Attack | CVE | Impact |
|---|---|---|
| NoPac | CVE-2021-42278 & CVE-2021-42287 | Standard User → Domain Administrator |
| PrintNightmare | CVE-2021-1675 & CVE-2021-34527 | SYSTEM Privilege Escalation |
| PetitPotam | CVE-2021-36942 | NTLM Relay → Domain Compromise |

---

# Attack 1 - NoPac

## Vulnerability Explanation

NoPac combines two Active Directory vulnerabilities:

- CVE-2021-42278
- CVE-2021-42287

The vulnerability abuses computer account naming and Kerberos authentication behavior.

A normal domain user can abuse these weaknesses to impersonate a Domain Controller account and obtain elevated privileges.

---

# NoPac Attack Chain

```text
Normal Domain User
        |
        ↓
Computer Account Manipulation
        |
        ↓
Kerberos Ticket Request
        |
        ↓
Administrator Impersonation
        |
        ↓
Domain Admin Access
