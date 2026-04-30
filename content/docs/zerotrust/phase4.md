---
title: "Phase 4: Bộ não ra quyết định (OPA)"
description: "Sử dụng Open Policy Agent (OPA) và ngôn ngữ Rego để xây dựng điểm ra quyết định chính sách (PDP)."
toc: true
authors: []
tags: ["OPA", "Rego", "Policy Engine", "Zero Trust"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-02-02'
lastmod: '2026-04-30'
draft: false
weight: 4
---

FleetDM giúp bạn biết thiết bị đang vi phạm gì, nhưng nó không nên là nơi quyết định toàn bộ hành động truy cập mạng. Phase này tách logic ra quyết định sang **OPA** để hệ thống linh hoạt, dễ sửa và dễ mở rộng hơn.

## Mục tiêu

- dựng OPA ở chế độ server,
- viết policy Rego đầu tiên,
- định nghĩa đầu vào và đầu ra rõ ràng,
- trả về quyết định đơn giản như `compliant` hoặc `quarantine`.

## Chuẩn bị

- dữ liệu posture từ Phase 3,
- một máy chủ hoặc container chạy OPA,
- quy ước thống nhất về tên policy vi phạm từ FleetDM,
- kế hoạch rõ ràng xem vi phạm nào sẽ dẫn đến cách ly.

## Bước 1: Khởi chạy OPA

Một cách nhanh gọn là chạy OPA bằng Docker:

```bash
docker run -d --name opa -p 8181:8181 openpolicyagent/opa run --server
```

Sau khi chạy, OPA sẽ lắng nghe API trên cổng `8181`. Từ lúc này, các thành phần khác có thể gửi JSON input để xin quyết định.

## Bước 2: Thiết kế contract input/output

Trước khi viết Rego, nên chốt format đầu vào để Orchestrator gọi nhất quán. Một input tối thiểu có thể gồm:

- hostname của endpoint,
- danh sách `failing_policies`,
- thông tin bổ sung như user, team hoặc nguồn dữ liệu nếu bạn cần.

Ví dụ tư duy:

- nếu `failing_policies` rỗng thì `decision = compliant`,
- nếu chứa policy bị cấm thì `decision = quarantine`.

Việc chốt contract sớm sẽ giúp Phase 6 đơn giản hơn rất nhiều.

## Bước 3: Viết policy Rego đầu tiên

Bạn có thể bắt đầu bằng một policy nhỏ, dễ đọc:

```rego
package posture

default decision = "compliant"

decision = "quarantine" if {
  input.failing_policies[_] == "Forbidden process running"
}
```

Policy này đủ tốt cho lab ban đầu vì nó thể hiện đúng điều bạn muốn chứng minh:

- OPA nhận input posture,
- OPA trả về một quyết định ngắn gọn,
- quyết định đó sẽ được OpenZiti thực thi ở phase sau.

## Bước 4: Nạp policy vào OPA

Sau khi lưu file `posture.rego`, nạp policy vào OPA:

```bash
curl -X PUT http://localhost:8181/v1/policies/posture --data-binary @posture.rego
```

Bạn cũng nên test nhanh bằng cách gửi một request JSON mẫu để xem OPA có thật sự trả về `quarantine` khi policy bị fail hay không.

## Bước 5: Nghĩ theo hướng mở rộng

Khi lab cơ bản đã chạy, bạn có thể mở rộng logic mà không phải đụng vào OpenZiti:

- cách ly khi firewall bị tắt,
- cách ly khi thiếu EDR,
- yêu cầu bản vá tối thiểu,
- cho phép ngoại lệ theo nhóm máy hoặc theo môi trường.

Đó chính là lợi ích lớn nhất của việc tách PDP ra thành một lớp độc lập.

## Điểm kiểm tra

Phase 4 hoàn tất khi:

- OPA chạy được ở chế độ server,
- policy Rego nạp thành công,
- input test trả về đúng `compliant` hoặc `quarantine`,
- bạn hiểu rõ rule nào đang dẫn tới hành động cách ly.

## Đầu ra của phase

Sau phase này, bạn cần có:

- endpoint OPA sẵn sàng nhận truy vấn,
- một policy Rego hoạt động,
- contract ra quyết định đủ ổn để Orchestrator gọi ở Phase 6.

Tiếp theo, bạn sẽ nối danh tính từ Keycloak sang OpenZiti ở Phase 5 để hoàn thiện lớp xác thực.
