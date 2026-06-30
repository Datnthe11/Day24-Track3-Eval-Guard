# Lab 24 — Production Eval + Guardrail Stack

Chào mừng các bạn đến với **Day 24**! 🎉

Hôm nay chúng ta sẽ xây dựng lớp **đánh giá (evaluation)** và **bảo vệ (guardrail)** hoàn chỉnh cho RAG pipeline mà các bạn đã xây dựng ở Day 18. Nếu Day 18 là "làm cho RAG chạy được", thì Day 24 là "làm cho RAG đáng tin cậy trong production".

---

## Bức tranh tổng thể

```
[Day 18 Pipeline của bạn]
        │
        ├──► Phase A: RAGAS 50q ──────────► Biết pipeline YẾU ở đâu
        │     (3 distributions)
        │
        ├──► Phase B: LLM-as-Judge ───────► Đo độ tin cậy của eval chính nó
        │     (pairwise + Cohen κ)
        │
        └──► Phase C: NeMo Guardrails ────► Bảo vệ pipeline khỏi inputs độc hại
              (Presidio PII + NeMo rails)
```

Sau lab này, các bạn sẽ có một **complete eval + guardrail stack** có thể deploy thẳng vào production.

---

## Yêu cầu tiên quyết

- Đã hoàn thành **Lab 18** (các module M1–M5 + pipeline.py đã chạy được)
- Python 3.11+
- Docker (chạy Qdrant)
- OpenAI API key

---

## Setup (15 phút — làm TRƯỚC khi tính giờ lab)

### Bước 1: Copy code từ Day 18

```bash
# Copy toàn bộ src/ từ Day 18 của bạn vào đây
cp -r <đường-dẫn-Day18>/src/m*.py src/
cp <đường-dẫn-Day18>/src/pipeline.py src/
```

> **Lưu ý:** M4 (`m4_eval.py`) của bạn cần đã implement xong `evaluate_ragas()` —
> Phase A sẽ dùng hàm này để chạy trên bộ test set 50 câu hỏi mới.

### Bước 2: Cài đặt môi trường

```bash
docker compose up -d                      # Khởi động Qdrant

pip install -r requirements.txt
python -m spacy download en_core_web_lg   # Cần cho Presidio PII detection

cp .env.example .env                      # Điền OPENAI_API_KEY vào .env
```

### Bước 3: Generate answers (quan trọng!)

```bash
python setup_answers.py
```

Script này sẽ chạy Day 18 pipeline của bạn trên **50 câu hỏi** và lưu kết quả vào `answers_50q.json`. Quá trình mất khoảng 5–10 phút. Đây là input cho Phase A.

---

## Cấu trúc thư mục

```
Day24-Track3-Eval-Guard/
│
├── src/
│   ├── m1_chunking.py      ← copy từ Day 18 của bạn
│   ├── m2_search.py        ← copy từ Day 18 của bạn
│   ├── m3_rerank.py        ← copy từ Day 18 của bạn
│   ├── m4_eval.py          ← copy từ Day 18 của bạn (★ cần implement xong)
│   ├── m5_enrichment.py    ← copy từ Day 18 của bạn
│   ├── pipeline.py         ← copy từ Day 18 của bạn
│   │
│   ├── phase_a_ragas.py    ★ BẠN IMPLEMENT — Tasks 1–4
│   ├── phase_b_judge.py    ★ BẠN IMPLEMENT — Tasks 5–8
│   └── phase_c_guard.py    ★ BẠN IMPLEMENT — Tasks 9–12
│
├── guardrails/
│   ├── config.yml          ← NeMo Guardrails config (đã có sẵn)
│   └── rails.co            ← Colang dialogue flows (có thể mở rộng)
│
├── data/                   ← Corpus HR policy (25 tài liệu tiếng Việt)
│
├── test_set_50q.json       ← 50 câu hỏi, 3 distributions
├── human_labels_10q.json   ← 10 nhãn nhân để tính Cohen κ
├── adversarial_set_20.json ← 20 inputs tấn công để test guardrail
│
├── setup_answers.py        ← Chạy pipeline → answers_50q.json
├── check_lab.py            ← Kiểm tra trước khi nộp
│
├── reports/
│   ├── ragas_50q.json      ★ auto-generated (Phase A)
│   ├── judge_results.json  ★ auto-generated (Phase B)
│   ├── guard_results.json  ★ auto-generated (Phase C)
│   └── blueprint.md        ★ Task 13 — bạn điền tay
│
└── analysis/
    ├── failure_clusters.md ★ Phân tích bottom-10 (điền sau Phase A)
    └── bias_report.md      ★ Phân tích bias (điền sau Phase B)
```

---

## Các giai đoạn (Phases)

### Giai đoạn A (Phase A) — Đánh giá Production với RAGAS (30 phút)

Chạy RAGAS trên **50 câu hỏi** với **3 phân phối (distributions)** để tìm ra điểm yếu của pipeline.

| Phân phối (Distribution) | Số câu | Đặc điểm |
|---|---|---|
| `factual` | 20 | Tra cứu chính sách đơn giản, 1 tài liệu |
| `multi_hop` | 20 | Kết hợp nhiều tài liệu, tính toán, suy luận |
| `adversarial` | 10 | Bẫy: xung đột phiên bản (v2023 vs v2024), bẫy phủ định |

**4 tasks cần thực hiện:** `group_by_distribution()` → `run_ragas_50q()` → `bottom_10()` → `cluster_analysis()`

```bash
python src/phase_a_ragas.py
# Đầu ra: reports/ragas_50q.json
```

### Giai đoạn B (Phase B) — LLM làm Giám khảo (30 phút)

Dùng LLM để so sánh cặp câu trả lời và đo độ đồng thuận với nhãn của con người.

- **Đánh giá theo cặp (Pairwise judge):** LLM chọn câu trả lời A hay B tốt hơn
- **Hoán đổi và lấy trung bình (Swap-and-average):** Đổi thứ tự A/B để phát hiện thiên vị vị trí (position bias)
- **Cohen's κ:** So sánh với 10 nhãn nhân trong `human_labels_10q.json`
- **Báo cáo thiên vị (Bias report):** Đo độ lệch vị trí (position bias) + độ lệch độ dài (verbosity bias)

```bash
python src/phase_b_judge.py
# Đầu ra: reports/judge_results.json
```

### Giai đoạn C (Phase C) — Hàng rào bảo vệ NeMo (30 phút)

Xây dựng lớp bảo vệ nhiều tầng trước và sau RAG pipeline.

```
Đầu vào → [Quét PII bằng Presidio] → [Hàng rào đầu vào NeMo] → RAG → [Hàng rào đầu ra NeMo] → Trả lời
```

- **Presidio:** Phát hiện CCCD, số điện thoại VN, email trong câu hỏi
- **Hàng rào đầu vào NeMo:** Chặn vượt rào (jailbreak), lạc đề (off-topic), tiêm prompt (prompt injection)
- **Hàng rào đầu ra NeMo:** Kiểm tra phản hồi trước khi trả về cho người dùng
- **Độ trễ P95 (P95 Latency):** Đo độ trễ từng tầng (Presidio ≈ <10ms, NeMo ≈ 200–500ms)

```bash
python src/phase_c_guard.py
# Đầu ra: reports/guard_results.json
```

---

## Bộ dữ liệu kiểm thử 50 câu hỏi

Bộ test set này được thiết kế đặc biệt để **kiểm tra sức chịu đựng (stress-test)** pipeline của bạn:

- **Thực tế (Factual):** Câu hỏi thẳng, nhưng corpus có nhiều phiên bản policy (v2023/v2024, v1/v2) → pipeline cần biết chọn phiên bản đúng
- **Suy luận nhiều bước (Multi-hop):** Tính toán lương thử việc, phí phạt tạm ứng, ngày phép tích lũy → cần kết hợp nhiều tài liệu
- **Tấn công (Adversarial):** Cố tình hỏi về policy cũ, dùng phủ định ("có nên tự xử lý không?"), hỏi VPN cá nhân → đây là những câu pipeline hay nhầm nhất

---

## Chạy kiểm thử (Tests)

```bash
pytest tests/ -v                     # Chạy toàn bộ test suite
pytest tests/test_phase_a.py -v      # Chỉ chạy Giai đoạn A
pytest tests/test_phase_b.py -v      # Chỉ chạy Giai đoạn B
pytest tests/test_phase_c.py -v      # Chỉ chạy Giai đoạn C
```

---

## Kiểm tra trước khi nộp bài

```bash
python check_lab.py
```

Danh sách kiểm tra (Checklist):
- [x] Các file mã nguồn Day 18 đã được chép vào `src/`
- [x] `answers_50q.json` đã được tạo
- [x] 0 TODOs còn lại trong `src/phase_*.py`
- [x] `reports/ragas_50q.json`, `judge_results.json`, `guard_results.json` đã có sẵn
- [x] `reports/blueprint.md` đã được điền đầy đủ
- [x] `pytest tests/` pass toàn bộ 100%

---

## Bàn giao (đẩy lên GitHub trước khi hết giờ)

1. **`src/phase_a_ragas.py`** — Tasks 1–4 (Đánh giá RAGAS)
2. **`src/phase_b_judge.py`** — Tasks 5–8 (Giám khảo LLM)
3. **`src/phase_c_guard.py`** — Tasks 9–12 (Hàng rào bảo vệ)
4. **`reports/blueprint.md`** — Task 13 (CI/CD blueprint, điền tay)
5. **`analysis/failure_clusters.md`** — Phân tích Giai đoạn A
6. **`analysis/bias_report.md`** — Phân tích Giai đoạn B

---

## Điểm số

| Giai đoạn | Điểm |
|---|---|
| Giai đoạn A: RAGAS (Tasks 1–4) | 30 |
| Giai đoạn B: Giám khảo (Tasks 5–8) | 35 |
| Giai đoạn C: Hàng rào (Tasks 9–12 + blueprint) | 35 |
| **Tổng** | **100** |
| Thưởng thêm (κ>0.6, tỷ lệ qua bài ≥18/20, tb tấn công<tb thực tế) | **+10** |

Chi tiết xem thêm: [`RUBRIC.md`](RUBRIC.md)

---

Chúc các bạn code vui! 🚀 Nếu gặp vấn đề, hãy gọi hỗ trợ — mentors luôn sẵn sàng hỗ trợ.
