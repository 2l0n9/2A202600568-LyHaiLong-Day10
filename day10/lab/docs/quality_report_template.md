# Quality report — Lab Day 10 (nhóm)

**run_id:** `2026-06-10T07-00Z`
**Ngày:** 10/06/2026

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước | Sau | Ghi chú |
|--------|-------|-----|---------|
| raw_records | 247 | 247 | Tổng số bản ghi đầu vào |
| cleaned_records | 46 | 35 | Đã fix lỗi luồng Deduplication |
| quarantine_records | 201 | 212 | Rác và câu trùng lặp đã bị loại bỏ |
| Expectation halt? | YES | NO | Các rule dọn dẹp đã giải quyết 100% violation |

---

## 2. Before / after retrieval (bắt buộc)

> Đính kèm hoặc dẫn link tới `artifacts/eval/before_after_eval.csv` (hoặc 2 file before/after).

**Câu hỏi then chốt:** Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi đơn được xác nhận?  
**Trước:** "Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc kể từ xác nhận đơn." (FAIL)
**Sau:** "Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng." (PASS)

**Merit (khuyến nghị):** versioning HR — `Nếu không có phản hồi với ticket P1 sau bao lâu thì hệ thống auto escalate?`

**Trước:** (Lấy sai sang Ticket P2 do Semantic Match yếu)
**Sau:** "Nếu không có phản hồi với ticket P1 trong 10 phút thì hệ thống auto escalate lên Senior Engineer." (Nhờ Document Expansion)

---

## 3. Freshness & monitor

> Kết quả `freshness_check` (PASS/WARN/FAIL) và giải thích SLA bạn chọn.

- **Kết quả:** FAIL
- **Log:** `freshness_check=FAIL {"latest_exported_at": "2026-04-11T00:00:00", "age_hours": 1447, "sla_hours": 24.0, "reason": "freshness_sla_exceeded"}`
- **Giải thích:** SLA được đặt là 24 giờ để đảm bảo chính sách luôn được cập nhật trong ngày. Tuy nhiên, dữ liệu lớn nhất trong file export có timestamp từ tháng 4/2026, chứng tỏ Ingestion Job đã không chạy trong nhiều tháng qua.

---

## 4. Corruption inject (Sprint 3)

> Mô tả cố ý làm hỏng dữ liệu kiểu gì (duplicate / stale / sai format) và cách phát hiện.

- **Cách làm:** Chạy `python etl_pipeline.py run --no-refund-fix --skip-validate`
- **Mục đích:** Bỏ qua khâu sửa thời hạn hoàn tiền từ 14 ngày về 7 ngày. Bỏ qua Expectation Halt để cố tình đẩy dữ liệu sai vào VectorDB.
- **Phát hiện:** Quá trình Eval RAG báo `hits_forbidden: true` cho câu hỏi về refund, chứng minh nếu Data Pipeline không dọn dẹp tốt, Model sẽ truy xuất nhầm thông tin độc hại.

---

## 5. Hạn chế & việc chưa làm

- Cần tích hợp Notification (nhắn tin Slack) khi Freshness bị FAIL để đội Data Ops phản ứng kịp thời.
