# External Reconnaissance & Enumeration

## Objective

The goal of external reconnaissance is to collect publicly available information about a target before performing any active testing. This information helps validate the engagement scope, identify potential attack vectors, and build resources for later stages of the penetration test.

---

# Information to Collect

During reconnaissance, gather the following information:

* Domain and subdomains
* DNS records (A, MX, NS, TXT, CNAME)
* IP addresses and ASN information
* Email address formats
* Employee names
* Public documents and metadata
* Technology stack
* Public source code repositories
* Leaked credentials from breach databases

---

# Common Tools

| Tool              | Purpose                         |
| ----------------- | ------------------------------- |
| dig               | DNS enumeration                 |
| nslookup          | DNS lookups                     |
| host              | DNS queries                     |
| whois             | Domain registration information |
| dnsrecon          | Passive DNS enumeration         |
| Sublist3r         | Subdomain discovery             |
| ExifTool          | Metadata extraction             |
| Dehashed          | Breach data search              |
| Have I Been Pwned | Email breach lookup             |

---

# Enumeration Workflow

```text
Passive Reconnaissance
        │
        ▼
Collect Public Information
        │
        ▼
Validate Findings
        │
        ▼
Build User & Target Lists
        │
        ▼
Begin Active Enumeration
```

---

# Important Rules

* Stay within the approved scope.
* Start with passive reconnaissance before active testing.
* Document all findings and evidence.
* Verify information using multiple sources.
* Do not scan or attack third-party infrastructure without authorization.

---

# Key Commands

```bash
# DNS Enumeration
dig domain.com ANY
dig domain.com MX
dig domain.com TXT
dig domain.com NS

# WHOIS
whois domain.com

# Passive Subdomain Enumeration
dnsrecon -d domain.com -t std
sublist3r -d domain.com

# Metadata Extraction
exiftool file.pdf
```

---

# Key Takeaways

* External reconnaissance provides valuable intelligence before interacting with the target.
* Small pieces of information such as email formats, public documents, or job postings can help build successful attack paths.
* Effective reconnaissance combines information from multiple public sources while remaining ethical and within the defined scope.
