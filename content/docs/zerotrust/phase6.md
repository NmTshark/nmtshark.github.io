---
title: "Phase 6: Posture Orchestrator"
description: "Lập trình kịch bản Python đóng vai trò tự động hóa luồng SOAR cho hệ thống C-ZTNA."
toc: true
authors: []
tags: ["Python", "Automation", "SOAR", "API"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-02-16'
lastmod: '2026-04-30'
draft: false
weight: 6
---

Nếu OPA là bộ não ra quyết định và OpenZiti là lớp thực thi, thì **Posture Orchestrator** chính là bộ điều phối. Đây là thành phần nối dữ liệu giữa FleetDM, OPA và OpenZiti để biến các policy posture thành hành động thực tế trên mạng.

## Mục tiêu

- lấy dữ liệu posture từ FleetDM,
- gửi input sang OPA để xin quyết định,
- map host sang identity tương ứng trong OpenZiti,
- cập nhật role posture và thu hồi session nếu cần,
- lưu log để phục vụ debug và báo cáo.

## Chuẩn bị

- FleetDM đã có host và policy hoạt động,
- OPA đã trả về quyết định ổn định,
- OpenZiti đã có identity theo convention `ep-<hostname>`,
- token hoặc API credential đủ quyền để gọi FleetDM và Ziti API.

## Luồng xử lý đề xuất

Một vòng lặp của Orchestrator nên làm lần lượt:

1. Gọi FleetDM API để lấy danh sách host online.
2. Đọc posture hoặc danh sách `failing_policies` của từng host.
3. Gửi JSON input sang OPA.
4. Nhận `decision` từ OPA.
5. Map hostname sang identity trong Ziti theo prefix `ep-`.
6. Giữ nguyên role nghiệp vụ, chỉ cập nhật role posture.
7. Nếu quyết định là `quarantine`, thu hồi session đang hoạt động.
8. Ghi log sự kiện và trạng thái mới.

Điểm mấu chốt là Orchestrator **không được phá hỏng role sẵn có**. Nó chỉ nên thay phần posture như:

- `#posture.compliant`
- `#posture.quarantine`

## Thiết kế file cấu hình

Nên gom toàn bộ endpoint và tham số vào `config.yaml` thay vì hard-code trong Python:

```yaml
poll_interval_seconds: 5

fleet:
  url: "https://fleet.lab.local:8443"
  api_token: "TOKEN_FROM_FLEET"

opa:
  url: "http://opa:8181"
  decision_path: "/v1/data/posture/decision"

mapping:
  ziti_identity_prefix: "ep-"

posture_attributes:
  compliant: "posture.compliant"
  quarantine: "posture.quarantine"

enforcement:
  revoke_sessions_on_quarantine: true
```

File này đủ để bạn đổi môi trường lab mà gần như không phải đụng đến logic code.

## Những điểm nên làm rõ trong code

Khi viết Orchestrator, nên tách thành các hàm nhỏ:

- lấy host từ FleetDM,
- trích xuất policy failing,
- gọi OPA,
- tìm identity trong Ziti,
- cập nhật role attributes,
- revoke session,
- ghi log SQLite hoặc CSV.

Việc tách hàm sẽ giúp debug dễ hơn nhiều khi một bước nào đó trả lỗi `401`, `404` hoặc timeout.

## Điểm kiểm tra

Phase 6 hoàn tất khi:

- Orchestrator chạy lặp ổn định,
- host vi phạm bị đổi từ `posture.compliant` sang `posture.quarantine`,
- session đang hoạt động bị thu hồi khi cần,
- log thể hiện rõ ai bị đổi trạng thái, lúc nào và vì lý do gì.

## Đầu ra của phase

Sau phase này, bạn cần có:

- một vòng lặp tự động hóa hoàn chỉnh,
- posture được phản ánh sang OpenZiti gần như theo thời gian thực,
- khả năng cách ly hoặc khôi phục endpoint mà không thao tác tay trên ZAC.

Từ đây, bạn đã sẵn sàng định nghĩa service và access policy thật sự ở Phase 7.
