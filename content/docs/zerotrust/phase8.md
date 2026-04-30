---
title: "Phase 8: Kịch bản Thực nghiệm (Demo)"
description: "Hướng dẫn thực hiện các kịch bản kiểm thử để đánh giá độ trễ và khả năng bảo vệ tự động của hệ thống."
toc: true
authors: []
tags: ["Testing", "Demo", "Incident Response", "Zero Trust"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-03-02'
lastmod: '2026-04-30'
draft: false
weight: 8
---

Sau khi hoàn tất phần dựng hạ tầng và policy, bạn cần một phase demo rõ ràng để chứng minh hệ thống không chỉ "đúng cấu hình" mà còn **đúng hành vi**. Phase này nên được chạy tuần tự, có quan sát, có log và có đầu ra đủ thuyết phục.

## Mục tiêu

- kiểm chứng luồng đăng nhập hợp lệ,
- chứng minh khả năng phát hiện vi phạm posture,
- chứng minh khả năng cách ly tự động,
- chứng minh khả năng khôi phục quyền truy cập sau khi thiết bị sạch trở lại.

## Cách chuẩn bị demo

Trước khi chạy từng kịch bản, nên mở sẵn:

- Ziti Desktop Edge trên máy client,
- dashboard FleetDM,
- log của Orchestrator,
- giao diện OPA nếu bạn có bật log,
- ZAC hoặc CLI để theo dõi role/session của identity.

Làm vậy sẽ giúp demo mạch lạc hơn nhiều vì người xem nhìn thấy nguyên nhân và hệ quả ở từng lớp.

## Kịch bản A: Truy cập hợp lệ

Mục tiêu của kịch bản này là chứng minh user đúng và máy sạch thì được cấp quyền.

Thao tác:

1. Import identity vào Ziti Desktop Edge.
2. Nhấn `Authorize`.
3. Đăng nhập Keycloak bằng tài khoản hợp lệ, ví dụ `alice.sales@lab.local`.
4. Truy cập `http://sales-web.demo`.

Kết quả mong đợi:

- identity chuyển sang trạng thái active,
- service Sales mở bình thường,
- FleetDM không ghi nhận policy failing,
- role posture hiện tại là `compliant`.

## Kịch bản B: Phát hiện vi phạm và cách ly

Mục tiêu là chứng minh hệ thống xác minh liên tục chứ không chỉ kiểm tra một lần lúc đăng nhập.

Thao tác:

1. Trong lúc đang truy cập `sales-web.demo`, bật tiến trình bị cấm như `xmrig.exe` hoặc mô phỏng một posture fail tương đương.
2. Chờ FleetDM quét lại.

Kết quả mong đợi:

- FleetDM đánh dấu host `failing`,
- Orchestrator gửi input sang OPA,
- OPA trả `quarantine`,
- Orchestrator đổi role posture và revoke session,
- truy cập đến `sales-web.demo` bị ngắt.

Nếu bạn có đo thời gian, hãy tách rõ:

- thời gian phát hiện vi phạm,
- thời gian OPA ra quyết định,
- thời gian OpenZiti thực thi ngắt kết nối.

## Kịch bản C: Self-remediation

Mục tiêu là chứng minh máy bị cách ly không bị "cắt khỏi thế giới", mà vẫn còn đường để tự sửa lỗi.

Thao tác:

1. Khi máy đang ở trạng thái `quarantine`, truy cập `http://remediate.demo`.
2. Làm theo hướng dẫn khắc phục trên trang remediation.
3. Dừng tiến trình vi phạm hoặc khôi phục cấu hình an toàn.

Kết quả mong đợi:

- `remediate.demo` vẫn truy cập được,
- dịch vụ nghiệp vụ vẫn bị khóa,
- sau chu kỳ quét tiếp theo, FleetDM trả trạng thái sạch,
- Orchestrator chuyển posture về `compliant`,
- quyền truy cập service nghiệp vụ được mở lại.

## Kịch bản D: Hết hạn phiên

Mục tiêu là chứng minh quyền truy cập không tồn tại vĩnh viễn.

Thao tác:

1. Chờ JWT hết hạn hoặc chủ động revoke session từ phía Keycloak hoặc Ziti.
2. Quan sát trạng thái trên client.

Kết quả mong đợi:

- session hiện tại bị vô hiệu,
- Ziti Desktop Edge yêu cầu authorize lại,
- người dùng phải lấy token mới mới tiếp tục truy cập được.

## Điểm kiểm tra

Phase 8 hoàn tất khi cả 4 kịch bản đều có thể tái hiện với kết quả nhất quán:

- user đúng, máy sạch thì vào được,
- máy vi phạm thì bị cách ly,
- máy bị cách ly vẫn vào được đường remediation,
- phiên hết hạn thì phải xác thực lại.

## Đầu ra của phase

Sau phase này, bạn cần có:

- bộ demo end-to-end hoàn chỉnh,
- log và màn hình minh chứng đủ rõ cho báo cáo hoặc bảo vệ,
- dữ liệu thực tế để tinh chỉnh polling interval, policy và UX của hệ thống.

Khi demo đã mượt, phase cuối cùng sẽ giúp bạn chốt lại checklist và hướng xử lý sự cố thường gặp.
