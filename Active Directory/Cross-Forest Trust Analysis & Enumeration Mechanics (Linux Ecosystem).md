# Cross-Forest Trust Analysis & Enumeration Mechanics (Linux Ecosystem)

## 🎯 Core Concepts

When assessing cross-forest trust relationships from a Linux platform, the underlying authentication and protocol mechanisms remain identical to Windows environments. The operational difference lies in utilizing non-domain-joined, cross-platform tools (such as Impacket or BloodHound-python) to interact with Active Directory LDAP, Kerberos, and RPC services across forest boundaries.

---

## 🧠 Cross-Forest Identity Vectors & Technical Mechanics

### 1. Cross-Forest Kerberos Service Ticket Requests
Under a bidirectional or inbound forest trust, an authenticated user in Forest A can query LDAP and request Service Principal Name (SPN) tickets for service accounts residing in Forest B.

* **Cross-Realm TGT Exchanges:** The local KDC in Forest A issues a referral TGT for Forest B's KDC. Forest B's KDC evaluates the request and issues a Kerberos TGS ticket encrypted with the target service account's secret key.
* **Offline Hash Analysis:** Service accounts utilizing weak passwords allow offline Kerberos ticket cracking, potentially exposing administrative credentials in the target forest.

---

### 2. Directory Resolution & Foreign Security Principal (FSP) Mapping
Active Directory uses domain-specific DNS for LDAP and Kerberos referrals across forest boundaries. Non-domain-joined Linux systems rely on proper DNS resolution to process cross-domain object relationships.

| Protocol / Component | Function in Cross-Forest Analysis |
| :--- | :--- |
| **`resolv.conf` / DNS Routing** | Directs LDAP/GC queries to specific Domain Controllers when target domains lack external resolution. |
| **Global Catalog (GC) LDAP** | Listens on port `3268`/`3269` to resolve forest-wide objects and multi-domain group memberships. |
| **Foreign Security Principals (FSP)** | Standard Active Directory objects representing external security principals added to local Domain Local groups. |

---

### 3. Graph-Based Trust & Membership Analysis
Enumerating cross-forest trust relationships involves collecting directory metadata from both forests to expose structural overlaps:

* **Cross-Forest Group Nesting:** Foreign Security Principals added to privileged Domain Local groups (e.g., `Builtin\Administrators`) inherit rights directly across trust boundaries.
* **Shared Identity Vectors:** Identifying accounts present in multiple domains exposes potential administrative credential reuse across distinct administrative boundaries.

---

## 🛡️ Defense, Hardening & Remediation

### 1. Restrict Cross-Forest Group Nesting & FSPs
* **Audit Foreign Security Principals:** Continuously inspect `CN=ForeignSecurityPrincipals` containers across all managed domains to detect unauthorized cross-forest group assignments.
* **Enforce Least Privilege Boundaries:** Avoid placing external accounts or foreign domain groups into local administrative groups (`Builtin\Administrators`, `Account Operators`).

### 2. Service Account Security & SPN Hardening
* **Deploy Group Managed Service Accounts (gMSAs):** Migrate traditional SPN-bound user accounts to gMSAs to eliminate weak passwords and protect against offline Kerberos ticket cracking.
* **Enforce Strong Password Policies:** Service accounts that cannot use gMSAs must utilize long, complex passwords (25+ characters) and robust encryption types (AES-128/AES-256).

### 3. Enforce Trust Isolation & Administrative Hygiene
* **Enforce SID Filtering:** Ensure SID Filtering (`quarantine:yes`) remains active on all inter-forest trust configurations to filter unauthorized `sIDHistory` values.
* **Prevent Administrative Credential Overlap:** Prohibit the reuse of passwords across administrative accounts operating in separate forests.

    # Verify quarantine status on inter-forest trusts
    netdom trust <LocalDomain> /domain:<ForeignDomain> /quarantine

    # Enforce SID Filtering across inter-forest trust boundaries
    netdom trust <LocalDomain> /domain:<ForeignDomain> /quarantine:yes
