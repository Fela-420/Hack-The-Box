# Password Spraying

## Overview

Password spraying is an authentication attack in which an attacker attempts a **small number of commonly used passwords across many different user accounts**. Unlike traditional brute-force attacks, the attacker avoids repeatedly targeting a single account, reducing the chance of triggering per-account lockout mechanisms.

The technique is commonly discussed in enterprise security because it targets environments where users choose weak or predictable passwords.

---

# How Password Spraying Differs from Brute Force

| Technique             | Description                                               | Typical Detection                                                                             |
| --------------------- | --------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **Brute Force**       | Many password guesses against a single account            | Often triggers account lockout after multiple failed attempts.                                |
| **Password Spraying** | One or a few common passwords tried against many accounts | May evade simple per-account lockout policies if monitoring focuses only on individual users. |

---

# Typical Attack Lifecycle

1. Obtain or infer a list of valid usernames.
2. Select a commonly used password.
3. Attempt authentication across multiple accounts.
4. Monitor for successful authentication.
5. Repeat after an appropriate interval if attempting another password.

---

# Why Password Spraying Can Be Effective

Organizations are vulnerable when:

* Users choose weak or predictable passwords.
* Default passwords remain unchanged.
* Password policies are weak.
* Multi-factor authentication (MFA) is not enforced.
* Authentication monitoring focuses only on individual accounts rather than organization-wide patterns.

---

# Common Sources of Usernames

Examples of publicly available or legitimate organizational sources include:

* Employee directories
* Company websites
* Public contact pages
* Professional networking profiles
* Public documents containing author metadata
* Standard corporate email naming conventions

---

# Common Weak Password Themes

Attackers frequently target passwords based on predictable patterns, such as:

* Generic default passwords
* Company-related names
* Seasonal naming conventions
* Years or simple numeric sequences
* Common keyboard patterns

These examples illustrate why organizations should prohibit predictable passwords through password policies and blocklists.

---

# Defensive Detection

Security teams should monitor for:

* Numerous failed logins across many different accounts.
* Authentication attempts using the same password pattern.
* Login attempts originating from a single source against multiple users.
* Unusual authentication activity outside normal business hours.
* Authentication attempts from unfamiliar geographic locations.
* Spikes in failed authentication events across the organization.

---

# Mitigation Strategies

Organizations can significantly reduce the risk of password spraying by implementing:

* Multi-factor authentication (MFA).
* Strong password policies.
* Password blocklists that prevent common or compromised passwords.
* Smart account lockout and rate-limiting mechanisms.
* Detection rules for distributed authentication failures.
* Centralized logging and continuous authentication monitoring.
* User security awareness training.

---

# Key Takeaways

* Password spraying differs from brute force by distributing authentication attempts across many accounts instead of repeatedly targeting one user.
* The attack primarily exploits weak password hygiene rather than software vulnerabilities.
* Modern defenses such as MFA, strong password policies, and centralized authentication monitoring are highly effective mitigations.
* Detecting coordinated authentication failures across multiple accounts is an important defensive capability for enterprise environments.
# Windows Domain Password Policy

## Overview

A password policy is a collection of security settings that define how user passwords are created, managed, and protected within a Windows domain. Properly configured password policies reduce the likelihood of unauthorized access and help defend against password guessing attacks.

Understanding an organization's password policy is an important part of security assessments, compliance audits, and Active Directory hardening.

---

# Objectives of a Password Policy

A well-designed password policy helps to:

* Enforce strong passwords.
* Prevent the use of weak or easily guessed credentials.
* Limit repeated authentication failures.
* Reduce the risk of credential-based attacks.
* Support organizational security and compliance requirements.

---

# Common Password Policy Settings

| Setting                           | Description                                                                                     |
| --------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Minimum Password Length**       | Specifies the minimum number of characters required for a password.                             |
| **Password Complexity**           | Requires combinations of uppercase, lowercase, numbers, and special characters.                 |
| **Password History**              | Prevents users from reusing recently used passwords.                                            |
| **Maximum Password Age**          | Defines how long a password remains valid before it must be changed.                            |
| **Minimum Password Age**          | Prevents users from immediately changing passwords repeatedly to bypass password history.       |
| **Account Lockout Threshold**     | Number of failed sign-in attempts before an account is temporarily locked.                      |
| **Account Lockout Duration**      | Length of time an account remains locked before automatic unlock or administrator intervention. |
| **Reset Account Lockout Counter** | Time after which failed login attempts are reset.                                               |

---

# Typical Enterprise Password Policy

Many organizations implement policies similar to:

| Setting                   | Example               |
| ------------------------- | --------------------- |
| Minimum Password Length   | 12–14 characters      |
| Password Complexity       | Enabled               |
| Password History          | 24 previous passwords |
| Maximum Password Age      | 90 days               |
| Account Lockout Threshold | 5–10 failed attempts  |
| Account Lockout Duration  | 15–30 minutes         |

Actual values vary depending on organizational security requirements.

---

# Administrative Audit Methods

Administrators can review password policies using Windows administrative tools such as:

* Active Directory Users and Computers
* Group Policy Management Console (GPMC)
* Local Security Policy
* Windows PowerShell cmdlets for Active Directory
* Microsoft management consoles and enterprise management platforms

These tools allow administrators to verify that password policies align with organizational security standards.

---

# Security Best Practices

Organizations should consider implementing:

* Passwords of at least 12–14 characters.
* Password blocklists to prevent common passwords.
* Multi-factor authentication (MFA).
* Smart account lockout protection.
* Monitoring for abnormal authentication activity.
* Regular password policy reviews.
* User security awareness training.

---

# Why Password Policies Matter

Weak password policies increase the likelihood of:

* Unauthorized account access.
* Credential stuffing attacks.
* Password guessing attacks.
* Password spraying attacks.
* Increased risk of domain compromise.

Strong password policies, combined with MFA and continuous monitoring, provide an important layer of defense against credential-based threats.

---

# Key Takeaways

* Password policies define how passwords are created, managed, and protected.
* Strong password policies significantly reduce credential-related security risks.
* Regular auditing helps ensure policies remain effective and aligned with organizational requirements.
* Password policies should be combined with MFA, monitoring, and user awareness to provide comprehensive protection.
