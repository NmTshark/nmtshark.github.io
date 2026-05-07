---
author: NmToan
date: 2026-05-02T00:00:00.000Z
lastmod: 2026-05-02T00:00:00.000Z
title: KubSTU Financial PDF Write-up
slug: kubstu-financial-pdf-write-up
featured: false
draft: false
tags:
  - Forensics
  - KubSTU
  - PDF
description: Write-up for the KubSTU financial.pdf challenge, focusing on embedded files, metadata, and decoy artifacts.
---

# Write-up: KubSTU - phân tích file `financial.pdf`

## Kết quả cuối cùng

**Flag thật:**

```text
KubSTU{PDF_M3t4d4t4_F0r3ns1cs_4dv4nc3d_Ch4ll3ng3_2025_S3cur3_Emb3dd3d_F1l3_3ncrypt10n_Pr0t0c0l}
```

---

## 1) Quan sát ban đầu

Khi chạy `binwalk` trên file PDF, ta thấy 3 thứ đáng chú ý:

```bash
binwalk financial.pdf
```

Kết quả:

- Offset `0x0`: PDF document
- Offset `0x84A`: ZIP archive data
- Offset `0x8E97`: Intel HEX data

Điều này dễ khiến mình nghĩ hướng làm là:
1. Tách ZIP ra
2. Giải nén ZIP
3. Phân tích “Intel HEX”

Nhưng đây chính là bẫy của đề.

---

## 2) Đọc nội dung PDF trước

Dùng `pdftotext` hoặc đọc text trong PDF trước:

```bash
pdftotext financial.pdf -
```

Ta thấy PDF ghi rất rõ:

- Có embedded archive
- Tên archive là `KUBGTU_FINANCIAL_DATA_2025.zip`
- Mật khẩu là:

```text
FinanceKubSTU2025!
```

Tức là chính PDF **cố tình dẫn người giải** đi theo hướng ZIP.

---

## 3) Vì sao ZIP là decoy?

Sau khi extract ZIP và giải nén bằng mật khẩu trong PDF, nội dung nhận được là một file text kiểu:

- `FAKE`
- `TEST`
- `DECOY`
- cảnh báo “you have been hacked”
- rất nhiều chuỗi trông giống flag nhưng đều giả

Ví dụ:

```text
KubSTU{FAKE_9010}
KubSTU{TEST_349}
KubSTU{TEST_295}
KubSTU{DECOY_856}
```

Các dấu hiệu cho thấy đây là mồi nhử:

1. Nội dung tự nhận nó là giả / decoy.
2. Có quá nhiều flag theo nhiều format khác nhau (`FAKE`, `TEST`, `DECOY`, `DUMMY`, `MOCK`).
3. Mật khẩu ZIP lại được in công khai trong PDF, nghĩa là đề đang muốn người chơi đi vào nhánh phụ.

Kết luận: **ZIP không chứa flag thật**, mà chỉ là tầng đánh lạc hướng.

---

## 4) Kiểm tra phần “không hiển thị” của PDF

Bước quan trọng là không chỉ đọc phần văn bản hiển thị trên trang, mà phải đọc **raw strings / metadata** của PDF.

Lệnh rất hữu ích:

```bash
strings -a financial.pdf | less
```

Hoặc lọc theo từ khóa:

```bash
strings -a financial.pdf | grep -nE 'HiddenAuditData|ArchivePassword|KubSTU|DATA\['
```

Kết quả cho thấy PDF có các trường metadata/custom field như:

```text
/ArchivePassword (FinanceKubSTU2025!)
/HiddenAuditData (...)
```

Ở đây, trường **`/HiddenAuditData`** là cực kỳ đáng nghi, vì:

- Không nằm trong nội dung hiển thị của 2 trang PDF
- Chứa rất nhiều dữ liệu “rác có chủ đích”
- Chứa hàng loạt chuỗi `KubSTU{FAKE...}`, `TEST`, `DECOY`, `DUMMY`, `MOCK`
- Xen lẫn nhiều chuỗi Base64

Nói cách khác: **flag thật không nằm trên trang PDF**, mà nằm trong **metadata ẩn**.

---

## 5) Tìm điểm bất thường trong `HiddenAuditData`

Trong đống dữ liệu mồi, ta tìm các chuỗi có khả năng là dữ liệu mã hóa/ẩn thật.

Một dòng nổi bật là:

```text
DATA[9376]="S3ViU1RVe1BERl9NM3Q0ZDR0NF9GMHIzbnMxY3NfNGR2NG5jM2RfQ2g0bGwzbmczXzIwMjVfUzNjdXIzX0VtYjNkZDNkX0YxbDNfM25jcnlwdDEwbl9QcjB0MGMwbH0="
```

Lý do dòng này đáng nghi hơn các dòng khác:

1. Nó là **một chuỗi Base64 hoàn chỉnh**, không bị cắt cụt.
2. Chuỗi bắt đầu bằng `S3Vi...`, đây là pattern hay gặp khi Base64 của chuỗi bắt đầu bằng `Kub...`.
3. Không giống các “flag giả” được viết thẳng ra văn bản như `KubSTU{FAKE_....}`.
4. Nó nằm lẫn trong metadata ẩn, rất hợp với kiểu giấu flag trong PDF forensic.

---

## 6) Giải mã Base64

Decode chuỗi đó:

```bash
echo 'S3ViU1RVe1BERl9NM3Q0ZDR0NF9GMHIzbnMxY3NfNGR2NG5jM2RfQ2g0bGwzbmczXzIwMjVfUzNjdXIzX0VtYjNkZDNkX0YxbDNfM25jcnlwdDEwbl9QcjB0MGMwbH0=' | base64 -d
```

Kết quả:

```text
KubSTU{PDF_M3t4d4t4_F0r3ns1cs_4dv4nc3d_Ch4ll3ng3_2025_S3cur3_Emb3dd3d_F1l3_3ncrypt10n_Pr0t0c0l}
```

Đây là flag thật.

---

## 7) Vậy “Intel HEX” ở `binwalk` là gì?

Đây là phần dễ gây hiểu nhầm.

`binwalk` báo:

```text
36503  0x8E97  Intel HEX data, record type: data
```

Nhưng khi soi raw strings quanh vùng này, ta thấy một chuỗi như:

```text
VAL276:9652200051
TEXT_58: "cy1e6bA2tk53twH69WN8GIN3n5kWY5T"
CODE25[:]WjhBSWdKb1h4bTgxMDJLZ0RIVmU0U2RnRUNxMXQ1MVNQUDF2Qk9TTW
```

`binwalk` có thể nhận diện nhầm một đoạn text có dấu `:` và pattern giống Intel HEX. Trong trường hợp này:

- Không có bằng chứng đây là một file Intel HEX hoàn chỉnh và hợp lệ
- `srec_cat` không extract được vì môi trường thiếu tool
- Quan trọng hơn: flag thật đã nằm trong metadata ẩn và decode ra hoàn chỉnh

Do đó, hướng “Intel HEX” ở đây rất có thể là **false positive** hoặc một decoy phụ thêm.

---

## 8) Tóm tắt hướng giải đúng

### Hướng sai dễ bị dụ

1. Dùng `binwalk`
2. Thấy ZIP -> extract ZIP
3. Dùng password trong PDF để mở ZIP
4. Nhận được file text chứa nhiều `FAKE/TEST/DECOY`
5. Tiếp tục mắc kẹt trong đống mồi

### Hướng đúng

1. Nhìn `binwalk` để biết PDF có dữ liệu nhúng
2. **Đọc nội dung PDF** để xác định ZIP + password chỉ là nhánh phụ
3. Dùng `strings -a financial.pdf`
4. Tìm các trường metadata/custom field
5. Phát hiện `/HiddenAuditData`
6. Trong đó tìm chuỗi Base64 hoàn chỉnh ở `DATA[9376]`
7. Decode Base64 -> ra flag thật

---

## 9) Các lệnh tái hiện đầy đủ

### Xem cấu trúc sơ bộ

```bash
binwalk financial.pdf
pdfinfo financial.pdf
```

### Đọc phần text hiển thị của PDF

```bash
pdftotext financial.pdf -
```

### Xem raw strings trong PDF

```bash
strings -a financial.pdf | less
```

### Lọc nhanh các điểm đáng chú ý

```bash
strings -a financial.pdf | grep -nE 'HiddenAuditData|ArchivePassword|KubSTU|DATA\['
```

### Decode chuỗi flag

```bash
echo 'S3ViU1RVe1BERl9NM3Q0ZDR0NF9GMHIzbnMxY3NfNGR2NG5jM2RfQ2g0bGwzbmczXzIwMjVfUzNjdXIzX0VtYjNkZDNkX0YxbDNfM25jcnlwdDEwbl9QcjB0MGMwbH0=' | base64 -d
```

---

## 10) Kết luận

Bài này là một bài **PDF forensic + decoy analysis**.

Ý tưởng chính của challenge:

- Cho người chơi thấy ZIP được nhúng trong PDF
- Cung cấp luôn password để người chơi nghĩ đó là con đường chính
- ZIP lại chỉ chứa flag giả
- Flag thật được giấu trong **custom metadata** của PDF, cụ thể là trường **`/HiddenAuditData`**
- Trong metadata có rất nhiều chuỗi mồi, nhưng chuỗi Base64 tại `DATA[9376]` mới là dữ liệu thật

**Flag cuối cùng:**

```text
KubSTU{PDF_M3t4d4t4_F0r3ns1cs_4dv4nc3d_Ch4ll3ng3_2025_S3cur3_Emb3dd3d_F1l3_3ncrypt10n_Pr0t0c0l}
```
---
author: NmToan
date: 2026-05-02T00:00:00.000Z
lastmod: 2026-05-02T00:00:00.000Z
title: KubSTU Financial PDF Write-up
slug: kubstu-financial-pdf-write-up
featured: false
draft: false
tags:
  - Forensics
  - KubSTU
  - PDF
description: Write-up for the KubSTU `financial.pdf` challenge, focusing on embedded files, metadata, and decoy artifacts.
---


