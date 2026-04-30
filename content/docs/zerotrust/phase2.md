---
title: "Phase 2: Thiết lập mạng lõi (OpenZiti)"
description: "Khởi tạo OpenZiti Controller và Edge Router để xây dựng vành đai bảo mật định nghĩa bằng phần mềm."
toc: true
authors: []
tags: ["OpenZiti", "SDP", "Overlay Network"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-01-19'
lastmod: '2026-04-30'
draft: false
weight: 2
---

Nếu Keycloak chịu trách nhiệm định danh, thì OpenZiti chịu trách nhiệm **mang danh tính đó vào luồng kết nối mạng**. Đây là nền móng để sau này bạn viết policy truy cập theo đúng user, đúng thiết bị và đúng posture.

## Mục tiêu

- dựng được controller và edge router,
- xác nhận data plane đã hoạt động,
- chuẩn hóa naming convention cho identity,
- tạo sẵn identity mẫu để dùng cho các phase tích hợp và demo.

## Chuẩn bị

- máy chủ hoặc VM chạy được OpenZiti,
- DNS hoặc IP ổn định cho controller và router,
- chứng chỉ hoặc cơ chế TLS phù hợp nếu bạn dùng domain thật,
- danh sách hostname endpoint dự kiến sẽ quản lý.

## Bước 1: Khởi tạo controller và edge router

Bạn có thể dùng `ziti-quickstart` hoặc triển khai bằng Docker Compose. Dù chọn cách nào, đích cuối cùng vẫn giống nhau:

- **Ziti Controller** quản lý identity, policy và chứng chỉ,
- **Ziti Edge Router** nhận kết nối từ client và chuyển tiếp lưu lượng qua overlay.

Sau khi dựng xong, kiểm tra:

- controller đã chạy ổn định,
- edge router đã join thành công,
- router xuất hiện `Online` trong ZAC hoặc CLI.

Không nên tạo service hay policy khi router còn chập chờn, vì lúc đó lỗi kết nối sẽ rất khó phân biệt là do policy hay do hạ tầng.

## Bước 2: Chuẩn hóa quy tắc đặt tên identity

Phase này rất quan trọng vì Orchestrator ở Phase 6 sẽ dựa vào tên để map host của FleetDM sang identity trong Ziti.

Khuyến nghị dùng chuẩn:

- **Identity name**: `ep-<hostname>`
- **External ID**: email của user tương ứng trên Keycloak
- **Role attributes**: tách rõ role nghiệp vụ và role endpoint

Ví dụ:

- `ep-WIN-HR-01`
- `ep-WIN-SALES-01`

Role ban đầu có thể là:

- `#role.hr`
- `#role.sales`
- `#endpoint`

Ở bước này, chưa nên gán `#posture.compliant` hay `#posture.quarantine`. Posture phải do Orchestrator quản lý, không nên set tay.

## Bước 3: Tạo identity mẫu

Bạn có thể tạo identity bằng ZAC hoặc CLI. Dùng CLI thường dễ lặp lại hơn khi cần viết tài liệu hoặc dựng lại lab:

```bash
ziti edge create identity "ep-WIN-HR-01" -a "role.hr,endpoint" -P default --external-id "alice.hr@lab.local"
ziti edge create identity "ep-WIN-SALES-01" -a "role.sales,endpoint" -P default --external-id "alice.sales@lab.local"
```

Mục tiêu ở đây không phải tạo thật nhiều identity, mà là xác minh convention đã đúng:

- tên identity khớp hostname,
- `external-id` khớp email ở Keycloak,
- role nghiệp vụ được gắn ngay từ đầu.

## Bước 4: Xác minh cơ chế enrollment

Sau khi tạo identity, hãy kiểm tra:

- export hoặc tải được enrollment token,
- client có thể nạp token đó,
- identity xuất hiện đúng trên controller.

Bạn chưa cần hoàn tất SSO ở phase này. Điều cần kiểm chứng là OpenZiti đã sẵn sàng tiếp nhận identity và cấp quyền ở mặt phẳng mạng.

## Điểm kiểm tra

Phase 2 hoàn tất khi:

- controller và router đều online,
- tạo được identity theo convention `ep-<hostname>`,
- `external-id` được map đúng với email người dùng,
- enrollment token có thể được tạo ra bình thường.

## Đầu ra của phase

Sau phase này, bạn cần có:

- một OpenZiti fabric hoạt động,
- identity mẫu cho từng nhóm người dùng,
- naming convention rõ ràng để Orchestrator dùng lại ở các phase sau.

Đây là điều kiện cần trước khi bạn đưa posture của thiết bị vào hệ thống ở Phase 3.
