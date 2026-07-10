# LLMNR NBT-NS Poisoning with Responder

**Module:** Network Attacks
**Attack Type:** LLMNR/NBT-NS Poisoning
**Objective:** Capture NetNTLMv2 authentication hashes from systems that fall back to LLMNR or NBT-NS name resolution when DNS cannot resolve a hostname.

---

# Overview

## What is LLMNR/NBT-NS Poisoning?

LLMNR (Link-Local Multicast Name Resolution) and NBT-NS (NetBIOS Name Service) are Windows name resolution protocols used when DNS cannot resolve a hostname.

When a computer cannot locate a requested host through DNS, it broadcasts a request across the local network asking if any device knows that hostname.

If an attacker-controlled system responds first, the victim may attempt to authenticate to it automatically using NTLM authentication. This allows the attacker to capture the user's **NetNTLMv2 hash**.

> **Note:** A NetNTLMv2 hash is not the user's plaintext password. It is a challenge-response value that can sometimes be cracked offline or used in certain relay attacks, depending on the environment.

---

# Attack Flow

```
Victim requests \\FileServer
          │
          ▼
DNS cannot resolve the hostname
          │
          ▼
Victim broadcasts an LLMNR/NBT-NS request
          │
          ▼
Responder replies pretending to be FileServer
          │
          ▼
Victim initiates NTLM authentication
          │
          ▼
Responder captures the NetNTLMv2 hash
          │
          ▼
Hash is saved for offline analysis
```

---

# Tools Used

| Tool            | Purpose                                                                     |
| --------------- | --------------------------------------------------------------------------- |
| Responder       | Listens for LLMNR/NBT-NS requests and captures NTLM authentication attempts |
| Hashcat         | Performs offline password cracking against captured hashes                  |
| John the Ripper | Alternative password cracking tool                                          |
| rockyou.txt     | Common password wordlist used during cracking                               |

---

# Lab Objective

Capture a NetNTLMv2 hash from a Windows host that attempts to resolve an unknown hostname using LLMNR/NBT-NS.

---

# Step 1 – Connect to the Lab Machine

Log in to the provided attack machine.

**Purpose**

Gain access to the system where Responder will be executed.

---

# Step 2 – Start Responder

Launch Responder on the correct network interface.

**Purpose**

Responder begins listening for:

* LLMNR requests
* NBT-NS requests
* WPAD requests (if enabled)

When a Windows host attempts name resolution, Responder replies and requests NTLM authentication.

---

# Step 3 – Wait for Authentication Requests

Allow Responder to monitor the network.

During this period:

* Windows clients broadcast unresolved hostname requests.
* Responder impersonates the requested host.
* Authentication attempts are captured automatically.

---

# Step 4 – View Captured Hashes

After stopping Responder, review the generated log files.

A captured NetNTLMv2 hash typically appears as:

```
username::DOMAIN:challenge:NTLMv2-response:blob
```

Example:

```
backupagent::INLANEFREIGHT:challenge:response:blob
```

---

# Understanding the Hash Format

| Field           | Description                                   |
| --------------- | --------------------------------------------- |
| Username        | Account attempting authentication             |
| Domain          | Windows domain                                |
| Challenge       | Server-generated challenge                    |
| NTLMv2 Response | Authentication response captured by Responder |
| Blob            | Additional authentication metadata            |

---

# Offline Password Cracking

Captured hashes can be tested against password wordlists using offline cracking tools.

Common tools include:

* Hashcat
* John the Ripper

The objective is to determine whether the user's password is weak enough to be recovered.

---

# Expected Result

Successful execution should produce one or more captured NetNTLMv2 hashes similar to:

```
username::DOMAIN:challenge:response:blob
```

These hashes can then be analysed offline.

---

# Common Issues

## No Hashes Captured

Possible causes:

* No Windows clients attempted name resolution.
* DNS resolved the hostname successfully.
* LLMNR/NBT-NS is disabled.
* Network segmentation prevents broadcast traffic.

---

## Password Cannot Be Recovered

Possible causes:

* Password is not present in the selected wordlist.
* Strong password policy.
* Additional cracking rules or larger wordlists may be required.

---

# Security Impact

If successful, an attacker may obtain:

* Usernames
* NetNTLMv2 authentication hashes
* Domain information

Recovered credentials may allow further movement within the environment if weak passwords are in use.

---

# Mitigation

Recommended defensive measures include:

* Disable LLMNR.
* Disable NetBIOS over TCP/IP where possible.
* Use strong, unique passwords.
* Enable SMB signing.
* Restrict NTLM usage where feasible.
* Monitor networks for LLMNR/NBT-NS poisoning activity.
* Deploy network intrusion detection capable of identifying rogue name-resolution responses.

---

# Key Takeaways

* LLMNR and NBT-NS are fallback name resolution protocols.
* Responder impersonates requested hosts on the local network.
* Windows clients may automatically attempt NTLM authentication.
* Captured NetNTLMv2 hashes can be analysed offline.
* Disabling legacy name resolution protocols significantly reduces exposure to this attack.
