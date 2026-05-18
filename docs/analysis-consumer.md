# Phân tích yêu cầu — vai Consumer

- Cặp đàm phán:
- Product: A / B
- Consumer service:
- Provider service:
- Người viết:
- Ngày:

---

## 1. Resource Consumer cần nhận/gửi

| Resource       | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
| -------------- | ------------------------ | --------------------------- | --------------------- |
| `<Resource 1>` |                          |                             |                       |
| `<Resource 2>` |                          |                             |                       |

---

## 2. API Consumer cần gọi

| Method | Path        | Lúc nào gọi? | Kỳ vọng response |
| ------ | ----------- | ------------ | ---------------- |
| POST   | `/...`      |              |                  |
| GET    | `/.../{id}` |              |                  |

---

## 3. Error case Consumer cần xử lý

Tối thiểu 5 case.

| Status | Consumer hiểu là gì?    | Consumer sẽ xử lý thế nào?        |
| -----: | ----------------------- | --------------------------------- |
|    400 | Request sai schema      | Sửa payload/log lỗi               |
|    401 | Thiếu token             | Refresh/cấu hình token            |
|    403 | Không đủ quyền          | Báo lỗi quyền truy cập            |
|    404 | Không tìm thấy resource | Hiển thị trạng thái không tồn tại |
|    409 | Xung đột nghiệp vụ      | Retry hoặc yêu cầu thao tác lại   |
|    422 | Vi phạm rule nghiệp vụ  | Hiển thị lý do cụ thể             |

---

## 4. Giả định bổ sung

- Giả định 1:
- Giả định 2:
- Giả định 3:

---

## 5. Câu hỏi cho Provider

1.
2.
3.

---

## 6. Rủi ro tích hợp

| Rủi ro                    | Tác động               | Đề xuất xử lý             |
| ------------------------- | ---------------------- | ------------------------- |
| Provider đổi kiểu dữ liệu | Consumer parse lỗi     | Chốt type/format/pattern  |
| Provider thiếu mã lỗi     | Consumer khó xử lý lỗi | Chuẩn hóa Problem Details |
