---
author: NmToan
date: 2026-05-02T00:00:00.000Z
lastmod: 2026-05-02T00:00:00.000Z
title: KubCTF Log Analysis Write-up
slug: kubctf Log-Analysis-Write-up
featured: false
draft: false
tags:
  - Forensics
  - KubSTU
  - SQLi
description: Write-up for the KubCTF log analysis challenge, detailing the attack timeline, techniques used, and indicators of compromise.
---
# Write-up: Web Server Compromise -> Database Server Pivot -> Data Theft

> Sanitized/defanged version: raw exploit payloads and shell commands have been neutralized for safe download and reading.

## 1. Challenge information

Trong quá trình security audit, log trên web server cho thấy attacker đã khai thác ứng dụng web, ghi web shell lên server, sau đó di chuyển sang database server và copy dữ liệu nhạy cảm.

Yêu cầu của challenge:

```text
Identify which vulnerability was used to gain initial access and what was uploaded.
Under which user did the attacker subsequently operate?
What was copied?
```

Flag đúng:

```text
KubSTU{SQLi,shell.php,dbadmin,confidential_data.sql}
```

---

## 2. Artefacts được cung cấp

Các artefact chính dùng để phân tích:

```text
access.log
error.log
auth.log
bash_history trên web service
bash_history trên database server
```

Những log quan trọng nhất đã được defang để tránh bị AV nhận diện nhầm:

```log
GET /index.php?id=1 UNION_SELECT 1,@@datadir
```

```log
GET /index.php?id=1 UNION_SELECT 1,'<PHP web shell payload - defanged>' INTO_OUTFILE '/var/www/html/uploads/shell.php'
```

```log
GET /uploads/shell.php?cmd=[id]
GET /uploads/shell.php?cmd=[read application config]
```

```bash
cp /var/lib/mysql/confidential_data.sql /tmp/[.backup_data]
```

---

## 3. Timeline tấn công

### 10:15:30 — SQL Injection reconnaissance

Attacker bắt đầu kiểm tra SQL Injection bằng payload:

```log
GET /index.php?id=1 UNION_SELECT 1,@@datadir
```

`@@datadir` là biến trong MySQL/MariaDB dùng để trả về đường dẫn thư mục dữ liệu của database. Việc query biến này cho thấy attacker đang xác nhận khả năng inject SQL và thu thập thông tin môi trường database.

User-Agent của request là:

```text
sqlmap/1.6.12
```

Điều này cho thấy attacker nhiều khả năng sử dụng công cụ tự động `sqlmap`.

Kết luận bước này:

```text
Vulnerability: SQL Injection
Affected parameter: id
Affected endpoint: /index.php
```

---

### 10:16:05 — Web shell được ghi lên server

Request tiếp theo cho thấy attacker dùng SQL Injection để ghi file PHP vào thư mục web-accessible:

```log
GET /index.php?id=1 UNION_SELECT 1,'<PHP command-execution payload - defanged>' INTO_OUTFILE '/var/www/html/uploads/shell.php'
```

Phần quan trọng của payload, đã được defang:

```sql
UNION_SELECT 1,'<PHP command-execution payload>'
INTO_OUTFILE '/var/www/html/uploads/shell.php'
```

Kỹ thuật này lợi dụng quyền ghi file của database process thông qua `INTO OUTFILE`. Attacker đã ghi nội dung PHP shell vào:

```text
/var/www/html/uploads/shell.php
```

Nội dung shell gốc là một PHP command execution one-liner. Trong bản sanitized này, payload không được ghi nguyên dạng để tránh bị Defender nhận diện nhầm.

Đây là một web shell rất đơn giản, cho phép attacker thực thi lệnh hệ thống qua tham số HTTP `cmd`.

Ví dụ đã defang:

```text
/uploads/shell.php?cmd=[id]
/uploads/shell.php?cmd=[read /etc/passwd]
/uploads/shell.php?cmd=[list files]
```

Kết luận bước này:

```text
Uploaded file: shell.php
Uploaded payload: PHP command execution web shell
```

---

### 10:16:15 — Attacker thực thi lệnh qua web shell

Sau khi shell được ghi thành công, attacker gọi:

```log
GET /uploads/shell.php?cmd=[id]
```

Request này kiểm tra user đang thực thi web shell. Trên Linux web server chạy Apache/PHP, user thường là:

```text
www-data
```

Tuy nhiên, đây chỉ là user ở giai đoạn web server. Challenge hỏi “under which user did the attacker subsequently operate?” trong bối cảnh attacker đã move sang database server, nên user cần lấy ở bước sau là `dbadmin`.

---

### 10:16:20 — Recon trên web server

Attacker tiếp tục liệt kê thư mục home của user web service:

```log
GET /uploads/shell.php?cmd=[list /home/www-data]
```

Điều này cho thấy attacker đã có remote command execution thông qua web shell và đang tìm thông tin có thể dùng để pivot.

---

### 10:16:25 — Đọc file cấu hình ứng dụng

Attacker đọc file cấu hình web app:

```log
GET /uploads/shell.php?cmd=[read /var/www/html/config.php]
```

Đây là bước rất quan trọng. File `config.php` thường chứa thông tin như:

```php
DB_HOST
DB_USER
DB_PASSWORD
DB_NAME
```

Từ đây attacker có thể lấy credential hoặc SSH key để truy cập database server.

Kết luận:

```text
Attacker likely obtained credentials or pivot information from /var/www/html/config.php
```

---

## 4. Pivot sang database server

Trong `auth.log`, có bằng chứng attacker đăng nhập vào database server bằng user `dbadmin`.

Dòng quan trọng:

```log
Mar 26 10:16:30 victim-db sshd[5680]: Accepted publickey for dbadmin from 192.168.1.10 port 54323 ssh-rsa SHA256:hK6cLRP4m5w60fHK1BGmWooBTXIWz+vtVHmuH/luoVQ
```

Dòng này xác nhận có SSH login thành công vào host `victim-db` bằng user:

```text
dbadmin
```

Ngay sau đó, session của `dbadmin` được mở:

```log
Mar 26 10:16:31 victim-db sshd[5681]: pam_unix(sshd:session): session opened for user dbadmin by (uid=0)
```

Tiếp theo, user `dbadmin` dùng `sudo` để mở root shell:

```log
Mar 26 10:16:35 victim-db sudo: dbadmin : TTY=pts/0 ; PWD=/home/dbadmin ; USER=root ; COMMAND=/bin/bash
```

Các dòng này chứng minh attacker đã hoạt động trên database server dưới user `dbadmin`, sau đó leo quyền sang root shell.

Kết luận cho câu hỏi:

```text
Under which user did the attacker subsequently operate?
Answer: dbadmin
```

---

## 5. Hoạt động trên database server

Trong bash history của database server có các lệnh:

```bash
mysql -u root -p
use webapp_db;
SELECT * FROM sensitive_info;
SHOW DATABASES;
SHOW TABLES;
DESCRIBE sensitive_info;
```

Các lệnh này cho thấy attacker đã truy cập MySQL/MariaDB, chọn database `webapp_db`, kiểm tra bảng và truy vấn dữ liệu trong bảng `sensitive_info`.

Sau đó xuất hiện lệnh quyết định:

```bash
cp /var/lib/mysql/confidential_data.sql /tmp/[.backup_data]
```

Lệnh này copy file:

```text
/var/lib/mysql/confidential_data.sql
```

sang:

```text
/tmp/.backup_data
```

Sau đó attacker kiểm tra file copy:

```bash
ls -la /tmp/
cat /tmp/[.backup_data]
```

Cuối cùng xoá dấu vết:

```bash
rm /tmp/[.backup_data]
history [-c]
```

Kết luận cho câu hỏi:

```text
What was copied?
Answer: confidential_data.sql
```

---

## 6. Attack chain tổng hợp

Toàn bộ chuỗi tấn công có thể tóm tắt như sau:

```text
1. Attacker dùng sqlmap kiểm tra SQL Injection tại /index.php?id=
2. Attacker dùng UNION SELECT ... INTO OUTFILE để ghi PHP web shell
3. Web shell được tạo tại /var/www/html/uploads/shell.php
4. Attacker gọi shell.php?cmd=... để thực thi lệnh hệ thống
5. Attacker đọc /var/www/html/config.php để lấy thông tin pivot
6. Attacker SSH sang database server bằng user dbadmin
7. Attacker dùng sudo để mở root shell
8. Attacker truy cập database và kiểm tra sensitive_info
9. Attacker copy /var/lib/mysql/confidential_data.sql sang /tmp/.backup_data
10. Attacker đọc file backup, xoá file tạm và xoá lịch sử lệnh
```

---

## 7. Indicators of Compromise

### Source IPs

```text
192.168.1.100
192.168.1.10
```

`192.168.1.100` xuất hiện trong web access log khi khai thác SQL Injection và gọi web shell.

`192.168.1.10` xuất hiện trong auth log khi SSH vào database server bằng user `dbadmin`.

### Web shell path

```text
/var/www/html/uploads/shell.php
```

### Suspicious HTTP requests

```text
/index.php?id=1 UNION SELECT 1,@@datadir
/index.php?id=1 UNION SELECT ... INTO OUTFILE ...
/uploads/shell.php?cmd=id
/uploads/shell.php?cmd=cat /var/www/html/config.php
```

### Suspicious database/server commands

```bash
mysql -u root -p
SELECT * FROM sensitive_info;
cp /var/lib/mysql/confidential_data.sql /tmp/.backup_data
cat /tmp/.backup_data
rm /tmp/.backup_data
history -c
```

### Stolen/copied file

```text
confidential_data.sql
```

---

## 8. Answers

| Question | Answer |
|---|---|
| Initial access vulnerability | `SQLi` |
| Uploaded file | `shell.php` |
| User attacker subsequently operated as | `dbadmin` |
| Copied file | `confidential_data.sql` |

Final flag:

```text
KubSTU{SQLi,shell.php,dbadmin,confidential_data.sql}
```
