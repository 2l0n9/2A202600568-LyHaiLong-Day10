# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `policy_refund_v4` | Export định kỳ | Chứa bản cũ (14 ngày làm việc) | `refund_no_stale_14d_window` |
| `hr_leave_policy` | Export định kỳ | Xung đột version (10 vs 12 ngày) | `hr_leave_no_stale_10d_annual` |
| `sla_p1_2026` | Export định kỳ | Không đồng bộ semantic ngữ nghĩa | `no_noise_phrases`, `no_stuttering_words` |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | Khóa chính duy nhất dùng để deduplicate |
| doc_id | string | Có | Mã tài liệu gốc (VD: policy_refund_v4) |
| chunk_text | string | Có | Đoạn văn bản sau khi đã được làm sạch rác |
| effective_date | date | Có | Ngày áp dụng chính sách |
| exported_at | datetime | Có | Ngày dữ liệu được export từ DB |

---

## 3. Quy tắc quarantine vs drop

> Record bị flag đi đâu? Ai approve merge lại?

- **Quy tắc:** Các record vi phạm expectations (VD: trùng lặp, thiếu chunk_id, văn bản chứa rác "Nội dung không rõ ràng", "sync lại dữ liệu") sẽ bị loại bỏ khỏi luồng chính.
- **Lưu trữ:** Lưu vào `artifacts/quarantine/quarantine_[run_id].csv`.
- **Merge lại:** Đội ngũ Data Engineering phân tích file quarantine, cập nhật `cleaning_rules.py` để xử lý ngoại lệ và chạy lại pipeline. Không ai được quyền sửa tay file cleaned CSV.

---

## 4. Phiên bản & canonical

> Source of truth cho policy refund: file nào / version nào?

- **Source of truth:** Các file CSV gốc được xuất từ hệ thống quản lý tài liệu. Phiên bản hiện hành dựa vào trường `effective_date`. Ví dụ: `policy_refund_v4` với ngày hiệu lực mới nhất (7 ngày hoàn tiền).
