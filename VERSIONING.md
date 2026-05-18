# Versioning Policy

- Provider: IoT Ingestion (B1)
- Phiên bản hợp đồng hiện tại: v1.0
- Ngày: 2026-05-18

---

## 1. Quy tắc đặt tên topic

Topic format: `<domain>.<service>.<resource>.<verb>`

| Topic | Semver | Mô tả |
|---|---|---|
| `campus.iot.sensor.reading.created` | v1 | Event sensor reading mới |
| `campus.iot.telemetry.ingested` | v1 | Telemetry đã nhận |
| `campus.iot.dlq` | — | Dead Letter Queue (không semver) |

Khi thay đổi payload schema không tương thích ngược → tăng major version (`.v2`), giữ topic cũ song song 1 release cycle.

---

## 2. Quy tắc thay đổi payload

### Thay đổi tương thích ngược (không tăng version)
- Thêm field optional mới
- Thêm giá trị enum mới cho field optional
- Thêm field mới vào discriminator

### Thay đổi breaking (tăng major version)
- Xóa hoặc đổi type field bắt buộc
- Đổi tên field bắt buộc
- Xóa giá trị enum đang dùng
- Đổi đơn vị normalize (SI)

---

## 3. Chính sách retry

| Thông số | Giá trị |
|---|---|
| Số lần retry | 3 |
| Exponential backoff | 1s → 2s → 4s |
| Backoff multiplier | 2x |

---

## 4. Dead Letter Queue

| Thông số | Giá trị |
|---|---|
| Topic DLQ | `campus.iot.dlq` |
| Retention | 7 ngày |
| Error type | `parse_error`, `missing_field`, `invalid_value` |

---

## 5. Deduplication

Consumer deduplicate theo `eventId` trong vòng **24 giờ** kể từ khi nhận event.

---

## 6. Consumer đang subscribe

| Consumer | Topic |
|---|---|
| Core Business (B6) | `campus.iot.sensor.reading.created` |
| Analytics (B5) | `campus.iot.telemetry.ingested` |

---

## 7. Lịch sử thay đổi

| Version | Ngày | Thay đổi |
|---|---|---|
| v1.0 | 2026-05-18 | Phiên bản đầu tiên, chốt contract với B5 và B6 |
