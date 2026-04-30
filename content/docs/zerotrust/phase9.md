---
title: "Phase 9: Xử lý sự cố & Checklist (Troubleshooting)"
description: "Tổng hợp các lỗi thường gặp trong quá trình triển khai và checklist kiểm tra chéo trước giờ demo."
toc: true
authors: []
tags: ["Troubleshooting", "Debug", "Checklist", "Zero Trust"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-03-09'
lastmod: '2026-04-30'
draft: false
weight: 9
---

Một hệ thống C-ZTNA có nhiều điểm nối giữa IAM, posture, policy engine và network fabric, nên lỗi thường không nằm ở một nơi duy nhất. Phase cuối này giúp bạn gom lại các lỗi phổ biến nhất và có checklist rõ ràng trước khi demo hoặc bàn giao.

## Mục tiêu

- khoanh vùng lỗi nhanh theo từng lớp,
- giảm thời gian debug trong lúc tích hợp,
- có checklist kiểm tra chéo trước giờ chạy thử hoặc trình bày.

## Nhóm lỗi 1: Đăng nhập Keycloak thành công nhưng Ziti không active

Biểu hiện:

- người dùng bấm `Authorize`,
- đăng nhập Keycloak thành công,
- nhưng identity trên Ziti Desktop Edge vẫn không active hoặc không mở được service.

Các điểm cần kiểm tra theo thứ tự:

1. token có claim `email` hay không,
2. `external-id` của identity có khớp đúng email đó hay không,
3. `Ext-JWT Signer` có đúng `issuer` và đúng nguồn public key hay không,
4. auth policy có thật sự cho phép signer đó hay không,
5. identity đang nằm trong auth policy nào.

Lưu ý:

- lỗi `issuer` sai hoặc thừa dấu `/` là nguyên nhân rất hay gặp,
- nhiều trường hợp user đăng nhập đúng nhưng Ziti từ chối vì map claim sai trường.

## Nhóm lỗi 2: Máy đã bị quarantine nhưng service vẫn còn truy cập được

Biểu hiện:

- role posture trên Ziti đã đổi sang `quarantine`,
- nhưng kết nối đang mở vẫn chưa rơi ngay.

Nguyên nhân thường gặp:

- chỉ đổi role nhưng chưa revoke session,
- kết nối TCP cũ vẫn tồn tại cho đến khi timeout,
- Orchestrator cập nhật tag nhưng không gọi bước thực thi ngắt phiên.

Hướng xử lý:

- bảo đảm Orchestrator có bước revoke session khi nhận `quarantine`,
- kiểm tra log xem request revoke có trả về thành công không,
- test lại với một phiên kết nối mới để tách lỗi policy và lỗi session tồn đọng.

## Nhóm lỗi 3: FleetDM có host nhưng OPA luôn trả kết quả không đúng

Biểu hiện:

- host đã online,
- policy đã fail,
- nhưng OPA vẫn luôn trả `compliant`, hoặc luôn trả `quarantine`.

Các điểm cần kiểm tra:

1. JSON input gửi sang OPA có đúng schema không,
2. tên policy fail trong FleetDM có khớp chuỗi mà Rego đang so sánh không,
3. bạn đang gọi đúng `decision_path` hay chưa,
4. policy Rego mới đã được nạp lại vào OPA chưa.

Nếu được, hãy log nguyên payload input/output để debug nhanh hơn thay vì chỉ log kết quả cuối.

## Nhóm lỗi 4: Orchestrator đổi sai role hoặc làm mất role cũ

Biểu hiện:

- sau khi update posture, máy mất luôn `#role.sales` hoặc `#role.hr`,
- policy nghiệp vụ không hoạt động như mong đợi.

Nguyên nhân:

- code cập nhật `roleAttributes` theo kiểu ghi đè toàn bộ,
- không tách role nghiệp vụ và role posture trước khi merge dữ liệu mới.

Hướng xử lý:

- chỉ xóa role posture cũ,
- giữ nguyên role nghiệp vụ hiện hữu,
- thêm role posture mới vào danh sách sau khi đã merge.

## Checklist trước khi demo

Trước giờ demo, nên rà nhanh toàn bộ các điểm sau:

- Keycloak đăng nhập được và token có claim `email`.
- OpenZiti controller và edge router đều online.
- Identity được đặt tên đúng convention `ep-<hostname>`.
- `external-id` khớp với email user trên Keycloak.
- FleetDM nhìn thấy host online và policy posture đổi trạng thái đúng.
- OPA nhận input và trả quyết định đúng với từng tình huống mẫu.
- Orchestrator cập nhật posture role mà không làm mất role nghiệp vụ.
- Khi `quarantine`, session đang hoạt động bị thu hồi thật sự.
- Dịch vụ nghiệp vụ bị chặn đúng lúc.
- Dịch vụ remediation vẫn truy cập được khi máy bị cách ly.
- Kịch bản khôi phục sau khi máy sạch đã được test lại ít nhất một lần.

## Kết luận

Nếu bạn đi đủ 9 phase theo đúng thứ tự, việc troubleshoot sẽ nhẹ hơn rất nhiều vì mỗi lớp đã có điểm kiểm tra riêng. Điều quan trọng nhất của bộ tài liệu này không chỉ là dựng xong hệ thống, mà là dựng theo cách **có thể giải thích, có thể kiểm chứng và có thể lặp lại**.
