# Active Directory Hardening & Defense Baseline

## 🎯 Core Concepts

Active Directory (AD) hardening focuses on restricting lateral movement, privilege escalation, and credential abuse across an enterprise network. Rather than relying solely on reactive security tools, establishing a hardened AD baseline reduces the internal attack surface through administrative containment, strict access controls, and comprehensive auditing.

---

## 🛡️ AD Hardening Domains: People, Process & Technology

### 1. Administrative Security & Identity Controls (People)
* **Tiered Administration Model:** Enforce administrative tiering (Tier 0 for Identity/Domain Controllers, Tier 1 for Enterprise Servers, Tier 2 for Workstations) to prevent high-privilege credentials from exposing hashes or tokens on lower-trust endpoints.
* **Protected Users Group:** Assign sensitive administrative accounts to the `Protected Users` group (introduced in Windows Server 2012 R2). This group enforces key security boundaries:
  * Disables NTLM authentication, DES, and RC4 encryption types.
  * Prevents CredSSP and Windows Digest plaintext credential caching in memory.
  * Restricts Kerberos TGT lifetime to 4 hours without delegation permissions.
* **Privileged Account Hygiene:** Implement Local Administrator Password Solution (LAPS) for unique workstation administrative credentials, disable the default `RID-500` administrator account, and enforce the principle of least privilege on Domain/Enterprise Admin groups.

---

### 2. Operational Asset & Lifecycle Policies (Process)
* **Active Directory Auditing:** Conduct periodic reviews of Organizational Units (OUs), Group Policy Objects (GPOs), FSMO role allocations, and external trust relationships.
* **Account Lifecycle Management:** Automatically disable or purge stale accounts, enforce account de-provisioning workflows for departing personnel, and decommission legacy operating systems cleanly.
* **Documentation & Asset Baselining:** Maintain accurate records of enterprise hosts, network topology, DNS/DHCP configurations, and software inventories to detect rogue or unmanaged endpoints.

---

### 3. Core Technical Mitigations & Configurations (Technology)

| Defensive Measure | Threat / Vector Mitigated | Technical Implementation / Controls |
| :--- | :--- | :--- |
| **Group Managed Service Accounts (gMSAs)** | Kerberoasting / Service Account Abuse | Replaces static passwords with 120-character auto-rotating passwords, neutralizing offline ticket cracking. |
| **SMB & LDAP Signing** | Relay & Man-in-the-Middle (MitM) Attacks | Requires cryptographic signatures on network traffic, breaking relay attacks. |
| **Machine Account Quota Reduction** | Unauthorized Machine Account Creation | Set `ms-DS-MachineAccountQuota` to `0` to prevent unprivileged users from provisioning new domain computers (mitigating RBCD & noPac vectors). |
| **Null Session & Anonymous Access Controls** | Unauthenticated Directory Enumeration | Configure `RestrictNullSessAccess` registry key to `1` to block unauthenticated null session queries. |
| **Spooler Service Hardening** | Print Spooler Exploitation | Disable `Spooler` service (`spoolsv.exe`) on Domain Controllers and critical infrastructure servers. |

---

## 📋 Security Controls & MITRE ATT&CK Mapping

    # Inspect Protected Users Group configuration via PowerShell
    Get-ADGroup -Identity "Protected Users" -Properties Name, Description, Members

    # Audit machine account quota setting across the domain
    Get-ADDomain | Select-Object Name, ms-DS-MachineAccountQuota

* **Internal Reconnaissance & Enumeration (T1595 / TA0006):** Configure host firewalls to suppress unnecessary responses, restrict anonymous LDAP queries, and deploy SIEM/EDR detection rule baselines for command-line directory queries.
* **Credential Access & Kerberoasting (T1558.003):** Enforce AES-128/256 encryption over legacy RC4 for Kerberos, enforce strong passphrase policies for legacy service accounts, and transition applicable accounts to gMSAs.
* **Poisoning & MitM (T1557):** Mandate SMB Message Signing (`RequireSecuritySignature = 1`) and LDAP Signing/Channel Binding across all clients and DCs.
* **Password Spraying (T1110.003):** Implement lockout policies, enable Multi-Factor Authentication (MFA) on exterior and internal portals, and monitor Domain Controller logs for Event IDs `4624` (Successful Logon) and `4625` / `4648` (Failed / Explicit Credential Logon patterns).
