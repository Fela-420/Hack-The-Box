# Understanding Snaffler & Finding Credentials in Active Directory

Snaffler is an Active Directory enumeration tool that searches accessible network shares for files that may contain sensitive information. It helps security professionals identify exposed credentials, configuration files, and other sensitive data that should not be accessible.

> **Note:** Snaffler is an enumeration tool, not an exploitation tool. It only accesses files that the current user is already authorized to read.

---

# What Can Snaffler Find?

Snaffler searches for files that commonly contain sensitive information, including:

- 🔑 Passwords
- 🔐 SSH Keys
- 🔒 Private Keys
- 🗄️ Database Connection Strings
- ⚙️ Configuration Files
- 🔑 API Keys
- 📄 Backup Files
- 📜 Certificates

---

# How Snaffler Works

## Step 1 – Enumerate Active Directory

Snaffler queries Active Directory to discover computers within the domain.

```
Domain
│
├── DC01
├── FILE01
├── WEB01
└── SQL01
```

---

## Step 2 – Enumerate SMB Shares

For each computer, Snaffler discovers available network shares.

Example:

```
\\FILE01\IT
\\FILE01\Finance
\\WEB01\Deploy
\\SQL01\Backups
```

---

## Step 3 – Check Permissions

Snaffler attempts to access only the files the current user has permission to read.

```
Current User
      │
      ├── Read ✔
      ├── Read ✔
      └── Access Denied ✘
```

---

## Step 4 – Hunt for Sensitive Files

Snaffler searches based on:

### Interesting Extensions

```
.config
.xml
.json
.ini
.env
.ppk
.key
.pem
.pfx
.kdb
.kdbx
```

### Interesting File Names

```
web.config
appsettings.json
passwords.txt
credentials.csv
connectionstrings.txt
backup.sql
```

### Sensitive Keywords

```
password
connectionString
User Id
API_KEY
SECRET
token
```

---

# Snaffler Color Coding

| Color | Meaning |
|--------|----------|
| 🔴 Red | High-value file (likely contains credentials) |
| 🟡 Yellow | Interesting file worth reviewing |
| 🟢 Green | Normal share or directory |
| ⚫ Black | Ignored system files/shares |

---

# Example Output

```
2022-03-31 12:17:19 [Share] {Green}
\\FILE01\IT

2022-03-31 12:17:19 [Share] {Green}
\\FILE01\IT\Web

2022-03-31 12:17:19 [File] {Red}
\\FILE01\IT\Web\web.config
```

A **Red** finding indicates a file that should be manually reviewed.

---

# Example: web.config

A common target is **web.config**, which may contain database connection strings.

```xml
<connectionStrings>
    <add
        name="DefaultConnection"
        connectionString="Server=SQL01;
        Database=AppDB;
        User Id=svc_app;
        Password=P@ssw0rd123!;" />
</connectionStrings>
```

Potentially exposed information:

- Database Server
- Database Name
- Database Username
- Database Password

---

# Example: appsettings.json

Modern .NET applications commonly use:

```json
{
  "ConnectionStrings": {
    "DefaultConnection":
      "Server=SQL01;Database=AppDB;User Id=svc_app;Password=P@ssw0rd123!"
  }
}
```

---

# Example: .env File

```env
DB_USER=admin
DB_PASS=Password123!
API_KEY=xxxxxxxx
JWT_SECRET=xxxxxxxx
```

---

# Running Snaffler

Example command:

```powershell
Snaffler.exe -s -d inlanefreight.local -v data
```

---

# Reviewing a File

If Snaffler identifies an interesting file:

```
\\FILE01\IT\Web\web.config
```

View it from Windows:

```cmd
type "\\FILE01\IT\Web\web.config"
```

Review the file for exposed credentials or sensitive configuration.

---

# Why It Matters

Poorly secured file shares may expose:

- Database credentials
- Service account passwords
- API keys
- Internal infrastructure information
- Application secrets

If these credentials are valid, they may enable further access within the environment.

---

# Typical Assessment Workflow

```
Obtain Domain User
        │
        ▼
Run Snaffler
        │
        ▼
Enumerate SMB Shares
        │
        ▼
Locate Sensitive Files
        │
        ▼
Review Exposed Secrets
        │
        ▼
Assess Security Impact
```

---

# Defensive Recommendations

Organizations should:

- Avoid storing passwords in plaintext configuration files.
- Use Windows Integrated Authentication where possible.
- Store secrets in a secure secrets manager.
- Use Group Managed Service Accounts (gMSAs).
- Apply least-privilege permissions to SMB shares.
- Regularly audit file shares for sensitive information.

---

# Key Takeaways

- Snaffler is an **Active Directory enumeration tool**.
- It **does not bypass permissions**.
- It identifies files that may expose credentials or sensitive information.
- The real security issue is **insecure credential storage** or **overly permissive file share permissions**, not the use of Snaffler itself.

---

## References

- Active Directory Enumeration
- SMB Share Enumeration
- Windows File Share Security
- Secure Secret Management
- Principle of Least Privilege
