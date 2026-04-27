---
author: NmToan
date: 2025-12-29T04:59:04.866Z
lastmod: 2026-04-27T00:00:00.000Z
title: HELP ME
slug:
featured: false
draft: false
tags:
  - Forensics
description: Write-up for a DNS exfiltration case from `incident-v3.pcapng`
---

# Incident Write-up: `incident-v3.pcapng`

**Flag:** `putcCTF{3xf1ltr4ted}`

## Overview

The goal was to analyze `incident-v3.pcapng`, determine how data was exfiltrated, and recover the final flag. After reviewing the capture, I found that the victim host was leaking data over DNS to `backup.site.lan`.

The payload was not stored in cleartext. Instead, it was:

1. split into blocks,
2. XORed with a changing key,
3. hex-encoded,
4. embedded into DNS query names.

Once the full DNS stream was reconstructed, the recovered file contained base64 strings. Decoding them revealed a shifted token, and shifting each alphanumeric character back by one produced the final flag.

## Main Artifacts

- `incident-v3.pcapng`
- `extracted_http/transmiter.py`
- `dns_queries.txt`
- `reconstruct_dns_exfil.py`
- `decode_strings.py`
- `recovered.bin`

## Analysis Plan

The solve path breaks into four main stages:

1. perform a quick scan of the capture,
2. extract and study the transmitter script,
3. reconstruct the leaked payload from DNS queries,
4. decode the final layers to recover the flag.

## 1. Quick Scan of the PCAP

I first looked for obvious flag-shaped strings:

```sh
strings incident-v3.pcapng | grep -o -E 'putcCTF\{[^}]+\}' || true
```

This returned repeated fake flags, so a deeper inspection was required.

Next, I checked which protocols were present:

```sh
tshark -r incident-v3.pcapng -T fields -e frame.protocols | sed 's/\./\n/g' | sort | uniq -c | sort -nr
```

DNS stood out as a likely candidate for covert data transfer.

## 2. Extract HTTP Objects

I exported the HTTP objects from the capture:

```sh
mkdir -p extracted_http
tshark -r incident-v3.pcapng --export-objects http,extracted_http
```

The most important file was:

- `extracted_http/transmiter.py`

This script explains exactly how the data was being sent.

## 3. Understand the Exfiltration Logic

From `transmiter.py`, the transmission flow is:

1. read the input file,
2. compute `id = md5(content)`,
3. split the file into 32-byte blocks,
4. XOR each block with `KEY`,
5. hexlify the result,
6. split the hex string into DNS-safe labels,
7. send queries of the form:

```text
{id}.{sequence}.{hexpart1}.{hexpart2}....backup.site.lan
```

The key detail is that the XOR key changes after every block:

```text
KEY = md5(data + KEY)
```

The sender also marks the end of transmission with:

```text
id.A.backup.site.lan
```

This means reconstruction must preserve the original block order and key evolution exactly.

## 4. Extract the Suspicious DNS Queries

Once the target domain was clear, I dumped all client-side queries to `backup.site.lan`:

```sh
tshark -r incident-v3.pcapng -Y 'dns.qry.name contains "backup.site.lan" && dns.flags.response == 0' -T fields -e dns.qry.name > dns_queries.txt
```

This produced the raw input needed for reconstruction.

## 5. Rebuild the Exfiltrated File

I wrote `reconstruct_dns_exfil.py` to automate the rebuild process. The script:

1. parses each line from `dns_queries.txt`,
2. extracts the sequence number,
3. joins the hex fragments,
4. unhexlifies them,
5. XOR-decrypts the data with the correct key,
6. updates the key with `md5(data + KEY)`,
7. concatenates all blocks into the final file.

Run:

```sh
python3 reconstruct_dns_exfil.py
```

Output:

- `recovered.bin`

The recovered file size was about `796 bytes`.

## 6. Inspect the Recovered Data

I then checked the printable strings:

```sh
strings recovered.bin
```

The file contained multiple base64-looking lines, so I decoded them with:

```sh
python3 decode_strings.py
```

This revealed a suspicious token:

```text
qvudDUG{4yg2mus5ufe}
```

It clearly resembled a shifted flag.

## 7. Decode the Final Token

Each alphanumeric character in the token had been shifted forward by one. Shifting them back gives the real flag:

```sh
python3 - <<'PY'
s = "qvudDUG{4yg2mus5ufe}"
out = []
for c in s:
    if 'a' <= c <= 'z' or 'A' <= c <= 'Z' or '0' <= c <= '9':
        out.append(chr(ord(c) - 1))
    else:
        out.append(c)
print(''.join(out))
PY
```

Result:

```text
putcCTF{3xf1ltr4ted}
```

## Conclusion

The full solve chain was:

1. identify DNS as the suspicious channel,
2. extract `transmiter.py` to understand the exfiltration format,
3. dump all DNS queries to `backup.site.lan`,
4. rebuild the original payload by reordering blocks and mirroring the XOR key schedule,
5. decode the embedded base64 strings,
6. shift the final token back by one character.

Final flag:

```text
putcCTF{3xf1ltr4ted}
```
