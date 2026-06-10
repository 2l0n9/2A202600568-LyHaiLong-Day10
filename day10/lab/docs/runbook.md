# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

> User / agent thấy gì? (VD: trả lời “14 ngày” thay vì 7 ngày)

Hệ thống RAG Agent trả lời sai thông tin chính sách cho người dùng. Ví dụ: Khách hàng hỏi thời hạn hoàn tiền, Agent trả lời "14 ngày" thay vì chính sách mới nhất là "7 ngày". Hoặc HR Agent trả lời nhân viên dưới 3 năm được 10 ngày phép (thay vì 12 ngày).

---

## Detection

> Metric nào báo? (freshness, expectation fail, eval `hits_forbidden`)

1. **Expectation fail:** Pipeline chạy bị dừng lại (HALT) ở log: `expectation[refund_no_stale_14d_window] FAIL`.
2. **Eval Metric:** Dashboard đánh giá hàng ngày báo cáo `hits_forbidden: true` và `contains_expected: false`.
3. **Freshness Metric:** Cảnh báo `freshness_sla_exceeded` do dữ liệu không được sync kịp.

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | Xem run_id gần nhất có `skipped_validate` = true hay không. |
| 2 | Mở `artifacts/quarantine/*.csv` | Kiểm tra xem các câu chứa "14 ngày" có bị flag vào quarantine hay không. Nếu không, rules cleaning có vấn đề. |
| 3 | Chạy `python eval_retrieval.py` | Kiểm tra điểm số của retriever, xác nhận Model đang query sai Vector. |

---

## Mitigation

> Rerun pipeline, rollback embed, tạm banner “data stale”, …

1. Tạm thời gắn banner "Hệ thống đang cập nhật, chính sách có thể chưa chính xác" trên UI của Agent.
2. Cập nhật `cleaning_rules.py` để bổ sung Regex lọc bỏ dữ liệu rác/cũ.
3. Chạy lại `python etl_pipeline.py run` để tạo Cleaned CSV mới.
4. Đảm bảo Idempotent Upsert chạy thành công để đè lại các Vector bị sai trong ChromaDB.

---

## Prevention

> Thêm expectation, alert, owner — nối sang Day 11 nếu có guardrail.

1. Bổ sung Expectation chặt chẽ (mức độ `halt`) để cấm tiệt chuỗi "14 ngày làm việc" lọt qua vòng Ingestion.
2. Tích hợp Guardrail vào prompt của Agent (Day 11) để cản Agent trả lời nếu retrieve ra văn bản đáng ngờ.
