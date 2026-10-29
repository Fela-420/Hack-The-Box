# Bug Bounty Hunting — Complete Module Notes

> A clean, consolidated reference covering bug bounty programs, writing reports,
> CVSS scoring, interacting with hosts, and worked examples.

---

## Table of Contents

1. [Bug Bounty Programs (BBP)](#1-bug-bounty-programs-bbp)
2. [BBP vs. VDP](#2-bbp-vs-vdp)
3. [Types of Programs](#3-types-of-programs)
4. [Code of Conduct](#4-code-of-conduct)
5. [Anatomy of a BBP](#5-anatomy-of-a-bbp)
6. [Finding Programs](#6-finding-programs)
7. [Writing a Good Report](#7-writing-a-good-report)
8. [CWE & CVSS](#8-cwe--cvss)
9. [CVSS v3.1 Base Metrics Reference](#9-cvss-v31-base-metrics-reference)
10. [Interacting with Organizations / BBP Hosts](#10-interacting-with-organizations--bbp-hosts)
11. [Worked Examples](#11-worked-examples)
12. [Quick Comparison Table](#12-quick-comparison-table)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. Bug Bounty Programs (BBP)

A **Bug Bounty Program** (also called a **Vulnerability Rewards Program — VRP**) is:

- A **crowdsourcing initiative** where individuals receive **recognition + compensation**
  for discovering and reporting software bugs.
- **Continuous, proactive security testing** that:
  - Supplements internal code audits & penetration tests
  - Completes an organization's vulnerability management strategy
- Often described as **"Continuous testing, constant protection."**
- Integrates seamlessly into an organization's existing development lifecycle.

---

## 2. BBP vs. VDP

> ⚠️ These terms must **NOT** be used interchangeably.

| Aspect | Vulnerability Disclosure Program (VDP) | Bug Bounty Program (BBP) |
|---|---|---|
| Purpose | Guidance on *how* to report vulnerabilities | **Incentivizes** discovering & reporting bugs |
| Reward | No monetary reward | **Monetary rewards** |

---

## 3. Types of Programs

### Private BBP
- Not publicly available — **invitation only**.
- Most programs **start private**, then go public once triaging is mature.
- Invitations based on:
  - Track record
  - Consistency of valid findings
  - Violation record
- Some may require a **background check**.

### Public BBP
- Open to the **entire hacking community**.

### Parent / Child Programs
- A parent company and its subsidiaries **share one bounty pool and one security team**.
- A subsidiary's program (child) is **linked to the parent**.

---

## 4. Code of Conduct

- Your **violation record always matters** — follow each program's policy strictly.
- The **Code of Conduct** (along with the program policy) establishes
  **expectations for behavior** during participation.
- Reading it carefully makes you **more effective and successful** in submissions.
- Goal: balance **professionalism + technical capability**.

---

## 5. Anatomy of a BBP

A typical bug bounty program includes:

| Element | Purpose |
|---|---|
| **Vendor Response SLAs** | When/how the vendor replies |
| **Access** | How to create/obtain research accounts |
| **Eligibility Criteria** | e.g., must be first reporter of a vulnerability |
| **Responsible Disclosure Policy** | Disclosure timelines & coordination |
| **Rules of Engagement** | Interaction rules |
| **Scope** | In-scope IPs, domains, vulnerabilities |
| **Out of Scope** | Excluded IPs, domains, vulnerabilities |
| **Reporting Format** | Required report structure |
| **Rewards** | Bounty amounts |
| **Safe Harbor** | Legal protection for researchers |
| **Legal Terms & Conditions** | Contractual terms |
| **Contact Information** | How to reach the team |

> On HackerOne, most of these live in the program's **"Policy"** section.
> **Read the policy + code of conduct meticulously — time is everything in bounty hunting.**

---

## 6. Finding Programs

- **HackerOne's Directory** — best resource to find:
  - Organizations with bounty programs
  - Contact info to report ethically found vulnerabilities

---

## 7. Writing a Good Report

### Purpose
- Document findings **clearly and concisely** for the security/triage team.
- Always include **step-by-step reproduction**.
- For less mature companies: translate technical issues into **business terms**.
- Well-formatted reports **reduce reproduction + triage time**.

### Essential Elements (order may vary)

| Element | What It Covers |
|---|---|
| **Vulnerability Title** | Type, affected domain/parameter/endpoint, impact |
| **CWE & CVSS Score** | Weakness type + severity |
| **Vulnerability Description** | Cause of the vulnerability |
| **Proof of Concept (POC)** | Clear, concise reproduction steps |
| **Impact** | Attacker capability; business impact + max damage |
| **Remediation** | Optional in bounties, good to include |

---

## 8. CWE & CVSS

### CWE — Common Weaknesses Enumeration
- Community-developed list of software/hardware **weakness types**.
- Serves as a **common language** and baseline for weakness identification,
  mitigation, and prevention.
- For a **vulnerability chain**, choose the CWE of the **initial vulnerability**.

### CVSS — Common Vulnerability Scoring System
- Published, worldwide standard for communicating **severity**.
- Use the **CVSS v3.1 Calculator** (focus on the **Base Score**).

---

## 9. CVSS v3.1 Base Metrics Reference

### Attack Vector (AV) — how it's exploited
| Value | Meaning |
|---|---|
| **Network (N)** | Remotely exploitable over the network |
| **Adjacent (A)** | Same physical or logical network (secure VPN included) |
| **Local (L)** | Local access (keyboard/terminal), SSH, or user interaction |
| **Physical (P)** | Physical interaction/manipulation |

### Attack Complexity (AC) — conditions beyond attacker control
| Value | Meaning |
|---|---|
| **Low (L)** | No special prep; repeatable without issue |
| **High (H)** | Special prep / information gathering needed |

### Privileges Required (PR) — access level needed
| Value | Meaning |
|---|---|
| **None (N)** | Unauthorized perspective |
| **Low (L)** | Standard user privileges |
| **High (H)** | Admin-level privileges |

### User Interaction (UI)
| Value | Meaning |
|---|---|
| **None (N)** | Attacker exploits independently |
| **Required (R)** | User must take some action |

### Scope (S)
| Value | Meaning |
|---|---|
| **Unchanged (U)** | Affects only vulnerable component / same security authority |
| **Changed (C)** | Can affect other components / beyond security scope |

### Confidentiality (C) — information access impact
| Value | Meaning |
|---|---|
| **None (N)** | No impact |
| **Low (L)** | Some loss; attacker has no control over what's obtained |
| **High (H)** | Total/serious loss; attacker has control over information |

### Integrity (I) — trustworthiness of information
| Value | Meaning |
|---|---|
| **None (N)** | No impact |
| **Low (L)** | Limited data modification; no control over consequence |
| **High (H)** | All/critical data modifiable; total loss of integrity |

### Availability (A) — accessibility of resources
| Value | Meaning |
|---|---|
| **None (N)** | No impact |
| **Low (L)** | Reduced performance; cannot fully deny service |
| **High (H)** | Severe/total loss; can deny service |

---

## 10. Interacting with Organizations / BBP Hosts

### Right After Submitting
- **Do NOT interact immediately.**
- Give the team time to process, validate, and ask questions.
- Check **vendor response SLAs** / **response efficiency metrics** for expected timing.
- **Do not spam** within a short period.

### If No Reasonable Response
- If submitted via a platform, contact **Mediation**.

### Once They Respond
- **Note the team member's username** and tag them in future communications.
- **Never use unofficial channels** (e.g., social media).

### Communication Style
- Professional report → professional communication.
- **Remain calm**; act like a security professional.

### Handling Disagreements (Severity / Bounty)
1. **Explain your rationale** — walk through each **CVSS metric value**.
2. **Review the policy & scope** — ensure compliance; confirm bounty aligns with policy.
3. **If unresolved** — contact **mediation** or similar platform service.

---

## 11. Worked Examples

### Example 1 — Stored XSS

| Field | Value |
|---|---|
| **Title** | Stored Cross-Site Scripting (XSS) in X Admin Panel |
| **CWE** | CWE-79 — Improper Neutralization of Input During Web Page Generation |
| **CVSS 3.1** | **5.5 (Medium)** |

**Description:** File upload mechanism ("Admin Info" → "Secure Data Transfer" → "Load of Data")
reflects and stores the **uploaded filename** without sanitization → stored & reflected XSS.

**Impact:** Credential theft, session hijacking, defacement, malicious redirects.
2. Another admin clicks **View** → JavaScript executes in their browser.

**CVSS Breakdown:**

| Metric | Value | Rationale |
|---|---|---|
| AV | Network | Internet-mounted attack |
| AC | Low | Just specify the payload |
| PR | High | Admin-level access needed |
| UI | None | Admins affected by browsing a page |
| S | Changed | Webserver → browser |
| C | Low | DOM access possible |
| I | Low | Slight integrity impact |
| A | None | No DoS via XSS |

---

### Example 2 — CSRF

| Field | Value |
|---|---|
| **Title** | Cross-Site Request Forgery (CSRF) in Consumer Registration |
| **CWE** | CWE-352 — Cross-Site Request Forgery |
| **CVSS 3.1** | **5.4 (Medium)** |

**Description:** Consumer registration lacks anti-CSRF protections. Attacker tricks a
victim into loading a page with a malicious request, inheriting the victim's identity.

**Impact:** Attacker can perform any action as the victim (limited by victim's privileges).
Specifically: register a fintech application and create an API key as the victim.

**POC:**
1. Intercept the "create fintech application" request → **no anti-CSRF tokens**.
2. Craft a malicious HTML page performing the cross-site POST to `/consumer-registration`.
3. Send to a victim with an active session.
4. Victim inadvertently creates an attacker-specific fintech application.
*(Could run in background if chained with XSS — finding 6.1.1.)*

**CVSS Breakdown:**

| Metric | Value | Rationale |
|---|---|---|
| AV | Network | Internet-mounted attack |
| AC | Low | Just trick a user with an open session |
| PR | None | No privileges needed |
| UI | **Required** | Victim must click a crafted link |
| S | **Unchanged** | Webserver → webserver |
| C | Low | Limited info obtainable |
| I | Low | Limited data modification (create app) |
| A | None | No DoS |

---

### Example 3 — RCE

| Field | Value |
|---|---|
| **Title** | IBM WebSphere Java Object Deserialization RCE |
| **CWE** | CWE-502 — Deserialization of Untrusted Data |
| **CVSS 3.1** | **9.8 (Critical)** |

**Description:** WebSphere server affected by insecure Java object deserialization →
remote arbitrary command execution. Base64-encoded serialized Java objects identified
by the **`rO0` header** on HTTPS port **8880**. Crafted SOAP request exploiting
**Apache Commons Collections (ACC)** with a ping command.

**Impact:** Command injection → command runs under current user's context.
If privileged (e.g., SYSTEM), can lead to **complete system takeover**.

**POC:**
1. Capture & decode a request to port 8880 → confirm serialized objects (`rO0` header).
2. Craft a SOAP request with a serialized Java object containing a **ping command**.
3. Send it → observe ping from WebSphere to attacker via **Wireshark** → RCE confirmed.

**CVSS Breakdown:**

| Metric | Value | Rationale |
|---|---|---|
| AV | Network | Internet-mounted attack |
| AC | Low | Just send a crafted request |
| PR | None | Unauthenticated |
| UI | None | No user action needed |
| S | Unchanged | Webserver → webserver |
| C | **High** | RCE → total info control |
| I | **High** | RCE → modify all/critical data |
| A | **High** | RCE → can power off server |

---

## 12. Quick Comparison Table

| Aspect | Stored XSS (Ex.1) | CSRF (Ex.2) | RCE (Ex.3) |
|---|---|---|---|
| **Score** | 5.5 (Medium) | 5.4 (Medium) | **9.8 (Critical)** |
| **CWE** | CWE-79 | CWE-352 | CWE-502 |
| **Attack Vector** | Network | Network | Network |
| **Attack Complexity** | Low | Low | Low |
| **Privileges Required** | High | None | None |
| **User Interaction** | None | Required | None |
| **Scope** | Changed | Unchanged | Unchanged |
| **Confidentiality** | Low | Low | **High** |
| **Integrity** | Low | Low | **High** |
| **Availability** | None | None | **High** |

---

## 13. Key Takeaways

- A **BBP** is **paid, continuous, crowdsourced security testing** — distinct from a **VDP**.
- **Read the policy + code of conduct meticulously** before participating.
- The **Code of Conduct** sets behavioral expectations; violations affect your record.
- A good report = **clear title + CWE/CVSS + description + reproducible POC + impact**.
- **CWE** = weakness type; **CVSS** = severity. Use the **CVSS v3.1 Base Score**.
- **Severity drivers**:
- **Privileges Required**, **User Interaction**, and **Scope** heavily influence scores.
- **High C/I/A across all three** = typically **Critical**.
- After submission: **wait patiently**, communicate **professionally via official channels**,
justify severity with **CVSS rationale**, verify **policy compliance**,
and escalate to **mediation** only when necessary.

---

*End of Module Notes — Bug Bounty Hunting Fundamentals*
Any administrator viewing uploaded files is a target.

**POC:**
1. Malicious admin uploads a file named:
