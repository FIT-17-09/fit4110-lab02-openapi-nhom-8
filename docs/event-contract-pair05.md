# Event Contract sơ bộ — Pair 05

> File dùng cho Lab 02, chưa cần AsyncAPI đầy đủ.

## 1. Thông tin dependency

| Mục | Giá trị |
|---|---|
| Dependency số | 05 |
| Producer | IoT Ingestion (B1) |
| Consumer | Core Business (B6) |
| Cơ chế | Queue async |
| Event/topic dự kiến | `sensor.reading.created` |
| Người ghi | B1 |
| Ngày | 2026-05-18 |

## 2. Mục đích nghiệp vụ

Khi IoT device gửi một reading mới (nhiệt độ, áp suất, ánh sáng, chất lượng không khí, độ ẩm), IoT Ingestion nhận và publish event `sensor.reading.created`. Core Business nhận event này để:
- Cập nhật trạng thái device
- Kiểm tra threshold, trigger alert nếu vượt ngưỡng
- Log lịch sử reading

Trigger: Device gửi HTTP/TCP request chứa raw sensor data → IoT Ingestion normalize → publish event.

## 3. Payload tối thiểu

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "sensor.reading.created",
  "occurredAt": "2026-05-18T12:00:00Z",
  "correlationId": "660e8400-e29b-41d4-a716-446655440001",
  "source": "iot-ingestion",
  "data": {
    "deviceId": "device-001",
    "sensorType": "temperature",
    "value": 25.5,
    "unit": "°C",
    "location": "building-a/floor-2/room-201"
  }
}
```

### Ràng buộc payload

| Field | Kiểu | Bắt buộc | Constraint |
|---|---|---|---|
| eventId | string (UUID v4) | Có | Do IoT Ingestion sinh |
| eventType | string | Có | Pattern: `sensor.<noun>.<verb>` |
| occurredAt | string (ISO 8601 UTC) | Có | Thời điểm device gửi |
| correlationId | string (UUID) | Có | Trace luồng; IoT Ingestion tự sinh nếu không có |
| source | string | Có | `iot-ingestion` |
| data.deviceId | string | Có | ID device |
| data.sensorType | string | Có | Enum: `temperature`, `pressure`, `light`, `air_quality`, `humidity` |
| data.value | number | Có | Đã normalize về SI |
| data.unit | string | Có | Enum: `°C`, `Pa`, `lux`, `ppm`, `μg/m³`, `%` |
| data.location | string | Có | Đường dẫn phân cấp |

## 4. Ràng buộc delivery

| Vấn đề | Quyết định |
|---|---|
| Event id bắt buộc | Có, `eventId` bắt buộc |
| CorrelationId | Có |
| Retry | Tối đa 3 lần, exponential backoff (1s, 2s, 4s) |
| Dead-letter queue | DLQ: `campus.iot.dlq`, retention 7 ngày |
| Duplicate | Có thể, Consumer idempotent (deduplicate theo eventId trong 24h) |

## 5. Issue chuyển sang Lab 03

1. Xác định broker cụ thể (RabbitMQ / Kafka) và cấu hình queue/topic
2. Quyết định DLQ routing và alerting khi có message vào DLQ
3. Mở rộng `sensorType` enum khi thêm loại sensor mới
