<!-- 
  ────────────────────────────────────────────────────────────────
  📋 DOC STATUS NOTE (added when integrating into smp-doc-process)
  
  Đây là DRAFT specification cho settlement lifecycle.
  Phân biệt rõ với Order Lifecycle (Doc 15 stage rules):
    - Order lifecycle = customer-facing 10-stage flow
    - Settlement lifecycle = backend money state machine
  
  Tham chiếu chéo:
    - Doc 16 Finance Ledger (journal entries cho mỗi state transition)
    - Doc 18 Event Catalog (events emit per state change)
    - Doc 15 Business Rules section "Payment Rules" và "Stages"
  
  Author: SMP Tech Lead/CTO · Date: 2026-05-28
  Status: DRAFT v1.2 · supersedes v1.0
  ────────────────────────────────────────────────────────────────
-->

# Doc 17 · Payment & Settlement Lifecycle · v1.2

**Version**: 1.2 · **Date**: 2026-05-28 · **Status**: DRAFT — matrix corrected · **Supersedes**: v1.0
**Audience**: Backend, Finance, BA, QC

> v1.2 sửa lỗi vòng review: `COD_PENDING` nay có đường vào (Option A), `VOIDED` làm rõ theo thời điểm tạo settlement, thêm rule **ledger↔settlement source-of-truth** và **COD-refund-before-remit**.

---

## 1. Hai aggregate (giữ v1.0)
ORDER lifecycle (customer-facing, 10-stage) tách khỏi SETTLEMENT lifecycle (máy tiền nội bộ). Nối qua event. `order.paid` chỉ là cờ hiển thị; sự thật tiền ở ledger (Doc 16 v1.2 §6).

## 2. Thời điểm tạo Settlement theo method (MỚI — sửa VOIDED)

| Method | Settlement tạo lúc | Ghi chú |
| --- | --- | --- |
| gateway / wallet / cod | `order.completed` | Cancel ≤ stage 06 xảy ra **trước** → KHÔNG có settlement → KHÔNG dùng `VOIDED` |
| **partner_wallet** (prepaid) | **stage 01** (auto-debit ví partner) | Method DUY NHẤT tạo settlement sớm; cancel sau prepay → đi `VOIDED` (hoàn về ví partner) |

→ `VOIDED` chỉ hợp lệ cho `partner_wallet` (hoặc edge: cancel sau `completed` nhưng trước thu tiền). Bỏ `VOIDED` khỏi các path gateway/wallet/cod thông thường.

## 3. States

| State | Ý nghĩa | SMP kiểm soát tiền? | customer_payment_status |
| --- | --- | --- | --- |
| `INIT` | Settlement vừa tạo | ❌ | "Chờ thanh toán" |
| `AWAITING_GATEWAY` | Đã tạo phiên cổng | ❌ | "Đang xử lý" |
| `COD_PENDING` | Đơn COD, **chờ thợ thu** | ❌ | "Thanh toán khi hoàn tất" |
| `COD_COLLECTED` | Thợ đã thu cash | ❌ (`agent_cod_receivable`) | **"Đã thanh toán (tiền mặt)"** |
| `SETTLED` | **Tiền thuộc kiểm soát SMP** | ✅ | "Đã thanh toán" |
| `REFUND_PENDING` / `PARTIALLY_REFUNDED` / `REFUNDED` | Hoàn | ✅/— | tương ứng |
| `FAILED` | Cổng lỗi / ví thiếu / COD không thu được | ❌ | "Thất bại" |
| `VOIDED` | Hủy trước thu (chỉ partner_wallet/edge) | — | "Đã hủy" |

`SETTLED` đạt khi: gateway webhook OK (HMAC) | wallet debit OK | **COD `cod_remitted`** (KHÔNG phải collected) | partner auto-debit OK.

## 4. Transition matrix (Option A — `COD_PENDING` có đường vào)

| Từ \ Sự kiện | `completed` | `init_gateway` | `init_cod` | `gateway_ok` | `gateway_fail` | `wallet_ok` | `wallet_insuf` | `tech_collect` | `tech_remit` | `refund_req` | `refund_partial` | `refund_full` | `cancel(partner)` |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| *(none)* | `INIT` | — | — | — | — | — | — | — | — | — | — | — | — |
| `INIT` (gateway) | — | `AWAITING_GATEWAY` | — | — | — | — | — | — | — | — | — | — | — |
| `INIT` (wallet) | — | — | — | — | — | `SETTLED` | `FAILED` | — | — | — | — | — | — |
| `INIT` (cod) | — | — | `COD_PENDING` | — | — | — | — | — | — | — | — | — | — |
| `INIT` (partner) | — | — | — | — | — | `SETTLED` | `FAILED` | — | — | — | — | — | `VOIDED` |
| `AWAITING_GATEWAY` | — | — | — | `SETTLED` | `FAILED` | — | — | — | — | — | — | — | — |
| `COD_PENDING` | — | — | — | — | — | — | — | `COD_COLLECTED` | — | — | — | — | — |
| `COD_COLLECTED` | — | — | — | — | — | — | — | — | `SETTLED` | `REFUND_PENDING`* | — | — | — |
| `FAILED` | — | `AWAITING_GATEWAY` | `COD_PENDING` | — | — | `SETTLED` | `FAILED` | — | — | — | — | — | — |
| `SETTLED` | — | — | — | — | — | — | — | — | — | `REFUND_PENDING` | — | — | — |
| `REFUND_PENDING` | — | — | — | — | — | — | — | — | — | — | `PARTIALLY_REFUNDED` | `REFUNDED` | — |
| `PARTIALLY_REFUNDED` | — | — | — | — | — | — | — | — | — | `REFUND_PENDING` | — | `REFUNDED` | — |

`*` `COD_COLLECTED → REFUND_PENDING`: refund đơn COD **trước remit** (§7). Sau refund này, `agent_cod_receivable` vẫn còn → thợ vẫn phải remit.

**Invariants**: `SETTLED` chỉ từ {AWAITING_GATEWAY, INIT-wallet/partner, COD_COLLECTED}. `REFUND_*` chỉ sau `SETTLED` hoặc `COD_COLLECTED` (case COD). `COD_PENDING`/`COD_COLLECTED` không tự `SETTLED` — phải có `tech_remit`.

## 5. Events (giữ v1.0, contract đầy đủ ở Doc 18)
`SettlementCreated · PaymentConfirmed · SettlementSettled · CODCollected · CODRemitted · RefundSettled · SettlementFailed`. Transport = **outbox** (chưa cần Kafka).

## 6. 5 câu hỏi lifecycle (giữ v1.0, chốt cứng)
1. Warranty effective tại `completed`, `payment_backed=false` tới `SETTLED`, void nếu không settle trong grace 7 ngày.
2. COD collected = UX-paid, kế toán chưa; `SETTLED` tại remit; tạo `agent_cod_receivable` ngay khi collect.
3. Loyalty earn tại `SETTLED` ở `pending`; confirm sau refund window 7d; refund → clawback.
4. Payout T+1 từ `SETTLED` (gateway: từ gateway_ok; COD: từ remit). Hold nếu complaint mở.
5. Refund sau payout → `agent_debt`, thu hồi từ earning tương lai (Doc 16 v1.2 §3.3–3.4).

## 7. Ledger↔Settlement SoT + COD refund (MỚI)
- **Source of truth**: ledger = tiền; settlement = quy trình (Doc 16 v1.2 §6). Daily reconcile bắt khớp; lệch → freeze + alert; ledger thắng về tiền.
- **Refund đơn COD trước remit**: đảo `cod_clearing → customer_wallet` (chưa recognize revenue nên sạch); `agent_cod_receivable` giữ nguyên → thợ vẫn remit, SMP thu hồi; thợ không remit → `agent_debt` (Doc 16 v1.2 §3.4).

## 8. Edge cases (giữ + bổ sung)
Webhook 2 lần → idempotent theo `gateway_ref`. COD collected quá SLA không remit → freeze toàn bộ payout thợ. Gateway settle về bank trễ → `Dr cash_bank/Cr cash_gateway`, không đổi state.

## 9. Changelog
| Version | Date | Changes |
| --- | --- | --- |
| 1.2 | 2026-05-28 | Option A matrix (COD_PENDING có đường vào); thời điểm tạo settlement theo method; VOIDED làm rõ; ledger↔settlement SoT; COD-refund-before-remit |
