---
title: "Phase 9: Xử lý sự cố & Checklist (Troubleshooting)"
description: "Tổng hợp các lỗi thường gặp trong quá trình triển khai và Checklist kiểm tra chéo trước giờ Demo."
toc: true
authors: []
tags: ["Troubleshooting", "Debug", "Checklist", "Zero Trust"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2023-11-09'
lastmod: '2023-11-09'
draft: false
weight: 9
---

Trong quá trình triển khai một hệ thống phân tán gồm nhiều "bánh răng" độc lập như C-ZTNA, việc xảy ra trục trặc ở các điểm nối là không thể tránh khỏi. Phase cuối cùng này sẽ cung cấp cho bạn các hướng dẫn khắc phục sự cố (Troubleshooting) nhanh chóng và một danh sách kiểm tra (Checklist) sống còn trước giờ Demo.

## 1. Xử lý các lỗi phổ biến (Troubleshooting)

Dưới đây là 2 lỗi thường gặp nhất liên quan đến việc xác thực và thực thi mạng.

### Lỗi 1: Đăng nhập Keycloak thành công nhưng Ziti Desktop Edge không Active
* **Biểu hiện:** Người dùng nhấn Authorize, đăng nhập Keycloak xong nhưng app Ziti Desktop Edge vẫn báo lỗi mạng hoặc giao diện cứ xám xịt.
* **Cách kiểm tra:** Đăng nhập vào Ziti Admin Console (ZAC), tìm Identity đó và xem thuộc tính `hasApiSession` có đang là `true` không.
* **Cách khắc phục:** * Nếu `hasApiSession = false`, có nghĩa là Controller đã từ chối Token.
  * Hãy kiểm tra lại xem Identity có đang nằm đúng ở `Default auth-policy` không.
  * Đảm bảo Signer Ext-JWT đã được Enable trên policy Mặc định này. 
  * Kiểm tra kỹ thông số Issuer và Audience trong cấu hình Signer, đặc biệt là lỗi dư dấu gạch chéo `/` ở cuối chuỗi URL của Keycloak.

### Lỗi 2: Trạng thái Posture đã đổi sang Quarantine nhưng ứng dụng vẫn truy cập được
* **Biểu hiện:** Máy bị đánh tag `Quarantine` trên ZAC, ping báo đứt, nhưng ứng dụng web đang mở sẵn thì load vẫn được thêm một lúc lâu.
* **Cách khắc phục:** * Theo lý thuyết mạng, khi đổi tag, các kết nối TCP *đang tồn tại* vẫn có thể sống lây lất cho đến khi bị Timeout