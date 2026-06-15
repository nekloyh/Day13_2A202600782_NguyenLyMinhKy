# Observathon – Report v1

**Sinh viên:** Nguyễn Lý Minh Kỳ  
**Phase:** public | **Sim:** 120/120 `ok`

---

## Quá trình xây dựng solution

### Bước 1 — Quan sát & chẩn đoán

Chạy practice sim lần đầu với config và prompt gốc để quan sát hành vi agent. Dùng `wrapper.py` để ghi log toàn bộ meta trả về từ `call_next()` (latency, token usage, tool count, status). Từ đó phát hiện các dấu hiệu bất thường: tỉ lệ lỗi cao, latency không có trần, agent gọi tool lặp lại, đầu ra chứa email/SĐT khách hàng, và tổng tiền sai.

### Bước 2 — Sửa config

Đối chiếu hành vi quan sát được với từng knob trong `config.json`. Xác định các tham số bị cài sai cố tình: `tool_error_rate=0.18`, `retry.enabled=false`, `timeout_ms=0`, `temperature=1.6`, `verbose_system=true`, `session_drift_rate=0.06`, `context_reset_every=0`, `loop_guard=false`, `redact_pii=false`, `normalize_unicode=false`, `catalog_override` chặn sản phẩm, `tool_budget=0`. Chỉnh lại toàn bộ về giá trị hợp lý.

### Bước 3 — Viết lại system prompt

Prompt gốc chỉ có 1 dòng ("Help the customer and give a total in VND") — không đủ để agent hoạt động đúng. Viết lại hoàn toàn với cấu trúc rõ ràng: STEPS tuần tự bắt buộc, công thức tính VND tường minh, luật từ chối khi hết hàng, luật không lặp PII, và luật bảo vệ khỏi injection qua trường GHI CHÚ.

### Bước 4 — Xây dựng wrapper

Bổ sung các tầng bảo vệ mà prompt và config không kiểm soát được: sanitize input (xóa lệnh giả trong GHI CHÚ trước khi gửi agent), cache kết quả để tránh gọi LLM lặp, retry tự động khi agent trả về lỗi, và redact PII trên output đầu ra làm lớp an toàn cuối.

### Bước 5 — Chạy public sim

Chạy lại với toàn bộ các thay đổi trên bộ public. Kết quả: 120/120 requests `ok`.

---

*Sẽ cập nhật sau khi fine-tune với private data (injection twist + paraphrased questions).*
