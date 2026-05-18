# Phân tích yêu cầu — vai Provider (IoT Ingestion)

- Cặp đàm phán: Pair 05 (IoT Ingestion → Core Business) + Pair 06 (IoT Ingestion → Analytics)
- Product: B
- Provider service: IoT Ingestion
- Consumer service: Core Business (Pair 05), Analytics (Pair 06)
- Người viết: NGUYỄN XUÂN PHÚC
- Ngày: 2026-05-18

---

## 1. Resource chính

### Pair 05 — IoT Ingestion → Core Business

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `sensor.reading.created` | Sự kiện cảm biến gửi dữ liệu đo mới | `deviceId` (string), `sensorType` (string), `value` (number), `unit` (string), `timestamp` (string ISO 8601) | `locationId` (string), `metadata` (object), `correlationId` (string) |
| `sensor.threshold.exceeded` | Sự kiện giá trị vượt ngưỡng cho phép | `deviceId`, `sensorType`, `value`, `threshold`, `unit`, `timestamp` | `locationId`, `severity` (string: low/medium/high), `correlationId` |

### Pair 06 — IoT Ingestion → Analytics

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `telemetry.ingested` | Sự kiện telemetry tổng hợp gửi về Analytics | `deviceId`, `sensorType`, `value`, `unit`, `timestamp` | `locationId`, `zoneId` (string), `batchId` (string), `metadata` |
| `device.status.changed` | Sự kiện trạng thái thiết bị thay đổi | `deviceId`, `status` (string: online/offline/error), `timestamp` | `locationId`, `reason` (string), `previousStatus` |

---

## 2. Action/API dự kiến

### Pair 05

| Method | Topic/Queue | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| POST (publish) | `campus.iot.sensor.reading.created` | Gửi event đọc cảm biến mới | Khi device gửi dữ liệu đo lên hệ thống |
| POST (publish) | `campus.iot.sensor.threshold.exceeded` | Gửi event vượt ngưỡng | Khi giá trị đọc vượt ngưỡng threshold đã cấu hình |

### Pair 06

| Method | Topic/Queue | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| POST (publish) | `campus.iot.telemetry.ingested` | Gửi telemetry tổng hợp | Khi IoT Ingestion nhận và xử lý xong dữ liệu từ device |
| POST (publish) | `campus.iot.device.status.changed` | Thông báo trạng thái device thay đổi | Khi device chuyển trạng thái (online→offline hoặc ngược lại) |

---

## 3. Error case

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| 400 | Payload sai định dạng (không đúng JSON hoặc thiếu field bắt buộc) | `{ "type": "https://campus.api/errors/invalid-payload", "title": "Invalid Payload", "status": 400, "detail": "Field 'deviceId' is required", "instance": "/telemetry/ingested" }` |
| 401 | Message thiếu hoặc sai correlation/request ID | `{ "type": "https://campus.api/errors/missing-correlation-id", "title": "Missing Correlation ID", "status": 401, "detail": "correlationId must be present in message headers", "instance": "/telemetry/ingested" }` |
| 422 | Payload đúng JSON nhưng giá trị không hợp lệ nghiệp vụ (value = NaN, sensorType không nằm trong danh mục) | `{ "type": "https://campus.api/errors/business-rule-violation", "title": "Business Rule Violation", "status": 422, "detail": "sensorType 'unknown' is not supported", "instance": "/sensor/reading/created" }` |
| 409 | Event trùng lặp (duplicate) — retry gửi lại cùng một event | `{ "type": "https://campus.api/errors/duplicate-event", "title": "Duplicate Event", "status": 409, "detail": "Event with id 'abc123' already processed", "instance": "/sensor/threshold/exceeded" }` |
| 503 | Queue/broker không khả dụng, không thể publish event | `{ "type": "https://campus.api/errors/broker-unavailable", "title": "Service Unavailable", "status": 503, "detail": "Message broker is temporarily unavailable. Event stored for retry.", "instance": "/telemetry/ingested" }` |

---

## 4. Giả định bổ sung

Ghi rõ những điểm user story chưa nói nhưng Provider cần giả định.

**Pair 05:**

- Đơn vị sensor (`unit`): dùng chuẩn SI cơ bản (°C cho nhiệt độ, Pa cho áp suất, lux cho ánh sáng, ppm cho chất lượng không khí). Nếu device gửi đơn vị khác, IoT Ingestion sẽ convert trước khi publish.
- Ngưỡng threshold: Core Business tự định nghĩa threshold riêng cho từng `sensorType` + `deviceId`; IoT Ingestion chỉ publish event khi threshold được cấu hình sẵn trong hệ thống.
- Latency chấp nhận: Core Business xử lý event trong vòng **30 giây** kể từ timestamp. Sau 30s, event có thể coi là outdated.
- Correlation ID: Consumer phải gửi kèm `correlationId` để trace request end-to-end; nếu thiếu, IoT Ingestion sẽ tự sinh UUID và log cảnh báo.
- Retry: nếu publish thất bại, hệ thống retry tối đa **3 lần** với exponential backoff (1s, 2s, 4s), sau đó đưa vào dead-letter queue.

**Pair 06:**

- Aggregation key: Analytics aggregate theo `deviceId` và `sensorType`, có thể drill-down theo `zoneId` nếu có.
- Batch: IoT Ingestion có thể gửi telemetry dạng batch (nhiều reading trong một message) nếu device gửi dữ liệu liên tục; mỗi batch có `batchId` duy nhất.
- Dead-letter: event payload không parse được → gửi vào dead-letter queue `campus.iot.dlq` để DevOps investigate, không retry vô hạn.
- Retention: Analytics expect data trong vòng **7 ngày** kể từ timestamp; sau 7 ngày data có thể bị archive hoặc xóa.

---

## 5. Câu hỏi cho Consumer

**Cho Core Business (Pair 05):**

1. Core Business cần nhận `locationId` bắt buộc hay tùy chọn? Nếu device không có location, có chấp nhận để trống không?
2. Ngưỡng threshold do ai quản lý — Core Business hay IoT Ingestion? Nếu Core Business quản lý, cơ chế cập nhật threshold như thế nào?
3. Core Business xử lý trễ (quá 30s) thì có cần notify IoT Ingestion không, hay chỉ bỏ qua event?
4. Event `sensor.threshold.exceeded` có cần gửi kèm giá trị threshold đã vượt không, hay chỉ cần thông báo deviceId + sensorType?

**Cho Analytics (Pair 06):**

5. Analytics aggregate theo `deviceId` hay theo `zoneId` là chính? Có cần cả hai không?
6. Batch event (nhiều reading trong một message) có được Analytics xử lý không, hay chỉ nhận từng event riêng lẻ?
7. Event `device.status.changed` có cần field `previousStatus` không, hay chỉ cần `status` hiện tại?
8. Analytics cần retention bao lâu — 7 ngày hay dài hơn? Dữ liệu lỗi (format sai) được đưa vào dead-letter hay bỏ qua hoàn toàn?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Tên event không thống nhất (Core Business dùng `sensor.reading.new` thay vì `sensor.reading.created`) | Consumer không nhận được event → mất dữ liệu | Chốt event name trong contract, version event name theo semver (e.g., `sensor.reading.created.v1`) |
| Đơn vị (`unit`) không đồng nhất giữa các device | Analytics aggregate sai giá trị | IoT Ingestion normalize về đơn vị chuẩn SI trước khi publish; liệt kê đơn vị hợp lệ trong contract |
| Duplicate event (retry không idempotent) | Core Business xử lý event 2 lần → trùng alert, sai thống kê | Mỗi event có `eventId` duy nhất; Consumer dùng idempotent key (eventId) để deduplicate |
| Event đến out-of-order | Analytics aggregate theo thời gian bị sai | Mỗi event có `timestamp` (ISO 8601, UTC); Consumer sort lại trước khi aggregate |
| Payload lớn (metadata chứa ảnh/base64) | Queue bị lag, timeout | Giới hạn payload size (recommend < 64KB); metadata không bắt buộc |
| Consumer không handle error đúng cách | Event bị drop không rõ lý do | Thống nhất dùng Problem Details JSON khi publish vào DLQ; log rõ eventId và lý do |
