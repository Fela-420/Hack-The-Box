# Windows Event Logs & Finding Evil: Study Notes

A consolidated reference covering SIEM concepts, the Elastic Stack, SOC operations, MITRE ATT&CK, SIEM use case development, Kibana visualizations, the triage process, Windows event logging, Sysmon, Event Tracing for Windows (ETW), PowerShell log analysis with Get-WinEvent, and the analyst decision framework. It includes the lab answers and reasoning from the associated skills assessments.

---

## Table of Contents

1. [SIEM Fundamentals](#1-siem-fundamentals)
2. [The Elastic Stack](#2-the-elastic-stack)
3. [SOC Fundamentals](#3-soc-fundamentals)
4. [MITRE ATT&CK](#4-mitre-attck)
5. [SIEM Use Case Development](#5-siem-use-case-development)
6. [SIEM Visualizations](#6-siem-visualizations)
7. [The Triage Process](#7-the-triage-process)
8. [Windows Event Logs](#8-windows-event-logs)
9. [Analyzing Attacks with Sysmon](#9-analyzing-attacks-with-sysmon)
10. [Event Tracing for Windows (ETW)](#10-event-tracing-for-windows-etw)
11. [Get-WinEvent: PowerShell Log Analysis](#11-get-winevent-powershell-log-analysis)
12. [Field and Query Reference](#12-field-and-query-reference)
13. [Skills Assessment: Worked Answers](#13-skills-assessment-worked-answers)
14. [Analyst Mindset and Key Takeaways](#14-analyst-mindset-and-key-takeaways)

---

## 1. SIEM Fundamentals

**Security Information and Event Management (SIEM)** software collects logs and events from many sources, normalizes and correlates them, and alerts security teams to potential threats in near real time.

The term combines SIM (Security Information Management) and SEM (Security Event Management). Gartner merged these into the SIEM category in 2005.

| Use case | Purpose |
|---|---|
| Log aggregation and normalization | Central visibility across IT |
| Threat alerting | Detect anomalies in real time |
| Contextualization and response | Filter noise and prioritize threats |
| Compliance | Support PCI DSS, HIPAA, GDPR, and ISO requirements |

Data flow:

```text
Ingest logs -> Normalize and aggregate -> Correlate and detect -> Alert, dashboard, or incident
```

A SIEM complements IDS and IPS; it does not replace them. Because SIEMs generate many alerts, tuning is essential.

---

## 2. The Elastic Stack

| Component | Role |
|---|---|
| Elasticsearch | Distributed JSON search and storage engine with a REST API |
| Logstash | Collects, transforms, and forwards logs |
| Kibana | Visualization and query interface (KQL) |
| Beats | Lightweight shippers such as Filebeat, Winlogbeat, and Metricbeat |

```text
Beats -> Logstash -> Elasticsearch -> Kibana
Beats -> Elasticsearch -> Kibana          (without Logstash)
```

SOC analysts work primarily in Kibana.

### Kibana Query Language (KQL)

| Syntax | Meaning |
|---|---|
| `field:value` | Match a field |
| `"free text"` | Search across fields |
| `A AND B`, `A OR B`, `NOT A` | Boolean logic |
| `field: value*` | Prefix wildcard |
| `field: *value*` | Contains wildcard |
| `field >= X` | Comparison (`>`, `>=`, `<`, `<=`) |

### Elastic Common Schema (ECS)

ECS is the shared field vocabulary across the stack. Prefer ECS fields for consistency and correlation.

- Use `.keyword` subfields for aggregations, sorting, and exact matches (for example, `user.name.keyword`).
- Use base fields for KQL text searches (for example, `user.name`).

---

## 3. SOC Fundamentals

A Security Operations Center (SOC) is a team and facility that continuously monitors, detects, and responds to security incidents.

Core process: **Triage, Containment, Eradication, Recovery.**

| Role | Responsibility |
|---|---|
| SOC Director | Strategy, budget, staffing |
| SOC Manager | Daily operations and incident coordination |
| Tier 1 Analyst | Monitor alerts, triage, escalate |
| Tier 2 Analyst | Deeper analysis and tuning |
| Tier 3 Analyst | Complex incidents and threat hunting |
| Detection Engineer | Build and tune detection rules |
| Incident Responder | Active response and forensics |
| Threat Intelligence Analyst | Research and share threat intelligence |
| Security Engineer | Build and maintain tooling |
| Compliance Specialist | Standards and audits |
| Awareness Coordinator | Training programs |

SOC maturity has progressed through three stages:

| Stage | Focus |
|---|---|
| SOC 1.0 | Network and perimeter, uncorrelated alerts |
| SOC 2.0 | Intelligence-driven, application-layer visibility |
| Cognitive SOC | Learning systems, business-aware detection |

---

## 4. MITRE ATT&CK

MITRE ATT&CK (Adversarial Tactics, Techniques, and Common Knowledge) documents attacker behavior.

- **Tactics** are the goals (the "why").
- **Techniques** are the methods (the "how").
- **Procedures** are specific implementations (the "how exactly").

Matrices cover Enterprise (Windows, macOS, Linux, cloud, and network), Mobile, and ICS.

Enterprise tactics, in typical intrusion order: Reconnaissance, Resource Development, Initial Access, Execution, Persistence, Privilege Escalation, Defense Evasion, Credential Access, Discovery, Lateral Movement, Collection, Command and Control, Exfiltration, and Impact.

ATT&CK supports detection and response, gap analysis, SOC maturity assessment, threat intelligence enrichment, behavioral analytics, red team planning, and training.

---

## 5. SIEM Use Case Development

A use case defines what to detect and how to alert on it. For example: "ten failed logons from one source within four minutes indicates brute force."

### Lifecycle

Requirements, data points, log validation, design, implementation, documentation, onboarding, testing, and fine tuning.

### Design parameters

- **Condition:** what triggers the alert.
- **Aggregation:** how events are grouped, which reduces false positives.
- **Priority:** severity and response SLA.

### Standard operating procedure (SOP) contents

- Conditions, aggregation, and priority
- Escalation matrix
- Reporting paths

### Build checklist

1. Understand the needs and risks.
2. Map the use case to the kill chain or ATT&CK.
3. Define time to detect (TTD) and time to respond (TTR).
4. Write the SOP.
5. Define the tuning process.
6. Build an incident response plan (IRP).
7. Set SLAs and OLAs.
8. Audit the process.
9. Maintain documentation and a knowledge base.

### Severity guidance

| Severity | When to use it |
|---|---|
| High | Clearly malicious, unusual for the environment, or high impact |
| Medium | Suspicious but possibly legitimate, such as an MSBuild network connection |
| Low | Informational or a minor pattern anomaly |

---

## 6. SIEM Visualizations

### Example 1: Failed logons for all users

**Purpose:** detect brute force.

- Time range: last 15 years
- Filter: `event.code : 4625`
- Index pattern: `windows*`
- Visualization: Table
- Rows: `user.name.keyword` and `host.hostname.keyword`
- Metric: Count

Refinements: rename columns clearly, add `winlog.logon.type.keyword`, sort by count descending, exclude specific usernames with filters, and exclude computer accounts with `NOT user.name: *$`.

### Example 2: Failed logons for disabled users

**Purpose:** detect attempts against disabled accounts, which can indicate leaked credentials being tested.

- Filters: `event.code : 4625` and `winlog.event_data.SubStatus : 0xC0000072`
- Rows: `user.name.keyword`, `host.hostname.keyword`
- Metric: Count

### Example 3: Successful RDP by service accounts

**Purpose:** service accounts should not use interactive RDP. This is a high-privilege risk.

- Filters: `event.code : 4624` and `winlog.logon.type : RemoteInteractive`
- Scope: `user.name: svc-*`
- Rows: `user.name.keyword`, `host.hostname.keyword`, `related.ip.keyword`
- Metric: Count

### Example 4: Users added to or removed from Administrators

**Purpose:** detect privilege escalation.

- Filters: `event.code : 4732 OR event.code : 4733` and `group.name : "administrators"`
- Rows: actor (`user.name.keyword`), member SID (`winlog.event_data.MemberSid.keyword`), group, action (`event.action.keyword`), and host
- Metric: Count
- Use an absolute time range. A relative range can produce weekly buckets when you aggregate by day.

### Best practices

- Set an absolute time range for day-level aggregation.
- Use `.keyword` for groupings, sorting, and aggregations.
- Use base fields for KQL text search.
- Save both the visualization and the dashboard.
- Exclude known noise in KQL, such as `*$` for computer accounts.

---

## 7. The Triage Process

Triage means evaluating and prioritizing alerts to decide whether to ignore them as false positives, investigate further, or escalate.

Escalate when you see a compromised critical system, an ongoing attack, a sophisticated technique, widespread impact, or a possible insider threat.

### Thirteen-step process

1. **Initial alert review:** metadata, IP addresses, affected systems, triggering rule.
2. **Classification:** severity, impact, and urgency.
3. **Correlation:** related alerts, IOCs, SIEM queries.
4. **Enrichment:** packet captures, memory, sandbox results, threat intelligence.
5. **Risk assessment:** asset criticality, data sensitivity, likelihood of lateral movement.
6. **Contextual analysis:** assets, existing controls, compliance impact.
7. **Incident response planning:** document, assign roles.
8. **Consultation with IT Operations:** planned changes, maintenance windows, known false positives.
9. **Response execution:** close if benign, or start incident response if suspicious.
10. **Escalation:** internal procedure first, then external parties (law enforcement, CERT) if needed.
11. **Continuous monitoring:** updates and communication.
12. **De-escalation:** once contained and documented.
13. **Process review:** update procedures from lessons learned.

### Decision framework

| Situation | Action |
|---|---|
| Matches baseline or expected pattern | Nothing suspicious |
| Unusual, and operational context is missing | Consult IT Operations |
| Clear policy violation or malicious indicator | Escalate to Tier 2 or Tier 3 |

---

## 8. Windows Event Logs

### Log categories

| Log | Purpose |
|---|---|
| Application | Application errors and information |
| System | Operating system events |
| Security | Authentication, authorization, and auditing |
| Setup | Installation activity |
| Forwarded Events | Events forwarded from other machines |

### Event anatomy

| Field | Meaning |
|---|---|
| Log name | Which log the event is in |
| Source | Software or component that generated it |
| Event ID | Unique identifier for the event type |
| Level | Information, Warning, Error, or Critical |
| Keywords | Audit Success or Audit Failure |
| User | Account involved |
| Logged | Timestamp |
| Computer | Machine name |
| XML data | Complete event data, including extended fields |

### Key Windows event IDs

| ID | Meaning |
|---|---|
| 1102 | Audit log cleared (anti-forensics) |
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4648 | Logon with explicit credentials |
| 4672 | Special privileges assigned |
| 4688 | Process creation |
| 4698 | Scheduled task created |
| 4719 | Audit policy changed |
| 4732 | Member added to a security-enabled group |
| 4733 | Member removed from a security-enabled group |
| 4738 | User account changed |
| 4771, 4776 | Kerberos or domain controller credential failure |
| 4907 | Auditing settings on an object changed |
| 5001 | Defender real-time protection changed |
| 5140 | Network share accessed |
| 5142 | Network share added |
| 5145 | Network share access checked |
| 7045 | Service installed |

### Logon types

| Code | Type | Meaning |
|---|---|---|
| 2 | Interactive | Local console |
| 3 | Network | SMB share access |
| 4 | Batch | Scheduled task |
| 5 | Service | Service account |
| 10 | RemoteInteractive | RDP |
| 11 | CachedInteractive | Cached domain credentials |

### SubStatus codes

| Code | Meaning |
|---|---|
| 0xC000006A | Bad password |
| 0xC0000064 | User does not exist |
| 0xC0000072 | Account disabled |
| 0xC0000234 | Account locked |

### Custom XML query

```xml
<QueryList>
  <Query Id="0" Path="Security">
    <Select Path="Security">*[System[(EventID=4625)]]</Select>
  </Query>
</QueryList>
```

---

## 9. Analyzing Attacks with Sysmon

### Why Sysmon

Sysmon logs activity that the Security event log does not capture: detailed process creation, image loads, network connections, process access, and more. It is configured with an XML file.

### Install and configure

```cmd
sysmon.exe -i -accepteula -h md5,sha256,imphash -l -n
sysmon.exe -c sysmonconfig-export.xml
```

Widely used community configurations include those by SwiftOnSecurity and Olaf Hartong's modular project.

### Key Sysmon event IDs

| ID | Event |
|---|---|
| 1 | Process creation |
| 3 | Network connection |
| 7 | Image (DLL) loaded |
| 8 | CreateRemoteThread (injection) |
| 10 | Process access |
| 11 | File created |
| 12, 13, 14 | Registry events |
| 22 | DNS query |

### Detection 1: DLL hijacking

An attacker places a malicious DLL next to a legitimate executable so the program loads it. In the lab, a reflective DLL was renamed to `WININET.dll`, copied with `calc.exe` into a writable folder, and executed.

Event 7 showed:

```text
Image       : calc.exe
ImageLoaded : WININET.dll
Signed      : false
```

Three indicators:
- `calc.exe` running outside System32
- `WININET.dll` loaded by `calc.exe` from outside System32
- An unsigned `WININET.dll`

### Detection 2: unmanaged PowerShell or C# injection

Running C# code requires the .NET CLR, which loads `clr.dll`, `clrjit.dll`, and `mscoree.dll`.

| Process | Expected |
|---|---|
| `powershell.exe` | Managed (loads CLR), expected |
| `spoolsv.exe` | Unmanaged, so CLR DLLs should not appear |

If `clr.dll` loads into an unmanaged process, suspect injection. Injection via `CreateRemoteThread` appears as Sysmon Event 8, for example with `SourceImage: rundll32.exe` and `TargetImage: Calculator.exe`.

### Detection 3: credential dumping from LSASS

Tools such as Mimikatz (`sekurlsa::logonpasswords`) read memory from `lsass.exe`. Sysmon Event 10 records the process access.

Suspicious signs:
- An unexpected tool accessing `lsass.exe`
- A `SourceUser` different from the `TargetUser`
- `GrantedAccess 0x1FFFFF` (full access, including `VM_READ`)
- `SeDebugPrivilege` requested

Legitimate LSASS access comes from antivirus and EDR products and from authentication processes.

---

## 10. Event Tracing for Windows (ETW)

### What ETW is

ETW is a high-speed, kernel-level tracing facility that captures events from user-mode applications and kernel-mode drivers. It is far more detailed than the event logs alone.

### Architecture

```text
Providers -> Sessions (controllers) -> Consumers or ETL files
```

| Component | Role |
|---|---|
| Controllers | Start and stop sessions, for example with `logman.exe` |
| Providers | Generate events |
| Consumers | Subscribe to events |
| Channels | Filter events |
| ETL files | Durable on-disk trace logs |

### Provider types

- **MOF:** legacy Managed Object Format
- **WPP:** kernel debug tracing
- **Manifest-based:** XML-described, the modern approach
- **TraceLogging:** API-based, minimal overhead

### logman commands

```cmd
logman.exe query -ets                       :: list active sessions
logman.exe query "EventLog-System" -ets     :: inspect a session
logman.exe query providers                  :: list all providers
logman.exe query providers | findstr "Winlogon"
logman.exe query providers Microsoft-Windows-Winlogon
```

The `-ets` flag is required to see live sessions.

### High-value providers

| Provider | Detects |
|---|---|
| Kernel-Process | Process injection and hollowing |
| Kernel-File | Unauthorized file access, ransomware |
| Kernel-Network | C2 and exfiltration |
| DotNETRuntime | Malicious .NET assembly loading |
| PowerShell | Script block logging |
| Kernel-Registry | Persistence |
| DNS-Client | DNS tunneling and C2 |
| CodeIntegrity | Unsigned drivers and code |
| Antimalware-Service, Antimalware-Protection | AV tampering |
| Threat-Intelligence | DFIR-grade threat intelligence (restricted to protected processes) |

### ETW vs. Sysmon

| Aspect | Sysmon | ETW |
|---|---|---|
| Scope | Curated | Kernel and user mode |
| Depth | High-level | Very granular |
| Providers | One driver | More than 1,000 |
| Typical use | Daily SOC monitoring | Deep DFIR and threat hunting |

### ETW in practice

SilkETW captures ETW data to JSON.

- **PPID spoofing:** Sysmon Event 1 shows a fake parent (`spoolsv.exe`), but the Kernel-Process provider reveals the real parent (`powershell.exe`).
- **Malicious .NET:** Sysmon Event 7 shows `clr.dll` and `mscoree.dll` loaded. The DotNETRuntime provider shows the exact interop call and method names, such as `GetTokenInformation`.

---

## 11. Get-WinEvent: PowerShell Log Analysis

Get-WinEvent queries large volumes of logs efficiently. It supports classic logs, modern logs, ETW, and `.evtx` files.

### Discovery

```powershell
Get-WinEvent -ListLog *
Get-WinEvent -ListProvider *
```

### Retrieve events

```powershell
Get-WinEvent -LogName 'System' -MaxEvents 50
Get-WinEvent -LogName 'Security' -Oldest -MaxEvents 30
Get-WinEvent -Path 'C:\path\to\file.evtx'
```

### FilterHashtable (fast)

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; ID=1,3}

Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-Sysmon/Operational'
  ID=1,3
  StartTime=$startDate
  EndTime=$endDate
}
```

The date range includes the start time and excludes the end time.

### FilterXPath (precise)

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-Sysmon/Operational' `
  -FilterXPath "*[System[EventID=3] and EventData[Data[@Name='DestinationIp']='<IP_ADDRESS>']]"
```

FilterXPath is generally faster than piping through `Where-Object`, because the filtering happens inside the event query.

### Property-based filtering

```powershell
# Show every property of one event
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; ID=1} -MaxEvents 1 |
  Select-Object -Property *

# Filter on a property index, here finding encoded PowerShell in the parent command line
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; ID=1} |
  Where-Object { $_.Properties[21].Value -like '*-enc*' } |
  Format-List
```

### Sysmon Event 1 property indexes

| Index | Field |
|---|---|
| 4 | Image |
| 5 | FileVersion |
| 10 | CommandLine |
| 12 | User |
| 20 | ParentImage |
| 21 | ParentCommandLine |

### XML parsing

```powershell
$xml = [xml]$_.ToXml()
$eventData = $xml.Event.EventData.Data
($eventData | Where-Object { $_.Name -eq 'DestinationIp' }).'#text'
```

---

## 12. Field and Query Reference

### Common ECS fields

| Field | Purpose |
|---|---|
| `event.code` | Windows event ID |
| `event.action` | Action, such as `added-member-to-group` |
| `event.outcome` | `success` or `failure` |
| `user.name`, `user.name.keyword` | Username (text and aggregation forms) |
| `host.name`, `host.hostname.keyword` | Machine name |
| `source.ip` | Source IP address |
| `related.ip`, `related.ip.keyword` | Related IP addresses |
| `group.name`, `group.name.keyword` | Group name |
| `winlog.logon.type` | ECS logon type |
| `winlog.event_data.LogonType` | Numeric logon type |
| `winlog.event_data.SubStatus` | Detailed failure reason |
| `winlog.event_data.MemberSid` | SID of the changed group member |
| `winlog.event_data.TargetUserName` | Target account |
| `winlog.channel.keyword` | Channel, such as `Security` |
| `process.name`, `process.parent.name` | Process information |

### KQL to DSL

| KQL | Elasticsearch DSL |
|---|---|
| `event.code: 4625` | `{"term": {"event.code": "4625"}}` |
| `user.name: admin*` | `{"wildcard": {"user.name.keyword": "admin*"}}` |
| `user.name: *admin*` | `{"wildcard": {"user.name.keyword": "*admin*"}}` |
| `A AND B` | `{"bool": {"filter": [A, B]}}` |
| `A OR B` | `{"bool": {"should": [A, B]}}` |
| `NOT A` | `{"bool": {"must_not": [A]}}` |
| `field >= X` | `{"range": {"field": {"gte": X}}}` |

### Common SOC queries

Failed logons, all users:

```json
{"query":{"bool":{"filter":[
  {"term":{"event.code":"4625"}},
  {"term":{"winlog.channel.keyword":"Security"}}
]}}}
```

Failed logons on disabled accounts:

```json
{"query":{"bool":{"filter":[
  {"term":{"event.code":"4625"}},
  {"term":{"winlog.event_data.SubStatus":"0xc0000072"}}
]}}}
```

RDP logons by service accounts:

```json
{"query":{"bool":{"filter":[
  {"term":{"event.code":"4624"}},
  {"term":{"winlog.logon.type.keyword":"RemoteInteractive"}},
  {"wildcard":{"user.name.keyword":"svc-*"}}
]}}}
```

Administrators group changes:

```json
{"query":{"bool":{"filter":[
  {"terms":{"event.code":["4732","4733"]}},
  {"term":{"group.name.keyword":"Administrators"}}
]}}}
```

Accounts with "admin" in the name:

```json
{"query":{"bool":{"filter":[
  {"term":{"event.code":"4625"}},
  {"wildcard":{"user.name.keyword":"*admin*"}}
]}}}
```

Exclude computer accounts:

```json
{"must_not":[
  {"wildcard":{"user.name.keyword":"*$"}}
]}
```

---

## 13. Skills Assessment: Worked Answers

### Environment context

- Cloud-hosted, with a closed DMZ.
- IT Operations has four people, all with privileged accounts.
- A privileged access workstation (PAW) is required for administrative activity.
- Service accounts carry a `-svc` suffix.
- Root cannot log in remotely, and sudo is required.

### SOC dashboard assessment

| # | Visualization | Answer | Reasoning |
|---|---|---|---|
| 1 | Failed logons, all users | Consult IT Operations | `sql-svc1` fails logons, which suggests a service misconfiguration |
| 2 | Failed logons, disabled accounts | Consult IT Operations | `anni` is disabled. Confirm whether that is expected |
| 3 | Failed logons, admins | Nothing suspicious | All events are on the PAW and domain controller |
| 4 | RDP by service accounts | Escalate to Tier 2 or 3 | `svc-sql1` does not follow the `-svc` naming convention |
| 5 | Admin group change | Consult IT Operations | The member appears only as a SID. Confirm the change with IT Operations |
| 6 | Admin logon not from PAW | Consult IT Operations | IT Operations is known to deviate from policy |
| 7 | SSH logins | Escalate to Tier 2 or 3 | Root SSH violates policy |

### Log analysis assessment

| # | Question | Answer | Evidence |
|---|---|---|---|
| 1 | DLL hijack process | Dism.exe | `C:\ProgramData\Dism.exe` loaded `C:\ProgramData\DismCore.dll` |
| 2 | Unmanaged PowerShell host | Calculator.exe | Loaded `clr.dll` and `clrjit.dll` |
| 3 | Injector process | rundll32.exe | Event 8: `rundll32.exe` injected into `Calculator.exe` via CreateRemoteThread |
| 4 | LSASS dumper | ProcessHacker.exe | Event 10 with `GrantedAccess 0x1FFFFF` |
| 5 | Logon after the dump | No | All 4624 events were SYSTEM logons of Type 5, with no suspicious IP addresses |
| 6 | Unusual parent process ID | WerFault.exe | `WerFault.exe` spawned `cmd.exe` with a spoofed parent |

### Other module answers

**Windows event logs, auditing modification**
- Q1: `TiWorker.exe` (Event 4907 on the WPF DLL)
- Q2: 10:23:50 (timestamp of the 4907 event)

**Sysmon and event logs, attacks**
- Q1: the DLL hijack file hash was recorded in the lab.
- Q2: the `clrjit.dll` hash was recorded in the lab.
- Q3: an NTLM hash was recorded in the lab.

**ETW, SilkETW and Seatbelt**
- Q1: `GetTokenInformation` (from the `ManagedInteropMethodName` field of DotNETRuntime)

**Get-WinEvent, share creation**
- Q1: 12:30:30 (the time of the `\\*\PRINT` share, from Event 5142)

---

## 14. Analyst Mindset and Key Takeaways

### Decision framework

```text
1. REVIEW   What does the alert say?
2. ENRICH   What is the context?
3. ASSESS   Is this expected behavior?
4. DECIDE
      Baseline                 -> nothing suspicious
      Operational noise        -> consult IT Operations
      Policy violation or IOC  -> escalate to Tier 2 or Tier 3
5. DOCUMENT everything, every time
```

### Golden rules

- Establish a baseline. Know what normal looks like.
- Naming conventions matter. Violations are red flags.
- Separate policy from practice. Known deviations call for consultation, not automatic escalation.
- Weigh context over count. Eight hits can be benign, and one can be critical.
- Use `.keyword` for aggregations and base fields for KQL.
- Document everything in the SOP, the IRP, the ticket, and your notes.

### Layered telemetry

| Layer | Strength | Weakness |
|---|---|---|
| Event logs | Authentication, group changes, services | Limited process and file detail |
| Sysmon | Process trees, image loads, process access | Single driver, curated scope |
| ETW | Deep kernel telemetry, interop calls | Complex, needs tools such as SilkETW |
| SIEM (Elastic) | Central correlation, dashboards | Requires tuning |

### Indicator quick reference

| Attack | Key indicator |
|---|---|
| DLL hijacking | Unsigned DLL loaded from outside System32 |
| Unmanaged PowerShell or C# | CLR DLLs in a non-managed process |
| Injection | Sysmon Event 8 (CreateRemoteThread), with Event 10 showing `0x1FFFFF` |
| LSASS dumping | Process access to `lsass.exe` with `VM_READ` |
| PPID spoofing | Unexpected parent-child pair |
| Credential misuse | Failed logons on a disabled account |
| Persistence | Event 7045 (service installed) or Event 4698 (scheduled task created) |

### Troubleshooting

| Issue | Fix |
|---|---|
| Empty results in PowerShell | Use `-FilterXPath` or `-FilterHashtable` instead of filtering after the fact |
| `.keyword` returns 0 | Keyword matching is case-sensitive. Match the exact case, or use the text field |
| Aggregation fails | Aggregate on the `.keyword` subfield |
| Wrong Sysmon property | Dump all properties and check the index numbers |
| Weekly buckets in Kibana | Set an absolute time range |
| No ETW data | Confirm the session with `logman query -ets`, and check that SilkETW is running |

---

## Conclusion

A SOC analyst's job is more than watching alerts. It involves:

- Understanding the available telemetry: event logs, Sysmon, and ETW
- Querying it efficiently with KQL, PowerShell, and the Elasticsearch DSL
- Correlating across sources and timelines
- Deciding with structure: triage, then escalate, consult, or close
- Documenting every decision
