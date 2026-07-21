# Active Directory Auditing & Security Posture Assessment

## 🎯 Core Concepts

Active Directory (AD) auditing provides visibility into organizational security posture, identifying structural misconfigurations, stale objects, and policy weaknesses. Combining point-in-time database snapshots with automated risk-scoring frameworks creates actionable telemetry to justify security remediation, track historical changes, and harden enterprise environments.

---

## 🧠 Auditing Frameworks & Visualization Tools

### 1. Directory State Preservation: Active Directory Explorer
**AD Explorer** (Sysinternals) enables direct navigation of the Active Directory database and supports offline snapshot generation for change auditing.

* **Offline Analysis & Comparison:** Captures a point-in-time copy of AD objects, attributes, and Access Control Lists (ACLs) to perform offline schema analysis or run delta comparisons between historical snapshots.
* **Non-Disruptive Assessment:** Allows security auditors to query directory properties without executing aggressive or noisy network scans against live Domain Controllers.

---

### 2. Risk-Based Posture Assessment: PingCastle
**PingCastle** uses a maturity model approach based on the Capability Maturity Model Integration (CMMI) framework to score domain risk levels and generate structural security maps.

| Function / Module | Operational Scope | Key Outputs |
| :--- | :--- | :--- |
| **Healthcheck (`1-healthcheck`)** | Calculates overall domain risk scores and evaluates core vulnerability exposure. | HTML report with risk metrics, misconfigurations, and anomalies. |
| **Domain Cartography (`3-carto`)** | Maps trust relationships across interconnected domains. | Visual trust maps illustrating cross-domain access pathways. |
| **Scanner Engine (`4-scanner`)** | Executes targeted checks (e.g., `aclcheck`, `localadmin`, `nullsession`, `zerologon`). | Granular endpoint and authorization policy findings. |

---

### 3. Group Policy Security Auditing: Group3r
**Group3r** automates the enumeration and analysis of Active Directory Group Policy Objects (GPOs) to detect privilege escalation vectors and weak security settings.

* **Targeted Finding Analysis:** Identifies insecure registry modifications, permissive User Rights Assignments, cleartext credentials in scripts, and misconfigured security policies (e.g., disabled LM Hash restrictions or missing LDAP signing mandates).
* **Execution Boundary:** Requires execution from a domain-joined host or within a valid domain user security context (e.g., via `runas /netonly`).

---

### 4. Bulk Directory Telemetry: ADRecon
**ADRecon** aggregates domain-wide metadata into consolidated reports (Excel/HTML/CSV) for high-coverage environment baselining.

* **Comprehensive Coverage:** Extracts domain trusts, site topologies, Fine-Grained Password Policies, SPNs, GPO links, DNS zones, and privileged group memberships in a single execution.
* **Stealth Considerations:** Generates high LDAP/RPC query volumes; primarily suited for authorized internal audits, baseline tracking, and assessment reporting rather than covert operations.

---

## 🛡️ Administrative Commands & Operational Workflow

    # Execute Group3r policy analysis and log output
    group3r.exe -f C:\AuditLogs\GPO_Audit.log

    # Execute PingCastle healthcheck scan against the target domain
    PingCastle.exe --server target.local --protocol ADWS

    # Execute ADRecon collection script from PowerShell
    Import-Module .\ADRecon.ps1; Invoke-ADRecon

    # Re-generate Excel summary report from raw ADRecon output folder
    Invoke-ADRecon -GenExcel -ColDir "C:\Tools\ADRecon-Report-20260722"

* **Documentation & Data Backing:** Export snapshot data and healthcheck scores directly into client deliverables to provide verifiable proof of security risks and secure executive backing for infrastructure remediation.
* **Continuous Monitoring:** Integrate periodic GPO and directory state audits into security operations to detect baseline configuration drift over time.
