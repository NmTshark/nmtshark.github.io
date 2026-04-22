---
title: "Phase 7: Dịch vụ và Chính sách Mạng (Zero Trust Rules)"
description: "Định nghĩa các Dark Services và cấu hình luật định tuyến mạng dựa trên ngữ cảnh kép của OpenZiti."
toc: true
authors: []
tags: ["Micro-segmentation", "Dark Network", "Policies", "Zero Trust"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2023-11-07'
lastmod: '2023-11-07'
draft: false
weight: 7
---

Đây là bước cấu hình cuối cùng trên mặt phẳng điều khiển. Chúng ta sẽ tạo ra các "Dịch vụ Tàng hình" (Dark Services) và viết luật (Policy) ép buộc hệ thống mạng phải thỏa mãn đồng thời cả điều kiện về Phân quyền (Role) VÀ Sức khỏe thiết bị (Posture).

## 1. Khởi tạo "Dịch vụ Tàng hình" (Dark Services)

Trong OpenZiti, dịch vụ mạng không được phơi bày ra Internet. Thay vào đó, chúng ta tạo ra các điểm neo logic. Chúng ta sẽ cấu hình 3 dịch vụ tương ứng với 3 ứng dụng đích:
1. `hr-peer`: Dịch vụ dành riêng cho bộ phận Nhân sự.
2. `sales-peer`: Dịch vụ dành riêng cho bộ phận Kinh doanh.
3. `remediate`: Trang web Helpdesk để nhân viên tự sửa lỗi (dịch vụ này luôn mở).

Đối với mỗi dịch vụ, bạn cần định nghĩa 2 thành phần bằng công cụ `ziti-cli`:
* **Intercept Config:** Là cổng vào (cửa trước). Nó định nghĩa địa chỉ DNS ảo trên máy nhân viên (Ví dụ: `sales-web.demo:80`).
* **Host Config (Bind):** Là cổng ra (cửa sau). Nó chỉ định cho Router nội bộ biết cần trút dữ liệu đến địa chỉ IP thật nào trong mạng LAN.

**Ví dụ lệnh tạo dịch vụ Sales:**
```bash
# 1. Tạo cấu hình Intercept
ziti edge create config sales-intercept intercept.v1 '{"protocols":["tcp"],"addresses":["sales-web.demo"],"portRanges":[{"low":80,"high":80}]}'

# 2. Tạo cấu hình Host
ziti edge create config sales-host host.v1 '{"protocol":"tcp","address":"<IP_MÁY_CHỦ_SALES>","port":80}'

# 3. Gộp lại thành Service và dán nhãn @sales-peer
ziti edge create service sales-peer --configs sales-intercept,sales-host --role-attributes "@sales-peer"
```
2. Thiết lập Chính sách Truy cập (Access Policies)
Sức mạnh thực sự của một hệ thống Context-Aware ZTNA nằm ở tham số toán tử logic (Semantic). Chúng ta sẽ sử dụng tham số này để chặn đứng sự leo thang đặc quyền.

A. Chính sách truy cập nghiệp vụ (Ví dụ: Sales Dial Policy)
Luật này quy định: Chỉ những máy tính có thẻ thuộc phòng Kinh doanh (#role.sales) VÀ phải mang nhãn An toàn (#posture.compliant) thì mới được phép kết nối. Chúng ta dùng toán tử AllOf (Tất cả điều kiện phải đúng).

Bash
ziti edge create service-policy sales-dial Dial \
  --identity-roles "#role.sales,#posture.compliant" \
  --service-roles "@sales-peer" \
  --semantic AllOf
B. Chính sách cho Trạm Y Tế (Remediation Dial Policy)
Trang Web Helpdesk nội bộ có nhiệm vụ hướng dẫn nhân viên cách diệt virus/sửa lỗi cấu hình. Do đó, nó phải hoạt động ngay cả khi máy tính đang bị dính mã độc. Chúng ta dùng toán tử AnyOf (Cho phép thẻ Compliant HOẶC Quarantine đều được truy cập).

Bash
ziti edge create service-policy remediate-dial Dial \
  --identity-roles "#posture.compliant,#posture.quarantine" \
  --service-roles "@remediate" \
  --semantic AnyOf
Kết quả đạt được
Với cấu hình trên, khi Posture Orchestrator (ở Phase 6) phát hiện mã độc và đổi nhãn của thiết bị từ compliant sang quarantine, thiết bị sẽ không còn thỏa mãn điều kiện AllOf của chính sách Sales nữa. Lập tức, kết nối tới ứng dụng Sales bị chặn đứng. Tuy nhiên, kết nối tới ứng dụng Remediation vẫn được duy trì vì nó thỏa mãn điều kiện AnyOf!