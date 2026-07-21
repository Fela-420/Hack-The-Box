# Child-to-Parent Domain Trust Security & Mechanics

## 📌 Conceptual Overview

In Active Directory multi-domain forests, child domains share a trust relationship with the forest root (parent) domain. By default, intra-forest trusts do not enforce strict boundary isolation because the entire forest—not individual child domains—is defined as the ultimate security boundary.

---

## 🧠 Key Mechanisms

### 1. The `sIDHistory` Attribute & `ExtraSIDs`
* **Purpose:** Designed to preserve resource access during domain migrations by keeping a record of an object's previous Security Identifiers (SIDs).
* **Kerberos Authorization:** When a KDC issues a Kerberos Ticket-Granting Ticket (TGT) or Service Ticket (TGS), it includes the user's Primary SID, Group SIDs, and any `sIDHistory` values within the **PAC (Privilege Attribute Certificate)**.
* **Intra-Forest Trust Behavior:** Because intra-forest trusts trust the KDCs of child domains to accurately report authorization data, foreign SIDs included in the PAC's `ExtraSIDs` field are evaluated by parent domain resource controllers without automatic filtering.

---

## 🔑 Data Points in Intra-Forest Trust Authorization

Understanding cross-domain authorization relies on five core identity elements:

| Data Element | Role in Authentication & Authorization |
| :--- | :--- |
| **Child Domain KRBTGT Key** | Used by the child domain KDC to encrypt and sign Kerberos tickets. |
| **Child Domain SID** | Base security identifier for identity objects local to the child domain. |
| **Parent/Forest Root SID** | Base security identifier for the root domain where forest-wide administrative groups reside. |
| **Enterprise Admins Group SID** | Relative ID `-519` appended to the forest root SID (`S-1-5-21-...-519`), granting forest-wide administration. |
| **Identity Principal Name** | The user identity requesting authorization within the ticket structure. |

---

## 🛡️ Defense, Detection & Mitigation

### 1. Enable SID Filtering (Quarantine)
While SID filtering is enabled by default on external/inter-forest trusts, it can also be evaluated for intra-forest trusts where legacy `sIDHistory` dependencies are no longer required:

    # Inspect quarantine status on a domain trust
    netdom trust <ChildDomain> /domain:<ParentDomain> /quarantine

    # Enable SID filtering across a trust boundary
    netdom trust <ChildDomain> /domain:<ParentDomain> /quarantine:yes

### 2. Architectural Containment
* **Treat the Forest as the Security Boundary:** Assume that administrative access over any child domain controller grants potential access to the entire forest root.
* **Restrict Administrative Scope:** Avoid using forest-wide administrative accounts for routine management inside child domains.
* **Protect the KRBTGT Account:** Implement strict access control lists (ACLs) to block unauthorized DCSync operations (`DS-Replication-Get-Changes-All`) and perform routine dual-resets of the `krbtgt` account password.

### 3. Monitoring & Auditing
* **Event ID 4768 / 4769:** Audit Kerberos TGT and TGS requests for unusual SID history requests or cross-domain ticket requests.
* **Monitor DCSync Operations:** Alert on non-domain-controller accounts requesting directory replication permissions.
