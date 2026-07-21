# Child-to-Parent Trust Analysis & Mechanics (Linux Ecosystem)

## 🎯 Core Concepts

When assessing domain trust configurations from a Linux host, the underlying theoretical mechanics of Kerberos PAC evaluation and `sIDHistory` trust boundaries remain identical to Windows environments. The primary difference lies in how protocol interactions (DRSUAPI, LSARPC, Kerberos, SMB) are initiated and processed using cross-platform toolsets.

---

## 🧠 Key Workflow Architecture & Automation Mechanics

### 1. Identity & Data Component Mapping
Elevating trust relationships between child and parent domains requires isolating five core directory attributes:

* **Child Domain `krbtgt` Key:** Encrypts and signs Kerberos TGTs for the local KDC.
* **Child Domain SID:** Base security identifier used to define local domain identity scopes.
* **Parent Forest Root SID:** Base security identifier for the root domain containing forest-wide administrative identities.
* **Enterprise Admins Relative ID (`-519`):** The RID appended to the forest root SID (`S-1-5-21-...-519`) identifying the principal group with forest-wide authorization.
* **Target Identity Principal:** The arbitrary identity embedded within the forged PAC structure.

---

### 2. Protocol Interactions & Automation Breakdown

Tools operating on non-Windows environments handle protocol tasks through specific RPC and Directory Replication interfaces:

| Protocol / Interface | Administrative Purpose | Security Impact |
| :--- | :--- | :--- |
| **DRSUAPI (`MS-DRSR`)** | Directory Replication Service Remote Protocol used for credential retrieval. | Used by controllers for domain replication; unauthorized invocation allows credential dumping. |
| **LSARPC (`MS-LSAT`)** | Local Security Authority Remote Protocol used for SID-to-Name translations. | Facilitates enumeration of domain SIDs and administrative group RIDs across trust boundaries. |
| **NRPC (`MS-NRPC`)** | Netlogon Remote Protocol used to identify domain controllers and forest topology. | Discovers forest FQDN and locates domain controller endpoints automatically. |
| **CCACHE Storage** | Credential Cache file format used by MIT Kerberos on Unix-like systems. | Stores Kerberos TGT/TGS structures for environment variable export (`KRB5CCNAME`). |

---

## 🛡️ Defense & Hardening Strategies

### 1. Forest Boundary Enforcement & SID Filtering
Organizations should enforce strict isolation on intra-forest trust paths where legacy application migrations are complete:

* **Quarantine Enforcement:** Enabling SID Filtering strips foreign SIDs (such as `ExtraSIDs`) from PACs crossing trust boundaries.
* **Forest Security Model:** Treat any child domain compromise as an implicit compromise of the entire forest unless strict boundaries are explicitly enforced and validated.

### 2. Operational Monitoring & Detection

```cmd
# Verify trust quarantine status via netdom
netdom trust <ChildDomain> /domain:<ParentDomain> /quarantine

# Enforce SID Filtering across intra-forest trusts
netdom trust <ChildDomain> /domain:<ParentDomain> /quarantine:yes
