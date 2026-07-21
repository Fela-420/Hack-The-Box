# Cross-Forest Trust Analysis & Security Boundaries (Windows)

## 🎯 Core Concepts

A **Forest Trust** establishes an authentication path between two distinct Active Directory forests. Depending on trust directionality (Inbound, Outbound, or Bidirectional) and scope, users in Forest A can request access to resources in Forest B. 

Unlike intra-forest trusts—where the forest root acts as the ultimate security boundary—inter-forest trusts enforce **SID Filtering** by default to prevent cross-forest privilege escalation via attributes like `sIDHistory`.

---

## 🧠 Cross-Forest Identity & Authorization Vectors

### 1. Cross-Domain Service Principal Name (SPN) Enumeration
In environments with inbound or bidirectional trusts, Active Directory allows authenticated accounts from Forest A to query LDAP objects and SPNs in Forest B.

* **Kerberos TGS Requests:** When a user requests a service ticket for an SPN in a trusted foreign forest, the local KDC issues a cross-realm TGT. The foreign KDC evaluates the request and returns a service ticket encrypted with the target service account's NTLM/AES key.
* **Impact of Weak Service Passwords:** If a service account in a trusted foreign forest uses a weak password, Kerberos ticket cracking allows offline recovery of plaintext credentials, granting access within that forest.

---

### 2. Foreign Security Principals (FSPs) & Domain Local Groups
In Active Directory architecture, group scope determines whether cross-forest principals can be added as members:

| Group Scope | Cross-Forest Principal Membership |
| :--- | :--- |
| **Global Group** | Accounts from the **same domain only**. |
| **Universal Group** | Accounts from **any domain within the same forest**. |
| **Domain Local Group** | Accounts from **any domain or trusted external forest**. |

When an account or group from Forest A is added to a Domain Local Group in Forest B, Active Directory creates a **Foreign Security Principal (FSP)** object inside `CN=ForeignSecurityPrincipals,DC=TargetDomain,DC=com`. 

* **Implicit Privilege Inheritance:** If a privileged account or administrative group from Forest A is explicitly nested inside a Domain Local Group (such as the local `Builtin\Administrators` group) in Forest B, compromising that account in Forest A grants equivalent administrative control in Forest B.

---

### 3. Cross-Forest `sIDHistory` Boundaries
Migration between forests can introduce security risks if trust configurations are altered:
* **Default Behavior:** Inter-forest trusts enable SID Filtering (`quarantine:yes`) by default, stripping foreign SIDs from the Kerberos PAC during cross-forest authentication.
* **Misconfiguration Risk:** If SID Filtering is intentionally disabled (`quarantine:no`) across a forest boundary to support legacy migrations, foreign SIDs present in `sIDHistory` will be honored by resource servers in the destination forest.

---

## 🛡️ Defense & Hardening Strategies

### 1. Enforce SID Filtering across Forest Trusts
Ensure that SID Filtering remains enabled on all external and inter-forest trust relationships to block unauthorized `ExtraSIDs` injection:

    # Verify SID filtering status on a cross-forest trust
    netdom trust <LocalDomain> /domain:<ForeignDomain> /quarantine

    # Enforce SID filtering (default for forest trusts)
    netdom trust <LocalDomain> /domain:<ForeignDomain> /quarantine:yes

### 2. Restrict Foreign Security Principal Nesting
* **Audit Foreign Group Memberships:** Regularly review FSP objects created within `CN=ForeignSecurityPrincipals` to detect unauthorized cross-forest group nesting.
* **Principle of Least Privilege:** Avoid adding privileged cross-forest accounts (e.g., Domain Admins from partner forests) into local administrative groups. Use explicit, role-based service accounts for cross-forest administrative tasks.

### 3. Password Hygiene & Service Account Hardening
* **Implement Managed Service Accounts (gMSAs):** Secure SPN-associated service accounts using Group Managed Service Accounts (gMSAs), which feature 120-character auto-rotating passwords, mitigating offline Kerberos ticket cracking attacks.
* **Prevent Credential Reuse:** Ensure local administrator passwords and administrative service account credentials are unique across distinct administrative forests.
