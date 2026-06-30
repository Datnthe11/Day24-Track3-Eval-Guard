# Báo cáo Phân Tích Độ Lệch (Bias Report) - Phase B (LLM-as-Judge)

Dựa trên kết quả so sánh cặp (pairwise judge) và phương pháp đánh giá chéo (swap-and-average) từ `reports/judge_results.json`, dưới đây là đánh giá độ ổn định và tính công bằng của LLM khi đóng vai trò giám khảo.

## 1. Hệ số đồng thuận Cohen's Kappa (κ)
* **Giá trị Cohen's κ:** `0.286`
* **Đánh giá:** Đây là mức đồng thuận (agreement) khá thấp giữa mô hình LLM đóng vai trò giám khảo và nhãn của con người. Điều này cho thấy bộ tiêu chí đánh giá cho mô hình Judge cần được hướng dẫn rõ ràng hơn hoặc LLM đang không thực sự hiểu cách chấm điểm của con người đối với một số lỗi chi tiết.

## 2. Độ lệch vị trí (Position Bias)
* **Tỷ lệ (Position Bias Rate):** `0.2` (2/10 trường hợp).
* **Đánh giá:** Tỷ lệ này khá thấp. Trong 10 câu hỏi, chỉ có 2 câu hỏi mà LLM thay đổi quyết định khi đổi chỗ hai câu trả lời (Answer A và Answer B). Nhận xét: LLM-as-Judge khá ổn định và ít bị thiên vị bởi thứ tự cung cấp đáp án.

## 3. Độ lệch độ dài (Verbosity Bias)
* **Tỷ lệ (Verbosity Bias):** `1.0` (100% các câu có kết quả quyết định).
* **Chi tiết:** Trong 8 trường hợp mà LLM có phân định được bên thắng/thua (quyết định không hòa), 8/8 lần mô hình đều chọn câu trả lời dài hơn làm câu chiến thắng (`b_wins_b_longer = 8`).
* **Đánh giá:** Đây là một độ lệch cực kỳ lớn (nghiêm trọng). Mô hình gần như mặc định cho rằng "câu trả lời dài hơn là câu trả lời tốt hơn" (Verbosity Bias). Điều này có thể giải thích tại sao Cohen's Kappa thấp (vì con người thường thích câu trả lời súc tích và đúng trọng tâm, trong khi mô hình thích sự dông dài).

## 4. Kết luận và Đề xuất cải tiến
* LLM Judge hiện tại mắc phải **Verbosity Bias** rất nặng. 
* **Đề xuất:** Cần điều chỉnh prompt cho LLM-as-Judge, yêu cầu mô hình phải phạt các câu trả lời quá dài dòng không cần thiết (penalty for verbosity) và ưu tiên tính súc tích (conciseness), đúng trọng tâm. Đồng thời nên thêm vài ví dụ few-shot minh họa việc chọn câu trả lời ngắn mà chính xác.
