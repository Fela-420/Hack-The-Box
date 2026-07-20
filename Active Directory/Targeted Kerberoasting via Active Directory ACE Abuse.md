# Targeted Kerberoasting via Active Directory ACE Abuse

## Objective

Understand which Active Directory Access Control Entry (ACE) permissions can be abused to perform a targeted Kerberoasting attack.

---

# Core Concept

In Active Directory, permissions over objects are controlled using **Access Control Lists (ACLs)**.

An ACL contains **Access Control Entries (ACEs)** that define:

- Who has access to an object
- What permissions they have
- What actions they can perform

Certain ACE permissions can allow an attacker to modify user objects and perform attacks such as Kerberoasting.

---

# Targeted Kerberoasting Logic

## What is Kerberoasting?

Kerberoasting is an attack where an attacker:

1. Identifies a user account with a Service Principal Name (SPN)
2. Requests a Kerberos service ticket (TGS)
3. Extracts the ticket hash
4. Cracks the password offline

Normally, only accounts configured with SPNs can be Kerberoasted.

---

# The Problem

What if a target user does not already have an SPN?

An attacker with the correct Active Directory permissions may be able to modify the user account and assign an SPN.

Once the SPN is added:
