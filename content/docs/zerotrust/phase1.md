---
title: "Phase 1: Triển khai Identity Provider (Keycloak)"
description: "Hướng dẫn cài đặt Keycloak và thiết lập cơ chế xác thực OIDC cho hệ thống Zero Trust."
toc: true
authors: []
tags: ["Keycloak", "IAM", "OIDC", "JWT"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-01-12'
lastmod: '2026-04-30'
draft: false
weight: 1
---

Phase này trả lời câu hỏi đầu tiên của Zero Trust: **người dùng là ai**. Mục tiêu là dựng một Identity Provider tập trung để các phase sau không phải tự quản lý user, password hay session riêng lẻ.

## Mục tiêu

- dựng Keycloak chạy ổn định,
- tạo một realm riêng cho lab,
- tạo OIDC client để OpenZiti dùng về sau,
- chuẩn hóa user và claim `email` để map với identity phía Ziti.

## Chuẩn bị

- 1 máy chủ hoặc VM chạy Docker và Docker Compose,
- cổng `8080` hoặc `8443` mở sẵn,
- PostgreSQL đi kèm nếu bạn muốn tách dữ liệu bền vững thay vì chạy chế độ dev,
- danh sách user demo dùng cho các phòng ban khác nhau.

## Bước 1: Khởi chạy Keycloak

Khuyến nghị chạy Keycloak bằng Docker Compose để dễ backup và nâng cấp. Tối thiểu bạn cần:

- 1 container Keycloak,
- 1 container PostgreSQL,
- biến môi trường admin cho lần đăng nhập đầu tiên.

Sau khi chuẩn bị xong `docker-compose.yml`, khởi chạy dịch vụ:

```bash
docker compose up -d
```

Chỉ chuyển sang bước tiếp theo khi:

- container Keycloak đã `Up` và ổn định,
- bạn truy cập được giao diện admin,
- đăng nhập được bằng tài khoản quản trị ban đầu.

## Bước 2: Tạo realm riêng cho dự án

Không nên dùng trực tiếp realm `master`. Hãy tạo một realm riêng, ví dụ `ztna-lab`, để:

- tách biệt cấu hình lab khỏi quản trị hệ thống,
- dễ export, backup và reset,
- tránh nhầm lẫn khi số lượng client và mapper tăng lên.

Realm này sẽ là namespace logic chung cho user, role và OIDC client của toàn bộ hệ thống C-ZTNA.

## Bước 3: Tạo OIDC client cho OpenZiti

Tạo một client mới với vai trò là đầu cuối đăng nhập cho Ziti Desktop Edge hoặc các identity người dùng.

Thông số nên thống nhất sớm:

- **Client ID**: `openziti-oidc`
- **Client type**: public client
- **Client authentication**: `Off`
- **Redirect URI**: dùng URI callback đúng với Ziti Desktop Edge, hoặc tạm dùng `*` trong lab nếu bạn đang thử nhanh

Việc đặt `public client` là hợp lý vì desktop app không phải nơi an toàn để lưu `client secret`.

## Bước 4: Tạo user mẫu

Nên tạo ít nhất 2 đến 3 user để test khác phòng ban, ví dụ:

- `alice.hr@lab.local`
- `alice.sales@lab.local`
- `tinh.sales@lab.local`

Khi tạo user, cần lưu ý:

- điền đầy đủ trường `email`,
- đặt password không ở trạng thái temporary nếu muốn demo nhanh,
- có thể gán thêm group hoặc role nội bộ của Keycloak nếu bạn muốn mở rộng về sau.

## Bước 5: Kiểm tra claim trong token

Điểm quan trọng nhất của phase này là **token phải chứa claim mà OpenZiti sẽ dùng để đối chiếu**. Trong kiến trúc này, claim đó là `email`.

Hãy kiểm tra trong `Client Scopes` hoặc mapper của client để bảo đảm:

- token có trường `email`,
- giá trị `email` đúng với user đăng nhập,
- các claim mặc định như `iss`, `aud`, `exp` cũng xuất hiện đúng chuẩn OIDC.

Nếu bỏ sót bước này, đến Phase 5 bạn sẽ đăng nhập Keycloak thành công nhưng Ziti vẫn từ chối tạo phiên.

## Điểm kiểm tra

Phase 1 được xem là hoàn tất khi:

- vào được realm `ztna-lab`,
- client `openziti-oidc` đã tồn tại,
- user demo đăng nhập được,
- token trả về có claim `email` đúng như mong đợi.

## Đầu ra của phase

Sau phase này, bạn cần có:

- realm `ztna-lab`,
- client `openziti-oidc`,
- bộ user demo,
- token OIDC có claim `email` dùng được cho bước tích hợp với OpenZiti.

Khi đã có đủ 4 đầu ra trên, bạn có thể chuyển sang dựng mặt phẳng mạng ở Phase 2.
