# Event Contract sơ bộ — Pair 06

> File dùng cho Lab 02, chưa cần AsyncAPI đầy đủ.

## 1. Thông tin dependency

| Mục | Giá trị |
|---|---|
| Dependency số | 06 |
| Producer | IoT Ingestion (B1) |
| Consumer | Analytics (B5) |
| Cơ chế | Queue async |
| Event/topic dự kiến | `telemetry.ingested` |
| Người ghi | B1 |
| Ngày | 2026-05-18 |

## 2. Mục đích nghiệp vụ

Khi IoT device gửi telemetry data, IoT Ingestion nhận và forward event `telemetry.ingested` tới Analytics. Analytics dùng event này để:
- Aggregate dữ liệu theo thời gian (time-series aggregation)
- Tính toán KPI (trung bình, min, max theo khoảng thời gian)
- Vẽ biểu đồ dashboard theo zone/building

Trigger: Tương tự Pair 05, event được forward sau khi IoT Ingestion nhận từ device.

## 3. Payload tối thiểu

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "telemetry.ingested",
  "occurredAt": "2026-05-18T12:00:00Z",
  "correlationId": "660e8400-e29b-41d4-a716-446655440001",
  "source": "iot-ingestion",
  "data": {
    "deviceId": "device-001",
    "sensorType": "temperature",
    "value": 25.5,
    "unit": "°C",
    "location": "building-a/floor-2/room-201",
    "batchId": null
  }
}
```

### Ràng buộc payload

| Field | Kiểu | Bắt buộc | Constraint |
|---|---|---|---|
| eventId | string (UUID v4) | Có | Do IoT Ingestion sinh |
| eventType | string | Có | Pattern: `sensor.<noun>.<verb>` |
| occurredAt | string (ISO 8601 UTC) | Có | Thời điểm device gửi (để sort) |
| correlationId | string (UUID) | Có | Trace luồng; IoT Ingestion tự sinh nếu không có |
| source | string | Có | `iot-ingestion` |
| data.deviceId | string | Có | ID device |
| data.sensorType | string | Có | Enum: `temperature`, `pressure`, `light`, `air_quality`, `humidity` |
| data.value | number | Có | Đã normalize về SI |
| data.unit | string | Có | Enum: `°C`, `Pa`, `lux`, `ppm`, `μg/m³`, `%` |
| data.location | string | Có | Đường dẫn phân cấp |
| data.batchId | string or null | Không | Dùng cho V2 batch; V1 để null |

## 4. Ràng buộc delivery

| Vấn đề | Quyết định |
|---|---|
| Event id bắt buộc | Có, `eventId` bắt buộc |
| CorrelationId | Có |
| Retry | Tối đa 3 lần, exponential backoff (1s, 2s, 4s) |
| Dead-letter queue | DLQ: `campus.iot.dlq`, retention 7 ngày |
| Duplicate | Có thể, Consumer idempotent (deduplicate theo eventId trong 24h) |
| Out-of-order | Analytics phải sort theo `occurredAt` trước khi aggregate |

## 5. Issue chuyển sang Lab 03

1. Xác định broker cụ thể và cấu hình topic/partition cho Analytics
2. Quyết định window aggregation (5 phút, 15 phút, 1 giờ) — Analytics sẽ define
3. Mở rộng V2: hỗ trợ `batchId` + mảng `readings[]` để giảm số lượng message
4. DLQ alerting và retry policy chi tiết cho batch mode
