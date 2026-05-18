# Phân tích yêu cầu — vai Provider

- Cặp đàm phán:
- Product: A / B
- Provider service:
- Consumer service:
- Người viết:
- Ngày:

---

## 1. Resource chính

| Resource       | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
| -------------- | ----- | ------------------- | ------------------- |
| `<Resource 1>` |       |                     |                     |
| `<Resource 2>` |       |                     |                     |

---

## 2. Action/API dự kiến

| Method | Path        | Mục đích | Consumer gọi khi nào? |
| ------ | ----------- | -------- | --------------------- |
| POST   | `/...`      |          |                       |
| GET    | `/.../{id}` |          |                       |

---

## 3. Error case

Tối thiểu 5 case.

| Status | Tình huống                                | Response body dự kiến |
| -----: | ----------------------------------------- | --------------------- |
|    400 | Payload sai định dạng                     | `Problem`             |
|    401 | Thiếu Bearer token                        | `Problem`             |
|    403 | Token hợp lệ nhưng không có quyền         | `Problem`             |
|    404 | Resource không tồn tại                    | `Problem`             |
|    409 | Xung đột nghiệp vụ                        | `Problem`             |
|    422 | Dữ liệu đúng JSON nhưng vi phạm nghiệp vụ | `Problem`             |

---

## 4. Giả định bổ sung

Ghi rõ những điểm user story chưa nói nhưng Provider cần giả định.

- Giả định 1:
- Giả định 2:
- Giả định 3:

---

## 5. Câu hỏi cho Consumer

1.
2.
3.

---

## 6. Rủi ro tích hợp

| Rủi ro                     | Tác động           | Đề xuất xử lý                         |
| -------------------------- | ------------------ | ------------------------------------- |
| Tên field không thống nhất | Consumer parse lỗi | Chốt naming trong `openapi.yaml`      |
| Payload lớn                | Timeout/mock lỗi   | Thống nhất content-type và size limit |
