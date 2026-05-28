# Doc 18 · Event Catalog · v1.0

**Version**: 1.0 · **Date**: 2026-05-28 · **Status**: DRAFT — *đóng blocker cross-service* · **Audience**: Backend, BA, QC, DevOps

> Settlement (Doc 17) và các feature (Doc 16) nối nhau qua event. Doc này là **contract** cho event — outbox là cơ chế vận chuyển, KHÔNG thay contract. Chưa cần Kafka: transport = **transactional outbox + in-process dispatcher** (lên v3.7 Kafka thì outbox là source).

---

## 1. Nguyên tắc (cross-cutting)

| Nguyên tắc | Quy định |
| --- | --- |
| **Envelope chuẩn** | Mọi event có cùng envelope (§2) |
| **Producer idempotency** | Outbox ghi cùng transaction với state change → at-least-once, không mất event |
| **Consumer idempotency** | Consumer dedup theo `event_id` (Redis `processed:{consumer}:{event_id}`, TTL 7d) → **exactly-once xử lý** |
| **Ordering** | Theo `ordering_key` (vd `order_id`) — event cùng key xử lý tuần tự |
| **Versioning** | `event_version` semver; thêm field = minor (backward-compat); đổi/xóa field = **major** (event mới `XxxV2`, chạy song song tới khi consumer migrate) |
| **Breaking-change rule** | KHÔNG đổi nghĩa/kiểu field cũ trong cùng major. Deprecate → mới → cutover |
| **Retry** | Exponential backoff: 1s,5s,30s,2m,10m (max 5) |
| **DLQ** | Quá retry → `dead_letter` table + alert; replay thủ công sau fix |
| **Replay** | Outbox/DLQ replay theo `event_id`; consumer phải idempotent để replay an toàn |

## 2. Envelope schema

```jsonc
{
  "event_id": "uuid",                 // idempotency key toàn cục
  "event_name": "SettlementSettled",
  "event_version": "1.0",
  "aggregate_type": "settlement",
  "aggregate_id": "settlement_123",
  "ordering_key": "order_456",        // xử lý tuần tự theo key
  "occurred_at": "2026-05-28T03:00:00Z",  // UTC
  "producer": "finance-svc",
  "trace_id": "...",                  // distributed tracing
  "payload": { /* per-event, §4 */ }
}
```

## 3. Bảng đăng ký event (registry)

| Event | Producer | Consumers | ordering_key | Mục đích |
| --- | --- | --- | --- | --- |
| `SettlementCreated` | finance | — | order_id | Khởi tạo settlement |
| `PaymentConfirmed` | finance | order-svc, notification | order_id | Set `order.paid` UX flag |
| `SettlementSettled` | finance | loyalty, finance(payout), quality(warranty) | order_id | Mốc tiền thật → trigger downstream |
| `CODCollected` | finance | finance(ledger), notification | order_id | Tạo `agent_cod_receivable` |
| `CODRemitted` | finance | finance(payout), reconcile | order_id | Xóa cod_liability, start payout clock |
| `RefundSettled` | finance | loyalty(clawback), finance(earnings), invoice | order_id | Hoàn tất refund |
| `SettlementFailed` | finance | order-svc, notification | order_id | Nhắc KH thử lại |
| `OrderStageChanged` | order | notification(chat bot), dispatch, finance | order_id | Chat system msg, trigger settlement |
| `OrderCompleted` | order | finance, quality(warranty) | order_id | Tạo settlement + warranty cert |
| `RatingCreated` | quality | agent(profile_mv) | agent_id | Refresh profile read model |
| `ComplaintOpened` | quality | finance(payout hold) | order_id | Hold earnings (BR-CMPL-003) |
| `WalletToppedUp` | finance | notification | customer_id | Thông báo nạp ví |
| `AgentFavorited` | agent | — | customer_id | Analytics |
| `WarrantyIssued` | quality | notification | order_id | Gửi chứng nhận BH |
| `PayoutPaid` | finance | notification, agent(earnings_mv) | agent_id | Cập nhật dashboard thu nhập |
| `SosTriggered` | quality | ops(alert), notification | order_id | Cảnh báo Ops 24/7 |

## 4. Payload contract (core finance events)

### `SettlementSettled` v1.0
```jsonc
{
  "settlement_id": "settlement_123", "order_id": "order_456",
  "method": "gateway|wallet|cod|partner_wallet",
  "amount_minor": 100000, "currency": "VND",
  "agent_id": "agent_7", "customer_id": "cust_9",
  "settled_at": "2026-05-28T03:00:00Z",
  "refund_window_until": "2026-06-04T03:00:00Z",
  "journal_entry_id": "je_555"        // link sang ledger (SoT)
}
```
Consumers: **loyalty** → earn `pending` (BR-LOYAL-001); **finance/payout** → start T+1 clock; **quality** → set warranty `payment_backed=true`.

### `CODCollected` v1.0
```jsonc
{ "settlement_id":"...", "order_id":"...", "agent_id":"...",
  "amount_minor":100000, "currency":"VND",
  "agent_cod_receivable_delta": 100000,   // naming chốt (Doc 16 v1.2)
  "collected_at":"..." }
```

### `RefundSettled` v1.0
```jsonc
{ "refund_id":"...", "order_id":"...", "customer_id":"...", "agent_id":"...",
  "type":"full|partial", "amount_minor":50000, "currency":"VND",
  "fault":"service_quality|no_show|overcharge|damage|duplicate|fraud|goodwill",
  "after_payout": true,                    // quyết định clawback vs reverse
  "destination":"wallet|source",
  "journal_entry_id":"je_777" }
```
Consumers: **loyalty** → clawback nếu earn confirmed / hủy nếu pending; **finance** → `agent_debt` nếu `after_payout && fault∈{thợ}`; **invoice** → e-invoice điều chỉnh.

### `ComplaintOpened` v1.0
```jsonc
{ "complaint_id":"...", "order_id":"...", "agent_id":"...",
  "category":"...", "opened_at":"..." }
```
Consumer: **finance/payout** → set earnings `on_hold` cho `agent_id` trên `order_id`.

## 5. Outbox + DLQ data model

```sql
CREATE TABLE outbox (
  id BIGINT PK, event_id CHAR(36) UNIQUE, event_name VARCHAR(48),
  aggregate_type VARCHAR(24), aggregate_id VARCHAR(64),
  ordering_key VARCHAR(64), payload JSON,
  status VARCHAR(12) DEFAULT 'pending',  -- pending|dispatched|failed
  attempts INT DEFAULT 0, next_retry_at TIMESTAMP NULL,
  created_at TIMESTAMP, dispatched_at TIMESTAMP NULL
);
CREATE TABLE dead_letters (
  id BIGINT PK, event_id CHAR(36), consumer VARCHAR(32),
  error TEXT, payload JSON, created_at TIMESTAMP, replayed_at TIMESTAMP NULL
);
-- consumer dedup: Redis SET processed:{consumer}:{event_id} EX 604800
```

**Dispatcher loop**: đọc `outbox WHERE status='pending' AND next_retry_at<=now ORDER BY id`, gọi consumer (in-process handler), thành công → `dispatched`; lỗi → tăng `attempts`, set `next_retry_at` (backoff); quá 5 → DLQ + alert.

## 6. Test checklist (cho QC)
- Replay 1 event 2 lần → consumer chỉ apply 1 (dedup).
- Producer crash giữa state-change và dispatch → outbox vẫn có event (cùng transaction).
- Out-of-order: `RefundSettled` tới trước `SettlementSettled` cùng order → consumer buffer/reject theo ordering_key.
- DLQ replay sau fix → đúng kết quả, không double-apply.

## 7. Changelog
| Version | Date | Changes |
| --- | --- | --- |
| 1.0 | 2026-05-28 | Envelope chuẩn; registry 16 event; payload contract core finance; outbox+DLQ; consumer dedup; versioning/breaking-change rule |
