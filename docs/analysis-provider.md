# Phân tích yêu cầu — vai Provider

- **Cặp đàm phán:** Pair 05 (IoT Ingestion → Core Business) & Pair 06 (IoT Ingestion → Analytics)
- **Product:** Smart Campus Operations Platform
- **Provider service:** IoT Ingestion (A1/B1)
- **Consumer service:** Core Business (A6/B6) và Analytics (A5/B5)
- **Người viết:** Hoàng Anh Minh
- **Ngày:** 18/05/2026

---

## 1. Resource chính

_Vì đây là giao tiếp Async, "Resource" ở đây được hiểu là các cấu trúc Message/Event Payload gửi vào Queue/Topic._

| Resource (Event Payload) | Mô tả                                                               | Thuộc tính bắt buộc                                                            | Thuộc tính tùy chọn                  |
| ------------------------ | ------------------------------------------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------ |
| `SensorReadingEvent`     | Payload khi sensor gửi dữ liệu đo đạc định kỳ (nhiệt độ, độ ẩm...). | `eventId`, `deviceId`, `sensorType`, `value`, `timestamp`                      | `unit`, `locationId`, `batteryLevel` |
| `ThresholdExceededEvent` | Payload cảnh báo nóng khi chỉ số vượt ngưỡng an toàn.               | `eventId`, `deviceId`, `sensorType`, `threshold`, `exceededValue`, `timestamp` | `locationId`                         |
| `DeviceStatusEvent`      | Payload cập nhật thiết bị online/offline.                           | `eventId`, `deviceId`, `status` (online/offline), `timestamp`                  | `lastPingTime`                       |

---

## 2. Action/API dự kiến

_Thay vì HTTP Method (GET/POST), bảng dưới đây ánh xạ sang hành vi Publish/Subscribe qua các Message Broker (RabbitMQ/Kafka)._

| Hành động (Method) | Topic/Routing Key (Path)        | Mục đích                     | Consumer lắng nghe/xử lý khi nào?                                 |
| ------------------ | ------------------------------- | ---------------------------- | ----------------------------------------------------------------- |
| `PUBLISH`          | `iot.sensor.reading.created`    | Feed telemetry định kỳ.      | Analytics consume liên tục để aggregate data; Core kiểm tra rule. |
| `PUBLISH`          | `iot.sensor.threshold.exceeded` | Kích hoạt cảnh báo tức thời. | Core Business consume ngay lập tức để trigger Notification.       |
| `PUBLISH`          | `iot.device.status.changed`     | Báo cáo trạng thái thiết bị. | Analytics lưu log; Core Business theo dõi health-check.           |

---

## 3. Error case (Sự cố xử lý Message & NACK)

_Consumer khi nhận message có thể gặp lỗi. Bảng này quy định cách map các lỗi logic nghiệp vụ tương đương với HTTP Status._

| Status | Tình huống (Kafka/RabbitMQ Consumer)                              | Cách xử lý / Trạng thái phản hồi dự kiến                                      |
| -----: | ----------------------------------------------------------------- | ----------------------------------------------------------------------------- |
|    400 | Payload nhận được sai cấu trúc (thiếu `deviceId` hoặc `value`).   | Ghi log cảnh báo, Reject message và đẩy vào Dead-Letter Queue (DLQ).          |
|    401 | Consumer (Client) rớt kết nối hoặc mất quyền truy cập vào Topic.  | Broker từ chối kết nối (`Authentication Failed`).                             |
|    404 | Tham chiếu `deviceId` không tồn tại trong CSDL của Core Business. | Drop message (bỏ qua), ghi log cảnh báo dữ liệu thiết bị rác (Orphaned Data). |
|    408 | Hệ thống Consumer quá tải, xử lý message quá thời gian (Timeout). | NACK (Negative Acknowledgement) để Broker tự động Retry lại sau.              |
|    422 | Payload chuẩn JSON nhưng `value` phi logic (vd: nhiệt độ -500°C). | Reject message, không Retry, đẩy vào DLQ để kỹ sư kiểm tra cảm biến.          |
|    429 | Lượng event đổ về quá lớn cùng lúc (Spike Traffic).               | Bật cơ chế Rate Limiting, Consumer chủ động pre-fetch số lượng nhỏ giọt.      |

---

## 4. Giả định bổ sung

_Những điểm user story chưa nói nhưng IoT Ingestion bắt buộc phải chốt hạ để bảo vệ hệ thống._

- **Giả định 1 (Cơ chế phân phối):** Hệ thống đảm bảo phân phối ít nhất 1 lần (`At-least-once delivery`). Do đó, có thể xảy ra tình trạng trùng lặp message. **Consumer bắt buộc phải tự implement cơ chế Idempotency** (dựa vào `eventId`) để tránh cộng dồn sai số liệu.
- **Giả định 2 (Đơn vị đo lường):** Nếu trong payload bị khuyết trường `unit`, Core Business và Analytics phải tự động ngầm hiểu theo chuẩn SI mặc định do IoT quy định (ví dụ: nhiệt độ là °C, độ ẩm là %).
- **Giả định 3 (Tính nhất thời):** Dịch vụ IoT Ingestion chỉ đóng vai trò Gateway tiếp nhận và đẩy vào Queue, không lưu trữ (persist) dữ liệu lịch sử lâu dài để tối ưu độ trễ. Việc lưu trữ thuộc trách nhiệm của Analytics.

---

## 5. Câu hỏi dành cho Consumer

1. **Dành cho Core Business:** Core sẽ chấp nhận độ trễ (TTL - Time To Live) của event tối đa bao lâu? (VD: Sự kiện nhiệt độ cao quá 5 phút mới tới nơi thì xử lý tiếp hay bỏ qua?).
2. **Dành cho Analytics:** Các bạn cần gửi gộp (Batch - VD 100 event/lần) để tối ưu ghi DB, hay muốn stream từng event một theo thời gian thực? Khóa aggregate chính là `deviceId` hay `locationId`?
3. **Câu hỏi chung:** Nếu định dạng dữ liệu lỗi không thể parse, Consumer sẽ drop luôn message hay muốn IoT Ingestion thiết lập sẵn một Dead-Letter Queue (DLQ) để lưu trữ báo cáo?

---

## 6. Rủi ro tích hợp

| Rủi ro                                       | Tác động                                                         | Đề xuất xử lý                                                                                 |
| -------------------------------------------- | ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Bất đồng định dạng thời gian (`timestamp`).  | Lỗi query dữ liệu, Analytics tổng hợp sai giờ.                   | Thống nhất 100% dùng chuẩn `ISO 8601 UTC` (vd: `2026-05-18T10:22:48Z`).                       |
| Trùng lặp event do mạng lag và Broker Retry. | Gửi thông báo rác hoặc tính toán KPI (Analytics) bị đúp số liệu. | Consumer check `eventId` trong cache Redis trước khi xử lý, nếu có rồi thì bỏ qua.            |
| Mất mát dữ liệu quan trọng khi hệ thống sập. | Bỏ lỡ sự kiện cháy/báo động (`threshold.exceeded`).              | Cấu hình Message Persistence (lưu ổ cứng) cho các topic quan trọng, thay vì chỉ lưu trên RAM. |
