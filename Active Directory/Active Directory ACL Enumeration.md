# Active Directory ACL Enumeration - Simple Guide

> **Educational Purpose:** This guide explains how security professionals identify misconfigured Active Directory permissions and understand privilege escalation paths during authorized security assessments.

---

# 🎯 What This Lesson Teaches

ACL Enumeration is the process of analyzing **Access Control Lists (ACLs)** in Active Directory to discover:

- Who has permissions over users, groups, and computers
- Dangerous delegated rights
- Permission chains that can lead to privilege escalation
- Paths from a low-privileged user to Domain Admin

The goal is to understand:

> "What can my current user control, and where can those permissions lead?"

---

# 📖 Understanding Active Directory ACLs

Every Active Directory object has permissions attached to it.

Examples:

- Users
- Groups
- Computers
- Organizational Units (OUs)

These permissions define who can:

- Read objects
- Modify attributes
- Change passwords
- Add members to groups
- Replicate domain data

A small permission mistake can create a path to full domain compromise.

---

# 🚨 Example Attack Path

```text
Starting User

wley
(Cracked Password)

        |
        | ForceChangePassword
        ↓

damundsen

        |
        | GenericWrite
        ↓

Help Desk Level 1
(Group)

        |
        | Nested Membership
        ↓

Information Technology
(Group)

        |
        | GenericAll
        ↓

adunn

        |
        | DCSync Rights
        ↓

DOMAIN ADMIN
