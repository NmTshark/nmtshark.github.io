---
title: "Hướng dẫn Triển khai C-ZTNA: Từ Zero đến Zero Trust"
description: "Tài liệu tổng quan về kiến trúc và các bước triển khai hệ thống Context-Aware Zero Trust Network Access (C-ZTNA) dành cho SME."
toc: true
authors: []
tags: ["Zero Trust", "OpenZiti", "Keycloak", "FleetDM", "OPA", "Security"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-01-12'
lastmod: '2026-03-31'
draft: false
---

Chào mừng bạn đến với tài liệu hướng dẫn triển khai hệ thống **Context-Aware Zero Trust Network Access (C-ZTNA)**. Dự án này nhằm thay thế mô hình bảo mật "lâu đài và hào nước" (Castle and Moat) truyền thống bằng triết lý "Không bao giờ tin tưởng, Luôn luôn xác minh".

Đây là trang gốc (root) chứa toàn bộ lộ trình xây dựng hệ thống từ con số 0.

## Kiến trúc Hệ thống Tổng thể

Hệ thống C-ZTNA được lắp ghép từ 5 thành phần mã nguồn mở độc lập, giao tiếp với nhau qua API để tạo thành một vòng lặp bảo mật khép kín (SOAR). Thay vì phụ thuộc vào một giải pháp đắt tiền duy nhất, kiến trúc này phân tách rõ ràng các vai trò:

1. **Keycloak (Identity Provider):** Chịu trách nhiệm quản lý định danh, xác thực người dùng (SSO) và cấp phát mã thông báo JWT.
2. **FleetDM & Osquery (Posture Agent):** Hệ thống thu thập tình trạng bảo mật của thiết bị điểm cuối, giám sát các tiến trình và cấu hình hệ điều hành theo thời gian thực.
3. **Open Policy Agent - OPA (Policy Decision Point):** Bộ não trung tâm đánh giá các chính sách truy cập dựa trên ngôn ngữ Rego, phân xử việc cấp phép hoặc cách ly.
4. **OpenZiti (Policy Enforcement Point):** Nền tảng mạng Overlay tạo các đường hầm vi phân đoạn (mTLS) và thực thi lệnh ngắt/mở kết nối ở tầng TCP/IP.
5. **Posture Orchestrator (Middleware):** Script Python đóng vai trò "dải băng keo", đồng bộ hóa quyết định từ OPA sang OpenZiti.

## Lộ trình Triển khai (Phase 1 - Phase 9)

Hệ thống được triển khai tuần tự qua 9 giai đoạn (Phases) để đảm bảo tính logic và dễ dàng khắc phục sự cố:

* **[Phase 1: Triển khai Identity Provider (Keycloak)](phase1.md)** - Xây dựng hệ thống định danh.
* **[Phase 2: Thiết lập mạng lõi (OpenZiti)](phase2.md)** - Xây dựng nền tảng mạng Overlay.
* **[Phase 3: Giám sát Posture (FleetDM & Osquery)](phase3.md)** - Cài đặt tác nhân giám sát thiết bị.
* **[Phase 4: Bộ não ra quyết định (OPA)](phase4.md)** - Viết luật đánh giá bảo mật.
* **[Phase 5: Tích hợp Xác thực (Keycloak + Ziti)](phase5.md)** - Ủy quyền đăng nhập bằng JWT.
* **[Phase 6: Posture Orchestrator](phase6.md)** - Lập trình luồng tự động hóa.
* **[Phase 7: Dịch vụ và Chính sách Mạng (Zero Trust Rules)](phase7.md)** - Định nghĩa Dark Services.
* **[Phase 8: Kịch bản Thực nghiệm (Demo)](phase8.md)** - Chạy thử nghiệm các luồng tấn công.
* **[Phase 9: Xử lý sự cố & Checklist](phase9.md)** - Các lỗi thường gặp và cách khắc phục.