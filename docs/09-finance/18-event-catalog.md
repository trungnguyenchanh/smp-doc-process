# Doc 18 · Event Catalog 
**Status**: DRAFT — *đóng blocker cross-service* · **Audience**: Backend, BA, QC, DevOps

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
 "event_id": "uuid", // idempotency key toàn cục
 "event_name": "SettlementSettled",
 "event_version": "1.0",
 "aggregate_type": "settlement",
 "aggregate_id": "settlement_123",
 "ordering_key": "order_456", // xử lý tuần tự theo key
 "occurred_at": "2026-05-28T03:00:00Z", // UTC
 "producer": "finance-svc",
 "trace_id": "...", // distributed tracing
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
| `StepAgentAssigned` | dispatch | agent(notification), audit | order_step_id | Thợ được assign vào step  |
| `StepLeadAccepted` | dispatch | finance, agent, audit | order_step_id | Lead accept step → unlock helper invitations  |
| `StepSplitOverridden` | dispatch | finance, audit | order_step_id | Lead override split_bps cho helper  |
| `StepEarningsCalculated` | finance | agent(earnings_mv), notification | order_step_id | Tính & post earnings cho N agents  |
| `WarrantyPackagePurchased` | warranty | finance, notification | customer_warranty_id | KH mua gói BH · trigger deferred revenue posting  |
| `WarrantyClaimOpened` | warranty | ops(approval queue), agent, notification | warranty_claim_id | KH yêu cầu claim · cần quota check + Ops approval cho repair  |
| `WarrantyClaimApproved` | warranty | order(create free order), agent(notification) | warranty_claim_id | Claim approved · tạo order với amount=0  |
| `WarrantyRevenueRecognized` | warranty | finance, audit | customer_warranty_id | Monthly cron post journal: deferred → revenue  |
| `WarrantyPackageExpired` | warranty | notification, finance | customer_warranty_id | Gói hết hạn · residual revenue recognized · suggest renew  |
| `MaintenanceScheduled` | warranty | notification, customer | customer_warranty_id | Auto-suggest KH đặt lịch vệ sinh định kỳ  |

## 4. Payload contract (core finance events)

### `SettlementSettled` 
```jsonc
{
 "settlement_id": "settlement_123", "order_id": "order_456",
 "method": "gateway|wallet|cod|partner_wallet",
 "amount_minor": 100000, "currency": "VND",
 "agent_id": "agent_7", "customer_id": "cust_9",
 "settled_at": "2026-05-28T03:00:00Z",
 "refund_window_until": "2026-06-04T03:00:00Z",
 "journal_entry_id": "je_555" // link sang ledger (SoT)
}
```
Consumers: **loyalty** → earn `pending` (BR-LOYAL-001); **finance/payout** → start T+1 clock; **quality** → set warranty `payment_backed=true`.

### `CODCollected` 
```jsonc
{ "settlement_id":"...", "order_id":"...", "agent_id":"...",
 "amount_minor":100000, "currency":"VND",
 "agent_cod_receivable_delta": 100000, // naming chốt (Doc 16 )
 "collected_at":"..." }
```

### `RefundSettled` 
```jsonc
{ "refund_id":"...", "order_id":"...", "customer_id":"...", "agent_id":"...",
 "type":"full|partial", "amount_minor":50000, "currency":"VND",
 "fault":"service_quality|no_show|overcharge|damage|duplicate|fraud|goodwill",
 "after_payout": true, // quyết định clawback vs reverse
 "destination":"wallet|source",
 "journal_entry_id":"je_777" }
```
Consumers: **loyalty** → clawback nếu earn confirmed / hủy nếu pending; **finance** → `agent_debt` nếu `after_payout && fault∈{thợ}`; **invoice** → e-invoice điều chỉnh.

### `ComplaintOpened` 
```jsonc
{ "complaint_id":"...", "order_id":"...", "agent_id":"...",
 "category":"...", "opened_at":"..." }
```
Consumer: **finance/payout** → set earnings `on_hold` cho `agent_id` trên `order_id`.

### `StepAgentAssigned` 
```jsonc
{
 "order_step_id": "ostep_123", "order_id": "order_456",
 "agent_id": "agent_789", "role": "lead|helper|specialist",
 "specialty": "electrician", // null nếu role=lead/helper generic
 "split_bps": 7000, // effective split (default or override)
 "is_override": false,
 "assigned_by": "agent_999", // ai assigned (lead or dispatch-svc)
 "assigned_at": "2026-06-15T03:00:00Z"
}
```
Consumers: **agent** → push notification mời thợ; **audit** → log assignment trail.

### `StepLeadAccepted` 
```jsonc
{
 "order_step_id": "ostep_123", "order_id": "order_456",
 "lead_agent_id": "agent_789",
 "accepted_at": "2026-06-15T03:30:00Z",
 "helpers_invitation_deadline": "2026-06-15T05:30:00Z" // BR-MA-007 · 2h SLA
}
```
Consumers: **finance** → mark step ready for in_progress; **agent** → enable lead UI để invite helpers; **audit** → state change.

### `StepSplitOverridden` 
```jsonc
{
 "order_step_id": "ostep_123", "order_id": "order_456",
 "overridden_by": "agent_lead_id",
 "changes": [
 { "agent_id": "agent_helper_X", "old_split_bps": 3000, "new_split_bps": 2000 },
 { "agent_id": "agent_helper_Y", "old_split_bps": 0, "new_split_bps": 1000 } // added
 ],
 "overridden_at": "2026-06-15T04:00:00Z",
 "reason": "added second helper" // free text optional
}
```
Consumers: **finance** → validate SUM still = 10000; **audit** → log full diff for compliance.

### `StepEarningsCalculated` 
```jsonc
{
 "order_step_id": "ostep_123", "order_id": "order_456",
 "step_revenue_minor": 300000, "currency": "VND",
 "step_commission_minor": 45000,
 "agents": [
 { "agent_id": "agent_A", "role": "lead", "split_bps": 4000, "amount_earned_minor": 18001 }, // lead absorbs +1 residual
 { "agent_id": "agent_C", "role": "specialist", "split_bps": 4000, "amount_earned_minor": 17999 },
 { "agent_id": "agent_D", "role": "helper", "split_bps": 2000, "amount_earned_minor": 9000 }
 ],
 "journal_entry_id": "je_888", // ref to journal_entries.id
 "calculated_at": "2026-06-15T10:00:00Z"
}
```
Consumers: **agent(earnings_mv)** → update materialized view; **notification** → push "Bạn đã nhận X VND từ step Y"; **audit** → trail full split.

## 5. Outbox + DLQ data model

```sql
CREATE TABLE outbox (
 id BIGINT PK, event_id CHAR(36) UNIQUE, event_name VARCHAR(48),
 aggregate_type VARCHAR(24), aggregate_id VARCHAR(64),
 ordering_key VARCHAR(64), payload JSON,
 status VARCHAR(12) DEFAULT 'pending', -- pending|dispatched|failed
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
