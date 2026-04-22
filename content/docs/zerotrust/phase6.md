---
title: "Phase 6: Posture Orchestrator"
description: "Lập trình kịch bản Python đóng vai trò tự động hóa luồng SOAR cho hệ thống C-ZTNA."
toc: true
authors: []
tags: ["Python", "Automation", "SOAR", "API"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2023-11-06'
lastmod: '2023-11-06'
draft: false
weight: 6
---

Nếu OPA là bộ não ra quyết định, OpenZiti là tay chân thực thi, thì **Posture Orchestrator** chính là hệ thần kinh. Đây là một đoạn mã (middleware) tự viết bằng Python, đóng vai trò kết nối luồng dữ liệu độc lập giữa FleetDM, OPA và OpenZiti thành một vòng lặp tự động hóa (SOAR).

## 1. Logic hoạt động của Orchestrator

Đoạn mã Python sẽ chạy ngầm dưới dạng Daemon, thực hiện chu trình 7 bước sau mỗi khoảng thời gian cố định (ví dụ: 5 giây/lần):

1. **Lấy dữ liệu (Polling):** Gọi API của FleetDM để lấy danh sách toàn bộ các Host đang online.
2. **Trích xuất:** Đọc dữ liệu Host Posture và tính toán danh sách các luật vi phạm (`failing_policies[]`) của từng máy.
3. **Tham vấn:** Đóng gói mảng vi phạm đó, gửi API sang cho OPA (Port 8181) và nhận về kết quả phán quyết (`compliant` hoặc `quarantine`).
4. **Ánh xạ danh tính:** Lấy Hostname của thiết bị từ FleetDM, ghép với tiền tố `ep-` để tìm kiếm Định danh (Identity) tương ứng trên OpenZiti Controller (như cấu trúc đặt tên ở Phase 2).
5. **Cập nhật Nhãn (Tagging):** Gọi Ziti API để cập nhật trường `roleAttributes`. **Lưu ý quan trọng:** Nó phải được lập trình để giữ nguyên các thẻ phòng ban cũ (như `#role.sales`), và chỉ ghi đè thay đổi đối với nhãn posture thành `#posture.compliant` hoặc `#posture.quarantine`.
6. **Thực thi ngắt mạng (Enforcement):** Nếu quyết định là `quarantine`, Orchestrator không chỉ đổi thẻ mà còn phải gọi API thu hồi lập tức (Revoke Session) các phiên dịch vụ đang active của thiết bị đó. Việc này đảm bảo ngắt mạng tức khắc thay vì chờ hệ thống tự Timeout.
7. **Lưu vết:** Ghi lại toàn bộ lịch sử biến đổi trạng thái vào cơ sở dữ liệu `orchestrator.db` (SQLite) và xuất ra file CSV để phục vụ báo cáo.

## 2. File cấu hình (`config.yaml`)

Để script Orchestrator linh hoạt mà không cần sửa code, bạn cần định nghĩa một file `config.yaml` chứa thông tin kết nối đến tất cả các thành phần:

```yaml
poll_interval_seconds: 5

fleet:
  url: "[https://fleet.lab.local:8443](https://fleet.lab.local:8443)"
  api_token: "MÃ_TOKEN_LẤY_TỪ_FLEETDM_ADMIN"

opa:
  url: "http://opa:8181"
  decision_path: "/v1/data/posture/decision"

mapping:
  mode: "hostname"
  ziti_identity_prefix: "ep-"

posture_attributes:
  compliant: "posture.compliant"
  quarantine: "posture.quarantine"

enforcement:
  # Cờ này cực kỳ quan trọng để đảm bảo TTE (Time-to-Enforce) ở mức mili-giây
  revoke_sessions_on_quarantine: true

Gemini đã nói
Dưới đây là nội dung hoàn chỉnh cho Phase 6. Ở giai đoạn này, chúng ta sẽ thiết lập "trái tim" tự động hóa của toàn bộ hệ thống. Tôi đã gắn tham số weight: 6 vào Frontmatter để đảm bảo đúng thứ tự trên blog của bạn.

Bạn hãy nhấn nút Copy ở góc trên bên phải khung dưới đây và dán vào file phase6.md:

Markdown
---
title: "Phase 6: Posture Orchestrator"
description: "Lập trình kịch bản Python đóng vai trò tự động hóa luồng SOAR cho hệ thống C-ZTNA."
toc: true
authors: []
tags: ["Python", "Automation", "SOAR", "API"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2023-11-06'
lastmod: '2023-11-06'
draft: false
weight: 6
---

Nếu OPA là bộ não ra quyết định, OpenZiti là tay chân thực thi, thì **Posture Orchestrator** chính là hệ thần kinh. Đây là một đoạn mã (middleware) tự viết bằng Python, đóng vai trò kết nối luồng dữ liệu độc lập giữa FleetDM, OPA và OpenZiti thành một vòng lặp tự động hóa (SOAR).

## 1. Logic hoạt động của Orchestrator

Đoạn mã Python sẽ chạy ngầm dưới dạng Daemon, thực hiện chu trình 7 bước sau mỗi khoảng thời gian cố định (ví dụ: 5 giây/lần):

1. **Lấy dữ liệu (Polling):** Gọi API của FleetDM để lấy danh sách toàn bộ các Host đang online.
2. **Trích xuất:** Đọc dữ liệu Host Posture và tính toán danh sách các luật vi phạm (`failing_policies[]`) của từng máy.
3. **Tham vấn:** Đóng gói mảng vi phạm đó, gửi API sang cho OPA (Port 8181) và nhận về kết quả phán quyết (`compliant` hoặc `quarantine`).
4. **Ánh xạ danh tính:** Lấy Hostname của thiết bị từ FleetDM, ghép với tiền tố `ep-` để tìm kiếm Định danh (Identity) tương ứng trên OpenZiti Controller (như cấu trúc đặt tên ở Phase 2).
5. **Cập nhật Nhãn (Tagging):** Gọi Ziti API để cập nhật trường `roleAttributes`. **Lưu ý quan trọng:** Nó phải được lập trình để giữ nguyên các thẻ phòng ban cũ (như `#role.sales`), và chỉ ghi đè thay đổi đối với nhãn posture thành `#posture.compliant` hoặc `#posture.quarantine`.
6. **Thực thi ngắt mạng (Enforcement):** Nếu quyết định là `quarantine`, Orchestrator không chỉ đổi thẻ mà còn phải gọi API thu hồi lập tức (Revoke Session) các phiên dịch vụ đang active của thiết bị đó. Việc này đảm bảo ngắt mạng tức khắc thay vì chờ hệ thống tự Timeout.
7. **Lưu vết:** Ghi lại toàn bộ lịch sử biến đổi trạng thái vào cơ sở dữ liệu `orchestrator.db` (SQLite) và xuất ra file CSV để phục vụ báo cáo.

## 2. File cấu hình (`config.yaml`)

Để script Orchestrator linh hoạt mà không cần sửa code, bạn cần định nghĩa một file `config.yaml` chứa thông tin kết nối đến tất cả các thành phần:

```yaml
poll_interval_seconds: 5

fleet:
  url: "[https://fleet.lab.local:8443](https://fleet.lab.local:8443)"
  api_token: "MÃ_TOKEN_LẤY_TỪ_FLEETDM_ADMIN"

opa:
  url: "http://opa:8181"
  decision_path: "/v1/data/posture/decision"

mapping:
  mode: "hostname"
  ziti_identity_prefix: "ep-"

posture_attributes:
  compliant: "posture.compliant"
  quarantine: "posture.quarantine"

enforcement:
  # Cờ này cực kỳ quan trọng để đảm bảo TTE (Time-to-Enforce) ở mức mili-giây
  revoke_sessions_on_quarantine: true 
3. Khởi chạy hệ thần kinh trung ương
Mở Terminal tại thư mục chứa source code của Orchestrator, cài đặt các thư viện cần thiết (requests, sqlite3...) và chạy lệnh:

Bash
python3 orchestrator.py
Hãy để Terminal này chạy ngầm (hoặc dùng tmux/screen). Khi bạn thấy Terminal in ra các dòng log quét (Polling) liên tục màu xanh, xin chúc mừng, vòng lặp tự động hóa của bạn đã chính thức hòa mạng!