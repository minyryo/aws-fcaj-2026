---
title: "Sự kiện 1 — Data Driven, AI Risen"
date: 2026-06-27
weight: 1
chapter: false
pre: " <b> 4.1. </b> "
---

### Thông tin sự kiện

| Thông tin | Chi tiết |
|-----------|---------|
| **Tên sự kiện** | Data Driven, AI Risen |
| **Thời gian** | 09:00, ngày 27/06/2026 |
| **Địa điểm** | Livestream |
| **Vai trò** | Người tham dự |

---

### Mô tả sự kiện

FCAJ Community Day tháng 6/2026 gồm năm bài trình bày tập trung vào các ứng dụng thực tiễn của Trí tuệ Nhân tạo trong hạ tầng đám mây, phát triển phần mềm và quy trình doanh nghiệp.

#### 1. Phát triển sự nghiệp và AI trong vận hành đám mây
*Diễn giả: Steve Tran (Cloud Thinker)*

Steve chia sẻ hành trình từ vị trí IT helpdesk đến AWS Solution Architect, nhấn mạnh rằng AI vừa tăng tốc độ lập trình vừa đòi hỏi kỹ sư phải có chuyên môn sâu hơn để quản lý các hệ thống phức tạp. Ông giới thiệu nền tảng AI agentic của Cloud Thinker, tự động hóa việc xử lý sự cố, kiểm thử bảo mật, kiểm soát chất lượng và tối ưu chi phí đám mây (FinOps).

#### 2. Xây dựng Voice AI Agent cho thị trường Việt Nam
*Diễn giả: Hieu Nghi (Renova Cloud), Kiet (Student Video Group), Trung (R AI)*

Phiên trình bày demo trực tiếp một Voice AI Agent được xây trên AWS Bedrock và phân tích các thách thức khi triển khai voice agent trong môi trường doanh nghiệp Việt Nam. Trung đề xuất pipeline tùy chỉnh Speech-to-Text → LLM → Text-to-Speech để xử lý chính xác giọng địa phương, ngắt hội thoại tự nhiên và các tác vụ tool-calling doanh nghiệp như tự động khóa thẻ ngân hàng.

#### 3. Tự động hóa khắc phục sự cố với AWS DevOps Agent
*Diễn giả: Bao và Nguyen (Cloud Kinetics)*

Các diễn giả trình bày cách AWS DevOps Agent giải quyết vấn đề log phân tán và thời gian phản hồi sự cố chậm bằng cách tự động điều tra cảnh báo, xác định nguyên nhân gốc rễ và đề xuất kế hoạch khắc phục từng bước. Demo trực tiếp cho thấy agent chẩn đoán thành công một cuộc tấn công DDoS giả lập trên ứng dụng thương mại điện tử chạy trên ECS trong vài phút thay vì vài giờ.

#### 4. Chuyển đổi quy trình Nhân sự với Amazon Q
*Diễn giả: Truong và Minh Anh (Noventiq)*

Minh Anh nêu các điểm đau của bộ phận HR như sàng lọc CV thủ công và rủi ro bảo mật khi tải dữ liệu ứng viên lên các công cụ AI công khai; Truong demo Amazon Q như một trợ lý AI bảo mật, có thể tùy chỉnh cho doanh nghiệp. Demo cho thấy Amazon Q tự động đọc nhiều CV, đối chiếu với mô tả vị trí Cloud Engineer và tạo báo cáo chấm điểm, đánh giá toàn diện.

#### 5. Bảo mật kết nối Amazon Q với Private MCP Server
*Diễn giả: Toan Nguyen và Nghi*

Phiên trình bày đề cập đến rủi ro bảo mật khi để lộ Model Context Protocol (MCP) server — cầu nối giữa các AI assistant như Amazon Q và các công cụ như Jira, Gmail, Zalo — ra internet công khai (DDoS, tấn công Man-in-the-Middle). Toan mô tả kiến trúc sử dụng AWS VPC connection, private subnet, interface endpoint và Route 53 resolver để đảm bảo toàn bộ lưu lượng AI và dữ liệu luôn nằm trong mạng AWS riêng tư.

---

### Kết quả và giá trị đạt được

- **Góc nhìn rộng hơn về tác động của AI đến sự nghiệp cloud:** Bài chia sẻ về sự nghiệp giúp tôi nhìn lại việc nâng cao kỹ năng — AI tăng năng suất cơ bản nhưng đồng thời đòi hỏi chuyên môn sâu hơn về hệ thống, từ đó thúc đẩy tôi đầu tư nhiều hơn vào việc học AWS.

- **Hiểu sâu hơn về kiến trúc voice/thông báo:** Pipeline Voice AI tùy chỉnh cho tiếng Việt (STT → LLM → TTS) là kiến thức nền hữu ích nếu ứng dụng đặt sân của nhóm mở rộng thông báo ra ngoài SNS sang voice hoặc chat — giờ tôi đã hiểu độ phức tạp kỹ thuật đằng sau.

- **Chiến lược tự động hóa phản hồi sự cố cho nhóm:** Khả năng của AWS DevOps Agent rút ngắn thời gian xử lý từ hàng giờ xuống vài phút rất phù hợp với chiến lược giám sát tương lai của dự án; điều này sẽ ảnh hưởng đến cách nhóm cấu hình CloudWatch alarm và runbook.

- **Mô hình triển khai AI nội bộ an toàn:** Demo Amazon Q trong HR cho thấy cách triển khai AI assistant trong khuôn khổ bảo mật doanh nghiệp — một mô hình tham chiếu hữu ích nếu nhóm tích hợp AI vào quy trình nội bộ mà không để lộ dữ liệu dự án nhạy cảm.

- **Nhận thức bảo mật cho tích hợp AI:** Hiểu lỗ hổng MCP server và giải pháp private subnet giúp nâng cao ý thức bảo mật API của nhóm, bổ sung cho kiến trúc API Gateway + Lambda mà nhóm đang xây dựng.
