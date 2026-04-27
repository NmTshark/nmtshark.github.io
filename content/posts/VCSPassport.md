---
author: NmToan
date: 2025-12-27T04:59:04.866Z
lastmod: 2026-04-27T00:00:00.000Z
title: VCSPassport BlueTeam CTF Gimme_your_point!
slug: VCSPassport-Blue-Team-CTF-Gimme-your-point
featured: false
draft: false
tags:
  - Forensics
  - VCSPassport
  - Pcap
description: Write-up for the VCSPassport challenge `Gimme_your_point`, covering PCAP analysis, malware unpacking, and recovery of stolen Chrome data.
---

> **Topic:** Network forensics and unpacking a disguised Windows payload  
> **Goal:** Start from the PCAP, trace the SharePoint compromise, analyze the malware, recover the stolen Chrome data, and reconstruct the flag

```text
Flag format: VCSPassport{...}
```

## Analysis Plan

When reviewing the PCAP, I focused first on the TCP streams to understand the attack chain. Two lines of investigation stood out quickly:

1. a suspicious ASPX upload against SharePoint,
2. a downloaded payload that looked like a certificate but clearly was not one.

## 1. Identify the Initial Access

One of the early streams showed evidence that the attacker uploaded a malicious ASPX file to the SharePoint server. This is a classic sign of a SharePoint remote code execution path using an uploaded webshell or crafted page.

That strongly suggests:

- the SharePoint server was compromised,
- the attacker gained code execution,
- additional payloads were then pulled onto the victim system.

## 2. Find the Downloaded Payload

In the next important stream, I saw a file called `raw_package` being downloaded. At first glance it looked like a certificate because it started with:

```text
-----BEGIN CERTIFICATE-----
```

However, when I base64-decoded a portion of the payload, I got another layer of text that again began with `-----BEGIN CERTIFICATE-----`. After decoding again, the header changed to:

```text
MZ
```

That is the standard magic value for a Windows PE executable.

So `raw_package` is really a Windows binary hidden behind multiple layers of base64 to make it look harmless.

## 3. Export the HTTP Objects

I then filtered the HTTP traffic and exported the transferred objects:

```text
File -> Export Objects -> HTTP
```

The resulting files showed that:

- `raw_package` is the downloaded payload,
- `ToolPane(1).aspx` is the malicious ASPX file used during exploitation,
- another file named `upload` is also highly suspicious.

When I examined `upload` with `strings` and `cat`, I found strings such as:

- `chrome_health_result`
- `chrome_service_result`

This strongly suggested that the file contained data stolen from Chrome. It also looked structurally like an archive, so I suspected it was a ZIP file with the extension changed.

## 4. Confirm That `upload` Is a Password-Protected ZIP

I renamed `upload` to `upload.zip` and tried to extract it. That confirmed it was indeed a ZIP file, but it was password-protected.

At that point I had two clear leads:

1. `raw_package` is the malware,
2. `upload.zip` contains the exfiltrated Chrome data.

To open the ZIP, the next logical step was to inspect the malware and recover the password.

## 5. Unwrap the Layers of `raw_package`

As established earlier, `raw_package` is a PE file wrapped in two layers of base64. I decoded the first layer with Python:

```python
import base64

input_path = '/home/kali/Downloads/vcsctf/pcapagain/raw_package'
output_path = 'decoded_output.txt'

with open(input_path, 'r') as f:
    encoded_data = f.read().replace('\n', '').replace('\r', '').strip()

decoded_data = base64.b64decode(encoded_data)

with open(output_path, 'wb') as f:
    f.write(decoded_data)
```

Then I decoded the second layer:

```python
import base64
import re

input_path = '/home/kali/Downloads/vcsctf/pcapagain/decoded_output.txt'
output_path = 'malware.exe'

with open(input_path, 'r', encoding='utf-8', errors='ignore') as f:
    raw_content = f.read()

clean_b64 = re.sub(r'[^A-Za-z0-9+/]', '', raw_content)
missing_padding = len(clean_b64) % 4
if missing_padding:
    clean_b64 += '=' * (4 - missing_padding)

decoded_data = base64.b64decode(clean_b64)

with open(output_path, 'wb') as f:
    f.write(decoded_data)
```

This produced:

- `malware.exe`

`file malware.exe` identified it as:

```text
PE32+ executable for MS Windows 5.02 (console), x86-64, 18 sections
```

## 6. Recover the ZIP Password from the Malware

A normal `strings malware.exe` run did not reveal much, but because this is a Windows binary, I expected some strings to be stored in UTF-16LE. So I switched to:

```bash
strings -el malware.exe
```

That immediately exposed the following fragments:

```text
_p4$$w0d
_S3(ReT
sup3r
```

The obvious reconstruction is:

```text
sup3r_S3(ReT_p4$$w0d
```

Using that password, I extracted `upload.zip` successfully and obtained:

- `chrome_health_result`
- `chrome_service_result`

## 7. Recover the Stolen Chrome Data

Further inspection showed that `chrome_health_result` is an SQLite database. I opened it with `sqlite3` or DB Browser for SQLite and queried the `logins` table:

```sql
SELECT * FROM logins;
```

Among the returned records, the important one was:

```text
https://flag.win.here/|||SH4R3_y0R_P$$wD||...
```

This is the most likely flag-bearing record because:

- the URL literally contains `flag`,
- the value `SH4R3_y0R_P$$wD` has the exact structure expected for a challenge secret.

## Conclusion

The full attack chain was:

1. the attacker exploited SharePoint using a malicious ASPX file,
2. a Windows payload was downloaded in a heavily disguised form,
3. the malware stole Chrome-related data,
4. the stolen data was archived into a password-protected ZIP,
5. the ZIP password was exposed through UTF-16LE strings inside the malware,
6. extracting the ZIP and querying the Chrome database revealed the flag value.

Recovered flag:

```text
VCSPassport{SH4R3_y0R_P$$wD}
```
