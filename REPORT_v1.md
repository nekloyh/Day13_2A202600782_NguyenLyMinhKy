# Observathon – Report v1

**Sinh viên:** Nguyễn Lý Minh Kỳ  
**Phase:** private | **Sim:** 80/80 `ok` | **Headline:** 92.95 / 100

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

### Bước 6 — Fine-tune với private data

Bộ private (80 request) thêm hai thử thách chính: (a) **prompt injection** nhúng trong trường `GHI CHU` với giá giả (`1.000.000 VND`), và (b) **paraphrase** câu hỏi theo nhiều cách diễn đạt khác nhau.

Các điều chỉnh so với public:
- Mở rộng regex nhận diện số lượng (`lấy/cần/muốn`, mẫu "N cái/chiếc") và điểm đến (`vận chuyển`) để bắt các paraphrase mới.
- Bổ sung retry logic trong wrapper: khi agent có điểm đến trong đơn nhưng không gọi `calc_shipping`, wrapper gọi lại một lần với chỉ thị tường minh — lấy phí ship thật từ trace thay vì tự tính.

**Kết quả private scorer (`score_private.json`):**

| Chỉ số | Điểm |
|---|---|
| **Headline** | **92.95** |
| correct (53/80) | 72.25% |
| quality | 83.35% |
| error | 100% |
| latency | 74.48% |
| cost | 0% ⚠ |
| drift | 52.4% |
| prompt | 79.11% |
| **diag_f1** | **100%** |

**Điểm mạnh:** error = 100% (retry + config), diag_f1 = 100% (chẩn đoán đủ 11 fault class).  
**Điểm yếu cần điều tra:** cost = 0% (có thể do sim không ghi nhận cost với `standard` tier); drift = 52.4% (session drift vẫn còn ảnh hưởng); correct = 72.25% (27 đơn sai — cần phân tích thêm từ `run_private_v3.json`).

---
