---
title: "Phase 3: Giám sát Posture (FleetDM & Osquery)"
description: "Cài đặt FleetDM Server và Osquery Agent để đánh giá tình trạng thiết bị theo thời gian thực."
toc: true
authors: []
tags: ["FleetDM", "Osquery", "Device Posture", "EDR"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-01-26'
lastmod: '2026-04-30'
draft: false
weight: 3
---

Zero Trust không chỉ hỏi "bạn là ai" mà còn hỏi thêm "thiết bị bạn đang dùng có an toàn không". Phase này xây dựng lớp quan sát posture để hệ thống có dữ liệu ra quyết định thay vì chỉ dựa vào danh tính.

## Mục tiêu

- dựng được FleetDM server,
- enroll được Osquery agent từ endpoint,
- nhìn thấy host online trong dashboard,
- tạo được ít nhất một policy posture có thể pass hoặc fail rõ ràng.

## Chuẩn bị

- máy chủ chạy FleetDM,
- Docker Compose hoặc bộ cài tương đương,
- endpoint Windows hoặc Linux để cài agent,
- kết nối mạng ổn định giữa endpoint và FleetDM,
- chứng chỉ TLS hợp lệ nếu bạn truy cập bằng domain.

## Bước 1: Khởi chạy FleetDM

Thông thường FleetDM sẽ chạy cùng các thành phần phụ trợ như:

- ứng dụng FleetDM,
- MySQL,
- Redis.

Khởi chạy stack:

```bash
docker compose up -d
```

Sau khi dịch vụ lên, truy cập giao diện FleetDM và hoàn tất:

- tạo tài khoản admin,
- cấu hình tổ chức hoặc team nếu cần,
- lấy `enroll secret` cho agent.

## Bước 2: Cài Osquery agent lên endpoint

Trên máy client, cài `osqueryd` rồi trỏ agent về FleetDM bằng:

- `--tls_hostname`
- `--enroll_secret_path`
- các tham số log và schedule theo chuẩn môi trường của bạn

Điểm quan trọng là endpoint phải enroll thành công và xuất hiện trong danh sách `Hosts`. Nếu không thấy host online, chưa nên viết policy vội.

## Bước 3: Tạo policy posture đầu tiên

Bắt đầu bằng một policy rất rõ ràng, dễ tái hiện kết quả, ví dụ phát hiện tiến trình bị cấm:

```sql
SELECT name, pid FROM processes WHERE name = 'xmrig.exe';
```

Ý tưởng ở đây:

- nếu query không trả bản ghi nào, host ở trạng thái sạch với policy đó,
- nếu query trả về kết quả, host bị đánh dấu `failing`.

Bạn có thể đặt polling interval ngắn trong lab, ví dụ 10 giây, để quan sát phản ứng nhanh hơn ở các phase demo.

## Bước 4: Kiểm tra dữ liệu posture trên host

Sau khi policy chạy, cần xác minh được 3 thứ:

- host có xuất hiện trong FleetDM,
- policy đã được áp vào host,
- trạng thái `passing` hoặc `failing` thay đổi đúng khi bạn bật hoặc tắt tiến trình kiểm thử.

Nếu policy không đổi trạng thái dù bạn đã mô phỏng hành vi vi phạm, hãy xử lý xong ở phase này. Đừng đẩy lỗi sang OPA hay Orchestrator.

## Điểm kiểm tra

Phase 3 được xem là hoàn tất khi:

- FleetDM dashboard truy cập ổn,
- endpoint enroll thành công,
- ít nhất một host hiện `Online`,
- ít nhất một policy posture có khả năng sinh kết quả `failing` để dùng cho bước tự động hóa.

## Đầu ra của phase

Sau phase này, bạn cần có:

- FleetDM server đang chạy,
- endpoint đã enroll,
- dữ liệu posture đủ tin cậy để chuyển cho OPA ở Phase 4.

Khi posture đã đo được và tái hiện được, bạn mới nên đưa logic ra quyết định vào OPA.
