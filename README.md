# CyberEd Portal — HackCamp 2026 Penetration Testing Exam

**Group 362**

This repository contains the complete writeups for the HackCamp 2026 Penetration Testing exam, covering three independent challenges across Linux privilege escalation, web application security, and Active Directory exploitation. Each writeup documents the full methodology — reconnaissance, exploitation, root cause analysis, impact, and remediation — for the corresponding target.

---

## Overview

| # | Task | Category | Vulnerability | Difficulty | Writeup |
|---|---|---|---|---|---|
| 1 | **Linux** | Web RCE → Privilege Escalation | Struts2 OGNL Injection (CVE-2017-5638) → sudo OpenVPN Abuse | Medium | [Task-1-Linux/README.md](Task-1-Linux/README.md) |
| 2 | **File Read** | Web Application Security | Path Traversal / Arbitrary File Read | Low | [Task-2-File-Read/README.md](Task-2-File-Read/README.md) |
| 3 | **Windows / AD** | Active Directory Exploitation | Zerologon (CVE-2020-1472) | Medium | [Task-3-Windows/README.md](Task-3-Windows/README.md) |

---

## Repository Structure

```
CyberEd-HackCamp-2026-PT-Exam/
├── Task-1-Linux/
│   └── README.md      # Struts2 RCE → sudo OpenVPN privilege escalation to root
├── Task-2-File-Read/
│   └── README.md      # Path traversal → arbitrary file read
├── Task-3-Windows/
│   └── README.md      # Zerologon → LDAP-based flag retrieval
└── README.md           # This file
```

---

## Task Summaries

### Task 1 — Linux
Exploited a critical Struts2 OGNL injection vulnerability (**CVE-2017-5638**) to obtain remote code execution as a low-privileged service account, then abused a misconfigured passwordless `sudo` rule on OpenVPN to escalate to `root`.

### Task 2 — File Read
Identified an unsanitized `file` parameter in an image-loading endpoint and exploited it via directory traversal to achieve arbitrary file read, retrieving the flag from `/etc/passwd`.

### Task 3 — Windows / Active Directory
Compromised a Domain Controller's machine account via **Zerologon (CVE-2020-1472)**, then used the resulting credentials to authenticate over NTLM and query Active Directory via LDAP, recovering the flag from a service account's `description` attribute.

---

## Methodology

Each writeup in this repository follows a consistent structure:

1. **Reconnaissance** — service and technology enumeration
2. **Vulnerability Identification** — how the flaw was discovered and confirmed
3. **Exploitation** — step-by-step technical walkthrough with commands and payloads
4. **Root Cause Analysis** — the underlying weakness that enabled the attack
5. **Impact Assessment** — real-world consequences of the vulnerability
6. **Remediation** — actionable recommendations to fix the issue
7. **Lessons Learned** — key takeaways from the engagement

---

## Disclaimer

These writeups were produced as part of an authorized educational penetration testing exam (HackCamp 2026, CyberEd Portal). All techniques were performed exclusively against designated lab targets provided for the exercise. Flags and target identifiers have been redacted or replaced with placeholders where required by the exam's disclosure policy.
