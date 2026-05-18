# Biên bản đàm phán hợp đồng API

- Provider: IoT Ingestion (B1)
- Phiên: v1.0
- Ngày: 2026-05-18

> Mỗi cặp đàm phán độc lập. Pair 05 đàm phán với Core Business (B6); Pair 06 đàm phán với Analytics (B5).

---

# PHẦN A — Pair 05 (IoT Ingestion → Core Business)

- Consumer: Core Business (B6)
- Trạng thái: **Đã chốt**

## Issue #1 — Tên event không thống nhất

- Raised by: Provider (IoT Ingestion)
- Event: `sensor.reading.created`
- Concern: Consumer có thể quen với tên event kiểu `sensor.reading.new`. Nếu đặt tên khác, Core Business sẽ không nhận được event.
- Proposal: Thống nhất format `sensor.<noun>.<verb>` với verb ở past participle. Version event bằng semver suffix (`.v1`).
- Resolution: **Accepted**
- Rationale: Quy tắc đặt tên nhất quán giúp developer dễ đoán event name, giảm lỗi subscribe sai topic.
- Impact: Core Business subscribe đúng topic `sensor.reading.created`.

## Issue #2 — Đơn vị sensor (unit) không đồng nhất

- Raised by: Provider (IoT Ingestion)
- Event: `sensor.reading.created`
- Concern: Device IoT có thể gửi đơn vị khác nhau (°C vs °F). Core Business đánh giá threshold sai nếu không normalize.
- Proposal: IoT Ingestion normalize về đơn vị SI trước khi publish. Enum hợp lệ: `°C`, `Pa`, `lux`, `ppm`, `μg/m³`, `%`. Không hợp lệ → DLQ.
- Resolution: **Accepted**
- Rationale: Normalize tại IoT Ingestion đảm bảo tính nhất quán. Core Business không phải tự convert.
- Impact: Payload schema ghi rõ enum unit. Core Business nhận giá trị đã normalize.

## Issue #3 — Thiếu idempotency key

- Raised by: Provider (IoT Ingestion)
- Event: Tất cả event
- Concern: Retry do broker/network lỗi → Core Business nhận event trùng, alert sai.
- Proposal: Mỗi event có `eventId` (UUID v4) do IoT Ingestion sinh. Core Business dùng `eventId` deduplicate trong 24 giờ.
- Resolution: **Accepted**
- Rationale: `eventId` là cách phổ biến nhất xử lý duplicate.
- Impact: Core Business thêm logic deduplicate theo `eventId`.

## Issue #4 — Thiếu correlationId cho trace

- Raised by: Provider (IoT Ingestion)
- Event: `sensor.threshold.exceeded`
- Concern: Core Business nhận `threshold.exceeded` nhưng không trace về event `sensor.reading.created` gốc.
- Proposal: Mỗi event có `correlationId`. Luồng `sensor.reading.created` → `sensor.threshold.exceeded` kế thừa cùng `correlationId`. Nếu không có → IoT Ingestion tự sinh UUID.
- Resolution: **Accepted**
- Rationale: Correlation ID giúp trace toàn bộ luồng xử lý từ device → IoT → Core Business.
- Impact: IoT Ingestion truyền `correlationId` giữa các event trong luồng.

## Issue #5 — Retention và DLQ policy

- Raised by: Provider (IoT Ingestion)
- Event: Tất cả event
- Concern: Payload lỗi → drop im lặng (mất dữ liệu) hoặc retry vô hạn (infinite loop).
- Proposal: Phân loại lỗi: (1) parse JSON → DLQ với `errorType: "parse_error"`; (2) thiếu required field → `errorType: "missing_field"`; (3) giá trị không hợp lệ → `errorType: "invalid_value"`. Retry tối đa 3 lần (1s, 2s, 4s). Retention DLQ: 7 ngày.
- Resolution: **Accepted**
- Rationale: DLQ tách biệt lỗi khỏi luồng chính, giúp DevOps investigate mà không block hệ thống.
- Impact: IoT Ingestion cấu hình DLQ `campus.iot.dlq` và retry policy.

### Pair 05 — Chốt hợp đồng v1.0

Provider sign-off (B1 — IoT Ingestion): **Nguyễn Phúc**  
Consumer sign-off (B6 — Core Business): **Mạnh Cường**  
Witness:    
Date: 2026-05-18

---

# PHẦN B — Pair 06 (IoT Ingestion → Analytics)

- Consumer: Analytics (B5)
- Trạng thái: **Đã chốt**

## Issue #1 — Tên event không thống nhất

- Raised by: Provider (IoT Ingestion)
- Event: `telemetry.ingested`
- Concern: Analytics có thể quen với tên event kiểu `sensor.telemetry.new`. Nếu đặt tên khác, Analytics sẽ không nhận được event.
- Proposal: Thống nhất format `sensor.<noun>.<verb>` với past participle. Semver suffix (`.v1`).
- Resolution: **Accepted**
- Rationale: Quy tắc đặt tên nhất quán giúp developer dễ đoán event name.
- Impact: Analytics subscribe đúng topic `telemetry.ingested`.

## Issue #2 — Đơn vị sensor (unit) không đồng nhất

- Raised by: Provider (IoT Ingestion)
- Event: `telemetry.ingested`
- Concern: Analytics aggregate sai giá trị nếu device gửi đơn vị khác nhau.
- Proposal: IoT Ingestion normalize về SI. Enum hợp lệ: `°C`, `Pa`, `lux`, `ppm`, `μg/m³`, `%`. Không hợp lệ → DLQ.
- Resolution: **Accepted**
- Rationale: Normalize tại IoT Ingestion đảm bảo tính nhất quán.
- Impact: Payload schema ghi rõ enum unit.

## Issue #3 — Thiếu idempotency key

- Raised by: Provider (IoT Ingestion)
- Event: Tất cả event
- Concern: Retry → Analytics aggregate sai vì nhận event trùng.
- Proposal: Mỗi event có `eventId` (UUID v4). Analytics dùng `eventId` deduplicate trong 24 giờ.
- Resolution: **Accepted**
- Rationale: `eventId` xử lý duplicate phổ biến nhất.
- Impact: Analytics thêm logic deduplicate.

## Issue #4 — Batch event hay event đơn?

- Raised by: Consumer (Analytics)
- Event: `telemetry.ingested`
- Concern: Analytics muốn event đơn để aggregate realtime, nhưng IoT Ingestion có thể gửi batch.
- Proposal: V1 (Lab 02): chỉ event đơn, mỗi event publish riêng. `batchId` thêm vào payload như optional field để chuẩn bị V2.
- Resolution: **Accepted**
- Rationale: Lab 02 giữ đơn giản. `batchId` optional không phá backward compatibility.
- Impact: Analytics chỉ handle event đơn V1.

## Issue #5 — Retention và DLQ policy

- Raised by: Provider (IoT Ingestion)
- Event: Tất cả event
- Concern: Payload lỗi → drop im lặng hoặc retry vô hạn.
- Proposal: Phân loại lỗi: parse error / missing field / invalid value → DLQ `campus.iot.dlq`. Retry 3 lần (1s, 2s, 4s). Retention 7 ngày.
- Resolution: **Accepted**
- Rationale: DLQ tách biệt lỗi khỏi luồng chính.
- Impact: IoT Ingestion cấu hình DLQ và retry.

## Issue #6 — Event out-of-order

- Raised by: Consumer (Analytics)
- Event: `telemetry.ingested`
- Concern: Event có thể đến không đúng thứ tự timestamp do network latency. Analytics aggregate sai nếu không sort.
- Proposal: Mỗi event có `timestamp` (ISO 8601, UTC). Analytics sort theo `timestamp` trước khi aggregate. IoT Ingestion ghi timestamp tại thời điểm device gửi.
- Resolution: **Accepted**
- Rationale: `timestamp` từ device đảm bảo thứ tự thực tế.
- Impact: Analytics cần logic sort theo `timestamp`.

### Pair 06 — Chốt hợp đồng v1.0

Provider sign-off (B1 — IoT Ingestion): **Nguyễn Phúc**  
Consumer sign-off (B5 — Analytics): **Lương Hương**  
Witness:    
Date: 2026-05-18
