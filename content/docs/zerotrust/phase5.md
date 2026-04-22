---
title: "Phase 5: Tích hợp Xác thực (Keycloak + Ziti)"
description: "Cấu hình External JWT Signer để OpenZiti chấp nhận quá trình đăng nhập thông qua Keycloak."
toc: true
authors: []
tags: ["Integration", "JWT Signer", "SSO", "OpenZiti", "Keycloak"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2023-11-05'
lastmod: '2023-11-05'
draft: false
weight: 5
---

Phase này là điểm giao thoa quan trọng nhất để ghép nối thành công Phase 1 (Định danh) và Phase 2 (Mạng lưới). Mục tiêu là phải làm cho Controller của OpenZiti tin tưởng và chấp nhận chuỗi JWT Token do Keycloak sinh ra thay vì dùng cơ chế mật khẩu nội bộ.

Hệ thống sử dụng cơ chế `Ext-JWT Signer` (Trình ký JWT bên ngoài) áp dụng trực tiếp lên `Default auth-policy` của OpenZiti.

## 1. Cấu hình External JWT Signer

Bạn sử dụng Ziti CLI hoặc giao diện ZAC để khai báo Keycloak làm nhà cung cấp JWT hợp lệ. Các thông số bắt buộc phải khớp tuyệt đối 100% với cấu hình của Keycloak:

* **Issuer:** URL gốc của realm Keycloak (Ví dụ: `https://keycloak.lab.local/realms/ztna-lab`). *Lưu ý không để dư dấu gạch chéo `/` ở cuối.*
* **Audience:** Bỏ trống hoặc thiết lập chuẩn theo cấu hình OIDC của Keycloak.
* **Client ID:** Bắt buộc nhập `openziti-oidc` (Client chúng ta đã tạo ở Phase 1).
* **Claims Property:** Nhập `email`. Đây là lệnh báo cho Ziti biết: "Hãy đọc trường Email trong token JWT, và đối chiếu xem nó có khớp với External ID của thiết bị đang xin kết nối không".

*Lưu ý: Bạn cũng cần lấy chứng chỉ public key (JWKS Endpoint) của Keycloak để Ziti có thể giải mã token. Sau khi tạo xong, hãy ghi lại **ID của Signer** này.*

## 2. Cập nhật Auth-Policy Mặc định

Theo thiết kế hệ thống tối ưu và ổn định nhất, thay vì tạo một Auth-Policy mới phức tạp, chúng ta sẽ **cập nhật trực tiếp Default auth-policy** để nó cho phép sử dụng Ext-JWT Signer.

Sử dụng công cụ `ziti-cli` để kích hoạt bằng câu lệnh sau:

```bash
# Đảm bảo Default policy chấp nhận trình ký bên ngoài
ziti edge update auth-policy default --primary-ext-jwt-allowed --primary-ext-jwt-allowed-signers <ID_CỦA_SIGNER_VỪA_TẠO>
```
3. Hiểu rõ luồng đăng nhập thực tế (Login Flow)
Việc cập nhật Default auth-policy giúp cho luồng trải nghiệm người dùng (UX) trên Ziti Desktop Edge (ZDE) diễn ra cực kỳ mượt mà và bảo mật:

Khởi tạo: Thiết bị được tạo Identity trên ZAC (như đã làm ở Phase 2) và người dùng tải file .jwt (Enrollment token) về máy.

Nạp chứng chỉ: Người dùng nạp file .jwt vào phần mềm Ziti Desktop Edge.

Xác thực SSO: Người dùng nhấn nút "Authorize" trên phần mềm. Lập tức ZDE sẽ mở trình duyệt web lên và trỏ về màn hình đăng nhập của Keycloak.

Cấp Token: Người dùng nhập User/Pass (hoặc quét MFA/FaceID) thành công. Keycloak ném JWT Token về lại cho ứng dụng ZDE.

Mở đường hầm: ZDE cầm Token đó trình diện cho Ziti Controller. Controller kiểm tra Token bằng Ext-JWT Signer vừa cấu hình. Nếu hợp lệ, một phiên làm việc (API Session) được tạo ra (hasApiSession=true) và đường hầm mạng được mở!

Bằng cách này, OpenZiti hoàn toàn không nắm giữ mật khẩu của người dùng, đảm bảo chuẩn Zero Trust tuyệt đối ở khâu quản trị danh tính.