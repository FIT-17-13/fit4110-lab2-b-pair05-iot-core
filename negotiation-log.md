# Biên bản đàm phán hợp đồng Event

- Cặp đàm phán: IoT Ingestion ↔ Core Business (Pair 05)
- Product: B
- Producer: IoT Ingestion (B1)
- Consumer: Core Business (Nhóm 12 — B6)
- Phiên: v1.0
- Ngày: 2026-05-19

---

## Issue #1 — Đơn vị đo lường của sensor value

- Raised by: Consumer (Core Business)
- Event: `sensor.reading.created`
- Concern: IoT Ingestion thu thập dữ liệu từ nhiều loại cảm biến khác nhau. Nếu không thống nhất đơn vị, Core Business sẽ so sánh sai với ngưỡng cảnh báo (VD: nhận Fahrenheit nhưng ngưỡng cấu hình theo Celsius).
- Proposal: Thống nhất đơn vị cố định cho từng loại sensor: `celsius` (nhiệt độ), `percent` (độ ẩm), `ppm` (khói/CO2), `boolean` (chuyển động).
- Resolution: Accepted
- Rationale: Đơn vị cố định giúp Core Business không cần convert, giảm lỗi logic. IoT Ingestion chịu trách nhiệm convert về đơn vị chuẩn trước khi publish event.
- Impact: IoT Ingestion phải implement lớp chuyển đổi đơn vị (unit converter) trước khi đẩy event vào queue. Trường `unit` trong payload là enum cố định.

---

## Issue #2 — Có cần trường locationId không?

- Raised by: Consumer (Core Business)
- Event: `sensor.reading.created`, `sensor.threshold.exceeded`
- Concern: Core Business cần biết cảm biến nào ở đâu để gửi alert cho đúng khu vực. Nếu chỉ có `deviceId` mà không có vị trí, Core phải tự query thêm thông tin thiết bị — gây chậm xử lý.
- Proposal: IoT Ingestion thêm trường `locationId` (pattern: `BUILDING-X-ROOM-NNN`) vào mỗi event.
- Resolution: Accepted
- Rationale: IoT Ingestion đã biết vị trí thiết bị khi đăng ký sensor. Đính kèm location trực tiếp trong event giúp Core Business xử lý ngay mà không cần gọi thêm API lookup.
- Impact: Thêm field `locationId` vào payload. IoT Ingestion cần duy trì bảng mapping `deviceId → locationId` trong registry nội bộ.

---

## Issue #3 — Xử lý event bị trễ (stale event)

- Raised by: Consumer (Core Business)
- Event: Tất cả events
- Concern: Nếu mạng IoT bị trễ hoặc queue bị nghẽn, Core Business có thể nhận event có `timestamp` cách đây rất lâu (VD: 30 phút trước). Nếu vẫn xử lý bình thường sẽ gây alert sai hoặc thống kê lệch.
- Proposal: Core Business sẽ tự bỏ qua (discard) event có `timestamp` cách thời điểm xử lý > 5 phút, đồng thời log cảnh báo `STALE_EVENT_DISCARDED`.
- Resolution: Accepted
- Rationale: 5 phút là ngưỡng hợp lý cho hệ thống giám sát realtime. Event quá cũ không còn giá trị để kích hoạt cảnh báo.
- Impact: Core Business thêm logic kiểm tra `occurredAt` trước khi xử lý. IoT Ingestion cam kết timestamp chính xác theo đồng hồ NTP.

---

## Issue #4 — Idempotency và xử lý event trùng lặp

- Raised by: Producer (IoT Ingestion)
- Event: Tất cả events
- Concern: Với cơ chế at-least-once delivery của Message Queue, cùng một sensor reading có thể được gửi 2 lần khi retry. Nếu Core Business tạo 2 alert cho cùng 1 sự kiện sẽ gây nhiễu hệ thống.
- Proposal: Mỗi event bắt buộc có `eventId` (UUID v4) duy nhất. Core Business kiểm tra `eventId` đã xử lý chưa trước khi thực thi logic nghiệp vụ.
- Resolution: Accepted
- Rationale: Đây là pattern chuẩn cho hệ thống phân tán (idempotent consumer). IoT Ingestion sinh UUID cho mỗi lần đọc cảm biến.
- Impact: Core Business cần bảng/cache lưu `eventId` đã xử lý (TTL 1 giờ để tiết kiệm bộ nhớ). IoT Ingestion đảm bảo không tái sử dụng `eventId`.

---

## Issue #5 — Thứ tự event (Ordering) và xử lý out-of-order

- Raised by: Consumer (Core Business)
- Event: `sensor.reading.created`
- Concern: Message queue không đảm bảo thứ tự giao hàng (FIFO) giữa các partition/consumer. Core Business có thể nhận reading mới trước reading cũ từ cùng 1 sensor, dẫn đến đánh giá ngưỡng sai (VD: nhận giá trị 35°C rồi mới nhận 50°C — alert bị chậm).
- Proposal: IoT Ingestion phát event theo thứ tự thời gian cho cùng 1 deviceId. Core Business duy trì `lastTimestamp` per deviceId và bỏ qua event có timestamp cũ hơn.
- Resolution: Accepted
- Rationale: Ordering per-device là đủ (không cần global ordering). Nếu dùng Kafka, có thể partition theo deviceId để đảm bảo thứ tự trong cùng partition.
- Impact: IoT Ingestion dùng `deviceId` làm partition key. Core Business lưu `lastProcessedTimestamp` cho mỗi device trong cache/DB.

---

## Issue #6 — Retention và thời gian sống của message trên queue

- Raised by: Cả hai bên
- Event: Tất cả events
- Concern: Nếu Core Business tạm thời down hoặc restart, các event chưa xử lý cần được giữ lại trên queue bao lâu? Nếu giữ quá lâu sẽ tốn tài nguyên broker, nếu giữ quá ngắn sẽ mất event.
- Proposal: Message retention trên queue là 12 giờ. Sau 12 giờ, message tự động bị xóa (TTL). Core Business cần xử lý hết backlog trong 12 giờ sau khi khôi phục.
- Resolution: Accepted
- Rationale: 12 giờ đủ để Core Business phục hồi từ sự cố thông thường (maintenance window, restart). Event sensor cũ hơn 12 giờ cũng không còn giá trị giám sát realtime.
- Impact: Cấu hình queue TTL = 12h trên broker. Core Business cần monitoring để phát hiện sớm khi consumer lag tăng cao.

---

## Issue #7 — Chuẩn hóa cấu trúc event theo CloudEvents

- Raised by: Cả hai bên
- Event: Tất cả events
- Concern: Payload cần cấu trúc nhất quán để dễ parse, route và debug. Nếu mỗi event có format riêng thì rất khó maintain khi hệ thống mở rộng.
- Proposal: Áp dụng chuẩn CloudEvents cho tất cả event, chia rõ metadata (eventId, eventType, occurredAt, source) và business data (nằm trong trường `data`).
- Resolution: Accepted
- Rationale: CloudEvents là chuẩn mở CNCF, được hỗ trợ bởi hầu hết Message Broker. Giúp middleware có thể lọc/route event dựa trên `eventType` mà không cần parse `data`.
- Impact: Mọi event đều có cấu trúc: `eventId`, `eventType`, `occurredAt`, `correlationId`, `source`, `data: {...}`. Các trường nghiệp vụ chỉ nằm trong `data`.

---

# Chốt hợp đồng sơ bộ v1.0

Producer sign-off: IoT Ingestion (B1)
Consumer sign-off: Nhóm 12 — Core Business (B6)
Witness (GV/TA):   _________________
Date:              2026-05-19
