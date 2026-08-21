# Footprinting & Enumeration – Complete Module Documentation

This document summarizes all concepts, methodologies, and practical techniques covered in the Footprinting module. It is intended as a one-stop reference for understanding how to systematically gather information about a target environment, identify attack surfaces, and extract useful data without triggering alarms.

---

## Table of Contents

1. [Footprinting Principles](#footprinting-principles)
2. [Enumeration Methodology – 6 Layers](#enumeration-methodology--6-layers)
3. [Service-Specific Footprinting](#service-specific-footprinting)
   - [DNS](#dns)
   - [FTP / TFTP](#ftp--tftp)
   - [SMB / Samba](#smb--samba)
   - [NFS](#nfs)
   - [SMTP](#smtp)
   - [IMAP / POP3](#imap--pop3)
   - [SNMP](#snmp)
   - [MySQL](#mysql)
   - [MSSQL](#mssql)
   - [Oracle TNS](#oracle-tns)
   - [IPMI](#ipmi)
   - [Linux Remote Management](#linux-remote-management-ssh-rsync-r-services)
   - [Windows Remote Management](#windows-remote-management-rdp-winrm-wmi)
4. [Cloud Resources & Staff OSINT](#cloud-resources--staff-osint)
5. [Lab Walkthrough – Easy Footprinting Lab](#lab-walkthrough--easy-footprinting-lab)
6. [Key Tools & Cheat Sheet](#key-tools--cheat-sheet)

---

## Footprinting Principles

- **There is more than meets the eye.** Always consider multiple viewpoints.
- **Distinguish between what you see and what you do not see.** Both visible and hidden information are valuable.
- **There are always ways to gain more information.** Understand the target, not just exploit it.

### Guiding Questions

| What we SEE | What we DON’T SEE |
|-------------|-------------------|
| What is it? | What is missing? |
| Why is it visible? | Why is it hidden? |
| What story does it tell? | What does its absence reveal? |
| What can we gain from it? | What does this indirectly expose? |
| How can we use it? | Apply the same logic. |

---

## Enumeration Methodology – 6 Layers

Think of enumeration as passing through six concentric walls. The goal is to find gaps (vulnerabilities) without brute-forcing blindly.

| Layer | Focus | Goal |
|-------|-------|------|
| 1. Internet Presence | Domains, subdomains, IPs, netblocks, cloud instances | Identify all external targets |
| 2. Gateway | Firewalls, IPS/IDS, EDR, VPNs, Cloudflare | Understand defenses |
| 3. Accessible Services | Service types, versions, ports, configurations | Understand the function of each service |
| 4. Processes | PID, data flow, sources, destinations | Find dependencies and data paths |
| 5. Privileges | Users, groups, permissions, restrictions | Discover overlooked rights |
| 6. OS Setup | OS version, patch level, config files, sensitive files | Assess administrative competence and internal security |

---

## Service-Specific Footprinting

### DNS

- **Ports:** 53 (TCP/UDP)
- **Key records:** A, AAAA, MX, NS, TXT, CNAME, PTR, SOA.
- **Zone transfer (AXFR):** If allowed, reveals all records.

**Commands:**

```bash
dig ns inlanefreight.htb @10.129.1.207
dig axfr inlanefreight.htb @10.129.1.207
dig any inlanefreight.htb @10.129.1.207
dig txt inlanefreight.htb @10.129.1.207
````

**Dangerous settings:**

* `allow-transfer` set to `any` or broad subnet.
* Version disclosure via `dig CH TXT version.bind @<IP>`.

---

### FTP / TFTP

* **FTP ports:** 21 (plain), 20 (data), 2121 (custom).
* **TFTP:** 69/UDP.
* **Anonymous login:** `anonymous:anonymous`.
* **Dangerous settings:** `anonymous_enable=YES`, `anon_upload_enable=YES`, `write_enable=YES`.

**Commands:**

```bash
ftp 10.129.1.207
nc -nv 10.129.1.207 21
nmap -p21 -sV -sC --script ftp-anon
```

---

### SMB / Samba

* **Ports:** 139 (NetBIOS), 445 (SMB over TCP).
* **Null session:** `smbclient -N -L //10.129.x.x`
* **Dangerous settings:** `guest ok = yes`, `browseable = yes`, `writable = yes`.

**Commands:**

```bash
smbclient -N -L //10.129.1.207
rpcclient -U "" -N 10.129.1.207
enum4linux-ng -A 10.129.1.207
```

---

### NFS

* **Ports:** 111 (RPC), 2049 (NFS).
* **Mounting:** `showmount -e <IP>`, then `mount -t nfs <IP>:/share /mnt/nfs -o nolock`.
* **Dangerous settings:** `no_root_squash`, `insecure`.

**Commands:**

```bash
nmap -p111,2049 -sV 10.129.x.x
showmount -e 10.129.x.x
sudo mount -t nfs 10.129.x.x:/var/nfs /mnt/nfs -o nolock
```

---

### SMTP

* **Ports:** 25, 465 (SMTPS), 587 (submission).
* **User enumeration:** `VRFY` and `EXPN` commands.
* **Open relay:** Test with `MAIL FROM` and `RCPT TO`.

**Commands:**

```bash
telnet 10.129.x.x 25
EHLO test
VRFY root
MAIL FROM: <test@test.com>
RCPT TO: <root>
```

---

### IMAP / POP3

* **IMAP:** 143 (plain), 993 (SSL).
* **POP3:** 110 (plain), 995 (SSL).
* **Authentication:** `LOGIN username password` (IMAP), `USER` / `PASS` (POP3).
* **Dangerous settings:** `auth_debug`, `auth_verbose`.

**Commands:**

```bash
openssl s_client -connect 10.129.x.x:993
1 LOGIN user pass
1 LIST "" *
1 SELECT INBOX
1 FETCH 1:* BODY[TEXT]
```

---

### SNMP

* **Ports:** 161/UDP (queries), 162/UDP (traps).
* **Versions:** v1, v2c (community strings in plaintext), v3 (encrypted).
* **Default community strings:** `public`, `private`.

**Commands:**

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 10.129.x.x
snmpwalk -v2c -c public 10.129.x.x 1.3.6.1.2.1.1
snmpget -v2c -c public 10.129.x.x 1.3.6.1.2.1.1.4.0
```

---

### MySQL

* **Port:** 3306.
* **Default credentials:** `root:""`, `root:root`, `admin:admin`.
* **Enumeration:** `show databases;`, `use <db>;`, `show tables;`.

**Commands:**

```bash
mysql -u root -p -h 10.129.x.x
nmap -p3306 -sV -sC --script mysql*
```

---

### MSSQL

* **Port:** 1433/TCP.
* **Default credentials:** `sa:""`, `sa:sa`.
* **Authentication:** Windows or SQL Server Authentication.

**Commands:**

```bash
nmap -p1433 --script ms-sql-info,ms-sql-empty-password 10.129.x.x
impacket-mssqlclient sa:""@10.129.x.x
```

```sql
SELECT name FROM sys.databases;
```

---

### Oracle TNS

* **Port:** 1521.
* **Default SIDs:** `XE`, `ORCL`, `PDB1`.
* **Default credentials:** `scott:tiger`, `system:manager`, `sys:change_on_install`.
* **Hash extraction:** `SELECT name, password FROM sys.user$;`

**Commands:**

```bash
nmap -p1521 --script oracle-sid-brute 10.129.x.x
sqlplus scott/tiger@10.129.x.x/XE as sysdba
./odat.py all -s 10.129.x.x
```

---

### IPMI

* **Port:** 623/UDP.
* **Default credentials:** `ADMIN:ADMIN` (Supermicro), `root:calvin` (Dell iDRAC).
* **Hash dumping (RAKP):** Metasploit `ipmi_dumphashes`.

**Commands:**

```bash
nmap -sU -p623 --script ipmi-version 10.129.x.x
```

```text
msfconsole
use auxiliary/scanner/ipmi/ipmi_dumphashes
set RHOSTS 10.129.x.x
run
```

---

### Linux Remote Management (SSH, Rsync, R-Services)

* **SSH:** 22/TCP – check weak configurations such as `PermitRootLogin` and `PasswordAuthentication`.
* **Rsync:** 873/TCP – enumerate shares with `rsync -av --list-only rsync://<IP>/`.
* **R-Services:** 512–514/TCP – `rlogin`, `rsh`, `rexec`; trust files include `/etc/hosts.equiv` and `.rhosts`.

**Commands:**

```bash
ssh -v user@10.129.x.x
rsync -av --list-only rsync://10.129.x.x/dev
rlogin 10.129.x.x -l user
```

---

### Windows Remote Management (RDP, WinRM, WMI)

* **RDP:** 3389/TCP – check NLA status.
* **WinRM:** 5985/HTTP, 5986/HTTPS – use `evil-winrm`.
* **WMI:** 135/TCP (initial) – use `wmiexec.py`.

**Commands:**

```bash
nmap -p3389 --script rdp* 10.129.x.x
evil-winrm -i 10.129.x.x -u user -p pass
wmiexec.py user:pass@10.129.x.x "hostname"
```

---

## Cloud Resources & Staff OSINT

### Cloud Resources

* Cloud storage such as S3 buckets, Azure blobs, and GCP storage can be exposed.
* Google dork example:

```text
intext:company inurl:amazonaws.com
```

* Tools:

  * GrayHatWarfare
  * domain.glass

### Staff OSINT

Potential sources include:

* LinkedIn
* Xing
* Job postings
* GitHub

Useful information may include:

* Technology stacks
* Usernames
* Email formats
* Internal tools
* Public repositories
* Cloud resources

---

## Lab Walkthrough – Easy Footprinting Lab

**Target:** `10.129.1.207`

**Domain:** `inlanefreight.htb`

**Credentials:** `ceil:qwer1234`

### Steps

1. **Nmap scan** revealed FTP on ports 21 and 2121, SSH on port 22, and DNS on port 53.

2. **DNS enumeration:**

```bash
dig axfr inlanefreight.htb @10.129.1.207
```

The zone transfer revealed subdomains including:

* `app`
* `internal`
* `mail1`
* `ns`

3. **FTP on port 2121** accepted the discovered credentials:

```text
ceil:qwer1234
```

4. Navigated to:

```text
/home/ceil/.ssh/
```

and downloaded:

```text
id_rsa
```

5. Set the private-key permissions:

```bash
chmod 600 id_rsa
```

6. Used the SSH key to access the server.

7. Found the flag at:

```text
/home/flag/flag.txt
```

**Flag content:**

```text
[submit your discovered flag here]
```

---

## Key Tools & Cheat Sheet

| Service    | Tool / Command                                            |
| ---------- | --------------------------------------------------------- |
| DNS        | `dig`, `nslookup`, `dnsrecon`, `dnsenum`                  |
| FTP        | `ftp`, `nc`, `wget`                                       |
| SMB        | `smbclient`, `rpcclient`, `enum4linux-ng`, `crackmapexec` |
| NFS        | `showmount`, `mount`, `nmap nfs*`                         |
| SMTP       | `telnet`, `smtp-user-enum`                                |
| IMAP/POP3  | `openssl`, `curl`, `hydra`                                |
| SNMP       | `snmpwalk`, `onesixtyone`, `braa`                         |
| MySQL      | `mysql`, `nmap mysql*`                                    |
| MSSQL      | `impacket-mssqlclient`, `sqsh`, `nmap ms-sql*`            |
| Oracle     | `sqlplus`, `odat.py`                                      |
| IPMI       | `nmap`, `msf ipmi_dumphashes`                             |
| SSH        | `ssh`, `ssh-audit`                                        |
| Rsync      | `rsync`                                                   |
| R-Services | `rlogin`, `rsh`, `rusers`                                 |
| RDP        | `xfreerdp`, `rdp-sec-check`                               |
| WinRM      | `evil-winrm`, `winrs`                                     |
| WMI        | `wmiexec.py`                                              |
| Cloud      | `awscli`, `az`, `gcloud`, `GrayHatWarfare`                |
| OSINT      | Google dorks, `theHarvester`, `Maltego`                   |

---

## Remember

Always obtain proper authorization before scanning or interacting with any target.

This guide is for educational and ethical security testing purposes only.

```
```
