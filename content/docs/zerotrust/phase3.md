---
title: "Phase 3: Giám sát Posture (FleetDM & Osquery)"
description: "Cài đặt FleetDM Server và Osquery Agent để đánh giá tình trạng thiết bị theo thời gian thực."
toc: true
authors: []
tags: ["FleetDM", "Osquery", "Device Posture", "EDR"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2023-11-03'
lastmod: '2023-11-03'
draft: false
weight: 3
---

Thành phần thứ ba của hệ thống có nhiệm vụ trả lời câu hỏi: "Thiết bị có an toàn không?". Chúng ta sử dụng FleetDM làm máy chủ quản lý tập trung và Osquery làm tác nhân giám sát.

## 1. Khởi chạy FleetDM Server

FleetDM được đóng gói qua Docker Compose, yêu cầu 3 container hoạt động song song:
1. **FleetDM App:** Giao diện quản trị và xử lý API.
2. **MySQL:** Lưu trữ thông tin thiết bị và cấu hình truy vấn.
3. **Redis:** Cache dữ liệu tăng tốc phản hồi.

Khởi chạy bằng lệnh:
```bash
docker compose up -d
```
Sau khi khởi chạy, truy cập `http://localhost:8443` để tạo tài
Truy cập giao diện web FleetDM (thường ở port 8443), khởi tạo tài khoản Admin và lấy chuỗi Enroll Secret.

2. Cài đặt Osquery Agent lên Client
Trên các máy ảo Windows/Linux đóng vai trò là Client:

Tải bộ cài đặt từ trang chủ Osquery.

Khởi chạy daemon osqueryd kèm theo cờ cấu hình:

--tls_hostname: Trỏ về địa chỉ URL của FleetDM Server.

--enroll_secret_path: Trỏ đến file chứa Enroll Secret.

Quay lại Dashboard của FleetDM, thiết bị sẽ hiển thị trong danh sách Hosts với trạng thái Online.

3. Thiết lập Policies (Luật kiểm tra)
FleetDM dùng cú pháp SQL để truy vấn thông tin hệ điều hành. Chuyển đến tab Policies trên FleetDM và thêm luật quét mã độc.

Ví dụ: Kiểm tra tiến trình đào coin XMRig:

SQL
SELECT name, pid FROM processes WHERE name = 'xmrig.exe';
Thiết lập Polling Interval là 10 giây. Nếu câu lệnh SQL trả về kết quả (có tiến trình đang chạy), FleetDM sẽ đánh dấu thiết bị là Failing (Vi phạm).


Nếu trong quá trình cài đặt Osquery Agent lên máy ảo mà gặp lỗi kết nối TLS với FleetDM, bạn cứ n