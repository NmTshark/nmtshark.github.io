---
title: "Phase 5: Tích hợp Xác thực (Keycloak + Ziti)"
description: "Cấu hình External JWT Signer để OpenZiti chấp nhận quá trình đăng nhập thông qua Keycloak."
toc: true
authors: []
tags: ["Integration", "JWT Signer", "SSO", "OpenZiti", "Keycloak"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-02-09'
lastmod: '2026-04-30'
draft: false
weight: 5
---

Đây là phase ghép hai nửa quan trọng nhất của hệ thống: **danh tính từ Keycloak** và **quyền truy cập mạng từ OpenZiti**. Khi phase này hoàn tất, người dùng sẽ không còn đăng nhập bằng tài khoản cục bộ của Ziti mà dùng luôn SSO qua Keycloak.

## Mục tiêu

- khai báo Keycloak là nhà phát hành JWT hợp lệ,
- để OpenZiti tin tưởng token từ Keycloak,
- map user dựa trên claim `email`,
- xác minh luồng đăng nhập từ Ziti Desktop Edge.

## Chuẩn bị

- Phase 1 đã có realm, client và user,
- Phase 2 đã có identity với `external-id`,
- issuer URL của Keycloak đã ổn định,
- bạn xác nhận token thật sự có claim `email`.

## Bước 1: Tạo External JWT Signer

Trong OpenZiti, bạn cần khai báo một `Ext-JWT Signer` để controller biết:

- token đến từ đâu,
- public key nào dùng để kiểm tra chữ ký,
- claim nào dùng để map sang identity.

Các thông số cần đặc biệt cẩn thận:

- **Issuer**: URL realm của Keycloak, ví dụ `https://keycloak.lab.local/realms/ztna-lab`
- **Client ID**: `openziti-oidc`
- **Claims property**: `email`
- **JWKS / public key source**: phải lấy đúng từ Keycloak

Một lỗi nhỏ như dư dấu `/` ở cuối `issuer` cũng có thể làm toàn bộ flow thất bại.

## Bước 2: Gắn signer vào auth policy

Sau khi có signer, bạn cần cho auth policy của OpenZiti chấp nhận signer đó. Trong lab đơn giản, cập nhật trực tiếp `default auth-policy` là cách dễ theo dõi nhất:

```bash
ziti edge update auth-policy default --primary-ext-jwt-allowed --primary-ext-jwt-allowed-signers <SIGNER_ID>
```

Mục tiêu của bước này là nói với controller rằng:

- identity thuộc policy này được phép dùng external JWT,
- signer nào là signer hợp lệ.

## Bước 3: Kiểm tra lại mapping giữa token và identity

Đây là chỗ dễ lệch nhất trong toàn bộ tài liệu. Cần bảo đảm:

- user đăng nhập có `email` trong token,
- identity trên Ziti có `external-id` đúng bằng email đó,
- auth policy đang áp đúng lên identity cần thử nghiệm.

Nếu một trong ba điểm trên sai, user vẫn đăng nhập Keycloak thành công nhưng identity trong Ziti sẽ không active.

## Bước 4: Test luồng đăng nhập thực tế

Luồng mong đợi nên diễn ra như sau:

1. Người dùng import enrollment token vào Ziti Desktop Edge.
2. Trên client, người dùng bấm `Authorize`.
3. Trình duyệt mở ra và chuyển hướng sang Keycloak.
4. Người dùng đăng nhập bằng tài khoản hợp lệ.
5. Keycloak trả JWT về cho Ziti Desktop Edge.
6. Ziti Controller xác minh token và tạo API session.

Khi thành công, identity sẽ chuyển sang trạng thái active và có thể dùng để truy cập service khi policy cho phép.

## Điểm kiểm tra

Phase 5 hoàn tất khi:

- `Ext-JWT Signer` được tạo thành công,
- auth policy đã cho phép external JWT,
- identity có `hasApiSession = true` sau khi người dùng authorize,
- không còn phụ thuộc vào cơ chế password nội bộ của Ziti.

## Đầu ra của phase

Sau phase này, bạn cần có:

- SSO giữa Keycloak và OpenZiti,
- mapping danh tính hoạt động ổn định bằng `email`,
- người dùng có thể đăng nhập Ziti bằng JWT.

Khi lớp xác thực đã xong, bạn có thể bắt đầu tự động hóa quyết định posture ở Phase 6.
