---
title: "Phase 4: Bộ não ra quyết định (OPA)"
description: "Sử dụng Open Policy Agent (OPA) và ngôn ngữ Rego để xây dựng điểm ra quyết định chính sách (PDP)."
toc: true
authors: []
tags: ["OPA", "Rego", "Policy Engine", "Zero Trust"]
categories: ["Tutorials", "Cybersecurity"]
series: ["Triển khai C-ZTNA"]
date: '2023-11-04'
lastmod: '2023-11-04'
draft: false
weight: 4
---

Trong kiến trúc Zero Trust, OPA đóng vai trò là Điểm Ra Quyết Định Chính Sách (Policy Decision Point - PDP). Nó giúp tách biệt hoàn toàn logic đánh giá rủi ro khỏi cấu hình hạ tầng mạng, mang lại sự linh hoạt tối đa cho hệ thống.

## 1. Khởi chạy Open Policy Agent

Open Policy Agent (OPA) là một engine mã nguồn mở cực kỳ nhẹ, được thiết kế để phân xử các chính sách (policy) trên toàn bộ stack công nghệ của bạn. OPA sẽ nhận dữ liệu (input) ở dạng JSON và trả về kết quả dựa trên các luật (rules) đã được định nghĩa.

Để khởi chạy OPA, cách nhanh nhất là sử dụng Docker và phơi port `8181` để lắng nghe các truy vấn API dạng RESTful từ Orchestrator (chúng ta sẽ cấu hình ở Phase 6).

Bạn chạy lệnh sau trong Terminal:

```bash
docker run -d --name opa -p 8181:8181 openpolicyagent/opa run --server  
```Lệnh này sẽ khởi động OPA ở chế độ server, sẵn sàng nhận các truy vấn chính sách từ các thành phần khác trong hệ thống.
2. Viết luật đánh giá bằng Rego (posture.rego)
Ngôn ngữ Rego của OPA sinh ra để truy vấn và xử lý dữ liệu JSON. Chúng ta sẽ viết và nạp vào OPA một file cấu hình mang tên posture.rego chứa logic phân xử rủi ro.

Logic cốt lõi của bài toán:

Mặc định trạng thái bảo mật của mọi thiết bị (decision) là compliant (Tuân thủ/An toàn).

OPA sẽ nhận đầu vào (Input) từ Orchestrator, trong đó chứa mảng failing_policies (danh sách các lỗi mà máy đó đang mắc phải lấy từ FleetDM).

Nếu mảng failing_policies có chứa tên luật vi phạm bảo mật, OPA sẽ lập tức chuyển biến kết quả thành quarantine (Cách ly).

Bạn tạo một file văn bản có tên posture.rego với nội dung như sau:

Đoạn mã
package posture

# 1. Trạng thái mặc định luôn là an toàn
default decision = "compliant"

# 2. Chuyển sang cách ly nếu trong mảng failing_policies có lỗi vi phạm
decision = "quarantine" {
    input.failing_policies[_] == "Forbidden process running"
}

# (Tùy chọn) Bạn có thể mở rộng thêm vô số điều kiện khác ở đây
# decision = "quarantine" {
#     input.failing_policies[_] == "Firewall is disabled"
# }
Nạp file cấu hình vào OPA
Sau khi tạo file, bạn cần nạp nó vào bộ nhớ của OPA thông qua API. Chạy lệnh cURL sau trong cùng thư mục chứa file posture.rego:

Bash
curl -X PUT http://localhost:8181/v1/policies/posture --data-binary @posture.rego
💡 Lợi ích của kiến trúc này: Bằng cách tách bạch logic đánh giá rủi ro ra một máy chủ OPA độc lập, đội ngũ IT Admin sau này muốn cấm thêm phần mềm "Torrent" hoặc ép buộc "Phải bật Windows Defender", họ chỉ cần thêm vài dòng code vào file Rego rồi chạy lệnh curl cập nhật lại. Ngay lập tức hàng trăm máy tính trong mạng sẽ áp dụng luật mới mà không cần phải đụng chạm đến hạ tầng mạng OpenZiti hay khởi động lại dịch vụ!