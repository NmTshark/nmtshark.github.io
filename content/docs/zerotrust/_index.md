---
title: "Hướng dẫn Triển khai C-ZTNA: Từ Zero đến Zero Trust"
description: "Tài liệu tổng quan về kiến trúc và lộ trình triển khai hệ thống Context-Aware Zero Trust Network Access (C-ZTNA) dành cho SME."
toc: true
authors: []
tags: ["Zero Trust", "OpenZiti", "Keycloak", "FleetDM", "OPA", "Security"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-01-12'
lastmod: '2026-04-30'
draft: false
---

Tài liệu này hướng dẫn triển khai một mô hình **Context-Aware Zero Trust Network Access (C-ZTNA)** theo hướng thực dụng: tách rõ từng thành phần, đi theo từng phase, kiểm tra được ở mỗi mốc và dễ mở rộng khi cần demo hoặc triển khai lab.

Thay vì đặt niềm tin vào vị trí mạng nội bộ, kiến trúc này bám theo nguyên tắc:

- không tin mặc định bất kỳ user hay thiết bị nào,
- xác minh liên tục danh tính và trạng thái thiết bị,
- chỉ cấp quyền tối thiểu cho đúng dịch vụ cần truy cập,
- thu hồi quyền ngay khi posture thay đổi.

## Kiến trúc tổng thể

Hệ thống gồm 5 khối chính:

1. **Keycloak**: quản lý danh tính, SSO, OIDC và phát hành JWT.
2. **OpenZiti**: tạo overlay network, cấp identity và thực thi policy ở tầng kết nối.
3. **FleetDM + Osquery**: thu thập posture của endpoint theo chu kỳ.
4. **OPA**: ra quyết định dựa trên policy Rego.
5. **Posture Orchestrator**: đồng bộ dữ liệu giữa FleetDM, OPA và OpenZiti để tự động cách ly hoặc khôi phục thiết bị.

## Luồng xử lý chính

1. User đăng nhập qua Keycloak để lấy JWT.
2. OpenZiti dùng JWT đó để xác thực và cấp phiên truy cập.
3. FleetDM liên tục kiểm tra posture của endpoint.
4. Orchestrator lấy kết quả posture, gửi sang OPA để xin quyết định.
5. Nếu OPA trả về `quarantine`, Orchestrator đổi role/tag trên OpenZiti và thu hồi session đang hoạt động.
6. Nếu thiết bị sạch trở lại, Orchestrator cấp lại trạng thái `compliant` để cho phép truy cập dịch vụ phù hợp.

## Lộ trình triển khai

Nên đi đúng thứ tự dưới đây để tránh cấu hình chồng chéo và dễ debug:

| Phase | Mục tiêu | Kết quả đầu ra |
| --- | --- | --- |
| [Phase 1](phase1.md) | Dựng Keycloak | Có realm, user và OIDC client dùng cho Ziti |
| [Phase 2](phase2.md) | Dựng OpenZiti | Có controller, edge router và identity mẫu |
| [Phase 3](phase3.md) | Dựng FleetDM + Osquery | Có host online và policy posture trả kết quả |
| [Phase 4](phase4.md) | Dựng OPA | Có policy Rego trả về `compliant` hoặc `quarantine` |
| [Phase 5](phase5.md) | Nối Keycloak với Ziti | User đăng nhập Ziti bằng JWT từ Keycloak |
| [Phase 6](phase6.md) | Viết Orchestrator | Posture tự đồng bộ sang Ziti |
| [Phase 7](phase7.md) | Tạo service và access policy | Chỉ đúng role + đúng posture mới truy cập được |
| [Phase 8](phase8.md) | Chạy demo end-to-end | Kiểm chứng luồng cấp quyền, cách ly và khôi phục |
| [Phase 9](phase9.md) | Troubleshooting và checklist | Có danh sách kiểm tra trước khi demo hoặc bàn giao |

## Cách đọc bộ tài liệu này

Mỗi phase đều được viết theo cùng một cấu trúc:

- **Mục tiêu**: phase này giải quyết bài toán gì.
- **Chuẩn bị**: các thành phần cần sẵn sàng trước khi làm.
- **Các bước triển khai**: thứ tự thao tác nên làm.
- **Điểm kiểm tra**: dấu hiệu để xác nhận phase đã ổn.
- **Đầu ra**: những gì cần có để chuyển sang phase tiếp theo.

## Khuyến nghị triển khai lab

- Dùng DNS nội bộ hoặc file `hosts` để đặt tên nhất quán cho `keycloak`, `fleet`, `opa` và các dịch vụ demo.
- Giữ chung một convention tên identity, ví dụ `ep-<hostname>`, ngay từ đầu.
- Tách rõ role nghiệp vụ và role posture, ví dụ `#role.sales` khác `#posture.compliant`.
- Chỉ demo automation sau khi đã xác minh từng thành phần chạy độc lập.

Nếu bạn đi tuần tự từ Phase 1 đến Phase 9, gần như toàn bộ lỗi khó sẽ được chặn từ sớm thay vì dồn về cuối lúc tích hợp.
