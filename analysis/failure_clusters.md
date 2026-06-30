# Báo cáo Nhóm Lỗi (Failure Clusters) - Phase A (RAGAS)

Dựa trên kết quả đánh giá 50 câu hỏi từ `reports/ragas_50q.json`, dưới đây là phân tích chi tiết về các điểm yếu của hệ thống RAG hiện tại.

## 1. Tổng quan điểm số
* **Trung bình (Average Score):** ~0.812 (tổng quan trên 50 câu).
* **Factual:** 0.896 (Hoạt động tốt nhất).
* **Multi-hop:** 0.761.
* **Adversarial:** 0.751.

## 2. Số liệu kém nhất (Worst Metric)
**Faithfulness** (Tính trung thực) là số liệu (metric) kém nhất trong phần lớn các câu hỏi thuộc top 10 câu bị đánh giá thấp nhất. Điều này cho thấy hệ thống đôi khi sinh ra các thông tin không có trong tài liệu nguồn (hallucination) hoặc trả lời sai lệch thông tin so với những gì RAG truy xuất được.

## 3. Phân bố lỗi chủ đạo (Dominant Failure Distribution)
* Phân bố có nhiều câu trả lời thất bại (bị đánh điểm thấp) nhất là **factual** theo phân tích ma trận (matrix), tuy nhiên các câu hỏi trong nhóm **multi_hop** và **adversarial** chiếm nhiều vị trí nhất trong Top 10 câu hỏi tệ nhất.
* Cụ thể, các câu hỏi `multi_hop` đòi hỏi kết hợp nhiều tài liệu để trả lời (như chính sách bảo hiểm kết hợp phụ cấp, so sánh v1.0 và v2.0) thường xuyên khiến mô hình trả lời thiếu chính xác.

## 4. Top 3 Lỗi Nghiêm Trọng Nhất (Từ Bottom 10)
1. **Câu 39 (multi_hop - avg 0.125):** "So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0..." → Lỗi *faithfulness*. Mô hình có thể bị nhầm lẫn giữa các phiên bản chính sách.
2. **Câu 30 (multi_hop - avg 0.375):** "So sánh quyền lợi bảo hiểm giữa nhân viên thử việc và nhân viên chính thức." → Lỗi *faithfulness*.
3. **Câu 33 (multi_hop - avg 0.375):** "Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024..." → Lỗi *faithfulness*. Lỗi tính toán nhiều bước.

## 5. Đề xuất cải tiến
* **Tinh chỉnh Prompt (Tighten System Prompt):** Ràng buộc chặt chẽ hơn để yêu cầu mô hình (LLM) chỉ được phép sử dụng thông tin từ văn bản đã được truy xuất, giảm tình trạng hallucination.
* **Xử lý xung đột phiên bản:** Nên thêm metadata vào các vector chunk (như `version: 2024`, `status: active`) và ưu tiên (boost) các tài liệu hiện hành trong quá trình truy xuất (retrieval) để tránh lấy nhầm chính sách cũ.
* **Hỗ trợ Multi-hop:** Sử dụng kỹ thuật chia nhỏ câu hỏi (query decomposition) để giải quyết các câu hỏi tính toán phức tạp (ví dụ: tách thành câu hỏi về "phụ cấp Manager" và "số ngày phép thâm niên 12 năm" trước khi tổng hợp lại).
