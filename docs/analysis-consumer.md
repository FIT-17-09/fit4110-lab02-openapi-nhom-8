# Phân tích yêu cầu — vai Consumer

- **Cặp đàm phán:** Pair 05 (IoT Ingestion → Core Business)
- **Product:** Smart Campus Operations Platform
- **Consumer service:** Core Business (A6/B6)
- **Provider service:** IoT Ingestion (A1/B1)
- **Người viết:** Nhóm 8
- **Ngày:** 18/05/2026

---

## 1. Resource Consumer cần nhận/gửi

| Resource              | Consumer dùng để làm gì?                                                                                                                                                                  | Field bắt buộc với Consumer                                                         | Field có thể tùy chọn                |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------ |
| `SensorReadingEvent`  | Nhận dữ liệu telemetry định kỳ từ các cảm biến (nhiệt độ, độ ẩm, chất lượng không khí) để đưa vào các Rule Engine đánh giá logic (ví dụ: kích hoạt điều hòa, thông gió).                  | `eventId`, `deviceId`, `sensorType`, `value`, `timestamp`                           | `unit`, `batteryLevel`, `locationId` |
| `ThresholdAlertEvent` | Nhận tín hiệu khẩn cấp khi cảm biến phát hiện chỉ số vượt ngưỡng an toàn (ví dụ: nhiệt độ > 60°C). Core Business sẽ dùng thông tin này để lập tức gọi Notification Service phát báo cháy. | `eventId`, `deviceId`, `sensorType`, `exceededValue`, `thresholdLimit`, `timestamp` | `urgencyLevel`                       |
| `DeviceHealthStatus`  | Nhận trạng thái online/offline/lỗi phần cứng của thiết bị để cập nhật lên Dashboard quản trị.                                                                                             | `deviceId`, `status`, `lastPing`                                                    | `errorCode`, `firmwareVersion`       |

---

## 2. API / Event Consumer cần gọi (Subscribe)

_Lưu ý: Đối với cơ chế Queue Async, phần "Method/Path" tương đương với hành động Subscribe lắng nghe trên các Topic/Routing Key cụ thể của Message Broker._

| Method      | Path (Topic/Queue)            | Lúc nào gọi / Lắng nghe?                                                                                             | Kỳ vọng response (Payload)                                                                     |
| ----------- | ----------------------------- | -------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `SUBSCRIBE` | `iot.sensor.telemetry.v1`     | Lắng nghe liên tục nền (Background Worker) 24/7.                                                                     | Payload chuẩn JSON chứa mảng hoặc đối tượng `SensorReadingEvent`. Trễ mạng (Latency) < 2 giây. |
| `SUBSCRIBE` | `iot.sensor.alert.v1`         | Lắng nghe ưu tiên cao (High Priority Queue).                                                                         | Nhận event khẩn cấp ngay lập tức (< 500ms), bắt buộc phải có thông tin `exceededValue`.        |
| `GET`       | `/api/v1/devices/{id}/status` | Gọi đồng bộ khi Core Business cần kiểm tra chéo (Double-check) trạng thái cảm biến trước khi kích hoạt báo động giả. | JSON chứa `DeviceHealthStatus`.                                                                |

---

## 3. Error case Consumer cần xử lý

_Bảng đối chiếu cách Core Business (Consumer) xử lý khi gặp các sự cố dữ liệu (Map theo HTTP Status Codes để dễ đàm phán)._

|  Status | Consumer hiểu là gì?                                                                                                    | Consumer sẽ xử lý thế nào?                                                                                                               |
| ------: | ----------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **400** | **Request/Event sai schema:** Payload từ IoT bị thiếu các trường bắt buộc (như mất `deviceId` hoặc sai định dạng JSON). | Bỏ qua (DROP) thông điệp đó, ghi log lỗi định dạng. Không tự động Retry để tránh kẹt Queue.                                              |
| **401** | **Thiếu token / Xác thực Broker thất bại:** Mất kết nối đến RabbitMQ/Kafka.                                             | Trigger cảnh báo nội bộ cho đội DevOps, tự động thử cấu hình lại Connection (Auto-reconnect).                                            |
| **404** | **Không tìm thấy resource:** Nhận được dữ liệu nhưng `deviceId` chưa từng được đăng ký trong CSDL của Core.             | Xác định đây là "thiết bị lạ/rác". Đẩy message vào một bảng tạm (Quarantine) và không chạy logic báo động.                               |
| **408** | **Timeout xử lý:** Core Business quá tải, không kịp chạy Rule Engine cho hàng ngàn event đổ về cùng lúc.                | Bắn tín hiệu NACK (Negative Acknowledge) để đẩy event về hàng đợi, tự động xử lý lại khi có tài nguyên.                                  |
| **409** | **Xung đột / Trùng lặp sự kiện:** Cùng một `eventId` được hệ thống IoT bắn sang 2 lần (do rớt mạng chập chờn).          | Áp dụng cơ chế Idempotency: Kiểm tra cache Redis, nếu `eventId` đã tồn tại thì trả về ACK để xóa message, tuyệt đối KHÔNG chạy lại Rule. |
| **422** | **Vi phạm rule nghiệp vụ:** Payload chuẩn JSON nhưng dữ liệu vô lý (VD: Nhiệt độ phòng là -900°C).                      | Không trigger báo động. Chuyển thông điệp này vào Dead-Letter Queue (DLQ) để kỹ sư IoT đi kiểm tra phần cứng.                            |

---

## 4. Giả định bổ sung

- **Giả định 1 (Chuẩn Thời gian):** Toàn bộ trường `timestamp` gửi từ IoT phải được định dạng theo chuẩn **ISO 8601 UTC** (VD: `2026-05-18T10:30:00Z`). Nếu Consumer nhận được event có timestamp quá trễ (cách thời điểm hiện tại hơn 5 phút), event đó sẽ bị hủy (Discard) vì đã lỗi thời.
- **Giả định 2 (Tính hợp lệ của thiết bị):** Consumer mặc định coi các `deviceId` gửi từ Provider đã được Provider xác thực sơ bộ ở tầng Gateway. Consumer sẽ chỉ map ID này với CSDL nội bộ để lấy ngữ cảnh (Location, Zone).
- **Giả định 3 (Đơn vị đo lường ngầm định):** Nếu Provider không gửi trường `unit`, Consumer sẽ tự động ngầm hiểu theo hệ SI chuẩn (Nhiệt độ = Độ C, Độ ẩm = %, Nồng độ khí = PPM).

---

## 5. Câu hỏi cho Provider (Dùng cho buổi đàm phán)

1. Provider có cơ chế gom cụm dữ liệu (Batching - ví dụ gửi 50 reading/lần) hay sẽ đẩy lẻ tẻ từng event một (Streaming)? Nếu là Streaming, làm sao Provider đảm bảo Core Business không bị DDOS khi hàng ngàn cảm biến cùng gửi data?
2. Trong trường hợp Broker hoặc mạng bị sập, hệ thống IoT Ingestion có lưu lại các thông điệp cảnh báo quan trọng (`ThresholdAlertEvent`) để gửi bù lại khi mạng khôi phục không?
3. Cấu trúc trường `value` cho cảm biến là kiểu số thực (`float`) hay chuỗi (`string`)? Liệu có khi nào `value` trả về `null` không (VD: khi cảm biến đang khởi động)?

---

## 6. Rủi ro tích hợp

| Rủi ro                                                                         | Tác động                                                                                                         | Đề xuất xử lý                                                                                                                                   |
| ------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **Provider đổi kiểu dữ liệu (Schema Drift):** Ví dụ đổi `deviceId` thành `id`. | Consumer không parse được JSON, toàn bộ luồng đánh giá Policy và báo động bị tê liệt.                            | Bắt buộc Provider phải tuân thủ nghiêm ngặt chuẩn định nghĩa trong `openapi.yaml` / AsyncAPI. Bất kỳ thay đổi nào phải nâng version (v1 -> v2). |
| **Mất mát các Message khẩn cấp:** Sự kiện cháy/vượt ngưỡng bị kẹt ở Broker.    | Không kịp kích hoạt thông báo, gây hậu quả nghiêm trọng về tài sản.                                              | Thống nhất phân luồng: Tạo một Topic ưu tiên riêng (`high-priority`) độc lập hoàn toàn với Topic gửi telemetry định kỳ.                         |
| **Thiếu thông tin `eventId` duy nhất:** Message không có ID rõ ràng.           | Consumer không thể kiểm tra trùng lặp (Idempotency), dẫn đến trigger cảnh báo cháy nhiều lần cho cùng 1 sự việc. | Provider bắt buộc phải sinh UUID chuẩn cho từng message ở trường `eventId`.                                                                     |
