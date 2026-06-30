# CI/CD Blueprint: RAG Eval + Guardrail Stack

**Sinh viên:** Nguyễn Thành Đạt - 2A202600771  
**Ngày:** 30/06/2026

---

## Kiến trúc Guard Stack (Guard Stack Architecture)

```
User Input
    │
    ▼ (~?ms P95)
[Presidio PII Scan]
    │ block if: Phát hiện VN_CCCD / VN_PHONE / EMAIL
    │ action:   trả về 400 + "PII detected in query" (Phát hiện thông tin cá nhân trong câu hỏi)
    ▼ (~?ms P95)
[NeMo Input Rail]
    │ block if: off-topic (lạc đề) / jailbreak (vượt rào) / prompt injection (tiêm prompt)
    │ action:   trả về 503 + từ chối trả lời
    ▼
[RAG Pipeline (Day 18)]
    │ M1 Chunk → M2 Search → M3 Rerank → GPT-4o-mini
    ▼
[NeMo Output Rail]
    │ flag if:  Phát hiện PII trong câu trả lời / nội dung nhạy cảm
    │ action:   Thay thế bằng câu trả lời an toàn
    ▼
User Response
```

---

## Ngân sách độ trễ (Latency Budget)

*(Dữ liệu sẽ được điền từ kết quả Task 12 — measure_p95_latency())*

| Layer | P50 (ms) | P95 (ms) | P99 (ms) | Budget |
|---|---|---|---|---|
| Presidio PII | 1603.6 | 3125.1 | 3125.1 | <10ms |
| NeMo Input Rail | 225.49 | 289.9 | 289.9 | <300ms |
| RAG Pipeline | ? | ? | ? | <2000ms |
| NeMo Output Rail | ? | ? | ? | <300ms |
| **Tổng cộng Guard** | 1830.87 | **3388.53** | 3388.53 | **<500ms** |

**Ngân sách OK không?** [ ] Có / [x] Không  
**Nhận xét:** Presidio PII đang là nút thắt cổ chai (bottleneck) nghiêm trọng với độ trễ P95 lên đến hơn 3 giây. Để tối ưu, cần loại bỏ các Regex pattern nhận diện PII không cần thiết, tối ưu code Presidio Analyzer, hoặc sử dụng các service microservice build bằng C++/Rust chuyên biệt thay vì bản cài Python chuẩn.

---

## Các Cổng CI/CD (Phải vượt qua trước khi merge vào main)

```yaml
# .github/workflows/rag_eval.yml
- name: RAGAS Quality Gate
  run: python src/phase_a_ragas.py
  env:
    MIN_FAITHFULNESS: 0.75
    MIN_AVG_SCORE: 0.65

- name: Guardrail Gate
  run: pytest tests/test_phase_c.py -k "test_adversarial_suite_pass_rate"
  # phải đạt ≥ 15/20 (75%)

- name: Latency Gate
  run: python -c "from src.phase_c_guard import measure_p95_latency; ..."
  # P95 tổng cộng < 500ms
```

---

## Bảng giám sát (Monitoring Dashboard) trên Production

| Metric | Ngưỡng Cảnh Báo | Hành Động |
|---|---|---|
| RAGAS faithfulness (lấy mẫu hàng ngày) | < 0.70 | Gọi hỗ trợ on-call |
| Tỷ lệ chặn tấn công (Adversarial block rate) | < 80% | Đánh giá các mẫu tấn công mới |
| Độ trễ P95 của Guard | > 600ms | Nâng cấp mô hình NeMo hoặc mở rộng (Scale) |
| Số lượng phát hiện PII | Tăng vọt >10/giờ | Kích hoạt cảnh báo bảo mật |

---

## Kết quả thực tế từ Lab

| Số liệu | Kết quả |
|---|---|
| RAGAS avg_score (50 câu) | 0.812 |
| Số liệu (metric) kém nhất | faithfulness |
| Nhóm lỗi (failure distribution) chính | factual |
| Hệ số Cohen's κ | 0.286 |
| Tỷ lệ vượt qua các truy vấn tấn công | 18 / 20 |
| Độ trễ P95 của hệ thống bảo vệ (Guard) | 3388.53 ms |

---

## Nhận xét & Cải tiến

> Hệ thống hoạt động khá tốt ở các thành phần nhận diện PII cơ bản thông qua Presidio nhờ tốc độ nhanh, cũng như NeMo Guardrails thể hiện được tính linh hoạt cao. Tuy nhiên, độ trễ (latency) của NeMo Guardrails có thể là một nút thắt (bottleneck) lớn nếu lưu lượng truy cập cao. Nếu đưa vào môi trường thực tế (production), tôi sẽ cân nhắc việc thay thế một số Guardrail nặng bằng các mô hình phân loại nhỏ gọn hơn hoặc tối ưu hóa (ví dụ: host local LLM chuyên dụng cho Guardrail thay vì dựa hoàn toàn vào API bên ngoài). Đồng thời, việc thêm cơ chế caching cho các truy vấn tấn công đã bị nhận diện cũng sẽ giảm tải đáng kể cho hệ thống.
