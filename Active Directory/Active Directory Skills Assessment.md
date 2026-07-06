# Active Directory Skills Assessment – Engagement Overview

## Scenario

In this assessment, I assume the role of a **Penetration Tester (Intern)** at **CAT-5 Security**. The objective is to perform an internal Active Directory penetration test for the fictional client **Inlanefreight** while demonstrating the ability to conduct an engagement independently.

The assessment simulates a real-world enterprise environment where multiple attack techniques must be chained together to achieve complete domain compromise.

---

# Client Information

**Client:** Inlanefreight

**Assessment Type:** Internal Active Directory Penetration Test

**Objective:** Evaluate the security posture of the client's Active Directory environment by identifying vulnerabilities, exploiting misconfigurations, escalating privileges, and ultimately obtaining Domain Administrator access.

---

# Assessment Objectives

The engagement consists of five primary objectives.

## 1. Domain Enumeration

The first phase focuses on understanding the Active Directory environment by identifying:

* Domains
* Domain Controllers
* Users
* Groups
* Computers
* Organizational Units (OUs)
* Group Policies (GPOs)
* Trust relationships
* Shares
* Permissions

The information gathered during this phase serves as the foundation for the remainder of the assessment.

---

## 2. Credential Discovery

After enumeration, the objective is to obtain valid credentials using various attack techniques, including:

* Password Spraying
* Kerberoasting
* Capturing NTLM hashes
* Credential harvesting
* Offline password cracking
* Misconfigured services
* Stored credentials

Successfully obtaining credentials enables further access throughout the environment.

---

## 3. Lateral Movement

Once an initial foothold has been established, the next objective is to move laterally across the network.

Possible techniques include:

* SMB
* WinRM
* Remote Desktop Protocol (RDP)
* Pass-the-Hash
* Pass-the-Ticket
* PsExec
* WMI
* Remote PowerShell

The goal is to compromise additional systems while expanding access within the domain.

---

## 4. Privilege Escalation

The assessment continues by identifying opportunities to increase privileges.

Typical escalation paths include:

* Local privilege escalation
* Misconfigured services
* Weak permissions
* Group membership abuse
* Active Directory ACL abuse
* Delegation misconfigurations

The objective is to progress from a standard domain user to Local Administrator and ultimately to Domain Administrator.

---

## 5. Domain Compromise

The final objective is to compromise the Domain Controller and obtain Domain Administrator privileges.

The assessment concludes by performing a DCSync attack to extract domain credential data, demonstrating complete compromise of the Active Directory environment.

---

# Scope

## In Scope

The following assets are authorized for testing.

| Target                        | Description                            |
| ----------------------------- | -------------------------------------- |
| INLANEFREIGHT.LOCAL           | Primary Active Directory domain        |
| LOGISTICS.INLANEFREIGHT.LOCAL | Child domain                           |
| FREIGHTLOGISTICS.LOCAL        | Separate forest with an external trust |
| 172.16.5.0/23                 | Internal network range                 |

---

## Out of Scope

The following assets and activities are excluded from the assessment.

* Any domains not explicitly listed.
* Phishing or social engineering attacks.
* Active attacks against the public Inlanefreight website.
* Any systems outside the defined scope.

---

# Assessment Scenarios

Two separate penetration testing scenarios are included.

## Scenario 1 – External Breach Simulation

The assessment begins as an external attacker with no initial access.

Objectives include:

* Perform passive reconnaissance.
* Identify an initial foothold.
* Gain internal network access.
* Continue attacking the Active Directory environment.

This simulates a real-world external compromise.

---

## Scenario 2 – Internal Attack Simulation

The assessment begins with an attack workstation already positioned inside the client's network.

Objectives include:

* Enumerate the Active Directory environment.
* Discover credentials.
* Move laterally.
* Escalate privileges.
* Achieve full domain compromise.

This reflects an insider threat or a post-compromise engagement.

---

# Rules of Engagement

The assessment follows these rules:

* Passive reconnaissance only against publicly accessible assets.
* Internal testing begins without prior knowledge of the environment.
* Password cracking is performed offline on authorized systems.
* Testing must not intentionally disrupt production services.
* Only assets explicitly included within scope may be targeted.

---

# Attack Flow

```text
Passive Reconnaissance
        │
        ▼
Domain Enumeration
        │
        ▼
Credential Discovery
        │
        ▼
Initial Foothold
        │
        ▼
Lateral Movement
        │
        ▼
Privilege Escalation
        │
        ▼
Domain Controller Compromise
        │
        ▼
DCSync
        │
        ▼
Domain Administrator
```

---

# Skills Covered

This assessment combines multiple Active Directory attack techniques into a complete penetration testing engagement.

Key areas include:

* Active Directory Enumeration
* Windows Enumeration
* SMB Enumeration
* LDAP Enumeration
* Kerberos Attacks
* Password Spraying
* Kerberoasting
* Hash Capture
* Lateral Movement
* Credential Reuse
* Privilege Escalation
* Pass-the-Hash
* Pass-the-Ticket
* Active Directory ACL Abuse
* DCSync
* Domain Compromise

---

# Conclusion

This assessment represents a full end-to-end Active Directory penetration test. Rather than focusing on a single attack, it requires chaining together enumeration, credential discovery, lateral movement, and privilege escalation techniques to achieve complete domain compromise. Successfully completing this assessment demonstrates the ability to conduct a realistic internal Active Directory engagement using industry-standard methodologies and attack techniques.
