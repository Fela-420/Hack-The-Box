# Domain Trusts Primer - Complete Guide

## 🎯 What Are Domain Trusts?
A **trust** is a relationship between two domains that allows users in one domain to access resources in another domain.

> **Analogy:** You have a key to your house (*your domain*). Your neighbor gives you a key to their house (*trust*). Now you can access both houses.

---

## 📋 Types of Trusts

| Trust Type | Description | Example |
| :--- | :--- | :--- |
| **Parent-Child** | Two domains in same forest, child trusts parent | `INLANEFREIGHT.LOCAL` $\rightarrow$ `LOGISTICS.INLANEFREIGHT.LOCAL` |
| **Tree-Root** | Between forest root and new tree root | New domain in same forest |
| **Cross-Link** | Between child domains (speeds up auth) | Between two child domains |
| **External** | Between two separate domains (non-transitive) | `INLANEFREIGHT.LOCAL` $\rightarrow$ `EXTERNAL.LOCAL` |
| **Forest** | Between two forest roots (transitive) | `INLANEFREIGHT.LOCAL` $\rightarrow$ `FREIGHTLOGISTICS.LOCAL` |
| **ESAE** | Bastion forest for AD management | Admin forest |

---

## 🔑 Key Trust Concepts

### Transitive vs Non-Transitive

| Transitive | Non-Transitive |
| :--- | :--- |
| Trust extends to child domains | Trust is only between the two domains |
| If A trusts B, and B trusts C $\rightarrow$ A trusts C | If A trusts B, and B trusts C $\rightarrow$ A does **NOT** trust C |
| Forest, parent-child, cross-link | External trusts |

### Direction

| Direction | Meaning |
| :--- | :--- |
| **One-Way** | Domain A trusts Domain B (users in B can access A, not vice versa) |
| **Bidirectional** | Both domains trust each other (users in either can access the other) |

---

## 🔍 How to Enumerate Trusts

### Method 1: ActiveDirectory Module (PowerShell)

    Import-Module ActiveDirectory
    Get-ADTrust -Filter *

**What to look for:**
* **Name** $\rightarrow$ Domain name
* **Direction** $\rightarrow$ BiDirectional / OneWay
* **IntraForest** $\rightarrow$ `True` = child domain
* **ForestTransitive** $\rightarrow$ `True` = forest/external trust
* **TrustType** $\rightarrow$ Uplevel / Downlevel

### Method 2: PowerView

    Import-Module .\PowerView.ps1

    # List all trusts
    Get-DomainTrust

    # Map all trusts (including from other domains)
    Get-DomainTrustMapping

### Method 3: Netdom (CMD)

    # List trusts
    netdom query /domain:inlanefreight.local trust

    # List domain controllers
    netdom query /domain:inlanefreight.local dc

    # List workstations/servers
    netdom query /domain:inlanefreight.local workstation

### Method 4: BloodHound
1. Run `Map Domain Trusts` query
2. Visualizes trust relationships

---

## 📊 Sample Trust Output

From `Get-DomainTrust`:

    SourceName      : INLANEFREIGHT.LOCAL
    TargetName      : LOGISTICS.INLANEFREIGHT.LOCAL
    TrustType       : WINDOWS_ACTIVE_DIRECTORY
    TrustAttributes : WITHIN_FOREST        ← Child domain
    TrustDirection  : Bidirectional        ← Both ways

    SourceName      : INLANEFREIGHT.LOCAL
    TargetName      : FREIGHTLOGISTICS.LOCAL
    TrustType       : WINDOWS_ACTIVE_DIRECTORY
    TrustAttributes : FOREST_TRANSITIVE    ← Forest trust
    TrustDirection  : Bidirectional        ← Both ways

---

## 🧠 Key Takeaways

* `INLANEFREIGHT.LOCAL` has two trusts:
  * `LOGISTICS.INLANEFREIGHT.LOCAL` (child domain, bidirectional)
  * `FREIGHTLOGISTICS.LOCAL` (forest trust, bidirectional)
* **Trusts can be exploited** – If we compromise one domain, we may be able to access the other.
* **Always check scope** – Make sure trusts are in scope before attacking.
