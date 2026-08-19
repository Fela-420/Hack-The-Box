# Network Enumeration with Nmap – Complete Module Summary

## Introduction
Nmap (Network Mapper) is an open-source tool for network discovery and security auditing.
It is used to:
- Discover live hosts on a network.
- Scan open ports and their states.
- Enumerate services, versions, and operating systems.
- Evade firewalls and intrusion detection/prevention systems (IDS/IPS).
- Automate interactions using the Nmap Scripting Engine (NSE).

## Installation
Nmap comes pre-installed on Kali Linux.
To install on other systems:
sudo apt install nmap      # Debian/Ubuntu
sudo yum install nmap      # RHEL/CentOS

## Basic Syntax
nmap [scan types] [options] <target>

## 1. Host Discovery (Finding Live Hosts)

- Disable port scan (`-sn`) – only ping to see if hosts are up.
- Scan a whole subnet:
sudo nmap 10.129.2.0/24 -sn -oA tnet

- Scan from a list (`-iL`):
sudo nmap -sn -iL hosts.lst

- Scan a single IP:
sudo nmap 10.129.2.18 -sn

- Force ICMP echo requests (`-PE`) and disable ARP (`--disable-arp-ping`):
sudo nmap 10.129.2.18 -sn -PE --disable-arp-ping

- View packet trace (`--packet-trace`) and reason (`--reason`) to understand why a host is marked up.

Key Point: On local networks, Nmap defaults to ARP ping – this often bypasses firewalls.

## 2. Port Scanning

### TCP Port States

| State | Meaning |
|---|---|
| open | Port is listening for connections. |
| closed | Port is accessible but no service is listening (responds with RST). |
| filtered | Firewall drops or rejects packets – Nmap gets no reply or an error. |
| unfiltered | Port is accessible but state cannot be determined (ACK scan only). |
| open\|filtered | Nmap can't differentiate (no response). |
| closed\|filtered | Idle scan only – can't tell if closed or filtered. |

### TCP Scan Types

- SYN scan (`-sS`) – default for root; half-open, fast and stealthy.
- Connect scan (`-sT`) – full TCP handshake; accurate but logged.
- ACK scan (`-sA`) – used to map firewall rules; sends ACK flag.
- Other scans: NULL (`-sN`), FIN (`-sF`), Xmas (`-sX`) – used to evade simple firewalls.

### UDP Scan

- UDP scan (`-sU`) – stateless, slower; often used for DNS, SNMP, etc.

### Port Specification

- Single port: `-p 22`
- Multiple ports: `-p 22,80,443`
- Range: `-p 22-445`
- All ports: `-p-` (65,535)
- Top ports: `--top-ports=10` or `-F` (top 100)

### Example: Full TCP scan with speed

sudo nmap 10.129.2.243 -p- -Pn -n --disable-arp-ping --min-rate 1000 -T5 --max-retries 2 -oN tcp_all.txt

## 3. Service Enumeration & Version Detection

- Version scan (`-sV`) – probes open ports to identify service name, version, and sometimes OS.
- Aggressive scan (`-A`) – combines OS detection, version detection, traceroute, and default NSE scripts.
- Verbosity (`-v` / `-vv`) shows open ports as they are found.
- Progress stats: press `[Space]` during scan or use `--stats-every=5s`.

### Banner Grabbing

- Nmap often misses full banners; manual connection with `nc` or `ncat` can reveal more.
- Example (SMTP):
nc -nv 10.129.2.28 25

- Use `--packet-trace` to see raw packets and banners.

## 4. Operating System Detection

- OS detection (`-O`) – uses TCP/IP stack fingerprinting.
- Use with `-Pn` if host is alive.
- Often combined with `-sV` or `-A`.

Example:
sudo nmap 10.129.2.80 -p 22,80 -sS -O -T2 -Pn -n

Output may give kernel version (e.g., Linux 4.15–5.19) – further service banners (SSH, HTTP) reveal the distribution (e.g., Ubuntu).

## 5. Nmap Scripting Engine (NSE)

NSE extends Nmap with Lua scripts organised into categories:

| Category | Purpose |
|---|---|
| auth | Authentication credentials. |
| broadcast | Host discovery via broadcast. |
| brute | Brute-force logins. |
| default | Default scripts (`-sC`). |
| discovery | Information gathering. |
| dos | Denial-of-service tests. |
| exploit | Exploit known vulnerabilities. |
| fuzzer | Fuzzing for bugs. |
| intrusive | May disrupt target. |
| malware | Detect malware. |
| safe | Non-intrusive. |
| version | Enhanced version detection. |
| vuln | Vulnerability checks. |

Usage:
- Default scripts: `-sC`
- Category: `--script <category>`
- Specific scripts: `--script script1,script2`

Example:
sudo nmap 10.129.2.28 -p 25 --script banner,smtp-commands

## 6. Performance Optimisation

- Timing templates (`-T<0-5>`):
  - `-T0` (paranoid)
  - `-T1` (sneaky)
  - `-T2` (polite)
  - `-T3` (normal, default)
  - `-T4` (aggressive)
  - `-T5` (insane)

- RTT timeouts:
  - `--initial-rtt-timeout <time>`
  - `--max-rtt-timeout <time>`

- Retries:
  - `--max-retries <n>` – default 10; lower for speed.

- Rate limiting:
  - `--min-rate <number>` – set minimum packets/sec.

Trade-off: Faster scans may miss hosts/ports.

## 7. Firewall & IDS/IPS Evasion

- ACK scan (`-sA`) – used to determine firewall rules; returns `unfiltered` for ports that respond with RST.
- Source port spoofing (`--source-port <port>`) – e.g., use port 53 (DNS) to bypass trusts.
- Decoys (`-D RND:<count>`) – hide real IP among fake ones.
- Fragmentation (`-f`) – split packets into smaller fragments to evade simple IDS.
- Spoofed source IP (`-S <IP>` + `-e <interface>`).
- DNS proxying (`--dns-server <ns>`) – use trusted DNS server for lookups.

Example:
sudo nmap -sS -p 80 -Pn -n -D RND:5 --source-port 53 -f 10.129.2.28

### Real-world scenario: Filtered port becomes open with `--source-port 53`

sudo nmap 10.129.2.28 -p 50000 -sS --source-port 53

## 8. Saving Results

- Normal output: `-oN <file>.nmap`
- Grepable output: `-oG <file>.gnmap`
- XML output: `-oX <file>.xml`
- All formats: `-oA <basename>`

Convert XML to HTML for readability:
xsltproc target.xml -o target.html

## 9. Practical Labs Conducted

### Easy Lab

- Target: `10.129.2.80`
- Goal: Identify OS.
- Used evasive scan: `-T2`, `-Pn`, `-n`, `--disable-arp-ping`, `-D RND:5`, `--source-port 53`, `-f`.
- OS discovered: Ubuntu (via SSH banner with version scan).

### Medium Lab

- Target: `10.129.2.48`
- Note: Must use UDP protocol.
- Goal: Find DNS server version.
- Used `dig` or `nmap -sU -p 53 --script dns-version`.
- Answer: DNS version string (e.g., `9.11.3-1ubuntu1`).

### Hard Lab

- Target: `10.129.152.12`
- Goal: Find flag from a service version.
- Steps:
  1. Fast targeted TCP scan on common and suspicious ports.
  2. Found open ports: 22, 80, 50000.
  3. Version scan revealed `50000/tcp open tcpwrapped` – manual connection needed.
  4. Use `nc` (or `ncat`) to connect to port 50000 to grab banner/flag.

Command used:
sudo nc -nv -p 53 10.129.152.12 50000

The flag is embedded in the banner. Submit it exactly as shown.

## 10. Summary of Key Flags

| Option | Description |
|---|---|
| `-sn` | Host discovery only (no port scan). |
| `-sS` | SYN stealth scan. |
| `-sT` | TCP Connect scan. |
| `-sU` | UDP scan. |
| `-sA` | ACK scan (firewall mapping). |
| `-sV` | Service/version detection. |
| `-O` | OS detection. |
| `-A` | Aggressive scan (OS, version, traceroute, default scripts). |
| `-p-` | All ports. |
| `-F` | Fast (top 100 ports). |
| `--top-ports=N` | Top N ports. |
| `-Pn` | Skip host discovery (assume host up). |
| `-n` | Disable DNS resolution. |
| `--disable-arp-ping` | Do not use ARP ping. |
| `--source-port` | Spoof source port. |
| `-D RND:<count>` | Use random decoys. |
| `-f` | Fragment packets. |
| `-T<0-5>` | Timing template. |
| `--min-rate` | Minimum packets per second. |
| `--max-retries` | Maximum retransmissions. |
| `--reason` | Show why a port is in a certain state. |
| `--packet-trace` | Show sent/received packets. |
| `-sC` | Default NSE scripts. |
| `--script <name/category>` | Run NSE scripts. |
| `-oN/-oG/-oX/-oA` | Output formats. |

## 11. Pro Tips

- Always save scan results (`-oA`) for later analysis.
- Use `-Pn -n --disable-arp-ping` when you know the host is alive to speed up scans.
- In CTF labs, suspicious high ports (e.g., 31337, 50000) often hold flags.
- Manual interaction (`nc`, `curl`, `dig`) often reveals more than automated scans.
- For evasion, combine `--source-port 53` with decoys and fragmentation, but avoid overdoing it (may trigger IPS).
- Monitor your own alert page (if available) to avoid being banned.

## 12. Resources

- Nmap Official Documentation: https://nmap.org/docs.html
- NSE Scripts Catalog: https://nmap.org/nsedoc/
- Nmap Performance Tuning: https://nmap.org/book/man-performance.html
- Firewall/IDS Evasion Techniques: https://nmap.org/book/man-bypass-firewalls-ids.html
```
