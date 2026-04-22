---
author: NmToan
date: 2025-12-29T04:59:04.866Z
lastmod: 2025-12-29T13:39:20.763Z
title: HELP ME 
slug:  
featured: false
draft: false
tags:
  - Forensics
  
description:  None 
  
---
# Incident writeup — incident-v3.pcapng

**Flag:** `putcCTF{3xf1ltr4ted}`

## Summary
A host exfiltrated a file over DNS to `backup.site.lan`. I reconstructed the transferred payload from DNS query names, decoded the contained base64 strings and recovered a Caesar-shifted flag. The final decoded flag is `putcCTF{3xf1ltr4ted}`.

## Artifacts
- **PCAP:** [incident-v3.pcapng](incident-v3.pcapng)
- **Extracted transmitter script:** [extracted_http/transmiter.py](extracted_http/transmiter.py#L1-L60)
- **DNS query dump:** [dns_queries.txt](dns_queries.txt)
- **Reconstruction script:** [reconstruct_dns_exfil.py](reconstruct_dns_exfil.py)
- **Base64 decoder:** [decode_strings.py](decode_strings.py)
- **Recovered binary:** [recovered.bin](recovered.bin)

## High-level steps
1. **Quick scan:** Search the pcap for obvious flag-like strings.
   - `strings incident-v3.pcapng | grep -o -E 'putcCTF\{[^}]+\}' || true`
   - This returned repeated fake flags, so deeper analysis was needed.

2. **Protocol enumeration:** Check what protocols are in the capture.
   - `tshark -r incident-v3.pcapng -T fields -e frame.protocols | sed 's/\./\n/g' | sort | uniq -c | sort -nr`
   - Observed lots of QUIC/TLS and significant DNS traffic.

3. **Export HTTP objects:** Pull any HTTP payloads (a transmitter script was present).
   - `mkdir -p extracted_http && tshark -r incident-v3.pcapng --export-objects http,extracted_http`
   - Found `extracted_http/transmiter.py` (the sender script). See [extracted_http/transmiter.py](extracted_http/transmiter.py#L1-L60).

4. **Understand the transmitter:** The script behavior (summary):
   - Reads the file and computes `id = md5(content)`.
   - Splits file into 32-byte chunks, XORs each chunk with a `KEY`, hexlifies the result, splits into 32-char labels and sends DNS queries of the form:
     - `{id}.{sequence}.{hexpart1}.{hexpart2}....backup.site.lan`
   - After each chunk the `KEY` is updated: `KEY = md5(data + KEY)` (so reconstruction must mirror this).
   - Sender signals end-of-transfer with `id.A.backup.site.lan`.

5. **Extract DNS queries:** Dump all client-side queries to `backup.site.lan` into a file.
   - `tshark -r incident-v3.pcapng -Y 'dns.qry.name contains "backup.site.lan" && dns.flags.response == 0' -T fields -e dns.qry.name > dns_queries.txt`

6. **Reconstruct transferred bytes:** I wrote `reconstruct_dns_exfil.py` which:
   - Parses `dns_queries.txt`, groups labels by the numeric `sequence` label,
   - Joins hex label parts, unhexlifies to bytes,
   - XORs with the same initial `KEY` and updates `KEY = md5(data + KEY)` for each block (to mirror the transmitter),
   - Concatenates decrypted blocks and writes `recovered.bin`.
   - Run: `python3 reconstruct_dns_exfil.py` → produced `recovered.bin` (size 796 bytes).

7. **Inspect recovered data:** Look at printable content and decode base64 blobs.
   - `strings recovered.bin` showed many base64-encoded lines.
   - I used `decode_strings.py` (included) to base64-decode each printable line from `recovered.bin`.
   - The decoded lines contained readable text and one suspicious token: `qvudDUG{4yg2mus5ufe}`.

8. **Decode final token:** The token is a single-character-left Caesar-like shift (each alphanumeric character is decremented by 1). Example:
   - `q->p, v->u, u->t, d->c` and `4->3, y->x` etc.
   - A small Python snippet to decode:

```sh
python3 - <<'PY'
 s = "qvudDUG{4yg2mus5ufe}"
 out = []
 for c in s:
     if 'a' <= c <= 'z' or 'A' <= c <= 'Z' or '0' <= c <= '9':
         out.append(chr(ord(c)-1))
     else:
         out.append(c)
 print(''.join(out))
PY
```

   - This yields the final flag: `putcCTF{3xf1ltr4ted}`.

## Commands (quick copy)
- `strings incident-v3.pcapng | grep -o -E 'putcCTF\{[^}]+\}' || true`
- `tshark -r incident-v3.pcapng -T fields -e frame.protocols | sed 's/\./\n/g' | sort | uniq -c | sort -nr`
- `tshark -r incident-v3.pcapng --export-objects http,extracted_http`
- `tshark -r incident-v3.pcapng -Y 'dns.qry.name contains "backup.site.lan" && dns.flags.response == 0' -T fields -e dns.qry.name > dns_queries.txt`
- `python3 reconstruct_dns_exfil.py`  # writes `recovered.bin`
- `python3 decode_strings.py`         # decodes base64 lines from `recovered.bin`

## Notes / improvements
- The transmitter used a per-block evolving key (md5 of previous payload+key). The reconstruction must mirror that exactly.
- If the DNS chunks are missing or out of order, the sequence label is used to order fragments.
- If you want, I can:
  - commit these scripts into a git branch,
  - add a small verification harness to automatically run reconstruction+decode,
  - or produce a small Dockerfile to reproduce the steps.

---

