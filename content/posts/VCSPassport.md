---
author: NmToan
date: 2025-12-27T04:59:04.866Z
lastmod: 2025-12-27T13:39:20.763Z
title: VCSPassport BlueTeam CTF Gimme_your_point!
slug:  VCSPassport_Blue_Team_CTF_Gimme_your_point
featured: false
draft: false
tags:
  - Forensics
  - VCSPassport
  - Pcap
description:  |
  "Write-up challenge Gimme_your_point từ VCSPassport về phân tích lưu lượng mạng và dịch ngược mã độc."
---

> **Chủ đề:** Phân tích lưu lượng mạng và dịch ngược mã độc powerShell được nguỵ trang trong lớp vỏ file thực thi 

> **Mục tiêu:** Chúng ta được cung cấp một file bắt gói tin mạng (pcap). Kịch bản cho thấy một máy chủ SharePoint đã bị tấn công. Kẻ tấn công đã xâm nhập, tải xuống mã độc, thu thập dữ liệu nhạy cảm từ trình duyệt Chrome và gửi dữ liệu đó ra ngoài. Nhiệm vụ là truy vết lại quá trình này để tìm lại dữ liệu bị đánh cắp (Flag)

```
Format Flag: VCSPassport{...}
```

## Phân tích file pcap

Như một thói quen mình bắt đầu phân tích file pcap bằng việc follow TCP stream để xem các luồng dữ liệu trao đổi giữa các máy thì tôi phát hiện ra một vài stream khá đáng chú ý:

Stream 1:
![TCP Stream 1](src/assets/images/vcspcap/tcp1.png)

Sau khi tìm hiểu thì tôi nhận thấy đây là một dấu hiệu của việc khai thác lổ hổng Remote Code Execution (RCE) trên SharePoint thông qua việc upload file ASPX độc hại lên server.

Stream 2:
![TCP Stream 2](src/assets/images/vcspcap/tcp2.png)

Tiếp tục phân tích stream tiếp theo thì tôi phát hiện có khả năng cao là attacker đang tải một mã độc về máy nạn nhân. File `raw_package` nhìn bề ngoài giống như một file Digital Certificate vì nó bắt đầu có header chuẩn là:
```
-----BEGIN CERTIFICATE-----
```
Tuy nhiên, sau khi tôi thử decode base64 một đoạn nhỏ của payload thì kết quả trả về khá bất ngờ khi nó tiếp tục là một đoạn text tiếp tục là bắt đầu với header `-----BEGIN CERTIFICATE-----`. Tiếp tục decode thêm một lần nữa thì tôi phát hiện ra header đã thay đổi thành `MZ` - đặc trưng của file thực thi Windows PE.

![Decode Base64](src/assets/images/vcspcap/decoded.png)

Dừng lại ở đây khi đã xác định được đây là một file thực thi. Tôi tiếp tục kiêm tra các gói tin HTTP khác để tìm kiếm thêm thông tin về file này.

![Các gói tin HTTP](src/assets/images/vcspcap/http.png)
Sau khi phân tích thì tôi tiến hành tải các file đã được truyền tải trong các gói tin HTTP về máy để phân tích tiếp bằng: `file --> export object --> save` 

Sau khi tải về tôi có được các file sau:

![Các file đã tải về](src/assets/images/vcspcap/file.png) 
Phân tích nhanh các file này tôi nhận thấy có một file tôi đã xác nhận là file thực thi `raw_package`. Ngoài ra còn các file khác như `ToolPane(1).aspx` là file ASPX độc hại được upload lên server SharePoint để khai thác lỗ hổng RCE được tìm thấy trong TCP Stream 1. Chú ý thêm là 2 file `upload`  và khi tôi dùng lệnh `strings` và `cat` để trích xuất chuỗi từ file `upload` thì tôi phát hiện ra khả năng đây là một file zip được đổi tên đuôi.
![Strings upload](src/assets/images/vcspcap/stringsupload.png)
![Cat upload](src/assets/images/vcspcap/catupload.png)

Phân tích một chút đoạn trên tôi nhận thấy có các chuỗi liên quan đến Chrome như `chrome_health_result` và `chrome_service_result`. Tôi đoán đây có thể là file zip chứa dữ liệu đánh cắp từ trình duyệt Chrome của nạn nhân. Và vì sao tôi có thể nhận biết thì theo tìm hiểu thì file zip có header chuẩn là `PK` và trong đoạn trích xuất chuỗi trên tôi đã thấy ký tự `P`, ngoài ra còn có đoạn `--e189ad38...` còn cho biết gói tin này đang gửi một file đính kèm ở đây là file zip. 
Vì vậy tôi đã thử tiến hành đổi đuôi file `upload` thành `upload.zip` và giải nén thử:
![Unzip upload](src/assets/images/vcspcap/unzipupload.png)

Để giải nén file zip này yêu cầu cần một password. Tới đây chúng ta đã có 2 luồng thông tin khá quan trọng:
1. File thực thi `raw_package` có thể là mã độc được tải về máy nạn nhân.
2. File zip `upload.zip` chứa dữ liệu đánh cắp từ trình duyệt Chrome.

Vì vậy tôi quyết định phân tích tiếp file thực thi `raw_package` để tìm kiếm manh mối về password giải nén file zip.  
## Phân tích mã độc

Như đã tìm hiểu trước đó chúng ta đã xác định được file `raw_package` là một file thực thi Windows PE được bọc bởi 2 lớp mã hoá base64. Tôi tiến hành sử dụng đoạn script Python sau để decode file này:

```python
import base64

input_path = '/home/kali/Downloads/vcsctf/pcapagain/raw_package'
output_path = 'decoded_output.txt'

try:
    with open(input_path, 'r') as f:
        encoded_data = f.read().replace('\n', '').replace('\r', '').strip()
    decoded_data = base64.b64decode(encoded_data)
    with open(output_path, 'wb') as f:
        f.write(decoded_data)

    print(f"Done")

except Exception as e:
    print(f"Lỗi: {e}")
```
Sau khi decode xong tôi được một file `decoded_output.txt` chứa nội dung của lần mã hoá thứ 2, tôi tiếp tục sử dụng đoạn script khác để decode tiếp lần nữa:

```python
import base64
import re

# ĐƯỜNG DẪN FILE CỦA BẠN (File chứa chuỗi TVqQAAM...)
input_path = '/home/kali/Downloads/vcsctf/pcapagain/decoded_output.txt'
output_path = 'malware.exe'

try:
    with open(input_path, 'r', encoding='utf-8', errors='ignore') as f:
        raw_content = f.read()
    clean_b64 = re.sub(r'[^A-Za-z0-9+/]', '', raw_content)

    # XỬ LÝ PADDING
    # Base64 bắt buộc độ dài chia hết cho 4. Nếu thiếu, thêm dấu '='
    missing_padding = len(clean_b64) % 4
    if missing_padding:
        clean_b64 += '=' * (4 - missing_padding)
        
    # GIẢI MÃ
    decoded_data = base64.b64decode(clean_b64)
    # GHI FILE
    with open(output_path, 'wb') as f:
        f.write(decoded_data)

    print(f"[+] Đã xuất file thành công: {output_path}")

except Exception as e:
    print(f"[-] Lỗi: {e}")
```
`Lưu ý là khi decode lần 2 ta nhớ thay đổi input_path thành decoded_output.txt và output_path thành tên file mới như malware.exe. Đồng thời xoá phần header -----BEGIN CERTIFICATE----- trước khi decode.`

Sau khi decode xong tôi đã có được file thực thi mã độc `malware.exe`. Tôi tiến hành điều tra và dùng các câu lệnh cơ bản để kiểm tra file này:
```
file malware.exe 
malware.exe: PE32+ executable for MS Windows 5.02 (console), x86-64, 18 sections
```
Trong quá trình phân tích nhanh bằng lệnh `strings` thì tôi không thu được kết quả gì đặc biệt. Tuy nhiên phải chú ý rằng đây là một file được thiết kế để chạy trên máy Windows vậy nên khả năng cao là chúng sử dụng bảng mã `UTF-16LE` thay vì bảng mã ASCII thông thường. Vì vậy tôi đã sử dụng lệnh `strings -el malware.exe` để trích xuất chuỗi với bảng mã UTF-16LE. 
(Các bạn có thể tìm hiểu kỹ hơn trên google)

![Strings UTF-16LE](src/assets/images/vcspcap/stringsutf16le.png)

Chú ý ngay 3 dòng đầu tiên có thể thấy
```
_p4$$w0d
_S3(ReT
sup3r
```     
Đây rất có thể là password để giải nén file zip `upload.zip` mà chúng ta đã tìm thấy ở phần trước. Tôi tiến hành thử ghép password này lại thành `sup3r_S3(ReT_p4$$w0d` và sử dụng để giải nén file zip:
```
unzip upload.zip                          
Archive:  upload.zip
warning [upload.zip]:  205 extra bytes at beginning or within zipfile
  (attempting to process anyway)
[upload.zip] chrome_health_result password: 
  inflating: chrome_health_result    
  inflating: chrome_service_result   
                                               
```
Quá trình giải nén thành công và tôi đã có được 2 file `chrome_health_result` và `chrome_service_result`. Tiếp tục phân tích 2 file này thì xác nhận đây chính là các file dữ liệu từ trình duyệt Chrome của nạn nhân và file `chrome_health_result` là một SQLite database. Tôi sử dụng công cụ DB Browser for SQLite để mở file này lên và tiến hành truy vấn dữ liệu trong các bảng.

```
sqlite3 chrome_health_result
SQLite version 3.46.1 2024-08-13 09:16:08
Enter ".help" for usage hints.
sqlite> SELECT * FROM logins;
https://flag.win.here/|||SH4R3_y0R_P$$wD||v20l3�� ��||https://flag.win.here/|13410266704228407|0|0|3|0|||||0|0||1|0||13410266704228407|||0|0|||0|0
https://local.domain.panel/|||admin||v20�䢣K^�H���Ϝ��Za��3EPUbxh+||https://local.domain.panel/|13410266724976314|0|0|3|0|||||0|0||2|0||13410266724976314|||0|0|||0|0
sqlite> 
```
Thực chất tới đây chúng ta có nhiều cách để tìm ra được cái dòng có khả năng là flag. Tôi nói chỉ là khả năng vì tôi đã làm lại bài này sau khi cuộc thi kết thúc và tôi không biết được đáp án chính xác là gì. Tuy nhiên tôi đã thử lấy dòng đầu tiên có chứa chuỗi `flag` trong URL và ghép vào định dạng flag được cung cấp ban đầu thì tôi đã có được flag hoàn chỉnh như sau:

```
VCSPassport{SH4R3_y0R_P$$wD}
```
Đây chỉ là suy đoán của tôi về flag của challenge này bởi vì tôi không thấy còn gì có thể phân tích thêm được nữa. Mục đích của bài viết này cũng là để chúng ta tham khảo học hỏi về cách chúng ta giải quyết một bài toán phân tích mã độc và lưu lượng mạng cũng như là cách giải quyết vấn đề. Nếu có sai sót mong các bạn thông cảm và góp ý để mình hoàn thiện hơn trong các bài viết sau và có bạn nào cũng tham gia vòng tuyển dụng và giải thành công thì hãy confimr cho mình nhé. 

Cảm ơn các bạn đã theo dõi!