# 🔐 Password Brute-Forcing & Credential Attacks – Complete Module Documentation

This document summarises everything learned, practised, and accomplished during the **Brute-Forcing & Password Attacks** module. It covers core concepts, tools, wordlist generation, filtering, real-world attack flows, and the two-part skills assessment.

---

## 1. What is Brute-Forcing?

Brute-forcing is a trial-and-error method used to guess passwords, login credentials, or encryption keys. The attacker systematically tries every possible combination of characters until the correct one is found.

**Key factors that determine success:**
- Password complexity (length and character set)
- Available computational power (guesses per second)
- Security defences (account lockouts, CAPTCHAs, rate-limiting)

---

## 2. Types of Brute-Force Attacks

| Attack Type | Description | Best Used When… |
| :--- | :--- | :--- |
| **Simple Brute Force** | Tries every possible combination of a given character set and length. | No prior information about the password is available. |
| **Dictionary Attack** | Uses a pre-compiled list of common passwords. | The target likely uses weak, common, or default passwords. |
| **Hybrid Attack** | Combines dictionary words with mutations (e.g., appending numbers or symbols). | Users make minor changes to common words (e.g., `Summer2023` → `Summer2024!`). |
| **Credential Stuffing** | Uses leaked credentials from other breaches to access other services. | Users reuse passwords across multiple accounts. |
| **Password Spraying** | Tries a small set of common passwords against many usernames. | Account lockout policies are in place; avoids triggering them. |
| **Rainbow Table Attack** | Uses pre-computed hash tables to reverse password hashes quickly. | You have a large set of hashes and enough storage space. |
| **Reverse Brute Force** | Tries a known password against many usernames. | A single password is suspected to be reused by many users. |
| **Distributed Brute Force** | Spreads the workload across many machines. | The password is highly complex and a single machine is too slow. |

---

## 3. Password Security Fundamentals

### The 4 Golden Rules of a Strong Password
1. **Length is KING** – at least 12 characters. Each extra character multiplies the search space exponentially.
2. **Complexity** – mix uppercase, lowercase, numbers, and symbols.
3. **Uniqueness** – never reuse the same password across different accounts.
4. **Randomness** – avoid dictionary words, personal information, or predictable patterns.

### Common Password Weaknesses
- Short passwords (< 8 characters)
- Common words or phrases (`password`, `admin`, `letmein`)
- Personal information (birthdays, pet names, phone numbers)
- Reusing the same password
- Predictable patterns (`123456`, `qwerty`, `p@ssw0rd`)

### Default Credentials – A Major Vulnerability
Many devices and services ship with default usernames and passwords (e.g., `admin:admin`, `cisco:cisco`). Attackers keep extensive lists of these defaults and can gain access instantly if they are not changed.

---

## 4. The Mathematics Behind Brute-Forcing

Combinations = (Character Set Size) ^ (Password Length)

| Password Length | Character Set | Combinations |
| :--- | :--- | :--- |
| 6 | Lowercase only | 26⁶ ≈ 300 million |
| 8 | Lowercase only | 26⁸ ≈ 200 billion |
| 8 | Lower + Upper + Numbers | 62⁸ ≈ 200 trillion |
| 12 | All ASCII printable (94 chars) | 94¹² ≈ 475 sextillion |

Even with a supercomputer (1 trillion guesses/sec), a 12-character full-complexity password would take **~15,000 years** to crack.

---

## 5. Tools Used

### Hydra – Network Login Cracker

**Syntax:**
```
hydra [login options] [password options] [target] [service]
```

**Common options:**
- `-l` or `-L` – single username or a file
- `-p` or `-P` – single password or a file
- `-t` – number of parallel tasks (threads)
- `-f` – stop after first successful login
- `-s` – specify a non-default port
- `-V` – verbose output

**Example – SSH brute-force:**
```
hydra -l satwossh -P passwords.txt 154.57.164.82 -s 31737 ssh -f
```

**Example – HTTP Basic Authentication:**
```
hydra -L usernames.txt -P passwords.txt 154.57.164.82 -s 30252 http-get / -f
```

**Example – Custom Login Form (http-post-form):**
```
hydra -L usernames.txt -P passwords.txt target -s port -f http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"
```

### Medusa – Parallel Login Brute-Forcer

**Syntax:**
```
medusa [target options] [credential options] -M module [module options]
```

**Example – SSH:**
```
medusa -h 154.57.164.82 -n 31737 -u satwossh -P passwords.txt -M ssh -t 3 -f
```

**Example – FTP:**
```
medusa -h 127.0.0.1 -u ftpuser -P passwords.txt -M ftp -t 5
```

**Key feature:** `-e ns` checks for empty passwords and username-as-password.

### Username Anarchy – Generate Username Variations
```
./username-anarchy Thomas Smith > thomas_usernames.txt
```
Generates variations like `thomas`, `smith`, `thomas.smith`, `t.smith`, `smitht`, `ts`, etc.

### CUPP – Common User Passwords Profiler
```
cupp -i
```
Interactive mode asks for personal details (name, birthdate, pet, partner, hobbies) and builds a custom password list (e.g., `Janey1990!`, `Spot2024@`, `jane@ahi`).

### Filtering with grep and Regular Expressions

To match password policies, filter wordlists:

| Policy Requirement | Grep Command |
| :--- | :--- |
| Minimum length 8 | `grep -E '^.{8,}$'` |
| At least one uppercase | `grep -E '[A-Z]'` |
| At least one lowercase | `grep -E '[a-z]'` |
| At least one digit | `grep -E '[0-9]'` |
| At least two special chars | `grep -E '([!@#$%^&*].*){2,}'` |
| Exclude common words | `grep -v -f dictionary.txt` |

**Example – full pipeline:**
```
grep -E '^.{8,}$' passwords.txt | grep -E '[A-Z]' | grep -E '[a-z]' | grep -E '[0-9]' | grep -E '([!@#$%^&*].*){2,}' > filtered.txt
```

---

## 6. Hands-On Labs – Attack Flows

### Lab 1: Cracking a 4-Digit PIN
- Target: Web endpoint `/pin?pin=0000`
- Script iterated from 0000 to 9999 and checked the response.
- Found the correct PIN and received a flag.

### Lab 2: Dictionary Attack on a Login Form
- Target: `/dictionary` endpoint with a POST parameter `password`.
- Used `500-worst-passwords.txt` from SecLists.
- Found the correct password and flag.

### Lab 3: Hybrid Attack – Filtering Wordlists
- Downloaded `darkweb2017_top-10000.txt`.
- Filtered with grep to match a password policy (min 8 chars, uppercase, lowercase, number).
- Reduced the list from 10,000 to 89 passwords, dramatically speeding up the attack.

### Lab 4: Basic HTTP Authentication (Part 1 of Skills Assessment)
- Target: `154.57.164.82:30252`
- Used Hydra with `http-get /` module.
- Found credentials: `admin:Admin123`.
- Logged in and received the username for Part 2: `satwossh`.

### Lab 5: SSH + FTP Pivoting (Skills Assessment Part 2)
1. Brute-forced SSH for `satwossh` → password `password1`.
2. Logged in via SSH, found FTP service running on port 21.
3. Brute-forced FTP with system usernames and `passwords.txt`.
4. Found that `satwossh:password1` worked for FTP as well.
5. Noticed `IncidentReport.txt` pointing to Thomas Smith – indicating another user (`thomas`).
6. Planned to brute-force SSH for `thomas` to retrieve the final flag.

---

## 7. Lessons Learned & Key Takeaways

- **Custom wordlists beat generic ones.** Tools like Username Anarchy and CUPP create targeted lists that dramatically improve success rates.
- **Filtering is essential.** Matching password policies with grep removes invalid candidates and makes attacks much faster.
- **Pivoting is powerful.** A foothold on SSH often reveals other services (FTP, internal web apps) that can be brute-forced from the inside.
- **Incident reports and system files** (like `/etc/passwd` and `/home/` directories) provide valuable intelligence. They reveal usernames and clues for further attacks.
- **Credentials are often reused.** The same password (`password1`) worked for both SSH and FTP for the same user. Reusing a password across services is a common and critical vulnerability.
- **Always check for other users.** If you get SSH access, look for other home directories – they often hold the next piece of the puzzle.

---

## 8. Final Answers from the Skills Assessment

**Part 1**
- Username for Part 2: `satwossh`
- Password for Basic Auth: `Admin123`

**Part 2**
- FTP username: `satwossh` (found via brute-forcing)
- Flag: (to be retrieved after brute-forcing `thomas` and reading `/home/thomas/flag.txt` or `/root/flag.txt`)

---

## 9. Useful Wordlists & Sources

| Wordlist | Description | Source |
| :--- | :--- | :--- |
| `rockyou.txt` | Millions of leaked passwords | SecLists / RockYou breach |
| `2023-200_most_used_passwords.txt` | Top 200 common passwords | SecLists |
| `500-worst-passwords.txt` | Extremely weak passwords | SecLists |
| `top-usernames-shortlist.txt` | Common usernames | SecLists |
| `darkweb2017_top-10000.txt` | 10,000 common passwords | SecLists |
| `passwords.txt` | Custom list from target | Retrieved via SCP/SSH |

---

## 10. Final Thoughts

Brute-forcing is a numbers game – but with smart wordlists, filtering, and pivoting, it becomes a precise and efficient attack methodology.

**As an attacker:** Use targeted lists, filter efficiently, and always pivot to find new services and users.

**As a defender:** Enforce strong password policies, use MFA, implement rate-limiting and lockout policies, and never rely on default credentials.
