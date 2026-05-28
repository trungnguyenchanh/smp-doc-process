# SMP QA · Test Plan + Test Cases + Bug Template

**Audience**: QC/QA team · **Updated**: v3.3

---

## 1. Test strategy overview

| Layer | Tool | Owner | Coverage target | Frequency |
|---|---|---|---|---|
| Unit test | Go testing | Backend Dev | 70% domain | Every commit (CI) |
| Integration test | Go + testcontainers | Backend Dev | Critical paths | Every commit (CI) |
| Contract test | Pact | Backend + Frontend | 100% endpoints | Per service repo CI |
| API E2E | Postman/Newman | QC | Smoke + critical flows | Every deploy |
| UI E2E | Playwright | QC | Top 10 user journeys | Every deploy to staging |
| Regression | Manual + automated | QC | Full regression suite | Pre-release |
| Performance | k6 | QC + DevOps | Critical endpoints | Weekly + pre-release |
| Security | OWASP ZAP + manual | Security | OWASP Top 10 | Monthly + pre-release |
| UAT | Manual | Business + select users | Real scenarios | Pre-pilot launch |

## 2. Test plan template (per release)

### 2.1 Header
```
Test Plan: SMP v3.3 Partner Platform
Release date: 2026-06-15
Test period: 2026-06-08 → 2026-06-14
QC Lead: <name>
Approvers: PM, Tech Lead
```

### 2.2 Scope
**In scope:**
- Partner CRUD (create, edit, KYC approval)
- Partner Owner đặt đơn private dispatch
- Wallet topup + transactions
- Partner monthly invoice generation
- 4 partner roles RBAC

**Out of scope (deferred to v3.4):**
- Multi-language UI cho partner
- Partner mobile app

### 2.3 Test environments
- Functional: `staging` (https://staging-api.smp.vn)
- Performance: `perf` (isolated, prod-sized data)
- Security: `staging`

### 2.4 Entry criteria
- All P0/P1 bugs from last release resolved
- All P2 bugs documented with workaround
- Migration scripts tested in dev
- Master data v3.3 loaded in staging
- Test accounts provisioned (8 partners, 7 partner_users)

### 2.5 Exit criteria
- 100% P0/P1 test cases pass
- ≥ 95% P2 test cases pass
- No new P0/P1 bugs open
- Performance metrics met (p95 < 2s for write endpoints)
- Security scan: no HIGH/CRITICAL findings

## 3. Test types

### 3.1 Functional test
Verify mỗi feature hoạt động đúng spec.

### 3.2 Integration test
Verify integration giữa các services + external (inside, wms).

### 3.3 Regression test
Re-test các features cũ để đảm bảo không break sau update.

### 3.4 UAT (User Acceptance Test)
Business team + select customers test real-world scenarios.

### 3.5 Performance test
- Load: dispatch latency < 2s p95 với 100 concurrent orders
- Stress: tăng 300% load để tìm breaking point
- Soak: chạy 24h liên tục, không leak memory
- Spike: 0 → 1000 RPS trong 30s

### 3.6 Security test
OWASP Top 10 + specific scenarios (xem doc Security testing).

## 4. Test case template

```
TC-ID: TC-ORDER-001
Title: Customer tạo đơn dịch vụ AC repair thành công
Module: Order
Type: Functional / Positive
Priority: P0 (critical)
Pre-condition:
  - Customer login với account "Nguyễn Văn A"
  - Customer có địa chỉ "15 Lê Lợi, Q.1, HCMC"
  - Service "Sửa điều hoà" đã được active
  - Có ≥ 1 thợ qualified online ở Q.1
Steps:
  1. Mở app SMP mobile
  2. Tap "Đặt dịch vụ" → "Sửa điều hoà"
  3. Confirm địa chỉ mặc định "15 Lê Lợi"
  4. Nhập note "Máy không lạnh"
  5. Tap "Đặt ngay"
Expected:
  - Đơn tạo thành công với order_id dạng "ord_..."
  - current_stage = "01_created"
  - UI redirect sang trang tracking
  - Sau 5 giây, dispatch trigger, current_stage = "02_dispatched_survey"
  - Push notification "Đang tìm thợ" gửi tới customer
Actual: <fill khi execute>
Status: Pass / Fail / Blocked
Defect ID: <link bug nếu fail>
Tested by: <name>
Tested at: <datetime>
```

## 5. Test cases sample · Order module

### TC-ORDER-001 · Tạo đơn happy path (P0)
Như mẫu trên.

### TC-ORDER-002 · Tạo đơn không có thợ qualified (P1)
**Pre**: Customer ở khu vực không có thợ (vd Côn Đảo)
**Steps**: Đặt đơn
**Expected**: API 422 `no_agent_available`, UI hiển thị "Hiện chưa có thợ ở khu vực này"

### TC-ORDER-003 · Tạo đơn với voucher hợp lệ (P0)
**Pre**: Voucher HOTSUMMER20 active (-20%, max 200k)
**Steps**: Tạo đơn 750k, áp voucher
**Expected**: Discount = 150k (20%×750k), total = 660k (gồm VAT 60k)

### TC-ORDER-004 · Tạo đơn với voucher hết hạn (P1)
**Pre**: Voucher EXPIRED20 hết hạn
**Steps**: Áp voucher
**Expected**: UI hiển thị "Mã không còn hiệu lực", discount = 0

### TC-ORDER-005 · Cancel đơn ở stage 01 (P1)
**Pre**: Đơn vừa tạo, stage 01
**Steps**: Tap "Huỷ đơn"
**Expected**: Stage → "cancelled", refund toàn bộ (nếu đã trả)

### TC-ORDER-006 · Cancel đơn sau khi survey paid (P1)
**Pre**: Đơn stage 05, đã trả phí khảo sát 80k
**Steps**: Cancel
**Expected**: Stage → "cancelled", refund 50% phí khảo sát = 40k (per policy)

### TC-ORDER-007 · Concurrent cancel + agent accept (P0)
**Pre**: Đơn stage 02, dispatch đang chạy
**Steps**: Customer cancel cùng lúc agent accept
**Expected**: Atomic: 1 thắng. Nếu cancel thắng, agent nhận "Đơn đã huỷ". Nếu accept thắng, customer nhận "Không huỷ được, thợ đã accept".

## 6. Test cases sample · Partner platform (v3.3)

### TC-PARTNER-001 · Partner Owner đặt đơn private dispatch (P0)
**Pre**: Login Partner Owner "Hùng AC Service", 8 agents qualified, wallet 8,500,000đ
**Steps**:
1. Tạo đơn dịch vụ AC, dispatch_visibility = private
2. Confirm
**Expected**:
- Đơn tạo với source = "partner_customer"
- Dispatch chỉ invite 8 agents của partner (verify qua admin monitor)
- Wallet trừ 750k → balance = 7,750,000đ
- partner_wallet_transactions row mới với type = "order_debit"

### TC-PARTNER-002 · Partner đặt đơn wallet không đủ (P1)
**Pre**: Wallet balance = 500,000đ
**Steps**: Tạo đơn 750k
**Expected**: API 402 `insufficient_balance`, UI hiển thị "Cần nạp thêm 250,000đ"

### TC-PARTNER-003 · Partner Finance export CSV (P2)
**Pre**: Login partner_finance role, có 87 transactions tháng 5
**Steps**: Filter month = 2026-05, tap "Export CSV"
**Expected**: File CSV download với 87 rows, cột chuẩn (txn_id, time, type, amount, order_id, balance_after, notes)

### TC-PARTNER-004 · RBAC: partner_finance không thấy được orders chi tiết (P0)
**Pre**: Login partner_finance
**Steps**: Truy cập trực tiếp URL `/partners/ptn_X/orders/ord_Y`
**Expected**: API 403 forbidden, không leak data

### TC-PARTNER-005 · Cross-partner data leak prevention (P0 · security)
**Pre**: Login partner_owner của "Hùng AC"
**Steps**: Truy cập URL của partner khác `/partners/ptn_freshair/orders`
**Expected**: API 403 forbidden, audit log entry "unauthorized_access_attempt"

## 7. Test cases · Dispatch engine

### TC-DISPATCH-001 · Round 1 ai accept trước thắng (P0)
**Pre**: Đơn ord_X dispatch, 3 thợ qualified
**Steps**: 3 thợ cùng nhận notification, agent T4K9 tap "Nhận" sau 5s, agent M2X9 tap "Nhận" sau 7s
**Expected**: T4K9 assigned, M2X9 nhận "Đơn đã có thợ khác nhận", API 409 conflict for M2X9

### TC-DISPATCH-002 · 60s timeout không ai accept → round 2 (P0)
**Pre**: Đơn ord_X round 1
**Steps**: Đợi 60s, không ai accept
**Expected**: Round 2 trigger, invite thêm thợ rộng zone (district lân cận)

### TC-DISPATCH-003 · 3 rounds fail → escalate to ops (P1)
**Pre**: Đơn ord_X
**Steps**: 3 rounds đều timeout
**Expected**: Stage chuyển sang "needs_manual_dispatch", task xuất hiện ở ops dashboard

### TC-DISPATCH-004 · WebSocket disconnect mid-dispatch (P1)
**Pre**: Thợ T4K9 online, dispatch invite
**Steps**: Tắt wifi của T4K9 trong lúc 60s window
**Expected**: Server detect disconnect sau 30s, không count T4K9 vào candidates round này

## 8. Test cases · Material BOM (v3.2)

### TC-BOM-001 · Tạo material variant với margin auto-calc (P1)
**Pre**: Material_type "Tụ điện 35μF" tồn tại, wms có SKU SKU-CAP-SAN-35 với cost 50k
**Steps**: Tạo variant Sanyo CBB60-35, sell_price 80k
**Expected**: variant created, cost_price_avg pulled từ wms = 50k, margin shown +60%

### TC-BOM-002 · Free-form material chờ verify (P0)
**Pre**: Thợ đang ở step, BOM expected có "tụ điện"
**Steps**: Thợ nhập free-form "Bơm xả Daikin TQ", qty 1, giá 280k
**Expected**: order_step_materials row mới với verify_status = "pending_verify", task xuất hiện ở ops material_verify page

### TC-BOM-003 · Ops verify free-form approve (P1)
**Pre**: Có 1 pending verify
**Steps**: Ops click "Approve off-catalog"
**Expected**: verify_status = "verified", audit log entry

### TC-BOM-004 · BOM variance > 5% flag review (P2)
**Pre**: BOM expected 0.5kg gas, thợ nhập 0.8kg
**Steps**: Submit
**Expected**: Tự động flag variance, hiện ở material_verify page

## 9. Test cases · Integration

### TC-INT-001 · webhook payment.succeeded valid HMAC (P0)
**Steps**: inside gửi webhook với signature đúng
**Expected**: Response 200, order.payment_status = "paid", dispatch survey trigger

### TC-INT-002 · webhook invalid HMAC (P0 · security)
**Steps**: Webhook với HMAC sai
**Expected**: Response 401, log security event, NO update

### TC-INT-003 · inside down → circuit breaker (P1)
**Pre**: inside mock timeout 5s
**Steps**: SMP cần customer data
**Expected**: Circuit breaker open sau N failures, UI fallback "Đang cập nhật", alert ops

### TC-INT-004 · wms stock check cache (P2)
**Steps**: Get variant stock 2 lần liên tiếp trong 30s
**Expected**: Lần 2 từ Redis cache, không gọi wms

## 10. Performance test criteria

### 10.1 Endpoints SLO
| Endpoint | Target p95 | Target p99 | Tool |
|---|---|---|---|
| `GET /api/v1/orders` (list) | < 300ms | < 800ms | k6 |
| `POST /api/v1/orders` | < 500ms | < 1500ms | k6 |
| `GET /api/v1/catalog/services` (cached) | < 100ms | < 300ms | k6 |
| Dispatch round (server-side) | < 2s | < 5s | Custom monitor |
| WebSocket message delivery | < 200ms | < 500ms | k6 |

### 10.2 Throughput
- Steady state: 100 RPS sustained 1h
- Peak: 500 RPS for 5 min
- Spike: 0 → 200 RPS in 10s, system stable

### 10.3 Resource usage
- Pod CPU < 70% under sustained load
- Pod RAM < 80% (no leak after 24h soak)
- MySQL connections < 80% of pool

### 10.4 k6 test script sample
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 50 },   // ramp up
    { duration: '5m', target: 100 },  // steady
    { duration: '2m', target: 200 },  // spike
    { duration: '5m', target: 100 },  // back to steady
    { duration: '2m', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function() {
  const res = http.get('https://staging-api.smp.vn/api/v1/orders?limit=20', {
    headers: { Authorization: `Bearer ${__ENV.TOKEN}` },
  });
  check(res, {
    'status 200': r => r.status === 200,
    'has items': r => JSON.parse(r.body).items !== undefined,
  });
  sleep(1);
}
```

## 11. Security test checklist (OWASP Top 10)

| ID | Check | Method |
|---|---|---|
| A01 | Broken access control · cross-partner data leak | Manual + automated |
| A02 | Cryptographic failures · JWT signed, secrets in Vault | Code review + scan |
| A03 | Injection (SQL, NoSQL, command) | Burp Suite + manual |
| A04 | Insecure design · rate limit, CSRF, idempotency | Manual review |
| A05 | Security misconfiguration · headers, error messages | OWASP ZAP |
| A06 | Vulnerable components · `govulncheck`, Trivy | CI scan |
| A07 | Authentication failures · brute force, weak passwords | Manual + tools |
| A08 | Software & data integrity · signed releases, SBOM | Cosign verify |
| A09 | Logging failures · audit log present | Manual + log review |
| A10 | SSRF · webhook URL validation | Burp |

## 12. Bug template

```
Bug ID: BUG-2026-1042
Title: [Partner] Wallet topup fail with bank_transfer method

Severity: P1 (high) / P0 (critical) / P2 (medium) / P3 (low)
Priority: P0 (must fix this sprint) / P1 / P2 / P3
Module: Partner > Wallet
Reproducibility: Always / Often / Sometimes / Once

Environment:
  - Build: v3.3.0-rc.2
  - Env: staging
  - Browser/App: Chrome 124 / iOS 17

Pre-condition:
  - Login partner_finance "Hùng AC Service"
  - Wallet balance 500,000đ

Steps to reproduce:
  1. Mở "Tài chính" → "Ví"
  2. Tap "Nạp tiền"
  3. Chọn "Chuyển khoản ngân hàng"
  4. Nhập amount 5,000,000đ
  5. Tap "Tạo yêu cầu"

Expected:
  - Tạo wallet_transaction row với type = "topup", status = "pending"
  - Hiển thị QR code chuyển khoản
  - Status quay về "pending bank confirmation"

Actual:
  - API 500 internal_error
  - UI hiển thị "Có lỗi xảy ra, vui lòng thử lại"
  - Backend log: `nil pointer dereference at wallet/service.go:142`

Attachments:
  - screenshot_500.png
  - har_export.har
  - backend_log_snippet.txt

Workaround: None

Discovered by: <QC name>
Discovered at: 2026-06-10 14:23
Assigned to: <dev name>
```

## 13. Severity matrix

| Sev | Definition | Examples | SLA fix |
|---|---|---|---|
| P0 (Critical) | System down, data loss, security breach | Login down, payment double-charge, PII leak | 4h |
| P1 (High) | Major feature broken, no workaround | Cannot create order, wallet calc wrong | 1 day |
| P2 (Medium) | Feature broken with workaround | UI typo, slow page (still works) | 1 sprint |
| P3 (Low) | Cosmetic, minor inconvenience | Color mismatch, slight misalignment | Backlog |

## 14. Test execution checklist (per release)

- [ ] Test plan reviewed + approved
- [ ] Test environment provisioned + seed data loaded
- [ ] All test cases assigned to QC team members
- [ ] P0/P1 cases executed first
- [ ] Bugs filed in tracker (JIRA/Linear)
- [ ] Daily bug triage with dev team
- [ ] Regression suite run before sign-off
- [ ] Performance baseline run + results compared
- [ ] Security scan run + findings reviewed
- [ ] Test report published to stakeholders
- [ ] Go/No-Go decision documented
