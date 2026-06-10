# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** F2
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Lý Hải Long | Ingestion / Cleaning ||

**Ngày nộp:** 10/06/2026
**Repo:** ``
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**
Pipeline ETL này xử lý quá trình nạp dữ liệu từ nguồn raw CSV (`data/raw/policy_export_dirty.csv`) - đại diện cho log dump của DB. 
1. **Ingest**: Đọc dữ liệu CSV thành Dict.
2. **Clean**: Đưa qua `cleaning_rules.py` để loại bỏ noise prefix, suffix, fix lặp từ, deduplicate các chunk text trùng nhau.
3. **Validate**: Dùng thư viện expect/assert ở `quality/expectations.py` để kiểm tra chất lượng (halt nếu còn rác).
4. **Embed**: Upsert vào ChromaDB (Idempotent update) và ghi ra các artifacts (cleaned, quarantine, manifest). Toàn bộ luồng được log lại bằng `run_id`.

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**
`python etl_pipeline.py run`

---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| `no_noise_phrases` (expectation) | 10 violations | 0 violations | Log `run_id=2026-06-10T06-43Z` |
| `no_stuttering_words` (expectation)| 2 violations | 0 violations | Log `run_id=2026-06-10T06-43Z` |
| `fix_deduplication_flow` (bug fix)| 46 cleaned records| 35 cleaned records | Log `run_id=2026-06-10T06-52Z` |

**Rule chính (baseline + mở rộng):**

- …

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**

_________________

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**
Nhóm đã chạy kịch bản inject bằng lệnh `python etl_pipeline.py run --no-refund-fix --skip-validate`. Lệnh này cố ý bỏ qua quá trình làm sạch (cleaning) cho chính sách hoàn tiền, cho phép dữ liệu cũ (hoàn tiền trong 14 ngày) lọt vào Vector Database. Sau đó, chạy file chấm điểm `grading_run.py` để lấy kết quả. 

**Kết quả định lượng (từ CSV / bảng):**
Khi chạy script so sánh `compare_evals.py` giữa 2 bản `before_fix` và `after_fix`, kết quả minh họa rất rõ rệt:

- ✅ **Câu hỏi:** Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi đơn được xác nhận?
- ❌ **BEFORE FIX (Dữ liệu bẩn):** Top-1 Retrieved lấy sai thông tin cũ: *"Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc kể từ xác nhận đơn."* -> Câu hỏi bị FAIL.
- ✨ **AFTER FIX (Sau khi dọn dẹp):** Top-1 Retrieved đã lấy đúng: *"Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng."* -> Câu hỏi PASS.

Sự khác biệt này minh họa rõ rệt tầm quan trọng của việc làm sạch dữ liệu trong pipeline trước khi đưa vào ứng dụng RAG. Mọi thuật toán tìm kiếm vector dù tốt đến đâu cũng sẽ vô dụng nếu bản thân văn bản đã sai lệch hoặc cũ (Garbage in, garbage out).

---

**Mô tả SLA & Freshness:**
Mỗi lần chạy pipeline, hệ thống sẽ xuất một file `manifest_[run_id].json`. File này chứa field `latest_exported_at` thu thập từ dòng mới nhất của data. Tool freshness check sẽ tính khoảng cách từ timestamp hiện hành tới timestamp lớn nhất của data.
Chúng tôi cài đặt SLA Freshness là 24 giờ (`FRESHNESS_SLA_HOURS=24`).
- **PASS:** Dữ liệu mới cập nhật dưới 24h.
- **FAIL:** Data quá cũ, quá 24h. Trong lần chạy `run_id=2026-06-10T07-00Z`, do mốc data lớn nhất là tháng 04/2026 nên hệ thống báo FAIL: `freshness_check=FAIL {"reason": "freshness_sla_exceeded"}`. Đây là dấu hiệu cho thấy Ingestion Job cần chạy lại.

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

Dữ liệu sạch sau khi được embed vào ChromaDB (`day10_kb`) hoàn toàn có thể đóng vai trò làm Knowledge Base trực tiếp cho Multi-Agent của Day 09. Các Agent (như HR Agent, IT Support Agent) có thể kết nối với ChromaDB này qua LangChain Tool để trả lời câu hỏi nhân viên. Nếu RAG retrieval ở Day 10 được cải thiện nhờ Data Cleaning, thì kết quả suy luận của Agent ở Day 09 cũng sẽ chính xác tương ứng. Việc tách collection giúp phân tách môi trường dev/prod và dễ dàng thử nghiệm nhiều phiên bản Cleaning Pipeline khác nhau.

---

## 6. Rủi ro còn lại & việc chưa làm

- Việc khắc phục sự yếu kém về Semantic Match của VectorDB hiện tại vẫn đang dùng Document Expansion thủ công bằng Regex. Cần tích hợp một model LLM (LLM as a Judge / Rewriter) để tự động hóa khâu xử lý nội dung.
- Quá trình Report chưa có giao diện trực quan (ví dụ: Metabase, Grafana) để theo dõi các tỉ lệ quarantine vs cleaned records.
