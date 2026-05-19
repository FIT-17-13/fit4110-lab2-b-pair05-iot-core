# Versioning — Event Contract: IoT Ingestion ↔ Core Business

## Phiên bản hiện tại

| Phiên bản | Ngày | Trạng thái | Ghi chú |
|---|---|---|---|
| v1.0.0 | 2026-05-19 | Signed-off | Hợp đồng sự kiện sơ bộ giữa IoT Ingestion (B1) và Core Business (B6) |

## Quy tắc versioning

Dùng **Semantic Versioning** (SemVer):

- **MAJOR** (x.0.0): Breaking change — thay đổi không tương thích ngược (VD: xóa field bắt buộc, đổi tên event)
- **MINOR** (1.x.0): Feature mới tương thích ngược (VD: thêm event mới, thêm field tùy chọn trong data)
- **PATCH** (1.0.x): Sửa lỗi, cập nhật description, không ảnh hưởng logic

## Lịch sử thay đổi

### v1.0.0 — 2026-05-19

- Khởi tạo Event Contract sơ bộ cho pair-05 (IoT Ingestion → Core Business)
- Events: `sensor.reading.created`, `sensor.threshold.exceeded`
- Payload: JSON chuẩn CloudEvents với `eventId`, `correlationId`, `deviceId`, `sensorType`, `value`, `unit`, `locationId`
- Đàm phán 6 issue (đơn vị đo, locationId, stale event, idempotency, tách event, CloudEvents)
- Đã sign-off Producer (B1) và Consumer (B6)
