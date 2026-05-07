---
author: NmToan
date: 2026-05-02T00:00:00.000Z
lastmod: 2026-05-02T00:00:00.000Z
title: KubCTF Challenge PCAP Write-up
slug: kubctf-challenge-pcap-write-up
featured: false
draft: false
tags:
  - Forensics
  - KubSTU
  - Pcap
description: Write-up for the challenge.pcap task, covering stream analysis, decoy protocols, and payload recovery.
---

# Challenge Write-up: `challenge.pcap`

## Tóm tắt

Bài cho một file `challenge.pcap` và mô tả rằng tài liệu tác chiến được truyền qua một **kênh mã hóa dùng custom protocol**.  
Mục tiêu là tìm được tài liệu thật trong PCAP và trích xuất **flag**.

**Flag cuối cùng:**

```text
KubSTU{p1ngu1n_0p_k4p1b4r0v5k_f4ll5}
```

---

## Ý tưởng tổng quát

Ban đầu PCAP có rất nhiều thứ gây nhiễu:

- HTTP responses nhìn “hợp lệ” nhưng nội dung rất lạ
- nhiều phiên FTP login thành công
- các directory listing lộ ra tên file như:
  - `intel_dump.bin`
  - `coordinates.gpx`
  - `mission_report.enc`
  - `roster.txt`
  - `map_kapibarovsk.dat`

Nhưng điểm mấu chốt là:

- **không có `RETR` trong FTP**
- **không export được file thật từ FTP-DATA**
- đề cũng nói rõ dữ liệu được truyền bằng **custom encrypted channel**

=> Kết luận: **HTTP và FTP chỉ là lớp decoy/manh mối**, còn payload thật nằm ở các stream nhị phân khác.

---

## Bước 1: Liệt kê các stream TCP đáng ngờ

Dùng script để gom payload theo từng stream TCP và lưu ra file `.raw`.

Ví dụ script:

```python
import dpkt
import socket
from pathlib import Path
from collections import defaultdict

PCAP = "challenge.pcap"
OUT = Path("extracted_streams")
OUT.mkdir(exist_ok=True)

targets = {9999, 7777, 54321, 12345, 31337, 4444, 5555, 8080, 5432}

streams = defaultdict(list)

with open(PCAP, "rb") as f:
    pcap = dpkt.pcap.Reader(f)
    for pkt_no, (ts, buf) in enumerate(pcap, start=1):
        eth = dpkt.ethernet.Ethernet(buf)
        if not isinstance(eth.data, dpkt.ip.IP):
            continue
        ip = eth.data
        if not isinstance(ip.data, dpkt.tcp.TCP):
            continue

        tcp = ip.data
        src = socket.inet_ntoa(ip.src)
        dst = socket.inet_ntoa(ip.dst)

        if tcp.sport not in targets and tcp.dport not in targets:
            continue

        key = (src, tcp.sport, dst, tcp.dport)
        streams[key].append((pkt_no, bytes(tcp.data)))

for key, packets in streams.items():
    src, sport, dst, dport = key
    payload = b"".join(data for _, data in packets if data)
    if not payload:
        continue

    first_pkt = packets[0][0]
    last_pkt = packets[-1][0]
    name = f"{src}_{sport}_to_{dst}_{dport}_{first_pkt}_{last_pkt}.raw"
    (OUT / name).write_bytes(payload)
    print(name, len(payload))
```

Kết quả nổi bật:

```text
172.20.0.2_43745_to_172.20.0.3_7777_87_100.raw 1720
172.20.0.2_36897_to_172.20.0.3_54321_227_236.raw 1133
172.20.0.2_43031_to_172.20.0.3_12345_317_328.raw 1760
172.20.0.2_45222_to_172.20.0.3_9999_524_533.raw 21
172.20.0.2_52936_to_172.20.0.3_31337_807_820.raw 3537
172.20.0.2_40428_to_172.20.0.3_4444_1030_1039.raw 757
172.20.0.2_42496_to_172.20.0.3_5555_1232_1241.raw 871
```

Trong số này, stream **port `9999`** và **port `31337`** là quan trọng nhất.

---

## Bước 2: Tìm secret từ auth channel trên port `9999`

Mở stream:

- `172.20.0.2_45222_to_172.20.0.3_9999_524_533.raw`
- và response tương ứng từ server

Thấy dữ liệu rõ ràng:

```text
PASS:IcyFl1pp3r$2026
ACK:OK
```

Đây là manh mối cực mạnh:

- `IcyFl1pp3r$2026` là **secret/password thật** của custom channel
- chưa chắc dùng trực tiếp ở lớp đầu tiên, nhưng chắc chắn phải giữ lại

---

## Bước 3: Phân tích stream quan trọng nhất trên port `31337`

Kiểm tra phần đầu file:

```bash
xxd -g 1 -l 48 172.20.0.2_52936_to_172.20.0.3_31337_807_820.raw
```

Kết quả:

```text
00000000: 58 46 45 52 4a 7f 2b 91 de 33 a8 5c e1 6d f0 19
00000010: 87 c4 55 3e 00 00 0d b9 1a 34 28 95 ca 33 a9 5c
00000020: 82 6d 12 07 1e 98 1d ed f1 b7 24 96 de 33 d9 4b
```

Giải thích:

- `58 46 45 52` = ASCII `XFER`
- tiếp theo là **16 byte**
- `00 00 0d b9` = số big-endian `3513`
- và stream có tổng độ dài `3537`

Từ đó suy ra format:

```text
4 bytes   magic = XFER
16 bytes  key / nonce / session material
4 bytes   length = 3513
3513 bytes payload
```

Tách payload:

```bash
dd if=172.20.0.2_52936_to_172.20.0.3_31337_807_820.raw of=31337.body bs=1 skip=24 status=none
wc -c 31337.body
```

Kết quả:

```text
3513 31337.body
```

Tức là parsing đã đúng.

---

## Bước 4: Thử đúng lớp mã hóa ngoài cùng

Lúc đầu có thể nghĩ tới AES/ChaCha/XOR với password, nhưng ở stream `31337` có một dấu hiệu rất đẹp:

- header chứa đúng **16 byte**
- sau đó là payload entropy cao
- challenge kiểu này rất hay dùng **repeating XOR** với key nằm ngay trong header

Ta thử XOR phần body bằng chính 16 byte ở `raw[4:20]`.

### Script giải lớp đầu

```python
from pathlib import Path

raw = Path("172.20.0.2_52936_to_172.20.0.3_31337_807_820.raw").read_bytes()

key = raw[4:20]
body = raw[24:]

dec = bytes(b ^ key[i % len(key)] for i, b in enumerate(body))

Path("stage1.zip").write_bytes(dec)
print("wrote stage1.zip", len(dec))
print(dec[:32])
```

Chạy xong, kiểm tra:

```bash
xxd -g 1 -l 32 stage1.zip
```

Kết quả:

```text
00000000: 50 4b 03 04 14 00 01 00 63 00 e2 1e 99 5c 48 d3
00000010: bb c8 0f 07 00 00 71 17 00 00 12 00 0b 00 6d 69
```

`50 4b 03 04` = `PK\x03\x04`

=> **Payload sau khi XOR là một file ZIP hợp lệ.**

---

## Bước 5: Nhận ra đây là AES-encrypted ZIP

Dù đã có ZIP, không thể giải bằng `unzip` thường, vì local file header cho thấy:

```text
14 00 01 00 63 00 ...
```

Trong đó:

- compression/encryption method là **99**
- đây là dấu hiệu của **WinZip AES encrypted ZIP**

Tức là:

- lớp ngoài cùng: XOR với key 16 byte trong header `XFER`
- lớp bên trong: AES ZIP
- password để giải nhiều khả năng chính là secret tìm được ở port `9999`

Password dùng để giải ZIP:

```text
IcyFl1pp3r$2026
```

---

## Bước 6: Viết script giải AES ZIP

Script giải:

```python
from pathlib import Path
import struct
import hashlib
import hmac
import zlib
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

ZIP_PATH = "stage1.zip"
PASSWORD = b"IcyFl1pp3r$2026"

def extract_winzip_aes(zip_bytes, outdir="out"):
    outdir = Path(outdir)
    outdir.mkdir(exist_ok=True)

    i = 0
    while i < len(zip_bytes):
        if zip_bytes[i:i+4] != b"PK\x03\x04":
            break

        ver, flag, method, mtime, mdate, crc, csize, usize, fnlen, extralen = struct.unpack(
            "<HHHHHIIIHH", zip_bytes[i+4:i+30]
        )

        name = zip_bytes[i+30:i+30+fnlen].decode()
        extra = zip_bytes[i+30+fnlen:i+30+fnlen+extralen]
        start = i + 30 + fnlen + extralen

        header_id, data_size = struct.unpack("<HH", extra[:4])
        vendor_ver = struct.unpack("<H", extra[4:6])[0]
        vendor = extra[6:8]
        strength = extra[8]
        actual_method = struct.unpack("<H", extra[9:11])[0]

        key_len = {1: 16, 2: 24, 3: 32}[strength]
        salt_len = {1: 8, 2: 12, 3: 16}[strength]

        salt = zip_bytes[start:start+salt_len]
        pwdver = zip_bytes[start+salt_len:start+salt_len+2]
        enc = zip_bytes[start+salt_len+2:start+csize-10]
        auth = zip_bytes[start+csize-10:start+csize]

        dk = hashlib.pbkdf2_hmac("sha1", PASSWORD, salt, 1000, dklen=2*key_len + 2)
        enc_key = dk[:key_len]
        auth_key = dk[key_len:2*key_len]
        verifier = dk[2*key_len:]

        if verifier != pwdver:
            raise ValueError(f"wrong password for {name}")

        calc_auth = hmac.new(auth_key, enc, hashlib.sha1).digest()[:10]
        if calc_auth != auth:
            raise ValueError(f"auth failed for {name}")

        aes = Cipher(algorithms.AES(enc_key), modes.ECB()).encryptor()
        out = bytearray()
        counter = 1
        for j in range(0, len(enc), 16):
            block = enc[j:j+16]
            ctr = struct.pack("<I", counter) + b"\x00" * 12
            keystream = aes.update(ctr)
            out.extend(bytes(a ^ b for a, b in zip(block, keystream)))
            counter += 1

        plain = bytes(out)

        if actual_method == 8:
            plain = zlib.decompress(plain, -15)

        (outdir / name).write_bytes(plain)
        print(f"extracted: {name} ({len(plain)} bytes)")

        i = start + csize

zip_bytes = Path(ZIP_PATH).read_bytes()
extract_winzip_aes(zip_bytes)
```

Chạy:

```bash
python3 extract_zip.py
```

Kết quả:

```text
extracted: mission_report.txt (6001 bytes)
extracted: roster.txt (1702 bytes)
extracted: map.txt (1578 bytes)
```

---

## Bước 7: Tìm flag trong các file đã giải

Dùng `grep`:

```bash
grep -R "KubSTU{" out
```

Kết quả:

```text
out/mission_report.txt:СЕКРЕТНЫЙ КОД ОПЕРАЦИИ: KubSTU{p1ngu1n_0p_k4p1b4r0v5k_f4ll5}
```

---

## Flag

```text
KubSTU{p1ngu1n_0p_k4p1b4r0v5k_f4ll5}
```

---

## Vì sao hướng này đúng?

### Vì sao bỏ qua FTP?
- Nhiều login success và listing chỉ là mồi nhử/manh mối
- Không có `RETR`
- Không có file content nào được tải qua FTP trong capture

### Vì sao chọn port `9999`?
- Stream này lộ ra secret trực tiếp:
  - `PASS:IcyFl1pp3r$2026`
- Đây là dấu hiệu của auth channel cho protocol bí mật

### Vì sao chọn port `31337`?
- Có header rất rõ: `XFER`
- Có field length hợp lệ `3513`
- Cấu trúc gói hoàn chỉnh và chặt chẽ nhất trong toàn bộ capture

### Vì sao thử XOR trước?
- Header chứa đúng 16 byte key material
- XOR lặp là kiểu custom obfuscation rất hay gặp trong CTF
- Kết quả ra ngay `PK\x03\x04`, chứng minh giả thuyết đúng

### Vì sao dùng `IcyFl1pp3r$2026` cho ZIP?
- Đó là password duy nhất được custom channel xác thực thành công
- ZIP sau XOR là AES-encrypted, nên password auth channel là ứng viên hợp lý nhất
- Thử vào thì giải được toàn bộ archive

---

## Tóm tắt chain giải bài

1. Mở `challenge.pcap`
2. Nhận ra HTTP/FTP phần lớn là decoy
3. Trích các TCP stream đáng ngờ
4. Tìm auth secret ở port `9999`:
   - `IcyFl1pp3r$2026`
5. Phân tích stream `31337`
6. Nhận ra format:
   - `XFER | 16-byte key | 4-byte length | payload`
7. XOR payload với 16-byte key trong header
8. Thu được `stage1.zip`
9. Nhận ra đây là AES ZIP
10. Dùng password `IcyFl1pp3r$2026` để giải
11. Extract được:
    - `mission_report.txt`
    - `roster.txt`
    - `map.txt`
12. Tìm thấy flag trong `mission_report.txt`

---

## Một-lệnh để nhớ

Nếu đã hiểu bài rồi, “xương sống” của lời giải chỉ là:

- lấy secret từ `9999`
- lấy key 16 byte từ header `XFER`
- XOR body để ra ZIP
- dùng secret giải AES ZIP
- grep flag trong `mission_report.txt`




