# Threat Hunting, Threat Intelligence, and Splunk: Course Notes

A master reference covering threat hunting fundamentals, the hunting process, threat intelligence, the Stuxbot hunts, Splunk and SPL basics, Splunk applications, real-world intrusion detection, TTP-based and analytics-based detection, and log source investigation.

---

## Table of Contents

1. [Threat Hunting Fundamentals](#1-threat-hunting-fundamentals)
2. [The Threat Hunting Process](#2-the-threat-hunting-process)
3. [Threat Hunting Glossary](#3-threat-hunting-glossary)
4. [Threat Intelligence Fundamentals](#4-threat-intelligence-fundamentals)
5. [Hunting for Stuxbot: Round 1](#5-hunting-for-stuxbot-round-1)
6. [Hunting for Stuxbot: Round 2](#6-hunting-for-stuxbot-round-2)
7. [Introduction to Splunk and SPL](#7-introduction-to-splunk-and-spl)
8. [Using Splunk Applications](#8-using-splunk-applications)
9. [Intrusion Detection with Splunk](#9-intrusion-detection-with-splunk)
10. [Detecting Attacker Behavior: TTPs](#10-detecting-attacker-behavior-ttps)
11. [Detecting Attacker Behavior: Analytics](#11-detecting-attacker-behavior-analytics)
12. [Splunk Skills Assessment](#12-splunk-skills-assessment)
13. [Log Sources and Investigation](#13-log-sources-and-investigation)
14. [Appendix A: Master Cheat Sheet](#appendix-a-master-cheat-sheet)
15. [Appendix B: Windows and Sysmon Event Reference](#appendix-b-windows-and-sysmon-event-reference)
16. [Appendix C: Splunk Field Names](#appendix-c-splunk-field-names)
17. [Appendix D: Curl Templates](#appendix-d-curl-templates)

---

## 1. Threat Hunting Fundamentals

Threat hunting is an active, human-led, hypothesis-driven practice that searches data for stealthy, advanced threats that evade existing controls. It is proactive and offensive in posture, not passive or alert-driven. Its main goal is to reduce **dwell time**, the gap between breach and detection, which is often weeks to months.

Hunting catches attackers earlier in the kill chain, before they entrench.

### Key facets

- Proactive strategy built on hypotheses, TTPs, and intelligence
- Reactive response to a verified incident
- Knowledge of the threat landscape and kill chain
- **Cognitive empathy**: thinking like the attacker
- Deep knowledge of the environment, assets, and normal activity
- High-fidelity data, tactical analytics, and advanced tooling

### Hunting and incident handling

| Incident phase | Hunter's role |
|---|---|
| Preparation | Rules of engagement, protocols |
| Detection and analysis | Validate IOCs, expand the investigation |
| Containment, eradication, recovery | Varies by organization |
| Post-incident | Recommend improvements |

### Team roles

| Role | Responsibility |
|---|---|
| Threat hunter | Proactive search |
| Threat intelligence analyst | Gather and analyze intelligence |
| Incident responder | Manage confirmed incidents |
| Forensics expert | Malware analysis, DFIR |
| Data analyst or scientist | Statistics, machine learning, data mining |
| Security engineer or architect | Secure infrastructure, hunting tooling |
| Network security analyst | Traffic patterns and anomalies |
| SOC manager | Coordination |

### When to hunt

1. New adversary or vulnerability intelligence appears.
2. New IOCs are linked to a known adversary.
3. Several network anomalies occur together.
4. During an active incident.
5. On a periodic, proactive schedule.

### Risk assessment and hunting

Risk assessment shows where to hunt. It prioritizes crown-jewel assets, informs the likely TTPs, highlights exploitable weaknesses, directs threat intelligence use, and refines incident response plans and controls.

---

## 2. The Threat Hunting Process

The hunting cycle has eight phases.

| Phase | What happens |
|---|---|
| 1. Setting the stage | Plan, enable logging, configure SIEM, EDR, and IDS |
| 2. Formulating hypotheses | Make specific, testable predictions |
| 3. Designing the hunt | Choose data sources, tools, IOCs, and queries |
| 4. Gathering and examining data | Collect and analyze, iteratively |
| 5. Evaluating findings | Confirm or refute; establish scope |
| 6. Mitigating threats | Isolate, remove, patch, block |
| 7. After the hunt | Document, share, update rules and playbooks |
| 8. Continuous learning | Review, improve, train |

### Example: hunting Emotet

- **Hypothesis:** Emotet is using compromised email accounts to send phishing emails with malicious macro documents.
- **Design:** Email, network, endpoint, and sandbox data. Search for suspicious subjects, attachments, C2 addresses, and hashes.
- **Gather and evaluate:** Email header analysis, traffic patterns, sandboxed samples. Confirm phishing and C2 activity.
- **Mitigate:** Isolate hosts, remove malware, clean mailboxes, block C2 domains.
- **After the hunt:** Update IOCs, detection rules, and playbooks. Move toward behavior-based and machine learning detection.

---

## 3. Threat Hunting Glossary

| Term | Meaning |
|---|---|
| Adversary | An unauthorized party: criminal, insider, hacktivist, or state actor |
| Threat | Intent + capability + opportunity |
| Intent | Why they target you |
| Capability | Tools, skills, money, resources |
| Opportunity | Favorable conditions such as leaked credentials or exposure |
| Campaign | Incidents sharing TTPs and goals |

### APT (advanced persistent threat)

Organized, well-resourced groups, often nation-states, that target high-value sectors over long periods. "Advanced" refers to sophisticated planning, not always high technology.

### TTPs

| Part | Meaning | Question |
|---|---|---|
| Tactics | Strategic goals | Why |
| Techniques | General methods | How |
| Procedures | Step-by-step implementation | Exactly how |

### Indicators

Data plus context is an indicator. IOCs (hashes, IPs, domains, executable names) are the most common form.

### Pyramid of Pain

| Level | Indicator | Difficulty to change |
|---|---|---|
| 1 | Hash values | Trivial |
| 2 | IP addresses | Easy |
| 3 | Domain names | Simple |
| 4 | Network and host artifacts | Annoying |
| 5 | Tools | Challenging |
| 6 | TTPs | Tough |

The higher you detect, the more pain you impose on the adversary.

### Diamond Model

| Component | Meaning |
|---|---|
| Adversary | The attacker |
| Capability | Tools and TTPs |
| Infrastructure | Servers, domains, IP addresses |
| Victim | The target |

The Kill Chain describes **stages** of an attack. The Diamond Model describes **components and their relationships**. They complement each other.

---

## 4. Threat Intelligence Fundamentals

Cyber threat intelligence (CTI) moves defense from reactive to proactive by giving the SOC knowledge of adversaries before attacks unfold.

### The four pillars

| Pillar | Meaning |
|---|---|
| Relevance | Applies to your organization or partners |
| Timeliness | Fresh intelligence is valuable; old intelligence decays |
| Actionability | Gives defenders a clear next step |
| Accuracy | Verified, or labeled with a confidence level |

Unactionable intelligence produces endless analysis with no output, sometimes called the "self-licking ice cream cone."

### Threat intelligence and threat hunting

| | Threat intelligence | Threat hunting |
|---|---|---|
| Nature | Predictive | Reactive and proactive |
| Goal | Anticipate adversary moves | Determine presence |
| Answers | Where, when, how, why | Are they here? Did we miss them? |
| Trigger | Continuous analysis | An event or incident |

The two feed each other. Adversary profiles inform hunts, and hunt findings refine intelligence.

### Three categories of CTI

| Type | Audience | Focus | Answers |
|---|---|---|---|
| Strategic | Executives | Long-term, business risk | Who and why |
| Operational | Mid-level management | Campaigns and TTPs | How and where |
| Tactical | SOC and IR | Immediate and technical | What to detect or block |

### Processing a tactical report

1. Understand the scope and narrative.
2. Classify IOCs: network, host, and email.
3. Map TTPs to MITRE ATT&CK.
4. Validate IOCs with sources such as VirusTotal and AlienVault OTX. Consider IOC age and false-positive rates.
5. Integrate IOCs into firewalls, EDR, IDS and IPS, and email gateways. Weigh business impact, since alerting is sometimes better than blocking. Follow change management.
6. Hunt proactively for IOCs and broader TTPs.
7. Monitor, improve detections, and share new IOCs with ISACs and ISAOs.

---

## 5. Hunting for Stuxbot: Round 1

### The threat

| Attribute | Detail |
|---|---|
| Motivation | Espionage |
| Targeting | Opportunistic |
| Initial access | Phishing |
| Impact | Full takeover and domain escalation |
| Risk | Critical |

### Attack chain

```text
Phishing email
  -> OneNote file (invoice.one), hosted on file-sharing services
  -> Batch file (invoice.bat) launched from the OneNote file
  -> PowerShell fetched from Pastebin
  -> RAT executable (default.exe)
  -> Registry Run key persistence
  -> Lateral movement (PsExec, WinRM)
  -> Credential theft and DCSync (Mimikatz)
```

### Known IOC categories

- OneNote lure files on file-sharing services
- Pastebin raw URLs used for PowerShell staging
- C2 endpoints on port 443
- SHA256 hashes of the malicious files

Exact values are in the original hunt material and are not reproduced here.

### Environment

About 200 employees in marketing, using Office, Gmail, Microsoft Edge, TeamViewer, and Active Directory GPOs. The SIEM is Elastic Stack. Logs include Windows, Sysmon, and PowerShell (`windows*`) plus Zeek (`zeek*`).

### Hunt walkthrough

| Step | Focus | Finding |
|---|---|---|
| 1 | Sysmon Event 15 for `invoice.one` | Downloaded through Edge to Bob's Downloads on WS001 |
| 2 | Event 11 for `invoice.one:Zone.Identifier` | Confirms an internet origin |
| 3 | Zeek DNS for the host | File-sharing lookups and a SmartScreen scan |
| 4 | Sysmon Event 1 for `invoice.one` | OneNote opened about six seconds later |
| 5 | Sysmon Event 1 with `ONENOTE.EXE` as parent | `cmd.exe` runs `invoice.bat` from a temp path |
| 6 | Sysmon Event 1 with `invoice.bat` in the parent command line | PowerShell downloads from Pastebin |
| 7 | Process ID of that PowerShell | Password spraying, `default.exe` dropped, ngrok DNS, DC query |
| 8 | Zeek connections to the C2 address | Activity continues; the ngrok address changes |
| 9 | Process name `default.exe` | Drops `svchost.exe`, `SharpHound.exe`, `payload.exe`, and a VBS script |
| 10 | Process name `SharpHound.exe` | Executed twice, using the "all" collection method |
| 11 | SHA256 of SharpHound | Found on WS001 and the PKI server |
| 12 | Events 4624 and 4625, LogonType 3, from the patient-zero IP | Failed administrator logons; successful `svc-sql1` logons |

### Key findings

| Finding | Detail |
|---|---|
| Patient zero | WS001, user `bob`, 192.168.28.130 |
| Persistence | `default.exe` |
| C2 | ngrok |
| Credential attack | Password spray. Administrator failed; `svc-sql1` succeeded |
| Lateral movement | PsExec (PSEXESVC) to the PKI server |
| AD reconnaissance | SharpHound, run twice |
| Masquerading | `svchost.exe` posing as a legitimate process |

### Skill assessment answers

| Question | Answer |
|---|---|
| VBS file dropped by `default.exe` | `XceGuhkzaTrOy.vbs` |
| Mimikatz arguments | `lsadump::dcsync /domain:eagle.local /all /csv, exit` |
| PowerShell tool for network shares | `PowerView` |

The Mimikatz answer comes from the `process.args` array, joined after the executable with commas. The PowerView functions `Invoke-ShareFinder`, `Find-DomainShare`, and `Get-NetShare` identify the tool.

---

## 6. Hunting for Stuxbot: Round 2

### New TTPs

- Supplementary tools deployed to `C:\Users\Public`
- Registry Run key persistence
- PowerShell Remoting for lateral movement, including to domain controllers

### Hunt 1: tool transfer to `C:\Users\Public`

Find a file in `C:\Users\Public` whose name starts with "r," and report the user.

```spl
index=* EventCode=11 file.path="C:\\Users\\Public*"
| table _time, host.name, user.name, file.name, file.path, process.name
```

Result: `Rubeus.exe`, created by `svchost.exe` on PKI.eagle.local under `svc-sql1`.

**Answer: `svc-sql1`**

### Hunt 2: Run key persistence

Find the first malicious Run key value and report its name. Ignore legitimate entries such as `MicrosoftEdgeAutoLaunch` and Office restart entries.

```spl
index=* EventCode=13 registry.path="*CurrentVersion\\Run*"
| table _time, host.name, user.name, registry.path, registry.value, registry.data.strings, process.name
| sort _time
```

The first malicious value was created by `powershell.exe` under user `bob` and pointed to `default.exe` in the user's Temp directory.

**Answer: `LgvHsviAUVTsIN`**

### Hunt 3: PowerShell Remoting to DC1

Find the PowerShell Remoting session toward DC1 and report the user.

```spl
index=* EventCode=4104 powershell.file.script_block_text="*Enter-PSSession*"
| table _time, host.name, winlog.user.name, powershell.file.script_block_text
```

A wildcard such as `*Enter-PSSession*DC1*` fails when DC1 is not literally after the cmdlet in the same string, for example when it is passed as a separate parameter. Search for the cmdlet alone, then review the results.

As a cross-check, explicit-credential events (4648) from PowerShell show the same activity.

**Answer: `svc-sql1`**

### Round 2 answer sheet

| Hunt | Field | Answer |
|---|---|---|
| 1 | `user.name` | `svc-sql1` |
| 2 | `registry.value` | `LgvHsviAUVTsIN` |
| 3 | `winlog.user.name` | `svc-sql1` |

---

## 7. Introduction to Splunk and SPL

Splunk ingests, indexes, searches, analyzes, and visualizes machine data. It supports SIEM, compliance, IT monitoring, observability, and data pipelines.

### Architecture

| Component | Role |
|---|---|
| Forwarders | Collect and send data |
| Indexers | Store data and process searches |
| Search heads | Coordinate searches and host the UI |
| Deployment server | Distributes forwarder configuration |
| Cluster manager | Coordinates indexer clusters |
| License manager | Licensing |

### Forwarder types

| Type | Description |
|---|---|
| Universal (UF) | Lightweight, no preprocessing |
| Heavy (HF) | Parses and routes; Enterprise only |
| HTTP Event Collector (HEC) | Token-based JSON or raw API |

### Key components

- Web interface for searching, alerting, and dashboards
- SPL, the search language
- Apps and add-ons from Splunkbase
- Technology add-ons (TAs), which handle data collection and field extraction
- Knowledge objects: fields, tags, event types, lookups, macros, data models, and alerts

### SPL basics

```spl
index="main" "UNKNOWN"
index="main" "*UNKNOWN*"
index="main" EventCode!=1
```

### Essential commands

| Command | Purpose |
|---|---|
| `fields` | Include or exclude fields |
| `table` | Format as a table |
| `rename` | Rename a field |
| `dedup` | Remove duplicates |
| `sort` | Sort results |
| `stats` | Aggregate |
| `chart` | Pivot-style aggregation |
| `eval` | Create or modify a field |
| `rex` | Regex extraction |
| `lookup` | Enrich from a CSV |
| `inputlookup` | Read a lookup file |
| `transaction` | Group related events |
| `earliest`, `latest` | Time filters |
| `top`, `rare` | Most and least common values |
| `eventcount` | Count events per index |
| `metadata` | List sourcetypes and sources |
| `fieldsummary` | Summarize fields |
| `bucket` | Create time buckets |

`stats` returns long-format results, one row per group. `chart` returns wide-format results, which suit time series.

### Transaction

```spl
index=main sourcetype="WinEventLog:Sysmon" (EventCode=1 OR EventCode=3)
| transaction Image startswith=eval(EventCode=1) endswith=eval(EventCode=3) maxspan=1m
| table Image
| dedup Image
```

### Subsearch

```spl
index=main sourcetype="WinEventLog:Sysmon" EventCode=1
  NOT [ search index=main sourcetype="WinEventLog:Sysmon" EventCode=1
        | top limit=100 Image
        | fields Image ]
| table _time, Image, CommandLine, User, ComputerName
```

### Lookup enrichment

```spl
index=main sourcetype="WinEventLog:Sysmon" EventCode=1
| rex field=Image "(?P<filename>[^\\\\]+)$"
| eval filename=lower(filename)
| lookup malware_lookup.csv filename OUTPUTNEW is_malware
| table filename, is_malware
```

### Data discovery

```spl
| eventcount summarize=false index=* | table index
| metadata type=sourcetypes index=* | table sourcetype
| metadata type=sources index=* | table source
sourcetype="WinEventLog:Security" | fieldsummary
index=* sourcetype=* | rare limit=10 index, sourcetype
```

---

## 8. Using Splunk Applications

An **app** is a full solution with UI, dashboards, and searches. A **TA** handles data collection and field extraction.

### Installation

1. Download the package from Splunkbase.
2. Go to Apps, then Manage Apps, then Install from file.
3. Upload the package, and upgrade if prompted.
4. Edit the app's macro so it points to your index and sourcetype.
5. Review the dashboards.

### Example: Sysmon app

The critical configuration is the macro:

```spl
index="main" sourcetype="WinEventLog:Sysmon"
```

### Fixing a broken panel

A panel that uses `sysmon EventCode=11 | top Computer` fails because the field is `ComputerName`. Use `top ComputerName`.

### Debugging checklist

1. Set the time range to All time.
2. Confirm the macro points to the correct index and sourcetype.
3. Confirm the field names exist.
4. Confirm the sourcetype matches.
5. Run the SPL manually.
6. Use `fieldsummary` to see the actual fields.

### Skill assessment answers

| Question | Answer |
|---|---|
| Complete the `net view` command | `net view /DOMAIN:uniwaldo.local` |
| SharpHound network connections | Count of EventCode 3 where `Image="*SharpHound.exe"` |

---

## 9. Intrusion Detection with Splunk

The real-world dataset contains about 581,000 events across several infections. Retrieve all of them with `index="main" earliest=0`.

### Search efficiency

| Query | Speed |
|---|---|
| `index="main" uniwaldo.local` | Fast |
| `index="main" *uniwaldo.local*` | Slow (leading wildcard) |
| `index="main" ComputerName="*uniwaldo.local"` | Fastest (targeted field) |

### Parent-child hunting

Start with parent-child pairs:

```spl
index="main" sourcetype="WinEventLog:Sysmon" EventCode=1
| stats count by ParentImage, Image
```

Narrow to suspicious children:

```spl
index="main" sourcetype="WinEventLog:Sysmon" EventCode=1
  (Image="*cmd.exe" OR Image="*powershell.exe")
| stats count by ParentImage, Image
```

Notepad spawning PowerShell stands out. Isolating it shows PowerShell downloading content from an internal IP on port 8080 under SYSTEM.

### Pivot on the IP address

```spl
index="main" 10.0.0.229 | stats count by sourcetype

index="main" 10.0.0.229 sourcetype="WinEventLog:Sysmon"
| stats count by CommandLine, host
```

The results showed a compromised Linux host and several Windows hosts pulling tools from the same server.

### DCSync detection

```spl
index="main" EventCode=4662 Access_Mask=0x100 Account_Name!=*$
```

GUIDs to match:
- `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2`: DS-Replication-Get-Changes-All
- `19195a5b-6da0-11d0-afd3-00c04fd930c9`: DS-Replication-Get-Changes

If DCSync occurred, rotate the krbtgt account, since a golden ticket may be possible.

### LSASS dump detection

```spl
index="main" EventCode=10 lsass
| stats count by SourceImage
```

Suspicious signs include unusual sources such as `notepad.exe` or `rundll32.exe`, `GrantedAccess=0x1FFFFF`, and an `UNKNOWN` entry in the call trace.

### UNKNOWN call trace alert

Filter progressively to remove legitimate callers:

```spl
index="main" CallTrace="*UNKNOWN*"
  SourceImage!="*Microsoft.NET*"
  CallTrace!=*ni.dll* CallTrace!=*clr.dll*
  CallTrace!=*wow64*
  SourceImage!="C:\\Windows\\Explorer.EXE"
| where SourceImage!=TargetImage
| stats count by SourceImage, TargetImage, CallTrace
```

Be aware of bypasses: any DLL with `ni` in its name, and injection through a signed process.

### Module answers

| Question | Answer |
|---|---|
| Other process that dumped LSASS | `rundll32.exe` |
| Misused DLL | `comsvcs.dll` |
| Temporary code execution host | `rundll32.exe` |
| C2 addresses | `10.0.0.186` and `10.0.0.91` |
| C2 port | 3389 (RDP) |

---

## 10. Detecting Attacker Behavior: TTPs

There are two approaches:

| Approach | Mindset |
|---|---|
| Known TTPs | Spot the known |
| Analytics | Spot the unusual |

The core pattern is:

```spl
index=main sourcetype=... EventCode=N <filter>
| stats count by <fields>
| sort - count
```

### Detection examples

**Reconnaissance with native tools:**

```spl
index="main" sourcetype="WinEventLog:Sysmon" EventCode=1
  (Image="*\\ipconfig.exe" OR Image="*\\net.exe" OR Image="*\\whoami.exe"
   OR Image="*\\netstat.exe" OR Image="*\\nbtstat.exe"
   OR Image="*\\hostname.exe" OR Image="*\\tasklist.exe")
| stats count by Image, CommandLine
| sort - count
```

**Payloads from whitelisted domains:**

```spl
index="main" sourcetype="WinEventLog:Sysmon" EventCode=22 QueryName="*github*"
| stats count by Image, QueryName
```

**PsExec, three angles:**

```spl
# Service ImagePath registry change (Event 13)
index="main" sourcetype="WinEventLog:Sysmon" EventCode=13
  Image="C:\\Windows\\system32\\services.exe"
  TargetObject="HKLM\\System\\CurrentControlSet\\Services\\*\\ImagePath"
| rex field=Details "(?<reg_file_name>[^\\\\]+)$"
| eval file_name=coalesce(file_name, lower(reg_file_name))
| stats values(Details) as RegistryDetails, count by file_name, ComputerName

# File creation (Event 11)
index="main" sourcetype="WinEventLog:Sysmon" EventCode=11 Image=System
| stats count by TargetFilename

# Named pipe (Event 18)
index="main" sourcetype="WinEventLog:Sysmon" EventCode=18 Image=System
| stats count by PipeName
```

**Archive creation:**

```spl
index="main" EventCode=11
  (TargetFilename="*.zip" OR TargetFilename="*.rar" OR TargetFilename="*.7z")
| stats count by ComputerName, User, TargetFilename
| sort - count
```

**Downloads by PowerShell and Edge:**

```spl
index="main" sourcetype="WinEventLog:Sysmon" EventCode=11 Image="*powershell.exe*"
| stats count by Image, TargetFilename
| sort + count

index="main" sourcetype="WinEventLog:Sysmon" EventCode=11
  Image="*msedge.exe" TargetFilename="*Zone.Identifier"
| stats count by TargetFilename
| sort + count
```

**Execution from the Downloads folder:**

```spl
index="main" EventCode=1
| regex Image="C:\\\\Users\\\\.*\\\\Downloads\\\\.*"
| stats count by Image
```

**Executables and DLLs outside Windows directories:**

```spl
index="main" EventCode=11
  (TargetFilename="*.exe" OR TargetFilename="*.dll")
  TargetFilename!="*\\windows\\*"
| stats count by User, TargetFilename
| sort + count
```

**Typosquatted tools:**

```spl
index="main" sourcetype="WinEventLog:Sysmon" EventCode=1
  (CommandLine="*psexe*.exe" OR ParentImage="*psexe*.exe" OR Image="*psexe*.exe")
  NOT (CommandLine="*PSEXESVC.exe" OR CommandLine="*PsExec64.exe")
| table Image, CommandLine, ParentImage, ParentCommandLine
```

**Traffic on non-standard ports:**

```spl
index="main" EventCode=3
  NOT (DestinationPort=80 OR DestinationPort=443 OR DestinationPort=22 OR DestinationPort=21)
| stats count by SourceIp, DestinationIp, DestinationPort
| sort - count
```

### Limitations

TTP detection is easy for attackers to evade, requires constant updates, and misses novel techniques. Pair it with analytics.

### Module answer

| Question | Answer |
|---|---|
| Password used during PsExec | `Password@123` |

Credential reuse let the attacker move laterally across several hosts.

---

## 11. Detecting Attacker Behavior: Analytics

### Core mechanism: streamstats

```spl
index="main" sourcetype="WinEventLog:Sysmon" EventCode=3
| bin _time span=1h
| stats count as NetworkConnections by _time, Image
| streamstats time_window=24h avg(NetworkConnections) as avg stdev(NetworkConnections) as stdev by Image
| eval isOutlier=if(NetworkConnections > avg + (0.5*stdev), 1, 0)
| search isOutlier=1
```

The anomaly rule is: value > average + (multiplier × standard deviation).

| Multiplier | Effect |
|---|---|
| 0.5 | Sensitive and noisy |
| 1.0 | Balanced |
| 1.5 | Conservative |
| 2.0 and above | Only extreme outliers |

### Detection examples

**Unusually long command lines:**

```spl
index="main" sourcetype="WinEventLog:Sysmon" Image=*cmd.exe
  ParentImage!="*msiexec.exe" ParentImage!="*explorer.exe"
| eval len=len(CommandLine)
| table User, len, CommandLine
| sort - len
```

**Frequency outliers for cmd.exe:**

```spl
index="main" EventCode=1 CommandLine="*cmd.exe*"
| bucket _time span=1h
| stats count as cmdCount by _time, User, CommandLine
| eventstats avg(cmdCount) as avg stdev(cmdCount) as stdev
| eval isOutlier=if(cmdCount > avg + 1.5*stdev, 1, 0)
| search isOutlier=1
```

**Processes loading many distinct DLLs:**

```spl
index="main" EventCode=7
  NOT (Image="C:\\Windows\\System32*")
  NOT (Image="C:\\Program Files (x86)*")
  NOT (Image="C:\\Program Files*")
  NOT (Image="C:\\ProgramData*")
| bucket _time span=1h
| stats dc(ImageLoaded) as unique_dlls_loaded by _time, Image
| where unique_dlls_loaded > 3
| stats count by Image, unique_dlls_loaded
| sort - unique_dlls_loaded
```

**Repeated process instances per host:**

```spl
index="main" sourcetype="WinEventLog:Sysmon" EventCode=1
| transaction ComputerName, Image
| where mvcount(ProcessGuid) > 1
| stats count by Image, ParentImage
```

### Choosing the right command

| Command | Scope | Use |
|---|---|---|
| `stats` | Whole result set | Simple aggregation |
| `eventstats` | Whole set, applied to each event | Compare each row to a global baseline |
| `streamstats` | Rolling window | Time-series anomaly detection |

### Tuning filters

```spl
NOT (Image="C:\\Windows\\System32*")
NOT (Image="C:\\Program Files (x86)*")
NOT (Image="C:\\Program Files*")
NOT (Image="C:\\ProgramData*")
ParentImage!="*msiexec.exe"
ParentImage!="*explorer.exe"
```

### Limitations

- Legitimate spikes cause false positives.
- Careful attackers stay under the threshold, causing false negatives.
- An attacker active during the baseline period becomes "normal."
- Thresholds must be tuned for each environment.

Analytics catches unknown TTPs. The combination of known TTPs, analytics, environment knowledge, and tuning gives the best detection.

### Module answer

| Question | Answer |
|---|---|
| Outlier for thread injection (2σ) | `randomfile.exe` (34 injections against an average of 10, with a threshold of about 32.8) |

---

## 12. Splunk Skills Assessment

### Full infection chain

```text
rundll32.exe demon.dll,havoc            <- Havoc C2 loader starts the infection
  -> randomfile.exe creates remote threads in rundll32.exe
  -> LSASS dump (rundll32 with comsvcs.dll MiniDump)
  -> C2 beacons to 10.0.0.91 and 10.0.0.186
  -> PsExec with Password@123 to 10.0.0.47
  -> RDP session (3389)
  -> DCSync as waldo, dumping domain hashes
  -> Full domain compromise
```

### Key answers

| Question | Answer |
|---|---|
| Process that created remote threads in rundll32 | `randomfile.exe` |
| Process that started the infection | `rundll32.exe` |

### The malicious command line

```text
rundll32.exe demon.dll,havoc
```

- `demon.dll` is a payload outside System32.
- `havoc` is an exported function of the Havoc C2 framework.
- The parent is a bare `cmd.exe`, consistent with a phishing or exploit launcher.
- The user is `waldo`, on host `DESKTOP-EGSS5IS`.

### Legitimate versus malicious rundll32

| Time | Command line | Verdict |
|---|---|---|
| 13:12:55 | `rundll32.exe AppXDeploymentExtensions.OneCore.dll,ShellRefresh` | Legitimate (Windows Store) |
| 13:25:24 | `rundll32.exe Windows.StateRepositoryClient.dll,...` | Legitimate (State Repository) |
| 13:33:31 | `rundll32.exe demon.dll,havoc` | Malicious |

Legitimate rundll32 usage loads DLLs from `C:\Windows\System32\`. A relative path is suspicious.

---

## 13. Log Sources and Investigation

Effective investigation begins with knowing what data you have. Before hunting, alerting, or investigating, answer these questions: What log sources exist? What events do they record? What fields do they expose? How do they relate to each other?

### Log source categories

| Category | Examples | Primary use |
|---|---|---|
| Endpoint and OS | Windows Event Logs, Sysmon, auditd | Process, file, registry, logon events |
| Network | Zeek, firewall, proxy, DNS | Connections, beacons, exfiltration |
| Application | IIS, Apache, SQL, AD | Application-level events |
| Identity | AD (4662, 4768), Azure AD | Authentication and privilege |
| Cloud | CloudTrail, Azure Activity | Cloud resource actions |
| Security tools | EDR, AV, IDS and IPS | Detection alerts |
| Custom | Applications, scripts, APIs | Business logic |

### Windows Event Logs

| Log | Contents |
|---|---|
| Security | Logons, privileges, directory service (4624, 4625, 4662, 4768, 4769) |
| System | Service and driver events (7045, 7036, 7034) |
| Application | Crashes and errors |
| PowerShell | Script blocks (4104), module logs |
| Sysmon | Deep endpoint telemetry |
| TaskScheduler | Scheduled task activity |
| TerminalServices | RDP sessions |

Common fields include `EventCode`, `ComputerName`, `User`, `LogName`, `SourceName`, `Message`, and `_raw`.

### Sysmon event codes

| Code | Name | Detection value |
|---|---|---|
| 1 | Process creation | Parent-child relationships and execution paths |
| 3 | Network connection | Beaconing and C2 |
| 7 | Image loaded | DLL hijacking |
| 8 | CreateRemoteThread | Injection |
| 10 | Process access | LSASS dumping |
| 11 | File created | Dropped tools |
| 12, 13, 14 | Registry | Persistence |
| 15 | FileCreateStreamHash | Mark-of-the-Web |
| 17, 18 | Pipe | PsExec and lateral movement |
| 22 | DNS event | C2 beacons and DNS abuse |
| 23 | File delete | Cleanup |

Key Sysmon fields: `Image`, `ParentImage`, `CommandLine`, `ParentCommandLine`, `ProcessId`, `ParentProcessId`, `ProcessGuid`, `User`, `TargetFilename`, `ImageLoaded`, `TargetObject`, `Details`, `SourceIp`, `DestinationIp`, `SourcePort`, `DestinationPort`, `QueryName`, and `QueryResults`.

### PowerShell logs

| Log | Event code | Content |
|---|---|---|
| Script block logging | 4104 | Full script text |
| Module logging | 4103 | Cmdlet execution |
| Transcription | 4104 or transcript | Full session |

Attackers rely heavily on PowerShell. These logs reveal encoded commands, download cradles, `Invoke-Expression`, `IEX`, and base64 blobs.

```spl
index=main sourcetype="WinEventLog:PowerShell" OR event.code=4104
| rex field=script_block_text "(?i)(Invoke-WebRequest|Invoke-Expression|IEX|DownloadString)"
| table _time, host, user, script_block_text
```

### Zeek network logs

| Log | Purpose |
|---|---|
| `conn.log` | All TCP and UDP connections |
| `dns.log` | DNS queries and answers |
| `http.log` | HTTP requests, URIs, and user agents |
| `ssl.log` | TLS handshakes and SNI |
| `files.log` | File transfers |
| `x509.log` | Certificates |
| `notice.log` | Anomalies and alerts |

Sysmon may not record browser traffic, so Zeek fills that gap on the network side.

### Investigation workflow across sources

```text
1. Endpoint log: spot a suspicious process or file
2. Network log (Zeek): see what it communicated with
3. Authentication log: see who logged on
4. Application or cloud log: see resource access
5. Correlate timelines and reconstruct the attack
```

The Stuxbot investigation followed this pattern. Sysmon Event 15 showed the download. Sysmon Event 1 showed the execution chain. Zeek DNS and connection logs showed the file hosting and C2 traffic. Security events 4624 and 4625 showed the credential spraying. No single source told the full story.

### SPL patterns by log source

```spl
# Windows Security logons
index=main sourcetype="WinEventLog:Security" EventCode=4624
| stats count by Account_Name, ComputerName
| sort - count

# Sysmon process relationships
index=main sourcetype="WinEventLog:Sysmon" EventCode=1
| stats count by Image, ParentImage
| sort - count

# Zeek connection volume
index=main sourcetype="zeek:conn"
| stats sum(orig_bytes) as out_bytes, sum(resp_bytes) as in_bytes by source.ip, destination.ip
| sort - out_bytes

# Zeek DNS queries
index=main sourcetype="zeek:dns"
| stats count by dns.question.name
| sort - count
```

### Data discovery workflow

1. **What data exists?** `| eventcount summarize=false index=* | table index`, then `| metadata type=sourcetypes index=* | table sourcetype`
2. **What does a sourcetype contain?** `sourcetype="<name>" | head 5 | table _raw`
3. **Which fields exist?** `sourcetype="<name>" | fieldsummary`
4. **Which event codes appear?** `sourcetype="<name>" | stats count by EventCode | sort - count`
5. **What is normal?** `sourcetype="<name>" | top limit=20 <field>` and `| rare limit=20 <field>`

### Useful cross-source pivots

- Process to network: Sysmon 1, 3, and Zeek conn
- Process to file: Sysmon 1 and 11
- Process to registry: Sysmon 1 and 13
- User to logon to host: Security 4624 and 4625, with LogonType 3
- DNS to IP to connection: Zeek dns and conn

### Common pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Wrong time range | Empty results | Set All time |
| Wrong index | Missing events | Use `index=*` to discover |
| Wrong field name | Empty results | Check with `fieldsummary` |
| Case sensitivity | Missed matches | Use `lower()` or `upper()` |
| Leading wildcards | Slow queries | Target specific fields |
| Missing macro | Broken app panel | Edit the macro |
| Sourcetype mismatch | Half the data missing | Check both raw and parsed forms |
| No Zeek data | No network view | Confirm the `zeek*` index exists |

---

## Appendix A: Master Cheat Sheet

### Investigation pivots

1. Start from an IOC: hash, filename, URL, or IP.
2. Follow the process tree from parent to child.
3. Follow the network through Zeek DNS, conn, and HTTP.
4. Pivot on the hash to find it on other hosts.
5. Pivot on the user through LogonType 3, 4648, 4624, and 4625.
6. Hunt TTPs, not just IOCs.

### Detection patterns

| Goal | Pattern |
|---|---|
| Known binary | `Image="*\\tool.exe"` |
| Known domain | `QueryName="*domain*"` |
| Suspicious location | `regex Image="C:\\\\Users\\\\.*\\\\Downloads.*"` |
| New executables | `EventCode=11 (TargetFilename="*.exe" OR "*.dll") TargetFilename!="*\\windows\\*"` |
| Typosquat | `Image="*partial*" NOT Image="*legitimate*"` |
| Port abuse | `NOT (DestinationPort=80 OR DestinationPort=443)` |
| Downloads | `TargetFilename="*Zone.Identifier"` |
| Rolling anomaly | `bin _time span=1h \| stats \| streamstats avg() stdev() \| eval isOutlier` |
| Repeated process | `transaction ComputerName, Image \| where mvcount(ProcessGuid) > 1` |

---

## Appendix B: Windows and Sysmon Event Reference

### Sysmon event codes

| Code | Name | Hunting value |
|---|---|---|
| 1 | Process creation | Parent-child anomalies |
| 2 | File creation time changed | Timestomping |
| 3 | Network connection | Beaconing and C2 |
| 4 | Sysmon state changed | Tampering |
| 5 | Process terminated | Sacrificial processes |
| 6 | Driver loaded | Bring-your-own-driver attacks |
| 7 | Image loaded | DLL hijacking |
| 8 | CreateRemoteThread | Injection |
| 10 | Process access | LSASS dumping |
| 11 | File created | Dropped tools |
| 12, 13, 14 | Registry | Persistence |
| 15 | FileCreateStreamHash | Mark-of-the-Web |
| 16 | Sysmon config changed | Tampering |
| 17, 18 | Pipe | PsExec |
| 22 | DNS event | Beaconing |
| 23 | File delete | Cleanup |
| 25 | Process tampering | Hollowing |

### Windows Security event codes

| Code | Meaning |
|---|---|
| 4624 | Successful logon (LogonType 3 is network) |
| 4625 | Failed logon |
| 4648 | Explicit credential use |
| 4662 | Directory service object access |
| 4688 | Process creation |
| 4768 | Kerberos TGT request |
| 4769 | Kerberos service ticket request |
| 7045 | Service installed |

### DCSync GUIDs

| GUID | Right |
|---|---|
| `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2` | DS-Replication-Get-Changes-All |
| `19195a5b-6da0-11d0-afd3-00c04fd930c9` | DS-Replication-Get-Changes |

---

## Appendix C: Splunk Field Names

**Universal:** `_time`, `_indextime`, `_raw`, `host`, `source`, `sourcetype`, `index`

**Windows and Sysmon:**
- Process: `Image`, `ParentImage`, `CommandLine`, `ParentCommandLine`, `ProcessId`, `ParentProcessId`, `ProcessGuid`
- User: `User`, `Account_Name`, `TargetUserName`, `winlog.user.name`
- Host: `ComputerName`, `Computer`, `host.name`
- Network: `SourceIp`, `DestinationIp`, `SourcePort`, `DestinationPort`
- File: `TargetFilename`, `ImageLoaded`, `file.name`, `file.path`
- Registry: `TargetObject`, `registry.path`, `registry.value`, `Details`
- Pipe: `PipeName`, `EventType`
- DNS: `QueryName`, `dns.question.name`, `dns.answers.data`
- Hash: `process.hash.sha256`, `Hashes`

**Zeek:** `source.ip`, `destination.ip`, `source.port`, `destination.port`, `dns.question.name`, `dns.answers.data`, `http.host`, `http.uri`, `http.user_agent`, `ssl.server_name`

**PowerShell:** `powershell.file.script_block_text`, `winlog.event_data.ScriptBlockText`

---

## Appendix D: Curl Templates

### Splunk REST API

```bash
curl -k -u <USER>:<PASSWORD> \
  "https://<SPLUNK_IP>:8089/services/search/jobs/export" \
  -d search="search index=* EventCode=4768 | stats count by Account_Name | sort - count | head 1" \
  -d output_mode=json
```

The free license disables remote REST management, so the Web UI may be required.

### Kibana console proxy

```bash
curl -s -XPOST 'http://<KIBANA_IP>:5601/api/console/proxy?path=windows%2A%2F_search&method=POST' \
  -H 'kbn-xsrf: true' \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 200,
    "sort": [{"@timestamp": "asc"}],
    "query": {
      "bool": {
        "filter": [
          { "term": { "event.code": "11" } },
          { "wildcard": { "file.path": "C:\\\\Users\\\\Public*" } }
        ]
      }
    },
    "_source": ["@timestamp","host.name","user.name","event.code","file.name","file.path","process.name"]
  }' | jq '.hits.hits[]._source'
```

---

## Conclusion

| Skill | Outcome |
|---|---|
| Conceptual | Threat hunting, TTPs, the kill chain, the Diamond Model |
| Process | The eight-phase hunting cycle |
| Splunk basics | SPL, stats, eval, rex, lookup, transaction |
| Splunk apps | Installation, macro configuration, panel debugging |
| Real-world detection | Finding intrusions across hundreds of thousands of events |
| TTP detection | Searching for known adversary behavior |
| Analytics detection | Rolling statistics and thresholds |
| Log source knowledge | Knowing which source tells which part of the story |
| Investigation | Correlating sources to reconstruct attack chains |

The working principle: assume nothing, verify everything, pivot aggressively, correlate across sources, document thoroughly, and keep learning.
