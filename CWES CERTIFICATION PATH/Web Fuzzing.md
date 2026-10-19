# Web Fuzzing Field Guide

A practical reference for fuzzing directories, parameters, virtual hosts, subdomains, and web APIs. It covers recursive fuzzing, output filtering, finding validation, API endpoint discovery, and a full skills-assessment walkthrough.

---

## Table of Contents

1. [What You Can Fuzz](#1-what-you-can-fuzz)
2. [Recursive Fuzzing](#2-recursive-fuzzing)
3. [Parameter and Value Fuzzing](#3-parameter-and-value-fuzzing)
4. [Virtual Host and Subdomain Fuzzing](#4-virtual-host-and-subdomain-fuzzing)
5. [Filtering Fuzzing Output](#5-filtering-fuzzing-output)
6. [Validating Findings](#6-validating-findings)
7. [Web APIs: Concepts](#7-web-apis-concepts)
8. [Identifying API Endpoints](#8-identifying-api-endpoints)
9. [API Fuzzing](#9-api-fuzzing)
10. [Skills Assessment Walkthrough](#10-skills-assessment-walkthrough)
11. [Cheat Sheet](#11-cheat-sheet)
12. [Safety and Ethics](#12-safety-and-ethics)

---

## 1. What You Can Fuzz

| Target | Tool mode | Example |
|---|---|---|
| Directories and files | `ffuf`, `gobuster dir` | `/FUZZ`, `/FUZZ.php` |
| Parameter values | `ffuf`, `wenum` | `?x=FUZZ`, `-d "y=FUZZ"` |
| Virtual hosts | `gobuster vhost` | `Host: word.domain.tld` |
| Subdomains | `gobuster dns` | `word.domain.tld` (DNS) |
| API endpoints | `webfuzz_api`, `ffuf` | `/FUZZ`, `/api/v1/FUZZ` |

---

## 2. Recursive Fuzzing

### Concept

Recursive fuzzing automatically drills into nested directories instead of fuzzing each level by hand.

### How it works

1. **Initial fuzzing.** Start at the web root, send the wordlist, and look for valid directories (200 or 301).
2. **Expansion.** For each hit, create a new branch (`/found/FUZZ`) and fuzz again.
3. **Iteration.** Repeat until a depth limit is reached or no new directories appear.

Think of it as a tree: the root is the trunk, directories are branches, and files are leaves.

### Command (ffuf)

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -ic \
  -u http://TARGET/FUZZ \
  -e .html \
  -recursion \
  -v
```

### Key flags

| Flag | Purpose |
|---|---|
| `-recursion` | Enable recursive fuzzing |
| `-recursion-depth N` | Limit how deep it goes (for example, 2) |
| `-ic` | Ignore comment lines (`#`) in the wordlist |
| `-e .php,.html` | Append extensions |
| `-rate N` | Requests per second, to protect the server |
| `-timeout` | Per-request timeout |

### Output pattern

```text
[Status: 301] .../level1          <- valid directory, queued
[INFO] Adding a new job: .../level1/FUZZ
[Status: 200] .../level1/index.html
[Status: 301] .../level1/level2   <- deeper directory, queued
```

At the deepest level, a file with an unusual size often holds the flag or sensitive content.

### Safety

Recursion is resource-heavy. Always set `-recursion-depth`, `-rate`, and `-timeout` so you do not overwhelm the target.

---

## 3. Parameter and Value Fuzzing

### Concept

Test how the application handles parameters and their values to find hidden functionality, logic flaws, or injection points.

### GET vs. POST

| | GET | POST |
|---|---|---|
| Location | URL (`?x=value`) | Request body |
| Visibility | Visible | Hidden from the URL bar |
| Typical use | Search and filters | Logins, sensitive data, uploads |
| Encoding | URL query string | `application/x-www-form-urlencoded` or `multipart/form-data` |

### Probe manually first

```bash
curl http://TARGET/get.php            # "Invalid parameter value x:"   (parameter missing)
curl "http://TARGET/get.php?x=1"      # "Invalid parameter value x: 1" (value rejected)
```

The error message tells you the parameter name. Always read it before fuzzing.

### Fuzz a GET value (wenum)

```bash
wenum -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  --hc 404 \
  -u "http://TARGET/get.php?x=FUZZ"
```

`--hc 404` hides 404 responses. Each wordlist entry replaces `FUZZ`. The valid value returns 200 OK instead of the "invalid" message.

### Fuzz a POST value (ffuf)

```bash
ffuf -u http://TARGET/post.php \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "y=FUZZ" \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200 -v
```

| Flag | Meaning |
|---|---|
| `-X POST` | Use the POST method |
| `-H "Content-Type: ..."` | Declare the body encoding |
| `-d "y=FUZZ"` | The fuzzable value goes in the body |
| `-mc 200` | Match only 200 OK |

### Validate the hit

```bash
curl "http://TARGET/get.php?x=VALID_VALUE"        # GET
curl -d "y=VALID_VALUE" http://TARGET/post.php    # POST
```

Key takeaway: GET uses `?x=FUZZ` in the URL, and POST uses `-d "y=FUZZ"` in the body. The valid value breaks the "invalid" pattern, usually as a change from a 40x to a 200.

---

## 4. Virtual Host and Subdomain Fuzzing

### Concepts

| | Virtual host (vhost) | Subdomain |
|---|---|---|
| Identified by | Host header (HTTP) | DNS records |
| Resolved by | Same IP, different name | DNS to a specific IP |
| Purpose | Multiple sites on one server | Separate sections or services of a domain |
| Risk | Exposed internal apps, sensitive data | Subdomain takeover when DNS is mismanaged |

### Map the vhost locally

```bash
echo "TARGET_IP inlanefreight.htb" | sudo tee -a /etc/hosts
```

`/etc/hosts` resolves top to bottom, and the first match wins. Stale entries from old labs can hijack your scans and cause timeouts. Clean them first:

```bash
sudo sed -i '/inlanefreight.htb/d' /etc/hosts
echo "TARGET_IP inlanefreight.htb" | sudo tee -a /etc/hosts
```

### VHost fuzzing (gobuster)

```bash
gobuster vhost \
  -u http://inlanefreight.htb:PORT \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  --append-domain \
  -t 100 \
  -xs 400
```

| Flag | Meaning |
|---|---|
| `vhost` | vhost mode |
| `-u` | Base URL |
| `--append-domain` | Critical. Appends the base domain to each word (`admin` becomes `admin.domain.tld`) |
| `-t 100` | Threads |
| `-xs 400` | Exclude status 400 as noise |

A `Status: 200` result means a valid, accessible vhost:

```text
Found: admin.inlanefreight.htb Status: 200 [Size: 100]
```

### Subdomain fuzzing (gobuster dns)

```bash
gobuster dns \
  --domain inlanefreight.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

In recent Gobuster versions, `-d` means delay, not domain. Use `--domain` or `--do` instead.

Each `Found:` line is a resolving subdomain:

```text
Found: support.inlanefreight.com   134.209.24.248
```

Key takeaway: vhosts hide behind the Host header on a shared IP, and subdomains hide behind DNS. A 200 response (vhost) or a resolving record (subdomain) is a valid lead.

---

## 5. Filtering Fuzzing Output

### Why

Fuzzers produce thousands of results. Without filtering, real findings get buried under 404 noise. Filtering is required, not optional.

### Default behavior

If you do not specify a matcher, ffuf matches only:

```text
200-299, 301, 302, 307, 401, 403, 405, 500
```

That is why 404s are hidden by default. `-mc all` disables this and floods the output.

### ffuf filters

| Flag | Purpose | Example |
|---|---|---|
| `-mc` | Match status codes | `-mc 200,204,400-499` |
| `-fc` | Filter status codes | `-fc 404,401,302` |
| `-fs` | Filter by size | `-fs 0-1023` |
| `-ms` | Match by size | `-ms 3456` |
| `-fw` | Filter by word count | `-fw 219` |
| `-mw` | Match word count | `-mw 5-10` |
| `-fl` | Filter by line count | `-fl 10` |
| `-ml` | Match line count | `-ml 20` |
| `-mt` | Match response time | `-mt >500` |

Combined examples:

```bash
# 200 only, exactly 427 words, size over 500 bytes
ffuf -u http://example.com/FUZZ -w wl.txt -mc 200 -fw 427 -ms >500

# Exclude common error codes
ffuf -u http://example.com/FUZZ -w wl.txt -fc 404,401,302

# Find slow endpoints (over 500 ms), which can indicate a logic issue
ffuf -u http://example.com/FUZZ -w wl.txt -mt >500
```

### Other tools

| Tool | Include codes | Exclude codes | Size | Words | Lines | Regex |
|---|---|---|---|---|---|---|
| Gobuster | `-s` (dir only) | `-b` (dir only) | `--exclude-length` | n/a | n/a | n/a |
| wenum | `--sc` | `--hc` | `--hs` / `--ss` | `--hw` / `--sw` | `--hl` / `--sl` | `--hr` / `--sr` |
| Feroxbuster | `-s` | `-C` | `-S` | `-W` | `-N` | `-X` |

wenum regex example:

```bash
wenum -w wl.txt --sr "admin\|password" -u https://example.com/FUZZ
```

Feroxbuster example:

```bash
feroxbuster --url http://example.com -w wl.txt -s 200 -S 10240 -X "error"
```

Key concept: a matcher decides what ffuf keeps, and a filter decides what it removes. Combine status, size, word count, and regex to narrow thousands of responses.

---

## 6. Validating Findings

### Why

Fuzzing produces leads, not confirmed bugs. Many hits are false positives. Validation turns a lead into a proven finding.

| Purpose | Outcome |
|---|---|
| Confirm | Prove the issue is real, not a false alarm |
| Understand impact | Assess severity and business risk |
| Reproduce | Give consistent steps so it can be fixed |
| Gather evidence | Provide proof for developers and stakeholders |

### Three-step manual verification

1. **Reproduce** the request with `curl` or a browser.
2. **Analyze** the response for error messages or unexpected content.
3. **Exploit carefully.** Keep it to a proof of concept with no damage. For example, for SQLi, return the database version, not the data.

Golden rule: prove the bug, don't damage the target. Prefer headers over content when confirming exposure.

### Worked example: `/backup/` directory

Step 1. Check whether the directory is browsable:

```bash
curl http://TARGET/backup/
```

An `Index of /backup/` page with file links confirms a directory listing.

Step 2. Validate with headers only, without downloading the file:

```bash
curl -I http://TARGET/backup/password.txt
```

| Header | Why it matters |
|---|---|
| `Content-Type` | Confirms the file type (`text/plain`, `application/sql`, `application/zip`) |
| `Content-Length` | Greater than 0 means real content. 0 means likely empty |

Example:

```text
Content-Type: text/plain;charset=utf-8   -> text file as expected
Content-Length: 171                      -> 171 bytes of real data
```

A browsable directory, a sensitive filename, and non-zero length are strong evidence of real exposure, and you never have to read the file.

### Responsible workflow

```text
Fuzz finds /backup/ (200)
   |
curl /backup/          -> confirm the listing
   |
curl -I /backup/file   -> headers only (type and size)
   |
Report. No data exfiltration needed.
```

---

## 7. Web APIs: Concepts

### What an API is

An API is the bridge between a server (data and functionality) and a client (browser, mobile app, or another server).

### The three API styles

| | REST | SOAP | GraphQL |
|---|---|---|---|
| Style | Architectural, stateless | Formal protocol | Query language and runtime |
| Endpoints | Many | One | One (usually `/graphql`) |
| Format | JSON or XML | XML envelope | Query in, JSON out |
| Methods | GET, POST, PUT, DELETE (CRUD) | Defined in the body | Queries and mutations |
| Strength | Simple, lightweight | Security, transactions, strict errors | No over- or under-fetching, strong typing |
| Best for | Most modern apps | Enterprise systems | Flexible data needs |

REST example:

```http
GET /users/123
```

SOAP example:

```xml
<soapenv:Envelope ...>
  <soapenv:Body>
    <tem:GetStockPrice>
      <tem:StockName>AAPL</tem:StockName>
    </tem:GetStockPrice>
  </soapenv:Body>
</soapenv:Envelope>
```

GraphQL example:

```graphql
query {
  user(id: 123) { name email }
}
```

### API vs. web server

| Feature | Web server | API |
|---|---|---|
| Purpose | Serve pages | Let applications exchange data |
| Format | HTML, CSS, JS | JSON or XML |
| Interaction | Users browse directly | Apps act on the user's behalf |
| Access | Usually public | Public, private, or partner |

Fuzzing shifts with the target:

```text
Web server  -> fuzz directories and files
Web API     -> fuzz endpoints and parameters
```

---

## 8. Identifying API Endpoints

### REST

Endpoints are hierarchical:

```text
/users          -> collection
/users/123      -> specific resource
```

| Parameter type | Location | Example |
|---|---|---|
| Query | After `?` | `/users?limit=10&sort=name` |
| Path | Embedded in the URL | `/products/{id}` |
| Body | POST, PUT, PATCH | `{"name":"New","price":99.99}` |

Discovery methods:
- **API documentation** (Swagger or OpenAPI, RAML). This is the most reliable source.
- **Network traffic** (Burp Suite, browser DevTools).
- **Parameter name fuzzing** (ffuf or wfuzz with wordlists).

### SOAP

SOAP uses a single endpoint. The operation is in the XML body, not the URL. A WSDL file describes everything: operations, input and output parameters, data types, and the endpoint URL.

Discovery methods: WSDL analysis (most reliable), Wireshark or tcpdump, and fuzzing malformed SOAP messages.

Example operation:

```xml
<lib:SearchBooks>
  <lib:keywords>cybersecurity</lib:keywords>
  <lib:author>Dan Kaminsky</lib:author>
</lib:SearchBooks>
```

### GraphQL

GraphQL uses a single endpoint, usually `/graphql`. Queries read data, and mutations write it.

| Component | Example |
|---|---|
| Field | `name`, `email` |
| Relationship | `posts` |
| Nested object | `posts { title body }` |
| Argument | `posts(limit: 5)` |

A mutation has an operation (`createPost`), arguments (`title: "..."`), and a selection set (`id`, `title`).

Discovery methods:
- **Introspection** (most reliable). Returns the complete schema.
- **Documentation UIs** such as GraphiQL or GraphQL Playground.
- **Traffic analysis.**

### Discovery at a glance

| Method | REST | SOAP | GraphQL |
|---|---|---|---|
| Docs or spec | Swagger, RAML | WSDL | Introspection, GraphiQL |
| Traffic | Burp, DevTools | Wireshark | Burp, DevTools |
| Fuzzing | ffuf or wfuzz on parameters | Malformed XML | Craft from the schema |

---

## 9. API Fuzzing

### What it is

API fuzzing is fuzzing tailored to API structure. You vary parameter values, headers, parameter order, and data types for each request. The goal is to trigger errors that expose input-validation flaws, injection, or authorization problems.

### Three types

| Type | Targets | Can reveal |
|---|---|---|
| Parameter | Query parameters, headers, body | SQLi, command injection, XSS, tampering |
| Data format | Structure and encoding of JSON or XML | Parsing errors, buffer overflows |
| Sequence | Order and timing of requests | Race conditions, IDOR, auth bypass |

In rough terms: fuzz values first, then formats, then workflow logic.

### Worked example: endpoint discovery

Step 1. Read the documentation:

```text
http://TARGET/docs
```

Swagger lists the documented endpoints. Documentation never tells the full story, though. Hidden endpoints exist because they are internal, obscure, or unfinished.

Step 2. Install and run an endpoint fuzzer:

```bash
git clone https://github.com/PandaSt0rm/webfuzz_api.git
cd webfuzz_api
pip3 install -r requirements.txt
python3 api_fuzzer.py http://TARGET
```

Step 3. Read the results:

```text
Total requests: 4730
404: 4727   <- noise
200: 2      <- hits
405: 1      <- method mismatch
```

| Status | Meaning |
|---|---|
| 404 | Noise |
| 200 | Valid endpoint |
| 405 | Endpoint exists, wrong HTTP method |

Step 4. Query the hidden endpoint:

```bash
curl http://TARGET/<hidden_endpoint>
```

### Beyond discovery

| Vulnerability | What fuzzing exposes |
|---|---|
| BOLA / IDOR | Changing IDs returns other users' objects |
| Function-level auth bypass | Manipulated parameters reach forbidden functions |
| SSRF | Injected values make the server issue unintended requests |

### API fuzzing workflow

```text
1. Read /docs (Swagger)       -> documented endpoints
2. Run endpoint fuzzer        -> undocumented endpoints
3. Interpret status codes     -> 200 valid, 405 wrong method, 404 noise
4. curl the hidden endpoint   -> data or flag
5. Fuzz parameters            -> BOLA, auth bypass, SSRF
```

Key takeaway: APIs expose more than they document. Swagger shows the intended surface, and fuzzing reveals the real one. Status codes are intelligence.

---

## 10. Skills Assessment Walkthrough

The chain runs: directory, then parameter, then vhost, then subdomain, then recursive paths, then flag. Every fuzz uses `common.txt`. Replace `TARGET` and `PORT` with your instance values.

### Step 1: Directory fuzzing on the root

```bash
ffuf -t 100 \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -u http://TARGET:PORT/FUZZ \
  -e .php,.html,.txt,.bak,.js \
  -recursion -recursion-depth 2 -rate 500
```

Found: `/admin` (301).

Wrong-target check: if you see `czcmdcvt`, `docs`, or `items`, you are on the API challenge instance, not the Skills Assessment. Spawn the correct instance.

### Step 2: Fuzz inside `/admin`

```bash
ffuf -t 100 \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -u http://TARGET:PORT/admin/FUZZ \
  -e .php
```

Found: `panel.php` (58 bytes).

### Step 3: Find the parameter

```bash
curl http://TARGET:PORT/admin/panel.php
# "Invalid parameter, please ensure accessID is set correctly"
```

### Step 4: Fuzz `accessID`, filtering the 58-byte false positive

```bash
ffuf -t 100 \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -u "http://TARGET:PORT/admin/panel.php?accessID=FUZZ" \
  -fs 58
```

Found: `getaccess`.

### Step 5: Read the vhost hint

```bash
curl "http://TARGET:PORT/admin/panel.php?accessID=getaccess"
# "Head on over to the fuzzing_fun.htb vhost..."
```

### Step 6: Add the vhost and fuzz subdomains

```bash
echo "TARGET_IP fuzzing_fun.htb" | sudo tee -a /etc/hosts

gobuster vhost \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  --append-domain \
  -u "http://fuzzing_fun.htb:PORT/" \
  -t 100 \
  -xs 403,400
```

Found: `hidden.fuzzing_fun.htb`.

```bash
echo "TARGET_IP hidden.fuzzing_fun.htb" | sudo tee -a /etc/hosts
grep fuzzing_fun /etc/hosts   # confirm exactly two clean lines
```

### Step 7: Follow the clue

```bash
curl http://hidden.fuzzing_fun.htb:PORT/
# "Wrong path, remember to be looking in /godeep"
```

### Step 8: Fuzz `/godeep`

```bash
ffuf -t 100 \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -u "http://hidden.fuzzing_fun.htb:PORT/godeep/FUZZ"
```

Found: `stoneedge` (301).

### Step 9: Fuzz deeper, recursively

```bash
ffuf -t 100 \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -u "http://hidden.fuzzing_fun.htb:PORT/godeep/stoneedge/FUZZ" \
  -e .php,.html,.txt,.bak,.js \
  -recursion -recursion-depth 2 -rate 500 \
  -fc 403
```

Found: `bbclone/`, then `typo3/`.

Final path: `/godeep/stoneedge/bbclone/typo3/`

### Step 10: Capture the flag

```bash
curl http://hidden.fuzzing_fun.htb:PORT/godeep/stoneedge/bbclone/typo3/
```

The flag is in the response body. Redacted here.

### Chain summary

```text
/ -> /admin -> panel.php -> accessID=getaccess
  -> fuzzing_fun.htb -> hidden.fuzzing_fun.htb
  -> /godeep -> /stoneedge -> /bbclone/typo3/ -> FLAG
```

---

## 11. Cheat Sheet

### Task reference

| Task | Command |
|---|---|
| Fuzz directories (recursive) | `ffuf -w WL -u URL/FUZZ -e .php -recursion` |
| Fuzz a GET value | `ffuf -w WL -u "URL?x=FUZZ"` |
| Fuzz a POST value | `ffuf -w WL -u URL -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "y=FUZZ"` |
| Fuzz vhosts | `gobuster vhost -u URL -w WL --append-domain -xs 400` |
| Fuzz subdomains | `gobuster dns --domain DOMAIN -w WL` |
| Filter 404s | `-fc 404` (ffuf), `-C 404` (feroxbuster), `--hc 404` (wenum) |
| Filter by size | `-fs 58` |
| Match only 200 | `-mc 200` |

### ffuf default matcher

```text
200-299, 301, 302, 307, 401, 403, 405, 500
```

### Status code intelligence

| Code | Meaning for fuzzing |
|---|---|
| 200 | Valid and accessible |
| 301, 302, 307 | Redirect, likely a directory. Follow it |
| 401, 403 | Exists but protected. Still a finding |
| 405 | Endpoint exists, wrong HTTP method |
| 404 | Not found. Noise |

### `/etc/hosts` hygiene

```bash
sudo sed -i '/domain.tld/d' /etc/hosts
echo "IP domain.tld" | sudo tee -a /etc/hosts
grep domain.tld /etc/hosts   # confirm one clean line
```

Duplicate or stale entries cause timeouts and misrouted scans.

### Fuzzing loop

```text
1. ENUMERATE   -> directories, parameters, vhosts, subdomains, endpoints
2. FILTER      -> remove noise (404, 403, baseline size)
3. IDENTIFY    -> read status codes and response sizes
4. VALIDATE    -> curl the hit, headers only
5. PIVOT       -> follow clues to the next target
6. REPEAT
```

---

## 12. Safety and Ethics

- Rate-limit your scans (`-rate`, `-t`) so you do not overwhelm servers.
- Depth-limit recursion with `-recursion-depth`.
- Validate with headers, not data exfiltration.
- Prove the bug as a proof of concept, without causing damage.
- Test only systems you are authorized to test.

---

## Final Notes

Every tool here automates the tedious parts. The skill is in reading responses, filtering noise, and following clues.

Fuzzing finds leads, and validation proves them.

APIs, vhosts, and recursion each hide behind a different surface: the Host header, DNS, nested paths, parameter values, and undocumented endpoints. Learn each surface, and you can map most targets.
