# Phân tích yêu cầu — vai Producer (IoT Ingestion)

- Cặp đàm phán: Pair 05 — IoT Ingestion → Core Business
- Product: B
- Producer service: IoT Ingestion (B1)
- Consumer service: Core Business (Nhóm 12 — B6)
- Người viết: Nhóm B1 (IoT Ingestion)
- Ngày: 2026-05-19

---

## 1. Event chính Producer sẽ phát

| Event | Mô tả | Trigger | Tần suất |
|---|---|---|---|
| `sensor.reading.created` | Dữ liệu đo mới từ cảm biến | Mỗi chu kỳ polling (5–30s) | Cao (hàng nghìn/phút) |
| `sensor.threshold.exceeded` | Giá trị cảm biến vượt ngưỡng cảnh báo | Khi value > threshold đã cấu hình | Thấp (chỉ khi bất thường) |

---

## 2. Payload field dự kiến

| Field | Kiểu | Bắt buộc | Mô tả |
|---|---|---|---|
| `deviceId` | string | ✅ | Mã cảm biến, pattern `SENSOR-NNN` |
| `sensorType` | string (enum) | ✅ | Loại: temperature, humidity, smoke, motion |
| `value` | number | ✅ | Giá trị đo được |
| `unit` | string (enum) | ✅ | Đơn vị: celsius, percent, ppm, boolean |
| `locationId` | string | ✅ | Vị trí: `BUILDING-X-ROOM-NNN` |
| `timestamp` | string (date-time) | ✅ | Thời điểm đo, ISO 8601 UTC |
| `threshold` | number | Chỉ threshold event | Ngưỡng bị vượt |
| `severity` | string (enum) | Chỉ threshold event | LOW, MEDIUM, HIGH, CRITICAL |

---

## 3. Giả định bổ sung

- Giả định 1: IoT Ingestion đã convert đơn vị về chuẩn thống nhất trước khi publish (VD: Fahrenheit → Celsius).
- Giả định 2: `deviceId` luôn đã được đăng ký trong device registry. Nếu sensor chưa đăng ký, IoT Ingestion sẽ không publish event.
- Giả định 3: Tần suất publish `sensor.reading.created` có thể rất cao (1 event/5 giây/sensor). Consumer cần thiết kế xử lý bất đồng bộ.

---

## 4. Câu hỏi cho Consumer

1. Core Business có muốn nhận mọi reading hay chỉ cần aggregated data (trung bình mỗi phút)?
2. Khi nhận `sensor.threshold.exceeded`, Core Business có tự tạo Alert luôn hay cần confirm thủ công?
3. Core Business có cần IoT Ingestion gửi event khi sensor offline (heartbeat timeout)?

---

## 5. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Sensor gửi dữ liệu rác (value = -999) | Core Business tạo alert sai | IoT Ingestion validate giá trị trước khi publish |
| Queue bị nghẽn giờ cao điểm | Event bị trễ, alert chậm | Tách topic riêng cho threshold exceeded (ưu tiên cao) |
| Sensor mất kết nối | Core Business không biết khu vực mất giám sát | Dự kiến thêm event `sensor.offline` ở Lab 03 |
