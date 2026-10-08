# Documentation and Reporting - Complete Module Notes

## 1. Introduction to Documentation and Reporting

### Why Documentation Matters
- Technical skills alone are not enough; soft skills back them up.
- Documentation is critical for policies, procedures, technical docs, pentest reports, and client deliverables.
- Good notes and evidence help you:
  - Recover from mistakes or failures.
  - Defend your actions.
  - Avoid blame.
  - Keep client trust.
  - Progress faster in your career.

### Pentest as a "Snapshot in Time"
- A pentest is a snapshot of the target network's security status at a specific time.
- Reports should include an overview section with:
  - Type of work performed.
  - Who performed it.
  - Source IP addresses used.
  - Special considerations (e.g., remote VPN, internal host).
  - Testing dates.
  - A disclaimer (e.g., "This report represents a snapshot in time...").

### Real-World Scenarios
1. **Exploding VM:** VM crashed, but daily backups and detailed notes saved the project.
2. **Ping of Death:** Client blamed testing for downing servers; logs and scope files proved otherwise.
3. **Slow as Molasses:** Hostile admin blamed scans for network slowdown; logs proved best practices were followed, and the real cause was debug mode on network devices.

---

## 2. Notetaking & Organization

### Suggested Notetaking Structure
- **Attack Path:** Outline of compromise path with screenshots/output.
- **Credentials:** Central place for compromised credentials/secrets.
- **Findings:** Subfolder per finding with narrative and evidence.
- **Vulnerability Scan Research:** Notes on scans tried.
- **Service Enumeration Research:** Services checked, failed attempts.
- **Web Application Research:** Interesting apps, subdomains, default creds tried.
- **AD Enumeration Research:** Step-by-step AD enumeration.
- **OSINT:** Open-source intelligence collected.
- **Administrative Information:** Contacts, PMs, POCs, RoE flags, to-do list.
- **Scoping Information:** In-scope IPs, URLs, provided credentials.
- **Activity Log:** High-level tracking of everything done.
- **Payload Log:** Payloads used, file hashes, upload locations, cleanup status.

### Notetaking Tools
- **Options:** Obsidian, OneNote, CherryTree, Notion, Outline, VS Code, Notepad++, Sublime Text, Evernote, GitBook, Cryptpad, Standard Notes.
- **Key Consideration:** Local vs Cloud storage. Check company policy for client data.
- **Recommended:** Obsidian (local), Outline (cloud/self-hosted). Both support Markdown.

### Logging
- Log all scanning and attack attempts. Keep raw tool output.
- **Tmux Logging:**
  - Saves everything typed into a Tmux pane to a log file.
  - Setup: Clone TPM, create `.tmux.conf`, add plugins (`tpm`, `tmux-sensible`, `tmux-logging`), source config, install plugins (`prefix + Shift + I`).
  - Start logging: `prefix + Shift + P`.
  - Retroactive logging: `prefix + Alt + Shift + P`.
  - Screen capture pane: `prefix + Alt + P`.
  - Set high history limit: `set -g history-limit 50000`.
- **Useful Plugins:** `tmux-sessionist`, `tmux-pain-control`, `tmux-resurrect`.

### Tracking Artifacts and Changes
- **Payload Log:** When used, which host, file path, cleanup status, file hash.
- **Account Creation/System Modifications:** Host IP, timestamp, description, location, application/service, account name/password.
- **Important:** Get written approval before risky changes.

### Evidence and Folder Structure
- **Evidence:** Client wants clear issues and evidence for validation. Prefer terminal output over screenshots.
- **Suggested Folder Structure:**
```
ACME-IPT/
├── Admin
├── Deliverables
├── Evidence/
│   ├── Findings/
│   ├── Scans/
│   │   ├── Vuln/
│   │   ├── Service/
│   │   ├── Web/
│   │   └── AD Enumeration/
│   ├── Notes/
│   ├── OSINT/
│   ├── Wireless/
│   ├── Logging output/
│   └── Misc Files/
└── Retest/
```

### Formatting and Redaction
- Redact credentials and PII.
- Do NOT use blur or pixelation (can be reversed).
- Use solid black bars or shapes. Edit the image directly, not just in Word.
- Annotate with arrows/boxes. Crop to relevant info.
- Include browser address bar or host info.
- **Terminal Output:** Redact passwords/hashes. Use placeholders like `<REDACTED>`. Highlight command and key output. Never alter output; use `<SNIP>` for cuts. Strip formatting before pasting into Word.
- **What NOT to Archive:** Don't bring down hosts, change passwords, make difficult-to-reverse changes, or collect unredacted PII/criminal info.

---

## 3. Types of Reports

### Assessment Types
- **Vulnerability Assessment:** Automated scan, no exploitation. Focus on themes, number of vulnerabilities, severity levels.
- **Penetration Testing:** Goes beyond scans. Perspectives: Black box, Grey box, White box. Styles: Non-evasive, Hybrid evasive, Evasive, Adversary simulation.
- **Inter-Disciplinary Assessments:** Purple Team, Cloud Focused, Comprehensive IoT, Web Application, Hardware.

### Report Types
- **Draft Report:** Client reviews before final version. Client may add management responses, adjust language.
- **Final Report:** Issued after client review and modifications. Some auditors do not accept draft reports.
- **Post-Remediation Report:** Retest only original findings and affected hosts. Put a time limit on retesting. Stay ethical; offer practical paths forward.
- **Attestation Report:** For vendors/customers needing proof a pentest was done. No full technical details. Usually 1-2 pages.

### Other Deliverables
- **Slide Deck:** Match language to audience. Use stories, do not fear-monger.
- **Spreadsheet of Findings:** Tabular version of findings. Easy sorting, data manipulation.
- **Vulnerability Notifications:** Used when a critical flaw is found. Minimum trigger: directly exploitable, internet-exposed, unauthenticated RCE, sensitive data exposure, weak/default credentials.

---

## 4. Components of a Report

### Prioritizing Efforts
- Filter out noise. Focus on high-impact issues first (RCE, sensitive data exposure, domain compromise).
- Do not waste most of your time validating minor informational findings. Group low-impact issues into categories.
- Lean on senior team members when stuck. Avoid rabbit holes and broken PoCs.

### Writing an Attack Chain
- **Purpose:** Show the full path from foothold to compromise. Connect findings together.
- **Include:** Summary of the chain, step-by-step walkthrough, command output, screenshots.
- **Example flow:** Responder -> Crack hash -> BloodHound -> Kerberoast -> Crack TGS -> Access SQL01 -> Dump LSA -> Access MS01 -> Steal TGT -> Pass-the-ticket -> DCSync -> Domain compromise.

### Executive Summary
**Audience:** Non-technical executives, budget decision-makers. Short attention span.

**Do:**
- Be specific with numbers (e.g., "25 occurrences" not "several").
- Keep it short (1.5-2 pages max).
- Describe what you accessed in plain terms (e.g., HR documents, banking systems).
- Describe what needs to improve at a process level.
- Optionally set expectations for remediation effort.

**Do Not:**
- Name or recommend specific vendors.
- Use acronyms.
- Over-focus on minor issues.
- Use obscure words.
- Reference technical sections.

**Vocabulary Changes:**

| Technical | Non-Technical |
|---|---|
| VPN, SSH | secure remote administration |
| SSL/TLS | secure web browsing technology |
| Hash | output used to validate file integrity |
| Password spraying | trying one common password across many accounts |
| Password cracking | converting a protected password back to readable form |
| Buffer overflow | attack that ran commands on the target |
| OSINT | public data gathering |
| SQL injection/XSS | unsafe user input manipulating the app |

### Summary of Recommendations
- Place before technical findings.
- List short, medium, and long-term recommendations.
- Tie each recommendation to a finding.
- Help client build a remediation roadmap.

### Findings
One of the most important sections. Shows your work, gives technical teams evidence to reproduce and fix.

### Appendices
- **Static Appendices:** Scope, Methodology, Severity Ratings, Biographies.
- **Dynamic Appendices:** Exploitation Attempts and Payloads, Compromised Credentials, Configuration Changes, Additional Affected Scope, Information Gathering, Domain Password Analysis.

### Report Type Differences
- **Internal Pentest:** Attack chain, internal compromise details.
- **External Pentest:** More OSINT, external footprint, externally exposed services.
- **Web App Security Assessment:** Focus on Executive Summary and Findings, OWASP Top 10.
- **Physical/Red Team/Social Engineering:** More narrative format.
- **Best Practice:** Create templates for each assessment type.

---

## 5. How to Write Up a Finding

### What Every Finding Needs
- Description of the finding and platforms affected.
- Impact if unresolved.
- Affected systems, networks, environments, or applications.
- Recommendation to fix it.
- Reference links.
- Steps to reproduce and evidence collected.
- Optional: CVE, OWASP/MITRE IDs, CVSS score, ease of exploitation, probability of attack.

### Showing Reproduction Steps Properly
- Do not assume the reader knows the tools.
- Break each step into its own figure.
- If setup is needed, show the full config.
- Write a narrative between figures.
- Offer alternative tools if they exist.
- **Make evidence defensible:** Do not just show a login prompt. Show the actual clear-text data in transit. Include URL, ifconfig, or ipconfig. Turn off bookmarks bar and unprofessional extensions. Redact credentials where possible.

### Effective Remediation Recommendations
**Bad:** Reconfigure your registry settings to harden against X.

**Good:** To fully remediate this finding, update the following registry hives with the specified values. Note: registry changes should be tested in a small group first. [list full path] Change value X to value Y.

**Bad:** Implement [expensive commercial tool] to address this finding.

**Good:** The vendor has published a workaround as an interim solution. A link is provided below. Commercial tools also exist but may be cost-prohibitive.

### Selecting Quality References
- Vendor-agnostic when possible.
- Thorough walkthroughs or mitigations.
- Not behind a paywall.
- Quick to the point.
- Clean websites, no ads or crypto miners.
- Ideally your own source material if possible.

### Poorly Written Finding
Sloppy formatting, missing CVSS score, description does not explain root cause, impact is vague, remediation is not actionable.

If the reader thinks "why do I care?" or "what do I do?", the finding is bad.

---

## 6. Reporting Tips and Tricks

### Build the Report As You Go
- Start organizing from day one.
- Fill in templated parts during long scans.
- Write Attack Chain and findings while testing.
- Capture evidence as you go.

### Templates
- Have a blank template for every assessment type.
- Never modify a previous client's report (risk of leaving old data).
- Use macros and placeholders in Word. Save templates as `.dotm`.

### MS Word Tips & Tricks
- Use Word for Windows, not Mac.
- Use Font Styles and Table Styles for global changes.
- Use built-in Captions for auto-numbering.
- Use Page Numbers, Table of Contents, List of Figures/Tables, Bookmarks.
- Set Language Settings for code/terminal font to ignore spelling/grammar.
- **Useful Hotkeys:** F4 (repeat), Ctrl+A then F9 (update fields), Ctrl+Alt+S (split window), Shift+F5 (jump to last revision).

### Automation
- Use macros for pop-ups (client name, dates, scope).
- Combine templates and remove unneeded sections via bookmarks.
- Save as `.dotm`.

### Reporting Tools / Findings Database
- Build a database of sanitized findings.
- **Tools:** Free (Ghostwriter, Dradis, VECTR, WriteHat), Paid (AttackForge, PlexTrac, Rootshell Prism).

### Misc Tips / Tricks
- Tell a story with the report.
- Write as you go.
- Stay organized, chronological notes.
- Show enough evidence, not too much.
- Annotate screenshots with arrows/boxes.
- Redact sensitive data with solid shapes, not blur.
- Redact unprofessional tool output.
- Check Hashcat output for crude words.
- Check grammar, spelling, formatting.
- Use raw command output when possible.
- Use solid console background, professional theme.
- Keep hostname/username professional.
- Establish QA process (at least one reviewer, ideally two).
- Establish a style guide.
- Use autosave and backups.
- Script and automate wherever possible.

### Client Communication
- **Start Notification Email:** Tester name, type/scope, source IP, testing dates, contacts.
- **Daily Stop Notification:** End of testing, high-level summary, report delivery expectations.
- **Other Communication:** Keep open dialogue. Ask about adding new scope. Stop and notify on critical findings. Be upfront about host outages. Give heads-up on Domain Admin.

### Presenting Your Report - Final Product
- **QA Process:** Sloppy report = questions about your work. At least one QA reviewer. Use QA checklist. Check grammar/spelling. Be careful with cloud grammar tools. Track changes. Draft then Final.
- **Report Review Meeting:** Give client a week to review. Offer a call to walk through findings. Use feedback to improve. Change DRAFT to FINAL after acceptance. Archive all data per retention policy.

---

## 7. Beyond the Module

### Ways to Practice
- Use notetaking tools on labs and boxes.
- Treat boxes like real engagements. Write professional reports.
- Read public pentest reports.
- Write up 1-2 findings per box.
- Start a personal blog.
- Share and get feedback.
- Use GitHub Pages and Git to document personal projects.

### Next Steps
- Complete Penetration Testing Process module.
- Complete Attacking Enterprise Networks module (capstone).
- Treat it like a real engagement and practice documentation and reporting.

### Bottom Line
- Practice documentation and reporting constantly.
- Use labs, boxes, blogs, and reports to refine your style.
- Better writing makes you more effective and more valuable.
- It's a skill you can only improve with deliberate practice.
