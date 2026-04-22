---
title: "Phase 2: Thiết lập mạng lõi (OpenZiti)"
description: "Khởi tạo OpenZiti Controller và Edge Router để xây dựng vành đai bảo mật định nghĩa bằng phần mềm (SDP)."
toc: true
authors: []
tags: ["OpenZiti", "SDP", "Overlay Network"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2026-01-19'
lastmod: '2026-01-26'
draft: false
weight: 2
---

Nếu Keycloak lo phần định danh, thì OpenZiti làm nhiệm vụ thực thi ở tầng mạng (Network Enforcement). Phase này tập trung vào việc thiết lập Mặt phẳng Điều khiển (Control Plane) và Mặt phẳng Dữ liệu (Data Plane).

## Khởi chạy Controller và Edge Router

Khuyến nghị sử dụng bộ cài đặt tự động `ziti-quickstart` hoặc cấu hình Docker Compose được cung cấp sẵn từ cộng đồng OpenZiti để khởi tạo mạng lõi.

Hệ thống yêu cầu ít nhất 2 thành phần phải hoạt động trơn tru:
1. **Ziti Controller:** Bộ não cấp phát chứng chỉ (PKI) và quản lý chính sách. Nó tuyệt đối không chuyển tiếp gói tin dữ liệu người dùng.
2. **Ziti Edge Router:** Cửa ngõ để máy trạm (Client) kết nối vào bằng đường hầm mã hóa mTLS.

Sau khi khởi chạy, hãy đăng nhập vào giao diện web Ziti Admin Console (ZAC) để xác nhận các Router đang ở trạng thái `Online`.

## Quy tắc đặt tên Định danh (Identity Naming Convention)

Trong OpenZiti, mỗi thiết bị hoặc máy chủ muốn tham gia mạng đều phải được cấp một Định danh (Identity). Để hệ thống tự động hóa (Orchestrator ở Phase 6) hoạt động chính xác, tên của Identity **bắt buộc** phải tuân thủ một chuẩn nhất định.

### Cấu trúc tiêu chuẩn
* **Tên Identity:** Phải bắt đầu bằng tiền tố `ep-` kèm theo tên Hostname của máy tính (Ví dụ: `ep-WIN-HR-01`). Orchestrator sẽ dùng tên này để tìm và cập nhật trạng thái.
* **External ID:** Sử dụng chính xác Email của người dùng đã tạo trên Keycloak (Ví dụ: `alice.hr@lab.local`). Điều này giúp liên kết thiết bị vật lý với người dùng logic.
* **Role Attributes (Thẻ nhãn):** Cấp sẵn các thẻ phòng ban ngay từ lúc khởi tạo. Ví dụ: `#role.hr` và `#endpoint`. Tuyệt đối chưa cấp thẻ `posture` ở bước này.

### Ví dụ khởi tạo qua Ziti CLI
Thay vì thao tác thủ công trên giao diện, bạn có thể dùng công cụ dòng lệnh `ziti-cli` để tạo hàng loạt:

```bash
# Tạo định danh cho nhân viên HR
ziti edge create identity "ep-WIN-HR-01" -a "role.hr,endpoint" -P default --external-id "alice.hr@lab.local"

# Tạo định danh cho nhân viên Sales
ziti edge create identity "ep-WIN-SALES-01" -a "role.sales,endpoint" -P default --external-id "alice.sales@lab.local"