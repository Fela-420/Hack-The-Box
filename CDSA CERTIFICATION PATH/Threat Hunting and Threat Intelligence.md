# Threat Hunting and Threat Intelligence: Reference Notes

A reference covering threat hunting fundamentals, the hunting process, core glossary terms, cyber threat intelligence, and two practical Stuxbot hunts performed in the Elastic Stack.

---

## Table of Contents

1. [Threat Hunting Fundamentals](#1-threat-hunting-fundamentals)
2. [The Threat Hunting Process](#2-the-threat-hunting-process)
3. [Threat Hunting Glossary](#3-threat-hunting-glossary)
4. [Threat Intelligence Fundamentals](#4-threat-intelligence-fundamentals)
5. [Practical Hunt: Stuxbot (Round 1)](#5-practical-hunt-stuxbot-round-1)
6. [Practical Hunt: Stuxbot (Round 2, Skills Assessment)](#6-practical-hunt-stuxbot-round-2-skills-assessment)
7. [Quick Reference](#7-quick-reference)

---

## 1. Threat Hunting Fundamentals

### 1.1 What threat hunting is

Threat hunting is an active, human-led, hypothesis-driven practice that searches network and endpoint data for stealthy, advanced threats that evade existing security controls.

- It is not a passive, alert-driven activity.
- It is proactive and offensive in posture.
- Its goal is to reduce dwell time, the period between breach and detection, which is often weeks to months.

### 1.2 Why it matters

- Traditional defense tools miss stealthy adversaries.
- Median dwell time is often three weeks or more.
- Hunting aims to catch attackers early in the kill chain, before they entrench.

### 1.3 Key facets

- A proactive, offensive strategy built on hypotheses, TTPs, and intelligence
- Reactive response to a verified incident, using artifacts and intelligence
- Understanding of the threat landscape and the kill chain
- Cognitive empathy with the attacker (an adversarial mindset)
- Deep knowledge of the organization's environment, assets, and normal activity
- Use of high-fidelity data, tactical analytics, and advanced hunting tools

### 1.4 Hunting and incident handling

| Incident phase | Hunter's role |
|---|---|
| Preparation | Define rules of engagement, protocols, and intervention rules |
| Detection and analysis | Validate IOCs, expand the investigation, find missed artifacts |
| Containment, eradication, recovery | Varies. May or may not be a hunter duty |
| Post-incident activity | Recommend security posture improvements |

Whether hunting and incident response are integrated or separate is an organizational decision.

### 1.5 Team structure

| Role | Responsibility |
|---|---|
| Threat hunter | Proactively searches for IOCs and threats |
| Threat intelligence analyst | Gathers and analyzes threat data and trends |
| Incident responders | Manage confirmed incidents end to end |
| Forensics experts | Malware analysis, reverse engineering, DFIR |
| Data analysts and scientists | Find patterns through statistics, machine learning, and data mining |
| Security engineers and architects | Design secure infrastructure and hunting-enabling tools |
| Network security analyst | Understands traffic patterns and anomalies |
| SOC manager | Coordinates the team and communication |

### 1.6 When to hunt

1. New adversary or vulnerability information appears.
2. New IOCs are linked to a known adversary.
3. Several network anomalies occur together or close in time.
4. During an active incident response.
5. On a regular schedule, as a proactive, continuous activity.

### 1.7 Risk assessment and hunting

Risk assessment tells you where to hunt. It helps you:

- Prioritize crown-jewel assets and other critical systems
- Understand the threat landscape and the TTPs likely to be used
- Identify vulnerabilities that attackers could exploit
- Direct how threat intelligence is applied
- Refine incident response plans and security controls

Common inputs include the SIEM, vulnerability scanners, penetration testing results, and threat intelligence platforms.

---

## 2. The Threat Hunting Process

### 2.1 The eight-phase cycle

Set the stage, formulate hypotheses, design the hunt, gather and examine data, evaluate findings, mitigate threats, document, and improve.

| Phase | What happens |
|---|---|
| 1. Setting the stage | Plan, study the threat landscape, enable logging, configure SIEM, EDR, and IDS |
| 2. Formulating hypotheses | Make specific, testable predictions from intelligence, alerts, or experience |
| 3. Designing the hunt | Choose data sources, methods, tools, IOCs, and queries |
| 4. Gathering and examining data | Collect and analyze logs, traffic, and endpoint data. This phase is iterative |
| 5. Evaluating findings | Confirm or refute the hypothesis. Establish behavior, scope, and impact |
| 6. Mitigating threats | Isolate, remove, patch, block, or reconfigure |
| 7. After the hunt | Document, share, update intelligence, detection rules, and IR playbooks |
| 8. Continuous learning | Review effectiveness, refine methods, train, and collaborate |

### 2.2 Example: hunting Emotet

| Phase | Application |
|---|---|
| Stage | Research Emotet TTPs, infection vectors, and likely targets |
| Hypothesis | Emotet is using compromised email accounts to send phishing emails with malicious Word documents that contain macros |
| Design | Email, network, and endpoint logs, plus sandbox results. Search for suspicious subject lines, attachments, C2 addresses, and hashes |
| Gather and examine | Email header analysis, network traffic patterns, sandboxed samples |
| Evaluate | Confirm the phishing activity and the C2 communication |
| Mitigate | Isolate hosts, remove the malware, clean compromised mailboxes, block C2 domains |
| After the hunt | Update Emotet IOCs, detection rules, and IR playbooks |
| Continuous learning | Move toward behavior-based and machine learning detection to keep pace with changing TTPs |

---

## 3. Threat Hunting Glossary

### 3.1 Core concepts

| Term | Meaning |
|---|---|
| Adversary | An unauthorized party seeking to infiltrate for money, espionage, intellectual property, or disruption |
| Adversary types | Cybercriminals, insiders, hacktivists, state-sponsored actors |
| Threat | Intent, capability, and opportunity together |
| Intent | Why the adversary wants to target you |
| Capability | Tools, skills, money, and resources |
| Opportunity | Favorable conditions such as leaked credentials, vulnerabilities, or exposure |
| Campaign | A group of incidents with similar TTPs and goals |

### 3.2 APT (advanced persistent threat)

- Organized groups, often nation-states
- Well resourced in money, people, and time
- Target high-value sectors such as government, healthcare, and defense
- "Advanced" does not always mean high-tech. It can mean sophisticated planning.
- "Persistent" means a long-term, determined effort.

### 3.3 TTPs (tactics, techniques, procedures)

The adversary's operational signature.

| Part | Meaning | Question answered |
|---|---|---|
| Tactics | High-level strategic goals | Why |
| Techniques | General methods used to reach those goals | How |
| Procedures | Step-by-step implementation | Exactly how |

Understanding TTPs shows how attackers get in, move laterally, and reach their objectives. It also supports building IOCs and detecting future attacks.

### 3.4 Indicators

Data plus context is an indicator. Raw data without context has little value.

IOCs (indicators of compromise) are digital traces of an intrusion, such as malicious file hashes, suspicious IP addresses, URLs and domains, and malicious executable or script names. Tracking them continually speeds up detection and response.

### 3.5 Pyramid of Pain

The pyramid ranks indicators by how hard they are for the adversary to change and how much that change hurts the adversary.

| Level | Indicator | Difficulty for the adversary to change |
|---|---|---|
| 1 (bottom) | Hash values | Trivial |
| 2 | IP addresses | Easy |
| 3 | Domain names | Simple |
| 4 | Network and host artifacts | Annoying |
| 5 | Tools | Challenging |
| 6 (top) | TTPs | Tough |

The higher you detect, the more pain you cause. Attackers must change significant behavior and infrastructure.

- **Hashes:** a tiny change to a file produces a new hash.
- **IP addresses:** easily masked with VPNs, proxies, or Tor.
- **Domains:** attackers use domain generation algorithms and dynamic DNS.
- **Network artifacts:** logs, packet captures, NetFlow, DNS queries.
- **Host artifacts:** registry keys, files, processes, DLLs, memory.
- **Tools:** malware, exploits, scripts, C2 frameworks.
- **TTPs:** the hardest to change and the most valuable to defenders.

### 3.6 Diamond Model of intrusion analysis

The model describes four connected components of an intrusion.

| Component | Meaning |
|---|---|
| Adversary | The attacker: an individual, group, or organization |
| Capability | Tools, TTPs, malware, exploits |
| Infrastructure | Servers, domains, IP addresses, botnets, C2 |
| Victim | The target: an individual, organization, or system |

The four components share bidirectional relationships. The adversary uses a capability, delivered through infrastructure, against a victim.

The Diamond Model and the Cyber Kill Chain complement each other. The kill chain describes stages of an attack. The Diamond Model describes components and their relationships.

Example: a cybercriminal group (adversary) targets a bank (victim). It uses spear-phishing (capability) sent from a botnet (infrastructure) to deliver a banking Trojan, which steals data.

---

## 4. Threat Intelligence Fundamentals

### 4.1 What cyber threat intelligence (CTI) is

CTI is insight that moves defense from reactive to proactive. It gives the SOC knowledge about adversaries before attacks unfold.

### 4.2 The four pillars of good CTI

| Pillar | Meaning | Test |
|---|---|---|
| Relevance | Applies to your organization or its partners | Is the affected software actually used here? |
| Timeliness | Fresh intelligence is valuable, and old intelligence decays | Are the IOCs still active? |
| Actionability | Gives defenders a clear next step | Can we act on it now? |
| Accuracy | Verified. Uncertain findings are labeled with a confidence level | Has it been cross-checked? |

Unactionable intelligence produces endless analysis with no output, sometimes called the "self-licking ice cream cone."

### 4.3 Threat intelligence and threat hunting

| | Threat intelligence | Threat hunting |
|---|---|---|
| Nature | Predictive | Reactive and proactive |
| Goal | Anticipate adversary moves | Determine whether the adversary is or was present |
| Answers | Where, when, how, and why | Are they here now? Did we miss them? |
| Trigger | Continuous analysis | An event or incident |

The two reinforce each other. Adversary profiles inform hunts, and hunt findings refine intelligence.

### 4.4 Three categories of CTI

| Type | Audience | Focus | Answers | Example |
|---|---|---|---|---|
| Strategic | Executives | Big picture, long term, business risk | Who and why | A report on a nation-state group |
| Operational | Mid-level management | Campaigns and TTPs in detail | How and where | Analysis of a ransomware campaign |
| Tactical | SOC, IR, defenders | Immediate and technical | What to detect or block | C2 IPs, hashes, registry keys, mutexes |

The three categories overlap. The most effective CTI analysts work across all of them.

### 4.5 Processing a tactical threat intelligence report

1. **Understand the scope and narrative.** Who is involved, and does it affect us?
2. **Identify and classify IOCs.**
   - Network: IP addresses, domains, URLs, C2 patterns, user agents, DNS patterns
   - Host: hashes, registry keys, mutexes, file paths, API calls
   - Email: sender addresses, subject lines, attachment types
   - Enrich with geolocation, WHOIS data, and associated domains
3. **Map the attack lifecycle** to MITRE ATT&CK.
4. **Analyze and validate IOCs.**
   - Cross-check with services such as VirusTotal and AlienVault OTX.
   - Consider the age of each IOC.
   - Add context. For example, a shared cloud IP address may block legitimate users.
   - Assess source reliability, allowlists, and false-positive rates.
5. **Integrate IOCs into security controls.**
   - Update firewalls, EDR, IDS and IPS, and email gateways.
   - Weigh business impact. Sometimes alerting is better than blocking.
   - Follow change management.
6. **Hunt proactively.** Search for the IOCs and the broader TTPs. For example, hunt for suspicious PowerShell even without a direct IOC match.
7. **Monitor and improve continuously.**
   - Trigger incident response when a hit occurs.
   - Improve detection rules and user education.
   - Share new IOCs and TTPs with ISACs and ISAOs.

---

## 5. Practical Hunt: Stuxbot (Round 1)

### 5.1 The threat

| Attribute | Detail |
|---|---|
| Actor | Organized cybercrime group "Stuxbot" |
| Motivation | Espionage. No ransomware, blackmail, or profit-driven data theft |
| Targeting | Opportunistic: anyone, at any time |
| Initial access | Phishing that uses social media, breach data, and corporate websites |
| Platform | Microsoft Windows |
| Impact | Full takeover and domain escalation |
| Risk | Critical |

### 5.2 Attack chain

```text
Phishing email (fake invoice)
  -> OneNote file (invoice.one), hosted on file-sharing services
  -> Batch file (invoice.bat) launched from a hidden OneNote button
  -> PowerShell fetched from Pastebin, run in memory
  -> RAT executable (default.exe)
  -> Persistence (executable on disk and registry Run keys)
  -> Lateral movement (PsExec, WinRM)
  -> Credential theft and DCSync (Mimikatz)
```

The RAT is modular and can capture screens, run Mimikatz, and provide an interactive command shell. Lateral movement uses Microsoft-signed PsExec and WinRM.

### 5.3 Known IOCs

Indicators included file-sharing URLs for the OneNote lure, Pastebin URLs for PowerShell staging, three C2 endpoints on port 443, and SHA256 hashes for the malicious files. Values are referenced in the original hunt material and are not reproduced here.

### 5.4 Environment

- About 200 employees in online marketing
- Office applications, Gmail, Microsoft Edge, and TeamViewer
- Active Directory with Group Policy Objects
- SIEM: Elastic Stack with Kibana
- Log sources: Windows audit logs, Sysmon, and PowerShell logs (index `windows*`), plus Zeek network logs (index `zeek*`)

### 5.5 Hunt steps

| Step | Query focus | Finding |
|---|---|---|
| 1 | Event 15 (FileCreateStreamHash) for `invoice.one` | File downloaded through Microsoft Edge to a user's Downloads folder on WS001 at 192.168.28.130 on March 26, 2023 |
| 2 | Event 11 for `invoice.one:Zone.Identifier` | Zone identifier confirms internet origin |
| 3 | Zeek DNS for the host's IP | Gmail, then a file-sharing service, then a SmartScreen scan |
| 4 | Sysmon Event 1 for `invoice.one` | OneNote opened about six seconds later |
| 5 | Sysmon Event 1 with `ONENOTE.EXE` as parent | `cmd.exe` ran `invoice.bat` from a temp directory |
| 6 | Sysmon Event 1 with `invoice.bat` in the parent command line | PowerShell downloaded content from Pastebin |
| 7 | Process ID of that PowerShell | Password spraying, `default.exe` dropped, ngrok DNS lookup, domain controller query |
| 8 | Zeek connections to the C2 IP | 24 connections, with activity continuing the next day. The ngrok address changed |
| 9 | Process name `default.exe` | Dropped `svchost.exe`, `SharpHound.exe`, `payload.exe`, and a VBS script |
| 10 | Process name `SharpHound.exe` | Executed twice on March 27, using the "all" collection method |
| 11 | SHA256 of `SharpHound.exe` | Same hash on WS001 and the PKI server |
| 12 | Logon events 4624 and 4625, type 3, from the patient-zero IP | Two failed administrator logons, and successful logons for `svc-sql1` on the PKI and PAW systems |

### 5.6 Round 1 findings

| Finding | Detail |
|---|---|
| Patient zero | WS001, user `bob`, 192.168.28.130 |
| Initial vector | Phishing email with `invoice.one` hosted on file.io |
| Execution | OneNote, then `invoice.bat`, then PowerShell fetched from Pastebin |
| Persistence | `default.exe` |
| C2 | Through ngrok |
| Credential attack | Password spraying. Administrator failed, `svc-sql1` succeeded |
| Lateral movement | PsExec (PSEXESVC) to the PKI server |
| AD reconnaissance | SharpHound run twice |
| Masquerading | `svchost.exe` posing as a legitimate process |

### 5.7 Round 1 skill assessment answers

| Question | Answer | Basis |
|---|---|---|
| Q1: VBS file dropped by `default.exe` | `XceGuhkzaTrOy.vbs` | File event from `default.exe` |
| Q2: Mimikatz arguments | `lsadump::dcsync /domain:eagle.local /all /csv, exit` | The `process.args` array, joined after the executable with commas |
| Q3: PowerShell tool for network shares | `PowerView` | Functions such as `Invoke-ShareFinder`, `Find-DomainShare`, and `Get-NetShare` belong to PowerView (PowerSploit) |

---

## 6. Practical Hunt: Stuxbot (Round 2, Skills Assessment)

### 6.1 New TTPs

- Deploys supplementary utilities to `C:\Users\Public`
- Uses registry Run keys for persistence
- Uses PowerShell Remoting for lateral movement, including to domain controllers

### 6.2 Hunt 1: lateral tool transfer to `C:\Users\Public`

Goal: find a file in `C:\Users\Public` whose name starts with "r", and report the user name.

Query for file creation events (Sysmon Event 11) under `C:\Users\Public`:

```bash
curl -s -XPOST 'http://<KIBANA_IP>:5601/api/console/proxy?path=windows%2A%2F_search&method=POST' \
  -H 'kbn-xsrf: true' -H 'Content-Type: application/json' \
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

Relevant hit: `Rubeus.exe` created in `C:\Users\Public` on PKI.eagle.local by process `svchost.exe`, under user `svc-sql1`.

**Answer: `svc-sql1`**

### 6.3 Hunt 2: registry Run key persistence

Goal: find the first malicious Run key value, and report the value name.

Query for registry value events (Sysmon Event 13) on Run keys:

```bash
curl -s -XPOST 'http://<KIBANA_IP>:5601/api/console/proxy?path=windows%2A%2F_search&method=POST' \
  -H 'kbn-xsrf: true' -H 'Content-Type: application/json' \
  -d '{
    "size": 200,
    "sort": [{"@timestamp": "asc"}],
    "query": {
      "bool": {
        "filter": [
          { "term": { "event.code": "13" } },
          { "wildcard": { "registry.path": "*CurrentVersion\\\\Run*" } }
        ]
      }
    },
    "_source": ["@timestamp","host.name","user.name","event.code","registry.path","registry.value","registry.data.strings","process.name"]
  }' | jq '.hits.hits[]._source'
```

Ignore legitimate entries, such as Microsoft Edge auto-launch and Office or Edge restart entries.

The first malicious entry was a randomized value name under the user's Run key, created by `powershell.exe` for user `bob`. Its data pointed to `default.exe` in the user's Temp directory.

**Answer: `LgvHsviAUVTsIN`**

Later malicious values included `KFbyzDpudqtCF`, `cgCfhHDMwmN`, and `ybZishBHbRKgfe`.

### 6.4 Hunt 3: PowerShell Remoting to DC1

Goal: find PowerShell Remoting toward DC1, and report the user name.

Query for script block logging (event 4104) containing `Enter-PSSession`:

```bash
curl -s -XPOST 'http://<KIBANA_IP>:5601/api/console/proxy?path=windows%2A%2F_search&method=POST' \
  -H 'kbn-xsrf: true' -H 'Content-Type: application/json' \
  -d '{
    "size": 200,
    "sort": [{"@timestamp": "asc"}],
    "query": {
      "bool": {
        "filter": [
          { "term": { "event.code": "4104" } },
          { "match_phrase": { "powershell.file.script_block_text": "Enter-PSSession" } }
        ]
      }
    },
    "_source": ["@timestamp","host.name","winlog.user.name","user.name","event.code","powershell.file.script_block_text"]
  }' | jq '.hits.hits[]._source'
```

A wildcard that requires `DC1` to appear directly after `Enter-PSSession` in the same string misses cases where the computer name is passed as a separate parameter. Search for the cmdlet alone, then review the results.

As a cross-check, explicit-credential events (4648) from PowerShell show the same activity:

```bash
curl -s -XPOST 'http://<KIBANA_IP>:5601/api/console/proxy?path=windows%2A%2F_search&method=POST' \
  -H 'kbn-xsrf: true' -H 'Content-Type: application/json' \
  -d '{
    "size": 200,
    "sort": [{"@timestamp": "asc"}],
    "query": {
      "bool": {
        "filter": [
          { "term": { "event.code": "4648" } },
          { "match_phrase": { "process.name": "powershell.exe" } }
        ]
      }
    },
    "_source": ["@timestamp","host.name","winlog.user.name","event.code","winlog.event_data.TargetUserName","winlog.event_data.TargetServerName"]
  }' | jq '.hits.hits[]._source'
```

**Answer: `svc-sql1`**

### 6.5 Round 2 answer sheet

| Hunt | Field | Answer |
|---|---|---|
| Hunt 1 | `user.name` | `svc-sql1` (Rubeus.exe) |
| Hunt 2 | `registry.value` | `LgvHsviAUVTsIN` |
| Hunt 3 | `winlog.user.name` | `svc-sql1` |

---

## 7. Quick Reference

### 7.1 Curl-to-Kibana template

```bash
curl -s -XPOST 'http://<KIBANA_IP>:5601/api/console/proxy?path=windows%2A%2F_search&method=POST' \
  -H 'kbn-xsrf: true' \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 200,
    "sort": [{"@timestamp": "asc"}],
    "query": { },
    "_source": ["@timestamp","host.name","event.code"]
  }' | jq '.hits.hits[]._source'
```

- Replace `windows%2A%2F_search` with `zeek%2A%2F_search` for network logs.
- Add `-u <USER>:<PASS>` if Kibana requires authentication.
- Use `jq '.hits.hits[]._source'` to flatten the output.

### 7.2 Sysmon event IDs

| Event ID | Meaning |
|---|---|
| 1 | Process creation |
| 3 | Network connection |
| 11 | File created |
| 13 | Registry value set |
| 15 | FileCreateStreamHash (browser downloads) |
| 22 | DNS query |

### 7.3 Windows Security event IDs

| Event ID | Meaning |
|---|---|
| 4624 | Successful logon (LogonType 3 = network) |
| 4625 | Failed logon |
| 4648 | Logon with explicit credentials |

### 7.4 PowerShell and WinRM event IDs

| Event ID | Meaning |
|---|---|
| 4104 | PowerShell script block logging |
| 91, 168 | WinRM operational events |

### 7.5 Common field names

- `@timestamp`, `host.name`, `user.name`, `winlog.user.name`
- `event.code`, `winlog.channel`, `message`
- `process.name`, `process.args`, `process.command_line`
- `process.parent.name`, `process.parent.command_line`
- `process.hash.sha256`
- `file.name`, `file.path`
- `registry.path`, `registry.value`, `registry.data.strings`
- `source.ip`, `destination.ip`, `destination.port`
- `dns.question.name`, `dns.answers.data`
- `powershell.file.script_block_text`
- `winlog.event_data.*` (LogonType, TargetUserName, TargetServerName, ScriptBlockText, IpAddress)

### 7.6 Wildcard escaping in curl bodies

- In a single-quoted shell body, write backslashes as `\\\\` in JSON so Lucene receives `\\`, which it reads as one literal backslash.
- Lucene wildcards are case-sensitive. `r*` will not match `Rubeus.exe`.

### 7.7 Investigation pivots

1. Start from an IOC: file hash, file name, URL, or IP address.
2. Follow the process tree from parent to child (Sysmon Event 1).
3. Follow the network through Zeek DNS, connection, and HTTP logs.
4. Pivot on the hash to find it on every host.
5. Pivot on the user through logon types 3 and 4648, and events 4624 and 4625.
6. Follow TTPs, not only IOCs. Hunt for PowerShell activity even without a specific IOC match.

### 7.8 Stuxbot attack chain summary

```text
Phishing -> OneNote (invoice.one) -> batch file (invoice.bat)
  -> PowerShell (Pastebin) -> default.exe (RAT)
  -> Registry Run key persistence -> ngrok C2
  -> Password spraying -> svc-sql1 compromise
  -> Rubeus, SharpHound, PsExec, WinRM
  -> DCSync (Mimikatz) -> domain compromise
```

### 7.9 Key hosts and accounts in this lab

| Entity | Role |
|---|---|
| WS001.eagle.local (192.168.28.130) | Patient zero, user `bob` |
| PKI.eagle.local | Lateral movement target |
| DC1 | Domain controller, target of PowerShell Remoting and DCSync |
| `bob` | Initial victim account |
| `svc-sql1` | Compromised service account |
| `administrator` | Password-spray target, failed |

### 7.10 Documentation rules for any hunt

- State the hypothesis clearly.
- List every query and its data source.
- Record the IOCs found.
- Map TTPs to MITRE ATT&CK.
- Document affected hosts, users, and timestamps.
- Share findings with IR and threat intelligence, and with ISACs where applicable.
- Feed results back into detection rules and IR playbooks.

---

## Conclusion

Threat hunting is a continuous, disciplined practice that combines:

- **Technical skill:** process trees, network pivots, and log analysis
- **Intelligence:** TTPs, IOCs, and threat context
- **Creativity:** hypothesis design and adversarial thinking
- **Rigor:** verification, documentation, and sharing

Effective hunters follow the process, hunt by TTPs as well as IOCs, pivot on any artifact, document and share every finding, and improve with every cycle.
