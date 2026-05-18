# Biên bản đàm phán hợp đồng API

- Cặp đàm phán: 5 (IoT Ingestion → Core Business)
- Product: Smart Campus Operations Platform
- Provider: IoT Ingestion (Nhóm A1/B1)
- Consumer: Core Business (Nhóm A6/B6)
- Phiên: v1.0
- Ngày: 2026-05-18

---

## Issue #1

- Raised by: Consumer (Core Business)
- Endpoint: Topic `sensor.reading.created`
- Concern: Core Business cần biết đơn vị đo lường (độ C, % độ ẩm) để so sánh với các ngưỡng (threshold) cấu hình trong database, tránh việc so sánh sai logic.
- Proposal: Phía Consumer đề xuất Provider gửi kèm trường `unit` trong payload, thay vì Consumer phải tự hardcode ánh xạ theo `sensorType`.
- Resolution: Accepted
- Rationale: Việc Provider gửi kèm `unit` giúp decouple logic, Consumer không cần biết quá nhiều về phần cứng thiết bị. Dữ liệu mang tính self-explanatory (tự giải thích) cao hơn.
- Impact: Cập nhật payload tối thiểu, bổ sung trường `unit` (ví dụ: `"unit": "Celsius"`).

---

## Issue #2

- Raised by: Consumer (Core Business)
- Endpoint: Topic `sensor.reading.created`
- Concern: Khi có cảnh báo (ví dụ cháy), Core Business cần biết vị trí cụ thể để kích hoạt chuông báo động ở khu vực đó, nhưng hiện tại payload thô chỉ có `deviceId`.
- Proposal: Yêu cầu IoT Ingestion gửi thêm `zoneId` (Khu vực) vào event data.
- Resolution: Accepted
- Rationale: Mặc dù Core Business có thể query database để tìm `zoneId` từ `deviceId`, nhưng việc này gây overhead (tốn tài nguyên query DB liên tục với hàng ngàn event/giây). Việc IoT Ingestion đính kèm sẵn `zoneId` từ lúc ingest dữ liệu giúp tăng tốc độ xử lý realtime.
- Impact: Payload bổ sung trường `"zoneId": "zone-building-A-floor-3"`.

---

## Issue #3

- Raised by: Provider (IoT Ingestion)
- Endpoint: Topic `sensor.reading.created`
- Concern: Do tính chất mạng không ổn định từ các gateway IoT, Provider có thể retry gửi lại một event nhiều lần (At-least-once delivery), dẫn đến việc queue nhận được nhiều message trùng lặp.
- Proposal: Consumer phải tự xử lý luỹ đẳng (Idempotency) để không trigger báo động 2 lần cho cùng 1 sự kiện.
- Resolution: Accepted
- Rationale: Đảm bảo tính nhất quán dữ liệu là trách nhiệm của Consumer khi đọc từ Queue.
- Impact: Provider cam kết luôn gửi kèm `eventId` (UUID v4) độc nhất cho mỗi sự kiện. Consumer sử dụng `eventId` này làm Idempotency Key (lưu Redis cache 5 phút) để check trùng.

---

## Issue #4

- Raised by: Consumer (Core Business)
- Endpoint: Topic `sensor.reading.created`
- Concern: Nếu hệ thống queue bị nghẽn (backpressure), Core Business có thể đọc được dữ liệu nhiệt độ đã diễn ra từ 10 phút trước (Stale data), việc phát cảnh báo lúc này không còn ý nghĩa và gây nhiễu.
- Proposal: Consumer sẽ chủ động drop (bỏ qua) các event có độ trễ quá lớn.
- Resolution: Accepted
- Rationale: Cảnh báo Smart Campus yêu cầu tính thời gian thực (Real-time). Dữ liệu cũ chỉ có tác dụng thống kê (Analytics) chứ không dùng để chạy Policy nghiệp vụ.
- Impact: Thống nhất logic: `CurrentTime - occurredAt > 2 phút` -> Consumer tự động drop message.

---

## Issue #5

- Raised by: Provider (IoT Ingestion)
- Endpoint: Topic `sensor.reading.created`
- Concern: Nếu format JSON từ IoT bị lỗi (do firmware thiết bị update sai) dẫn đến thiếu các field bắt buộc, việc Consumer parse lỗi có thể làm crash service Core Business.
- Proposal: Consumer phải bắt Exception (try-catch) khi parse JSON và đẩy message lỗi sang một hàng đợi đặc biệt (Dead-letter Queue - DLQ).
- Resolution: Accepted
- Rationale: Đảm bảo tính Resiliency (khả năng phục hồi) cho Consumer. Broker không bị nghẽn bởi các "poison pill" (tin nhắn độc hại).
- Impact: Chốt cơ chế DLQ sơ bộ. Chi tiết cấu hình DLQ sẽ được đặc tả rõ bằng AsyncAPI ở Lab 03.

---

## Issue #6

- Raised by: Provider (IoT Ingestion)
- Endpoint: Topic `sensor.reading.created`
- Concern: Tần suất gửi 1 event/giây từ hàng ngàn thiết bị sẽ tạo ra hàng trăm ngàn message, có nguy cơ làm sập Message Broker (RabbitMQ/Kafka).
- Proposal: IoT Ingestion đề xuất gom cụm (Batching) các event lại, cứ 10 giây sẽ gửi 1 mảng (array) chứa nhiều reading thay vì gửi lẻ tẻ.
- Resolution: Rejected (Cho Lab 02) -> Chuyển thành Issue Lab 03
- Rationale: Việc xử lý Batching làm tăng độ phức tạp cho Consumer trong Lab 02 (phải bóc tách mảng). Hai bên thống nhất Lab 02 giữ luồng single-event để pass kiểm thử cơ bản.
- Impact: Giữ nguyên cấu trúc JSON Object đơn. Đưa bài toán Batching vào danh sách "Issue chuyển sang Lab 03" để tiếp tục đàm phán.

---

# Chốt hợp đồng v1.0

Provider sign-off: Nguyễn Xuân Phúc (Đại diện IoT Ingestion)
Consumer sign-off: ... (Đại diện Core Business)
Witness (GV/TA): [Để trống cho Giảng Viên ký]
Date: 2026-05-18

---

## Ghi chú warning nếu Spectral còn cảnh báo

_(Ghi chú: Vì cặp dependency này sử dụng Queue async, Lab 02 chưa yêu cầu viết OpenAPI 3.1 nên không chạy tool kiểm tra Spectral. Bảng dưới đây dùng để ghi nhận các cảnh báo liên quan đến validation schema chuẩn bị cho AsyncAPI ở Lab 03)._

| Warning                                  | Lý do chấp nhận tạm thời                                                  | Kế hoạch sửa                                                                    |
| ---------------------------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Thiếu chuẩn hóa kiểu dữ liệu cho `value` | Lab 02 thống nhất bằng văn bản tạm thời. Chưa dùng tool chặn strict type. | Lab 03 sẽ viết AsyncAPI mô tả rõ `value` là `number (float)` thay vì `string`.  |
| Thiếu giới hạn enum cho `sensorType`     | Hiện tại chấp nhận mọi chuỗi string do IoT gửi lên.                       | Lab 03 sẽ viết AsyncAPI giới hạn enum: `["temperature", "humidity", "motion"]`. |
