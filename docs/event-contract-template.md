# Event Contract sơ bộ — dùng cho dependency Queue async

> File này chỉ dùng cho các cặp Queue async ở Lab 02 để ghi nhận thỏa thuận ban đầu. Đặc tả chi tiết bằng AsyncAPI sẽ chuyển sang Lab 03.

## 1. Thông tin dependency

- **Dependency số:** 5
- **Producer:** IoT Ingestion (A1/B1)
- **Consumer:** Core Business (A6/B6)
- **Cơ chế:** Queue async
- **Event/topic dự kiến:** `sensor.reading.created`
- **Người ghi:** Nhóm 8
- **Ngày:** 2026-05-18

## 2. Mục đích nghiệp vụ

Event này được trigger tự động liên tục (ví dụ: mỗi 10 giây) từ các thiết bị cảm biến (nhiệt độ, độ ẩm, cảnh báo khói) đặt tại các khu vực trong Smart Campus. Service Core Business (Consumer) sẽ subscribe topic này để đánh giá các rule policy theo thời gian thực.

_Ví dụ:_ Nếu `sensorType` là `temperature` và `value` > 45°C, Core Business sẽ lập tức thay đổi trạng thái khu vực và kích hoạt luồng cảnh báo sang Notification Service.

## 3. Event name / topic

| Mục         | Giá trị                           |
| ----------- | --------------------------------- |
| Event name  | `sensor.reading.created`          |
| Topic/queue | `smartcampus.iot.sensor.readings` |
| Producer    | `iot-ingestion-service`           |
| Consumer    | `core-business-service`           |

## 4. Payload tối thiểu

```json
{
  "eventId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "eventType": "sensor.reading.created",
  "occurredAt": "2026-05-18T11:00:00Z",
  "correlationId": "req-9876-abc-1234",
  "source": "iot-ingestion-service",
  "data": {
    "deviceId": "temp-sensor-lab-ai-01",
    "zoneId": "zone-building-A-floor-3",
    "sensorType": "temperature",
    "value": 39.5,
    "unit": "Celsius",
    "batteryLevel": 85
  }
}
```

## 5. Ràng buộc cần thống nhất

| Vấn đề                             | Quyết định tạm thời                                                                                                                                                                            |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Event id có bắt buộc không?        | **Có** (Bắt buộc dùng UUID v4 để đảm bảo tính duy nhất).                                                                                                                                       |
| Có cần correlationId không?        | **Có** (Bắt buộc để trace log phân tán qua hệ thống giám sát).                                                                                                                                 |
| Có cho phép gửi trùng event không? | **Có thể** (Do cơ chế at-least-once delivery và network retry). Nhóm Core Business thống nhất sẽ dùng `eventId` làm Idempotency Key để tự động bỏ qua event trùng lặp (lưu cache trong Redis). |
| Độ trễ (Stale data)                | Dữ liệu trễ quá 2 phút so với `occurredAt` sẽ bị Core Business tự động drop vì không còn giá trị cảnh báo realtime.                                                                            |
| Retry khi lỗi                      | Core Business sẽ retry tối đa 3 lần nếu lỗi kết nối database nội bộ, sau đó đẩy sang DLQ. Ghi rõ ở Lab 03.                                                                                     |
| Dead-letter queue (DLQ)            | Nếu IoT gửi payload sai định dạng JSON, Core Business sẽ không làm crash app mà đẩy thẳng message đó sang DLQ để log lại. Ghi rõ ở Lab 03.                                                     |

## 6. Issue chuyển sang Lab 03

1. **Xử lý Out-of-order events (Sự kiện đến sai thứ tự):** Nếu event lúc 08:01 đến queue sau event lúc 08:02 do rớt mạng cục bộ tại cảm biến, Core Business cần có cơ chế (như so sánh `occurredAt`) để không bị ghi đè trạng thái nghiệp vụ cũ lên trạng thái mới.
2. **Tối ưu băng thông bằng Batching:** Với hàng ngàn sensor gửi dữ liệu mỗi giây, việc đẩy từng message lẻ có thể làm quá tải Message Broker. Hai nhóm cần đàm phán việc IoT Ingestion gom batch (ví dụ: array chứa 50 readings/message) ở Lab 03.
3. **Đặc tả Schema Validation:** Cần viết AsyncAPI schema để quy định chặt chẽ kiểu dữ liệu (ví dụ: `value` bắt buộc là `float`, `sensorType` giới hạn trong `enum: ["temperature", "humidity", "motion"]`).
