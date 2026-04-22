---
title: "Phase 8: Kịch bản Thực nghiệm (Demo)"
description: "Hướng dẫn thực hiện 4 kịch bản kiểm thử để đánh giá độ trễ và khả năng bảo vệ tự động của hệ thống."
toc: true
authors: []
tags: ["Testing", "Demo", "Incident Response", "Zero Trust"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2023-11-08'
lastmod: '2023-11-08'
draft: false
weight: 8
---

Mọi thiết lập về định danh, mạng lưới, giám sát và tự động hóa đã hoàn tất! Để chứng minh hệ thống C-ZTNA hoạt động hoàn hảo trong môi trường thực tế, bạn cần chạy tuần tự 4 kịch bản kiểm thử (Test Cases) sau đây.

## Kịch bản A: Truy cập hợp lệ (Happy Path)

Kịch bản này chứng minh luồng đăng nhập Zero Trust cơ bản thông qua cơ chế SSO của Keycloak.

* **Thao tác:** 1. Người dùng mở ứng dụng Ziti Desktop Edge (ZDE) trên máy trạm.
  2. Click vào nút **Authorize**. Trình duyệt sẽ tự động bật lên và chuyển hướng đến trang đăng nhập của Keycloak.
  3. Đăng nhập bằng tài khoản hợp lệ, ví dụ: `alice.sales@lab.local`.
* **Kết quả kỳ vọng:** * Identity trên giao diện ZDE chuyển sang trạng thái `Active`. 
  * Người dùng mở trình duyệt và truy cập thành công vào tên miền ảo `http://sales-web.demo`.

## Kịch bản B: Tấn công mã độc (Vi phạm Posture)

Kịch bản này chứng minh khả năng "Xác minh liên tục" (Continuous Verification) và tốc độ ngắt mạng tự động (Automated SOAR) của hệ thống.

* **Thao tác:** Trong lúc người dùng đang làm việc bình thường trên trang Sales, hãy chạy một phần mềm bị cấm (ví dụ: bật file giả lập mã độc đào coin `XMRig` hoặc tắt Windows Defender) trên máy trạm đó.
* **Kết quả kỳ vọng:** 1. Khoảng vài giây sau, Terminal của Posture Orchestrator nhảy log thông báo nhận được sự kiện vi phạm từ FleetDM.
  2. OPA trả về phán quyết `quarantine`. Orchestrator lập tức gọi Ziti API để đổi nhãn thiết bị và thu hồi Session.
  3. Đường hầm mạng đến `sales-web.demo` **bị cắt đứt ngay lập tức** (Website quay vòng vòng rồi báo lỗi *Site can't be reached*). 
  *Thời gian đo lường từ lúc phát hiện đến khi ngắt mạng (TTE) chỉ rơi vào khoảng ~35 mili-giây, tổng thời gian tự động phản hồi ~10.12 giây.*

## Kịch bản C: Hạ cánh mềm (Self-Remediation)

Kịch bản này chứng minh tính nhân văn và tối ưu trải nghiệm người dùng (UX) của kiến trúc. Thay vì cắt Internet hoàn toàn, hệ thống mở đường cho nhân viên tự cứu lấy mình.

* **Thao tác:** Ngay khi bị mất mạng Sales (ở Kịch bản B), người dùng gõ URL của trang Trạm y tế: `http://remediate.demo` (hoặc `helpdesk.demo` tùy bạn đặt tên).
* **Kết quả kỳ vọng:** * Trang web Helpdesk vẫn hiển thị bình thường dù máy đang bị cách ly. 
  * Nội dung trang web hiển thị thông báo lỗi (ví dụ: *Hệ thống phát hiện phần mềm đào coin*). 
  * Người dùng làm theo hướng dẫn, tự tay tắt tiến trình `XMRig` đi. 
  * Đợi khoảng 10 giây để FleetDM quét lại và báo máy "sạch". Orchestrator tự động cấp lại thẻ `compliant`. Mạng Sales tự động thông luồng trở lại mà không cần gọi IT!

## Kịch bản D: Hết hạn phiên (Session Timeout)

Kịch bản này chứng minh hệ thống không cấp quyền truy cập vĩnh viễn, ngăn chặn rủi ro Session Hijacking.

* **Thao tác:** Đợi cho đến khi JWT Token hết hạn (theo thời gian Expiration cấu hình trên Keycloak), hoặc Admin chủ động ép hết hạn (Revoke) phiên đăng nhập từ phía máy chủ Keycloak.
* **Kết quả kỳ vọng:** Controller của OpenZiti tự động quét và cắt đứt mọi kết nối mạng hiện tại. Ziti Desktop Edge chuyển sang trạng thái xám và yêu cầu người dùng phải click Authorize lại từ đầu để lấy Token mới.

---