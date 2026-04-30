---
title: "Phase 7: Dịch vụ và Chính sách Mạng (Zero Trust Rules)"
description: "Định nghĩa các Dark Services và cấu hình luật định tuyến mạng dựa trên ngữ cảnh kép của OpenZiti."
toc: true
authors: []
tags: ["Micro-segmentation", "Dark Network", "Policies", "Zero Trust"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-02-23'
lastmod: '2026-04-30'
draft: false
weight: 7
---

Đây là phase biến toàn bộ phần nền tảng trước đó thành **quyền truy cập thật sự**. Thay vì mở cổng dịch vụ ra mạng nội bộ hoặc Internet, bạn sẽ tạo các dark service và chỉ cho phép đúng identity phù hợp kết nối đến.

## Mục tiêu

- tạo service đại diện cho ứng dụng nội bộ,
- tách rõ dịch vụ nghiệp vụ và dịch vụ remediation,
- viết access policy kết hợp cả role nghiệp vụ lẫn posture,
- chứng minh micro-segmentation hoạt động đúng.

## Chuẩn bị

- OpenZiti fabric đang chạy ổn định,
- identity đã có role nghiệp vụ,
- Orchestrator đã có thể gắn posture role,
- ứng dụng đích đã sẵn sàng trong LAN hoặc môi trường lab.

## Bước 1: Xác định nhóm service cần công bố

Trong mô hình lab tối thiểu, bạn nên tách thành 3 nhóm:

- `hr-peer`: dịch vụ cho khối nhân sự,
- `sales-peer`: dịch vụ cho khối kinh doanh,
- `remediate`: dịch vụ hướng dẫn khắc phục sự cố.

Mỗi nhóm service nên đại diện cho một nhu cầu truy cập rõ ràng. Cách tách này sẽ giúp policy dễ đọc hơn và demo cũng trực quan hơn.

## Bước 2: Tạo intercept và host config

Mỗi service thường cần hai phần:

- **Intercept config**: tên miền ảo và port mà client sẽ thấy,
- **Host config**: IP thật và port thật của ứng dụng phía sau.

Ví dụ cho dịch vụ Sales:

```bash
ziti edge create config sales-intercept intercept.v1 '{"protocols":["tcp"],"addresses":["sales-web.demo"],"portRanges":[{"low":80,"high":80}]}'
ziti edge create config sales-host host.v1 '{"protocol":"tcp","address":"<IP_SALES_SERVER>","port":80}'
ziti edge create service sales-peer --configs sales-intercept,sales-host --role-attributes "@sales-peer"
```

Khi tạo xong, dịch vụ đã tồn tại trên fabric nhưng chưa ai truy cập được cho đến khi bạn viết policy.

## Bước 3: Viết policy cho dịch vụ nghiệp vụ

Đây là nơi kiến trúc context-aware thể hiện rõ nhất. Ví dụ, chỉ máy của phòng Sales và đang sạch mới được vào ứng dụng Sales:

```bash
ziti edge create service-policy sales-dial Dial \
  --identity-roles "#role.sales,#posture.compliant" \
  --service-roles "@sales-peer" \
  --semantic AllOf
```

Ý nghĩa của `AllOf`:

- phải là người/thiết bị thuộc Sales,
- đồng thời phải đang ở posture `compliant`.

Thiếu một trong hai điều kiện là bị từ chối.

## Bước 4: Viết policy cho dịch vụ remediation

Service remediation nên hoạt động ngay cả khi endpoint bị cách ly. Ví dụ:

```bash
ziti edge create service-policy remediate-dial Dial \
  --identity-roles "#posture.compliant,#posture.quarantine" \
  --service-roles "@remediate" \
  --semantic AnyOf
```

Ý nghĩa của `AnyOf`:

- máy sạch truy cập được,
- máy đang bị quarantine cũng vẫn truy cập được để tự xử lý sự cố.

Đây là điểm làm cho mô hình của bạn mềm hơn, thực tế hơn và thân thiện hơn với người dùng cuối.

## Điểm kiểm tra

Phase 7 hoàn tất khi:

- service đã được tạo thành công,
- policy của service nghiệp vụ dùng cả role và posture,
- policy remediation vẫn cho phép máy bị cách ly truy cập,
- người dùng khác phòng ban hoặc sai posture không thể chui qua dịch vụ không thuộc quyền của họ.

## Đầu ra của phase

Sau phase này, bạn cần có:

- dark service cho từng ứng dụng,
- service policy rõ ràng theo nguyên tắc quyền tối thiểu,
- đường remediation luôn sẵn để hỗ trợ tự khôi phục.

Đây là bước cuối về cấu hình. Phase 8 sẽ dùng toàn bộ stack để chạy demo end-to-end.
