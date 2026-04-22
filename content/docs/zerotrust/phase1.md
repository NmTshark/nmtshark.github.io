---
title: "Phase 1: Triển khai Identity Provider (Keycloak)"
description: "Hướng dẫn cài đặt Keycloak và thiết lập cơ chế xác thực OIDC cho hệ thống Zero Trust."
toc: true
authors: []
tags: ["Keycloak", "IAM", "OIDC", "JWT"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-01-12'
lastmod: '2026-01-19'
draft: false
weight: 1
---

Trong Phase này, chúng ta sẽ thiết lập Keycloak để trả lời câu hỏi đầu tiên của Zero Trust: "Bạn là ai?". Keycloak sẽ đóng vai trò là "cửa khẩu" duy nhất để cấp phép danh tính cho toàn bộ hệ thống.

## Khởi chạy Keycloak với Docker

Để đảm bảo tính cô lập và dễ dàng quản lý, Keycloak sẽ được chạy dưới dạng container cùng với cơ sở dữ liệu PostgreSQL.

Bạn cần tạo một file `docker-compose.yml` định nghĩa service Keycloak, phơi port `8080` (hoặc `8443` nếu dùng HTTPS) và thiết lập các biến môi trường cấu hình tài khoản Admin mặc định. Sau khi cấu hình xong, chạy lệnh `docker compose up -d` để khởi động dịch vụ. Đảm bảo container trạng thái `Healthy` trước khi tiếp tục.

## Cấu hình Realm và OIDC Client

Sau khi đăng nhập vào Keycloak Admin Console, chúng ta không sử dụng Realm `master` mà phải tạo một không gian làm việc riêng biệt.

1. **Tạo Realm:** Khởi tạo một Realm mới với tên `ztna-lab`. Đây sẽ là ranh giới logic cho toàn bộ user và ứng dụng của dự án.
2. **Tạo Client:** OpenZiti Desktop Edge sẽ giao tiếp với Keycloak thông qua giao thức OpenID Connect (OIDC). Bạn cần tạo một Client mới với cấu hình:
   * **Client ID:** `openziti-oidc` (Ghi nhớ ID này để cấu hình Ziti ở Phase 5).
   * **Client Authentication:** Chuyển sang `Off` (Thiết lập thành Public client vì Ziti Desktop Edge là ứng dụng cài trên máy khách, không thể lưu trữ Client Secret an toàn).
   * **Valid Redirect URIs:** Đặt là `*` trong môi trường lab (hoặc cấu hình URI callback cụ thể của Ziti Desktop Edge nếu triển khai thực tế).

## Tạo Người dùng và Claims Mapping

Hệ thống OpenZiti trong kiến trúc này yêu cầu ánh xạ (mapping) người dùng thông qua trường `email`. Do đó, cấu hình JWT Token phải được tinh chỉnh.

### 1. Khởi tạo User Demo
Truy cập mục Users và tạo các tài khoản đại diện cho các phòng ban khác nhau:
* `alice.hr@lab.local` (Phòng Nhân sự)
* `alice.sales@lab.local` (Phòng Kinh doanh)
* `tinh.sales@lab.local` (Phòng Kinh doanh)

Lưu ý: Bắt buộc phải điền đầy đủ trường Email và thiết lập mật khẩu (tắt trạng thái Temporary) cho các user này.

### 2. Cấu hình Claims Mapping
Mặc định, JWT Token của Keycloak có thể không chứa đầy đủ các Claim (thuộc tính) mà OpenZiti cần. 
Trong phần **Client Scopes** của client `openziti-oidc`, bạn cần kiểm tra và thêm bộ ánh xạ (Mapper). Đảm bảo mapper `email` được đưa vào JWT Token một cách rõ ràng. Token này khi sinh ra sẽ chứa thông tin user id, phân quyền và thời gian sống (expiration time) để Ziti Controller có thể kiểm chứng sau này.