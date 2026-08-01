---
title: "Sự kiện 3 — FCAJ x Agentic AI Build Week"
date: 2026-07-25
weight: 3
chapter: false
pre: " <b> 4.3. </b> "
---

### Thông tin sự kiện

| Trường | Chi tiết |
|-------|---------|
| **Tên sự kiện** | FCAJ x Agentic AI Build Week — *Show Up. Build. Pitch. WIN!* |
| **Thời gian** | Thứ Bảy, ngày 25 tháng 7, 2026 |
| **Địa điểm** | Tầng 26, Bitexco Tower, 02 Hải Triều, Phường Sài Gòn, TP. Hồ Chí Minh |
| **Vai trò** | Người tham dự (Offline) |

---

### Tóm tắt sự kiện

FCAJ x Agentic AI Build Week là một hackathon cường độ cao, phối hợp với **JI Fund**. Trong khoảng thời gian rất ngắn (chừng 24–48 giờ), các đội phải brainstorm, build và pitch một sản phẩm AI hoạt động được trước hội đồng giám khảo.

### Bài học chính

- **Quản lý scope mang tính quyết định.** Dưới áp lực thời gian gắt gao, cách làm hiệu quả là giữ scope thật nhỏ và giải quyết đúng một pain point cụ thể (MVP), thay vì liên tục thêm tính năng.
- **Execution quan trọng hơn ý tưởng.** Một ý tưởng hay gần như vô nghĩa nếu không có demo chạy được — có một sản phẩm hoạt động để trình bày với giám khảo là yếu tố quyết định lớn nhất.
- **Teamwork là biết gạt cái tôi sang một bên.** Làm việc trong tình trạng thiếu ngủ và áp lực buộc mỗi đội phải phân chia vai trò rõ ràng và tin vào quyết định của nhau; nếu không, mâu thuẫn nội bộ có thể phá hỏng cả dự án.

---

### Phần trình bày & điểm nhấn của các đội

**1. One Team**
Một *conversational ordering agent dùng AI* cho các thương hiệu F&B (ví dụ KFC). Đội xây dựng một chatbot đa kênh, cho phép khách đặt món ngay trong các app nhắn tin như Zalo — không cần cài app riêng, không cần tạo tài khoản. Dựa trên **Amazon Bedrock** và **AgentCore**, bot ghi nhớ ngữ cảnh người dùng, xử lý hội thoại tự nhiên và xử lý đơn hàng an toàn, loại bỏ rào cản (friction) trong luồng đặt hàng.

**2. Signal Scout**
Một *agent competitive-intelligence và chiến lược kinh doanh*. Hệ thống thu thập và phân tích dữ liệu công khai rải rác (báo cáo tài chính, thay đổi cơ cấu) của đối thủ để giúp nhà chiến lược đánh giá liệu áp dụng mô hình tương tự có sinh lời hay không — tự động dự báo doanh thu tiềm năng và rủi ro, giảm mạnh thời gian lãnh đạo phải nghiên cứu thủ công.

**3. Team Plan**
Một *công cụ sinh kiến trúc AI cho Solutions Architect*. Hướng đến các yêu cầu gấp của khách hàng, công cụ biến prompt ngôn ngữ tự nhiên cùng các chính sách nghiệp vụ thành một sơ đồ kiến trúc AWS tức thì trên draw.io — đồng thời ước tính chi phí và sinh Infrastructure-as-Code (Terraform) có thể deploy ngay.

**4. Team 3K — "Sheper"**
Một *hệ thống theo dõi và quản lý đám đông dùng AI*. Sử dụng các mô hình computer-vision **YOLO** và **ByteTrack**, hệ thống phân tích luồng camera thời gian thực tại những nơi đông người (sân bay, siêu thị) để phát hiện và dự đoán ùn tắc, sau đó tự động cảnh báo nhân viên và đề xuất phương án giải tỏa điểm nghẽn.

**5. Six Pillars**
Một *Adaptive Workflow Engine* cho phòng chống rửa tiền (AML) trong các tổ chức tài chính. Đóng vai trò co-pilot cho analyst, hệ thống tự động hóa việc điều tra tẻ nhạt đối với các cảnh báo giao dịch false-positive — dùng nhiều sub-agent để đối chiếu hồ sơ KYC, dòng tiền và sanction blacklist — rút một cuộc điều tra 3 giờ xuống còn vài phút, mà vẫn giữ human-in-the-loop cho khâu phê duyệt cuối.

---

### Kết quả và giá trị thu được

- **Củng cố kỷ luật MVP** mà tôi đang áp dụng cho dự án của nhóm — dưới deadline, việc giữ scope thật chặt (làm phần đặt sân + thanh toán cốt lõi trước, mở rộng sau) chính là cách nhóm giữ được tiến độ cho hệ thống court-booking.
- **Tiếp xúc trực tiếp với hệ sinh thái agentic-AI / Bedrock** — chứng kiến các agent kiểu production được dựng trên Bedrock + AgentCore chỉ trong 48 giờ giúp tôi mở rộng hình dung về những gì hiện đã khả thi, và về việc luồng thông báo SNS/Lambda của nhóm sau này có thể tiến hóa sang giao diện hội thoại.
- **Một hình mẫu cụ thể về teamwork dưới áp lực cao** — bài học "gạt cái tôi, phân chia vai trò, tin vào quyết định của nhau" phản ánh đúng cách nhóm năm người của chúng tôi phân chia quyền sở hữu (BE / FE / CI-CD), và là thói quen đáng mang theo cả sau kỳ thực tập.
