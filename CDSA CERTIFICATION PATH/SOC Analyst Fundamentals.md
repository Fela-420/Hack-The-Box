# SOC Analyst Fundamentals: Study Notes

A consolidated reference covering SIEM concepts, the Elastic Stack, SOC operations, MITRE ATT&CK, SIEM use case development, Kibana visualizations, the triage process, and practical Elasticsearch querying with curl. Each Kibana example includes its Elasticsearch DSL equivalent.

---

## Table of Contents

1. [SIEM Fundamentals](#1-siem-fundamentals)
2. [The Elastic Stack](#2-the-elastic-stack)
3. [SOC Fundamentals](#3-soc-fundamentals)
4. [MITRE ATT&CK](#4-mitre-attck)
5. [SIEM Use Case Development](#5-siem-use-case-development)
6. [SIEM Visualizations](#6-siem-visualizations)
7. [The Triage Process](#7-the-triage-process)
8. [Querying Elasticsearch with curl](#8-querying-elasticsearch-with-curl)
9. [Key Field Reference](#9-key-field-reference)
10. [Skills Assessment: Worked Answers](#10-skills-assessment-worked-answers)
11. [Troubleshooting Cheat Sheet](#11-troubleshooting-cheat-sheet)
12. [Common SOC Queries](#12-common-soc-queries)
13. [From Alert to Decision](#13-from-alert-to-decision)

---

## 1. SIEM Fundamentals

**Security Information and Event Management (SIEM)** software collects logs and events from many sources, normalizes and correlates them, and alerts security teams to potential threats.

The term combines SIM (Security Information Management) and SEM (Security Event Management). Gartner introduced the category in 2005.

| Use case | Purpose |
|---|---|
| Log aggregation and normalization | Central visibility across IT |
| Threat alerting | Detect anomalies in near real time |
| Contextualization and response | Filter noise and prioritize real threats |
| Compliance | Support PCI DSS, HIPAA, GDPR, and ISO requirements |

Data flows through a SIEM in this order:

```text
Ingest logs -> Normalize and aggregate -> Correlate and detect -> Alert, dashboard, or incident
```

A SIEM complements IDS and IPS. It does not replace them. SIEMs also generate many alerts, so tuning is essential to prevent false-positive fatigue.

---

## 2. The Elastic Stack

| Component | Role |
|---|---|
| Elasticsearch | Distributed JSON search and storage engine with a REST API |
| Logstash | Collects, transforms, and transports logs |
| Kibana | Visualization and query interface (KQL) |
| Beats | Lightweight shippers such as Filebeat and Winlogbeat |

Typical data flow:

```text
Beats -> Logstash -> Elasticsearch -> Kibana
Beats -> Elasticsearch -> Kibana          (without Logstash)
```

SOC analysts work primarily in Kibana, querying the indices that Elasticsearch stores.

### Kibana Query Language (KQL)

| Syntax | Meaning |
|---|---|
| `field:value` | Match a field |
| `"free text"` | Search across fields |
| `A AND B` | Both conditions |
| `A OR B` | Either condition |
| `NOT A` | Negation |
| `field: value*` | Prefix wildcard |
| `field: *value*` | Contains wildcard |
| `field >= 5` | Comparison (`>`, `>=`, `<`, `<=`) |

### Elastic Common Schema (ECS)

ECS is the shared field vocabulary across the stack. Prefer ECS fields for consistency, correlation, and future compatibility.

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
| Tier 2 Analyst | Deeper analysis, pattern recognition, tuning |
| Tier 3 Analyst | Complex incidents and threat hunting |
| Detection Engineer | Build and tune detection rules |
| Incident Responder | Active response and forensics |
| Threat Intelligence Analyst | Research and share threat intelligence |
| Security Engineer | Build and maintain tooling |
| Compliance Specialist | Standards and audits |
| Awareness Coordinator | Training programs |

SOC maturity has progressed through several stages:

| Stage | Focus |
|---|---|
| SOC 1.0 | Network and perimeter, uncorrelated alerts |
| SOC 2.0 | Intelligence-driven, application-layer visibility, threat intelligence |
| Cognitive SOC | Learning systems, business-aware detection |

---

## 4. MITRE ATT&CK

MITRE ATT&CK (Adversarial Tactics, Techniques, and Common Knowledge) documents attacker behavior.

- **Tactics** are the goals (the "why").
- **Techniques** are the methods (the "how").
- **Procedures** are specific implementations (the "how exactly").

Matrices cover Enterprise (Windows, macOS, Linux, cloud, and network), Mobile, and ICS.

Enterprise tactics, in order of a typical intrusion: Reconnaissance, Resource Development, Initial Access, Execution, Persistence, Privilege Escalation, Defense Evasion, Credential Access, Discovery, Lateral Movement, Collection, Command and Control, Exfiltration, and Impact.

ATT&CK supports detection and response, gap analysis, SOC maturity assessment, threat intelligence enrichment, behavioral analytics, red team planning, and training.

---

## 5. SIEM Use Case Development

A use case defines what to detect and how to alert on it. For example, "ten failed logons from one source within four minutes indicates brute force."

### Lifecycle

Requirements, data points, log validation, design, implementation, documentation, onboarding, testing, fine tuning.

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

Build steps:
- Time range: last 15 years
- Filter: `event.code : 4625`
- Index pattern: `windows*`
- Visualization type: Table
- Rows: `user.name.keyword` (username) and `host.hostname.keyword` (logged by)
- Metric: Count

Refinements:
- Rename columns clearly.
- Add `winlog.logon.type.keyword` for logon type.
- Sort by count, descending.
- Exclude specific usernames with filters.
- Exclude computer accounts: `NOT user.name: *$ AND winlog.channel.keyword: Security`

Key point: use `.keyword` for aggregations and base fields for KQL text searches.

### Example 2: Failed logons for disabled users

**Purpose:** detect attempts against disabled accounts, which can indicate leaked credentials being tested.

Filters:
- `event.code : 4625`
- `winlog.event_data.SubStatus : 0xC0000072` (account disabled)

Rows: `user.name.keyword`, `host.hostname.keyword`. Metric: Count.

### Example 3: Successful RDP by service accounts

**Purpose:** service accounts should not use interactive RDP. This is a high-privilege risk.

Filters:
- `event.code : 4624`
- `winlog.logon.type : RemoteInteractive`

Service account scope in KQL:

```kql
user.name: svc-*
```

Rows: `user.name.keyword`, `host.hostname.keyword`, `related.ip.keyword`. Metric: Count.

### Example 4: Users added to or removed from Administrators

**Purpose:** detect privilege escalation.

Filters:
- `event.code : 4732 OR event.code : 4733`
- `group.name : "administrators"`

Rows:
- `user.name.keyword` (actor)
- `winlog.event_data.MemberSid.keyword` (member SID)
- `group.name.keyword` (target group)
- `event.action.keyword` (added or removed)
- `host.name.keyword` (machine)

Metric: Count. Use an absolute time range (for example, from a fixed start date to now). Relative ranges can produce weekly buckets when you aggregate by day.

### Visualization best practices

- Set an absolute time range when aggregating by day.
- Use `.keyword` for groupings, sorting, and aggregations.
- Use base fields for KQL text searches.
- Save both the visualization and the dashboard.
- Exclude known noise with KQL, such as `*$` for computer accounts.

---

## 7. The Triage Process

Triage means evaluating and prioritizing alerts to decide whether to ignore them as false positives, investigate further, or escalate.

Escalate when you see:
- A compromised critical system
- An ongoing attack
- A sophisticated technique
- Widespread impact
- A possible insider threat

### Ideal 13-step process

1. **Initial alert review:** metadata, IP addresses, affected systems, triggering rule.
2. **Classification:** severity, impact, and urgency.
3. **Correlation:** related alerts, IOCs, SIEM queries, threat intelligence.
4. **Enrichment:** packet captures, memory, sandbox results, threat intelligence, reconnaissance.
5. **Risk assessment:** asset criticality, data sensitivity, compliance exposure, likelihood of lateral movement.
6. **Contextual analysis:** assets, existing security controls, compliance impact.
7. **Incident response planning:** document, assign roles, coordinate teams.
8. **Consultation with IT Operations:** changes, maintenance windows, known false positives.
9. **Response execution:** close if benign, or start incident response if suspicious.
10. **Escalation:** internal procedure first, then external parties (law enforcement, CERT) if needed.
11. **Continuous monitoring:** updates and communication.
12. **De-escalation:** once contained, closed, and documented.
13. **Process review:** update procedures from lessons learned.

### Decision framework

| Situation | Action |
|---|---|
| Matches baseline or expected pattern | Nothing suspicious |
| Unusual, and the operational context is missing | Consult IT Operations |
| Clear policy violation or malicious indicator | Escalate to Tier 2 or Tier 3 |

---

## 8. Querying Elasticsearch with curl

Confirm Elasticsearch is reachable:

```bash
curl -s http://<TARGET>:9200/
```

### Basic search

```bash
curl -s -X POST "http://<TARGET>:9200/<INDEX>/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": { "match_all": {} },
    "size": 5
  }'
```

### Count only

```bash
curl -s -X POST "http://<TARGET>:9200/<INDEX>/_count?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"match_all":{}}}'
```

### KQL to DSL translation

| KQL | Elasticsearch DSL |
|---|---|
| `event.code: 4625` | `{"term": {"event.code": "4625"}}` |
| `user.name: admin*` | `{"wildcard": {"user.name.keyword": "admin*"}}` |
| `user.name: *admin*` | `{"wildcard": {"user.name.keyword": "*admin*"}}` |
| `A AND B` | `{"bool": {"filter": [A, B]}}` |
| `A OR B` | `{"bool": {"should": [A, B], "minimum_should_match": 1}}` |
| `NOT A` | `{"bool": {"must_not": [A]}}` |
| `field >= X` | `{"range": {"field": {"gte": X}}}` |

### Terms aggregation: failed logons by user

```bash
curl -s -X POST "http://<TARGET>:9200/windows/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "query": {
      "bool": {
        "filter": [
          { "term": { "event.code": "4625" } },
          { "term": { "winlog.channel.keyword": "Security" } }
        ],
        "must_not": [
          { "wildcard": { "user.name.keyword": "*$" } }
        ]
      }
    },
    "aggs": {
      "users": {
        "terms": { "field": "user.name.keyword", "size": 100 }
      }
    }
  }'
```

### Date histogram

```bash
curl -s -X POST "http://<TARGET>:9200/windows/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "query": { "match_all": {} },
    "aggs": {
      "per_day": {
        "date_histogram": {
          "field": "@timestamp",
          "calendar_interval": "day",
          "format": "yyyy-MM-dd"
        }
      }
    }
  }'
```

### Nested aggregations: user, host, then source IP

```bash
curl -s -X POST "http://<TARGET>:9200/windows/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "query": {
      "bool": {
        "filter": [
          { "term": { "event.code": "4624" } },
          { "term": { "winlog.logon.type.keyword": "RemoteInteractive" } },
          { "wildcard": { "user.name.keyword": "svc-*" } }
        ]
      }
    },
    "aggs": {
      "by_user": {
        "terms": { "field": "user.name.keyword", "size": 100 },
        "aggs": {
          "by_host": {
            "terms": { "field": "host.hostname.keyword", "size": 20 },
            "aggs": {
              "by_ip": {
                "terms": { "field": "related.ip.keyword", "size": 20 }
              }
            }
          }
        }
      }
    }
  }'
```

---

## 9. Key Field Reference

### Windows Security event codes

| Code | Meaning |
|---|---|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4732 | Member added to a security-enabled local group |
| 4733 | Member removed from a security-enabled local group |

### ECS and Winlogbeat fields

| Field | Meaning |
|---|---|
| `event.code` | Windows event ID |
| `event.action` | Action, such as `added-member-to-group` |
| `event.outcome` | `success` or `failure` |
| `user.name` | Actor or target username |
| `user.name.keyword` | Same, for aggregations |
| `host.name`, `host.hostname` | Machine name |
| `host.hostname.keyword` | Same, for aggregations |
| `source.ip` | Source IP address |
| `related.ip` | Array of related IP addresses |
| `related.ip.keyword` | Same, for aggregations |
| `group.name`, `group.name.keyword` | Group name |
| `winlog.logon.type` | ECS logon type: Interactive, Network, RemoteInteractive |
| `winlog.event_data.LogonType` | Numeric Windows logon type: 2, 3, 10 |
| `winlog.event_data.SubStatus` | Detailed failure reason, such as `0xC0000072` for a disabled account |
| `winlog.event_data.MemberSid` | SID of the changed group member |
| `winlog.channel.keyword` | Channel, such as `Security` |

### Logon types

| Code | ECS name | Meaning |
|---|---|---|
| 2 | Interactive | Local console |
| 3 | Network | Network share access |
| 10 | RemoteInteractive | RDP |

### SubStatus codes

| Code | Meaning |
|---|---|
| 0xC000006A | Bad password |
| 0xC0000064 | User does not exist |
| 0xC0000072 | Account disabled |
| 0xC0000234 | Account locked out |

---

## 10. Skills Assessment: Worked Answers

### Environment context from senior analyst notes

- The environment is cloud-hosted, and the DMZ is closed.
- IT Operations has four people, all with high privileges.
- IT Operations often uses the default admin account despite policy.
- Endpoints are hardened to CIS baselines, with limited application whitelisting.
- Administrative activity requires a privileged access workstation (PAW).
- Linux servers are legacy and mostly idle.
- The root account cannot log in remotely, and sudo is required.
- Service accounts carry a `-svc` suffix under a strict naming convention.

### Answers

| # | Visualization | Answer | Reasoning |
|---|---|---|---|
| 1 | Failed logons, all users | Consult IT Operations | `sql-svc1` fails logons, which suggests a service misconfiguration |
| 2 | Failed logons, disabled user | Consult IT Operations | `anni` is disabled. Confirm whether that status is expected |
| 3 | Failed logons, admin users | Nothing suspicious | All events are on the PAW and domain controller, which is expected admin behavior |
| 4 | RDP by service account | Escalate to Tier 2 or 3 | `svc-sql1` does not match the `-svc` convention, so it may be an unauthorized account |
| 5 | User added or removed from a local group | Consult IT Operations | Member appears only as a SID. Confirm whether IT Operations made the change |
| 6 | Admin logon not from PAW | Consult IT Operations | Admins are known to deviate from policy. Confirm before escalating |
| 7 | SSH logins | Escalate to Tier 2 or 3 | Root SSH violates policy, since root is blocked for remote logon |

### Reasoning patterns

- Naming convention violations are treated as suspicious and escalated.
- Known operational noise, such as PAW or domain controller activity and IT Operations habits, goes to consultation.
- Unknown operational context means consulting first.
- A clear policy violation, such as root SSH, means escalating.

---

## 11. Troubleshooting Cheat Sheet

### Zero results: diagnostic steps

Check the total count with no filters:

```bash
curl -s -X POST "http://<TARGET>:9200/windows/_count?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"match_all":{}}}'
```

Inspect a raw document to see the actual field names and values:

```bash
curl -s -X POST "http://<TARGET>:9200/windows/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{"size": 3, "query": {"term": {"event.code": "4625"}}}'
```

Check the mapping if a `.keyword` subfield does not exist:

```bash
curl -s "http://<TARGET>:9200/windows/_mapping/field/user.name?pretty"
```

### Common pitfalls

| Pitfall | Fix |
|---|---|
| `term` on a `.keyword` field returns 0 | Keyword matching is case-sensitive. Match the exact case, or use `match` |
| `wildcard` on a text field fails | Use the `.keyword` subfield |
| Aggregation fails with "fielddata disabled" | Aggregate on the `.keyword` subfield |
| Date histogram shows weekly buckets | Set an absolute time range in Kibana |
| `user.name: admin*` returns 8 in KQL but 4 in DSL | KQL searches the analyzed text field (case-insensitive). `.keyword` is case-sensitive |
| `related.ip` is empty | Try `source.ip` or `winlog.event_data.IpAddress` |

### Query types

| Query | Behavior |
|---|---|
| `term` | Exact value on a keyword field, case-sensitive |
| `match` | Analyzed, case-insensitive, tokenized |
| `wildcard` | Pattern match, typically on a keyword field |
| `match_phrase` | Analyzed phrase, order-sensitive |

---

## 12. Common SOC Queries

Failed logons, all users:

```json
{"query":{"bool":{"filter":[
  {"term":{"event.code":"4625"}},
  {"term":{"winlog.channel.keyword":"Security"}}
]}}}
```

Failed logons, disabled accounts:

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

Users added to or removed from Administrators:

```json
{"query":{"bool":{"filter":[
  {"terms":{"event.code":["4732","4733"]}},
  {"term":{"group.name.keyword":"Administrators"}}
]}}}
```

Accounts with "admin" anywhere in the name:

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

## 13. From Alert to Decision

```text
1. REVIEW   What does the alert say?
2. ENRICH   What is the context?
3. ASSESS   Is this expected behavior?
4. DECIDE
      Expected or baseline         -> nothing suspicious
      Operational noise            -> consult IT Operations
      Policy violation or IOC      -> escalate to Tier 2 or Tier 3
5. DOCUMENT everything, every time
```

### Golden rules

- Establish a baseline. Know what normal looks like.
- Naming conventions matter. Violations are red flags.
- Separate policy from practice. Known deviations call for consultation, not automatic escalation.
- Weigh context over count. Eight hits can be benign, and one can be critical.
- Use `.keyword` for aggregations and base fields for KQL.
- Document everything in the SOP, the IRP, the ticket, and your notes.
