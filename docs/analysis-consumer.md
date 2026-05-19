# Phân tích yêu cầu — vai Consumer (Core Business)

- Cặp đàm phán: Pair 05 — IoT Ingestion → Core Business
- Product: B
- Consumer service: Core Business (Nhóm 12 — B6)
- Producer service: IoT Ingestion (B1)
- Người viết: Nhóm 12
- Ngày: 2026-05-19

---

## 1. Event Consumer cần nhận

| Event | Consumer dùng để làm gì? | Field bắt buộc với Consumer |
|---|---|---|
| `sensor.reading.created` | Lưu trữ dữ liệu cảm biến, phục vụ thống kê và phát hiện xu hướng bất thường | deviceId, sensorType, value, unit, locationId, timestamp |
| `sensor.threshold.exceeded` | Tự động tạo Alert trong hệ thống, gửi thông báo qua Notification service | deviceId, sensorType, value, unit, threshold, severity, locationId, timestamp |

---

## 2. Luồng xử lý khi nhận event

| Event | Consumer sẽ làm gì? | Thời gian xử lý dự kiến |
|---|---|---|
| `sensor.reading.created` | Lưu vào DB sensor_readings, cập nhật dashboard realtime | Batch mỗi 10 giây |
| `sensor.threshold.exceeded` | Tạo Alert (severity tương ứng), trigger event `alert.created` cho Notification và Analytics | Xử lý ngay (< 500ms) |

---

## 3. Error case Consumer cần xử lý

| Tình huống | Consumer xử lý thế nào? |
|---|---|
| Event trùng lặp (cùng eventId) | Bỏ qua, log `DUPLICATE_EVENT_SKIPPED` |
| Event quá cũ (timestamp > 5 phút trước) | Discard, log `STALE_EVENT_DISCARDED` |
| deviceId không tồn tại trong hệ thống | Bỏ qua, log `UNKNOWN_DEVICE`, không tạo alert |
| value nằm ngoài khoảng hợp lý (VD: nhiệt độ = -500°C) | Bỏ qua, log `INVALID_SENSOR_VALUE` |
| Queue consumer bị crash giữa chừng | Message không được ACK, broker tự redeliver |

---

## 4. Giả định bổ sung

- Giả định 1: Core Business chỉ cần nhận event đã được IoT Ingestion validate sơ bộ. Dữ liệu rác hoặc sensor chưa đăng ký không nên xuất hiện trên queue.
- Giả định 2: `sensor.threshold.exceeded` luôn đi kèm severity, Core Business dùng severity này để quyết định mức độ ưu tiên của Alert.
- Giả định 3: Core Business sẽ không gửi response/ACK nghiệp vụ lại cho IoT Ingestion (fire-and-forget từ góc IoT).

---

## 5. Câu hỏi cho Producer

1. Khi sensor offline > 5 phút, IoT Ingestion có gửi event `sensor.offline` không?
2. Có bao nhiêu sensor tối đa trong hệ thống? (Để Core Business ước lượng throughput)
3. IoT Ingestion có filter bớt reading không quan trọng trước khi publish không? (VD: giá trị không thay đổi so với lần trước)

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| IoT gửi quá nhiều reading (flood) | Core Business bị nghẽn, lag dashboard | IoT Ingestion throttle hoặc gom batch |
| Đồng hồ sensor lệch múi giờ | Timestamp sai, event bị discard nhầm | IoT Ingestion sync NTP trước khi publish |
| Sensor gửi giá trị dao động quanh ngưỡng | Spam alert liên tục (flapping) | Dự kiến Lab 03: thêm debounce logic |
