# Phân tích yêu cầu — vai Provider (IoT Ingestion)

- **Cặp đàm phán:** 05 & 06 (Mở rộng luồng quản lý thiết bị)
- **Product:** Smart Campus Operations Platform
- **Provider service:** IoT Ingestion Service (A1/B1)
- **Consumer service:** Core Business (A6/B6) & Analytics (A5/B5)
- **Người viết:** Nhóm 8
- **Ngày:** 2026-05-18

---

## 1. Resource chính

| Resource      | Mô tả                                                       | Thuộc tính bắt buộc                          | Thuộc tính tùy chọn                           |
| :------------ | :---------------------------------------------------------- | :------------------------------------------- | :-------------------------------------------- |
| **Sensor**    | Đại diện cho các thiết bị vật lý (nhiệt độ, độ ẩm, cổng...) | `sensorId`, `sensorType`, `zoneId`, `status` | `firmwareVersion`, `lastMaintenance`, `model` |
| **Telemetry** | Dữ liệu đo lường tức thời từ cảm biến gửi về.               | `sensorId`, `value`, `unit`, `timestamp`     | `batteryLevel`, `signalStrength`              |

---

## 2. Action/API dự kiến

Bên cạnh luồng Async chính, Provider cung cấp các Endpoint REST để Consumer quản lý cấu hình:

| Method    | Path                   | Mục đích                                             | Consumer gọi khi nào?                                   |
| :-------- | :--------------------- | :--------------------------------------------------- | :------------------------------------------------------ |
| **POST**  | `/sensors`             | Đăng ký thiết bị IoT mới vào hệ thống.               | Khi Admin thêm cảm biến mới vào một khu vực.            |
| **GET**   | `/sensors/{id}/status` | Lấy giá trị mới nhất (last-known) của cảm biến.      | Khi Core Business cần kiểm tra trạng thái ngay lập tức. |
| **PATCH** | `/sensors/{id}`        | Cập nhật thông tin hoặc tạm dừng nhận tin từ sensor. | Khi thiết bị cần bảo trì hoặc thay đổi zone.            |

---

## 3. Error case

Hệ thống xử lý các tình huống lỗi để đảm bảo tính sẵn sàng cao (High Availability).

| Status  | Tình huống                                                 | Response body dự kiến                                                             |
| :------ | :--------------------------------------------------------- | :-------------------------------------------------------------------------------- |
| **400** | Payload đăng ký thiếu trường bắt buộc (`zoneId`).          | `{ "error": "Invalid_Request", "message": "Missing required field: zoneId" }`     |
| **401** | Gọi API thiếu API Key hoặc Token không hợp lệ.             | `{ "error": "Unauthorized", "message": "Access denied" }`                         |
| **404** | Không tìm thấy cảm biến với ID yêu cầu.                    | `{ "error": "Not_Found", "message": "Sensor ID does not exist" }`                 |
| **409** | Trùng lặp `sensorId` đã tồn tại trên hệ thống.             | `{ "error": "Conflict", "message": "Duplicate sensor ID" }`                       |
| **422** | Dữ liệu JSON đúng nhưng đơn vị (`unit`) không được hỗ trợ. | `{ "error": "Unprocessable_Content", "message": "Unsupported measurement unit" }` |

---

## 4. Giả định bổ sung

- **Giả định 1:** Mọi thiết bị IoT đều được gán cứng một `sensorId` duy nhất tại xưởng sản xuất (hoặc khi onboard).
- **Giả định 2:** Dữ liệu thời gian sử dụng chuẩn ISO 8601 (UTC+7) để đồng bộ giữa các service.
- **Giả định 3:** Service IoT Ingestion có bộ đệm Redis để lưu trạng thái cuối của sensor, giảm tải cho Database chính khi Consumer truy vấn GET.

---

## 5. Câu hỏi cho Consumer

1. **Về phía Core Business:** Các bạn có cần chúng tôi lọc (filter) bớt dữ liệu nhiễu trước khi đẩy vào Queue không, hay muốn nhận toàn bộ raw data?
2. **Về phía Analytics:** Tần suất lấy dữ liệu trạng thái qua API (Sync) là bao nhiêu để chúng tôi thiết lập Rate Limit phù hợp?
3. **Chung:** Nếu hệ thống Queue (RabbitMQ/Kafka) bảo trì, các bạn có chấp nhận mất dữ liệu trong thời gian đó hay chúng tôi phải lưu tạm vào DB?

---

## 6. Rủi ro tích hợp

| Rủi ro                 | Tác động                                              | Đề xuất xử lý                                                                                |
| :--------------------- | :---------------------------------------------------- | :------------------------------------------------------------------------------------------- |
| **Nghẽn Queue**        | Dữ liệu đến Consumer bị trễ (lag), mất tính realtime. | Thiết lập cảnh báo ngưỡng (Threshold) và xem xét cơ chế Batching ở Lab 03.                   |
| **Sai lệch định dạng** | Consumer không parse được dữ liệu đo lường.           | Chốt chặt chẽ Schema trong file `event-contract` và dùng Spectral để kiểm tra.               |
| **Thiết bị gửi lỗi**   | Cảm biến hỏng gửi giá trị ảo (ví dụ nhiệt độ 1000°C). | Provider bổ sung lớp Validation đơn giản để chặn các giá trị phi lý trước khi đẩy vào Queue. |
