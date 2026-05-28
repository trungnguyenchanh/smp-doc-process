# SMP Business Rules

**Audience**: BA, PM, Dev, QC, Ops · **Updated**: v3.3

---

## Mục đích

Tài liệu này tập trung **mọi business rule** rải rác trong SPEC v3.x. Khi dev code, QC test, ops vận hành — đây là **single source of truth** cho logic.

Mỗi rule có ID `BR-<category>-<num>` để reference từ user story, test case, code comment.

---

## 🔮 Migration plan v4.0 · Markdown rules → `rules_engine.yaml`

> **Quan trọng**: Format hiện tại (Markdown prose) phù hợp cho v3.x đọc/review thủ công. Khi sang v4.0, toàn bộ rules sẽ migrate sang **YAML** + load runtime qua `antonmedv/expr` (xem [Doc 05 · Glossary section O](./05-glossary.md#o--rules-engine-terms--v40)).

### Tại sao migrate?

| Vấn đề hiện tại (v3.x Markdown) | Giải pháp v4.0 (YAML + expr) |
|---|---|
| Thay đổi rule = code change + deploy | Update ConfigMap → hot-reload trong 30s |
| Rule khác nhau theo country = nhiều `if` trong Go code | YAML có `when: country == 'VN'` |
| Khó test rule độc lập | Unit test YAML rule với mock context |
| BA muốn xem rule phải đọc Go code | BA đọc YAML, hiểu ngay |
| Khó audit "rule X có hiệu lực từ khi nào" | YAML version trong Git, blame thấy ai sửa khi nào |

### Rule ID convention mới · v4.0

```text
<country_code>.<category>.<num>
```

- `country_code`: ISO 3166-1 alpha-2 lowercase (`vn`, `us`, `cn`, `th`, `id`, `sg`)
- `category`: `dispatch`, `pricing`, `payment`, `kyc`, `stage`, `material`, `partner`, `integration`, `quality`, `notification`, `retention`
- `num`: 3 digit zero-padded (`001`, `025`, `100`)

**Examples**:
- `vn.dispatch.001` — Round timeout Vietnam (60s)
- `us.dispatch.001` — Round timeout US (90s, do thợ rải rác hơn)
- `vn.pricing.007` — VAT 10% Vietnam
- `sg.pricing.007` — GST 9% Singapore
- `*.kyc.001` — Áp dụng tất cả country (wildcard)

**Mapping với BR-* cũ**: rule `BR-DISP-001` v3.x → `vn.dispatch.001` v4.0.

### Format comparison · 1 rule trong 2 format

**Format hiện tại (v3.x Markdown)**:
```markdown
### BR-DISP-001 · Round timeout
Mỗi dispatch round có timeout **60 giây** (production), **30 giây** (dev/staging)
để thợ accept. Nếu không có ai accept → tự động chuyển sang round tiếp theo.
```

**Format mới (v4.0 YAML)**:
```yaml
- id: vn.dispatch.001
  name: "Round timeout Vietnam"
  category: dispatch
  description: "Mỗi dispatch round có timeout 60s production, 30s dev/staging."
  enabled: true
  priority: 100
  when: "context.country_code == 'VN'"
  then:
    round_timeout_seconds: 60
  overrides:
    - when: "context.env in ['dev', 'staging']"
      then:
        round_timeout_seconds: 30
  changelog:
    - { version: "3.0", date: "2026-01-15", author: "BA team", change: "Initial" }
```

### Migration roadmap

| Phase | Khi nào | Action |
|---|---|---|
| **Phase 0** | v3.4 (now) | Giữ Markdown. Doc này là source. |
| **Phase 1** | v3.5 (Q3 2026) | Build tool: parse Markdown → generate YAML draft. Review thủ công. Deploy ConfigMap parallel với code rules cũ. |
| **Phase 2** | v3.6 (Q3 2026) | Switch code over: dispatch-engine + finance-svc + pricing logic đọc từ YAML. Markdown thành reference. |
| **Phase 3** | v4.0 (Q4 2026) | YAML = source of truth. Markdown auto-generate từ YAML cho human-readable. |

### Cấu trúc file `rules_engine.yaml` (preview cho v4.0)

```yaml
version: "4.0"
last_updated: "2026-11-01"
maintainers: ["ba-team@smp.vn"]

# Context schema · các biến rule có thể reference
context_schema:
  country_code: string    # 'VN', 'US', 'CN'
  env: string              # 'dev', 'staging', 'prod'
  order: object            # full order entity
  agent: object            # full agent entity
  customer: object         # full customer entity
  partner: object | null   # nếu order qua partner
  now: timestamp           # UTC timestamp

rules:
  # === DISPATCH ===
  - id: vn.dispatch.001
    name: "Round timeout"
    # ... (như trên)

  - id: vn.dispatch.002
    name: "Max rounds"
    when: "context.country_code == 'VN'"
    then:
      max_rounds: 3

  # === PRICING ===
  - id: vn.pricing.001
    name: "Labor price multiplier by level"
    when: "context.country_code == 'VN'"
    then:
      multiplier_map:
        junior: 0.8
        mid: 1.0
        senior: 1.3

  - id: vn.pricing.007
    name: "VAT Vietnam"
    when: "context.country_code == 'VN'"
    then:
      vat_rate: 0.10

  - id: sg.pricing.007
    name: "GST Singapore"
    when: "context.country_code == 'SG'"
    then:
      vat_rate: 0.09

  # ... 80+ rules
```

### Đọc rules trong code Go (v4.0 preview)

```go
import "github.com/expr-lang/expr"

type RulesEngine struct {
    rules []Rule
    cache map[string]*vm.Program // compiled expressions
}

func (e *RulesEngine) Evaluate(category string, ctx Context) map[string]any {
    var matched []Rule
    for _, r := range e.rules {
        if r.Category != category || !r.Enabled {
            continue
        }
        program := e.cache[r.ID]
        result, err := expr.Run(program, ctx)
        if err == nil && result.(bool) {
            matched = append(matched, r)
        }
    }
    // sort by priority, merge `then` results
    return mergeThens(matched)
}

// Usage trong dispatch-engine:
ctx := Context{CountryCode: "VN", Env: "prod", Order: order}
result := engine.Evaluate("dispatch", ctx)
timeout := result["round_timeout_seconds"].(int) // 60
```

---

## A · Dispatch Rules

### BR-DISP-001 · Round timeout
Mỗi dispatch round có timeout **60 giây** (production), **30 giây** (dev/staging) để thợ accept. Nếu không có ai accept → tự động chuyển sang round tiếp theo.

### BR-DISP-002 · Max rounds
Tối đa **3 rounds** (production), sau đó đơn được **escalate** sang ops dispatch tay.

### BR-DISP-003 · Round expansion
- **Round 1**: invite tất cả agent qualified online trong cùng `district` của customer
- **Round 2**: mở rộng sang các district lân cận (within 5km hoặc cùng `coverage_zone`)
- **Round 3**: invite mọi agent qualified trong `city`, kể cả offline (push notification)

### BR-DISP-004 · Qualified agent definition
Agent qualified để nhận đơn khi:
- `agents.status = 'active'`
- `agents.is_online = true` (round 1-2)
- Có `agent_skills` matching service step's `skill_id` với `level >= step.min_level`
- KYC level ≥ basic
- Không bị suspended trong tháng

### BR-DISP-005 · Private dispatch logic (v3.3)
Nếu `orders.source = 'partner_customer'` AND `dispatch_visibility = 'private'`:
- **Tất cả** rounds **chỉ** dispatch cho agents thuộc `partner_id` của đơn
- Nếu 3 rounds fail → escalate to ops (NOT mở rộng cho thợ partner khác)

### BR-DISP-006 · Open dispatch with partner preference (v3.3)
Nếu `dispatch_visibility = 'open'` AND `partner_id != NULL`:
- **Round 1**: dispatch cho agents của partner trước
- **Round 2-3**: mở rộng cho tất cả agents qualified

### BR-DISP-007 · Surge multiplier
Khi `coverage_zone.surge_multiplier > 1.0`:
- `labor_price` × multiplier applied to all steps in zone đó
- Hiển thị notice cho customer trước khi đặt: "Khu vực này đang phụ phí giờ cao điểm +XX%"
- Multiplier tính lúc create order, **không** đổi khi đơn đang chạy

### BR-DISP-008 · Manual dispatch override
Ops admin với scope `dispatch.manage` có thể assign đơn cho **bất kỳ** agent active, không cần qualified theo skill — nhưng phải có lý do (audit log).

### BR-DISP-009 · Concurrent invitation cap
1 agent không nhận quá **2 invitation đồng thời**. Nếu agent đang xử lý 2 đơn → bị exclude khỏi round.

---

## B · Pricing Rules

### BR-PRICE-001 · Labor price by level
`labor_price` cho 1 step = `steps.labor_price_l<N>` với N = level của agent assigned.

Nếu chưa biết level (lúc estimate trước khi assign): dùng `labor_price_l3` (mặc định).

### BR-PRICE-002 · Material sell price
`material_variants.sell_price` do **SMP set**, **không** lấy từ wms cost. SMP control margin.

### BR-PRICE-003 · Quote subtotal
```text
labor_subtotal = SUM(order_steps.labor_price)
material_subtotal = SUM(order_step_materials.unit_price * quantity)
survey_fee = service.survey_fee (default 80,000đ)
discount = applied voucher amount (max voucher.max_discount)
vat = (labor + material + survey - discount) * 10%
total = labor + material + survey - discount + vat
```

### BR-PRICE-004 · Survey fee policy
- Customer trả survey fee TRƯỚC khi Survey Agent xuất phát (default 80,000đ)
- Survey fee:
  - Hoàn 100% nếu Survey Agent **không đến** trong cam kết (no-show)
  - Hoàn 50% nếu customer **từ chối** báo giá (compensate cho thợ đã đi)
  - Không hoàn nếu khách **duyệt** báo giá (đã được trừ vào tổng đơn)

### BR-PRICE-005 · Voucher discount
- Customer nhập voucher_code → SMP gọi `inside.vouchers.validate(code, customer_id, amount)`
- inside trả về `discount_amount` (có thể là % hoặc fixed)
- SMP áp dụng discount tối đa `voucher.max_discount`
- Voucher chỉ áp dụng 1 lần per đơn

### BR-PRICE-006 · Surge applies to labor only
Surge multiplier áp dụng cho `labor_price`, **không** áp dụng cho `material_price`.

### BR-PRICE-007 · Partner pricing override (v3.3)
Nếu partner có custom pricing config:
- `partners.customer_config.labor_discount_percent`: giảm % labor cho partner đó
- `partners.customer_config.no_material_markup`: nếu true, partner trả material theo cost_price (không markup)

Default: không override.

### BR-PRICE-008 · VAT
- VAT 10% áp dụng cho **mọi** đơn cho cá nhân
- Đơn B2B với partner_type = 'business': VAT 10% với invoice xuất tên công ty partner
- Đơn `source = 'contract'`: VAT theo terms hợp đồng (có thể 8% hoặc khác)

### BR-PRICE-009 · Rounding
- Tất cả calculations giữ precision integer (VND no decimal)
- Discount/percent rounding: ROUND HALF UP (vd 12,505 → 12,510 nếu round to 10)

---

## C · Payment Rules

### BR-PAY-001 · Payment timing
- **Customer direct**: thanh toán **sau khi duyệt báo giá** (stage 06)
- **Partner customer (prepaid wallet)**: trừ wallet ngay khi tạo đơn (stage 01)
- **Partner customer (monthly invoice)**: ghi nợ vào invoice, không trừ ngay

### BR-PAY-002 · Refund logic
| Cancel at stage | Refund |
|---|---|
| 01-02 (chưa có thợ) | 100% nếu đã trả |
| 03-04 (thợ đang đến) | 100% trừ phí khảo sát |
| 05 (đã khảo sát, chờ duyệt) | 100% trừ 50% phí khảo sát (40k giữ lại) |
| 06-07 (đã duyệt, chờ thực hiện) | 100% labor + 100% material (nếu wms cho cancel reservation) |
| 08 (đang sửa) | Pro-rata: trả labor đã làm + material đã dùng, refund phần còn lại |
| 09-10 (đã xong) | Không refund (trừ trường hợp dispute) |

### BR-PAY-003 · Partner wallet topup
- Topup methods: bank_transfer, e-wallet
- Bank transfer: tạo wallet_transaction status = 'pending', confirm sau khi tài chính SMP verify (manual)
- E-wallet: confirm tự động qua webhook
- Minimum topup: 1,000,000đ
- Maximum topup per transaction: 100,000,000đ (above cần manual approval)

### BR-PAY-004 · Partner wallet debit
Khi tạo đơn `partner_customer`:
1. Pre-check `wallet_balance >= total_amount`
2. Atomic: lock wallet row + insert wallet_transaction (type='order_debit') + update balance
3. Nếu order cancelled: insert wallet_transaction (type='refund') hoàn lại

### BR-PAY-005 · Insufficient wallet
- API trả 402 `insufficient_balance` với detail
- UI hiển thị "Cần nạp thêm <X>đ"
- Không allow tạo đơn nếu thiếu

---

## D · KYC Rules

### BR-KYC-001 · Agent KYC levels
| Level | Required docs | Allowed actions |
|---|---|---|
| `pending` | None | View pool, can't accept |
| `basic` | CCCD front+back, portrait selfie, bank statement | Accept orders, max value 5M đ/đơn |
| `full` | basic + skill certificate + insurance | All orders, no value cap |

### BR-KYC-002 · KYC progression
- Agent đăng ký → pending
- Upload docs basic → review → approve/reject (manual ops)
- Nếu approve → status = active, level = basic
- Sau 10 đơn completed + rating ≥ 4.0 → có thể upgrade to full (request)
- Upgrade to full requires submit thêm certificate + insurance

### BR-KYC-003 · Partner KYC levels (v3.3)
| Level | Required docs | Allowed actions |
|---|---|---|
| `pending` | None | View only, can't create orders |
| `basic` | GPKD/CCCD rep, STK ngân hàng, bằng nghề (1 thợ chính) | Tạo đơn, max 50 đơn/tháng |
| `full` | basic + hợp đồng dịch vụ + bảo hiểm trách nhiệm | Tạo đơn không giới hạn, supplier mode |

### BR-KYC-004 · KYC verification responsibility
- Agent KYC: SMP ops verify (theo doc image quality + cross-check info)
- Partner KYC: SMP ops Partner Squad verify, có thể cần outsource thẩm định business

### BR-KYC-005 · Rejection process
- Reject → status không đổi, gửi notification với reason
- User có thể re-upload (max 3 lần)
- Sau 3 lần fail → require manual call/meeting

---

## E · Order Stage Transition Rules

### BR-STAGE-001 · Allowed transitions
Chỉ các transitions sau là valid (state machine):

```text
01_created → 02_dispatched_survey (auto, sau khi payment OK)
02_dispatched_survey → 03_survey_accepted (khi agent tap "Nhận")
02_dispatched_survey → 02_dispatched_survey (round 2, 3)
02_dispatched_survey → needs_manual_dispatch (sau 3 rounds fail)
03_survey_accepted → 04_arrived_survey (khi agent check-in GPS gần customer)
04_arrived_survey → 05_surveyed (agent submit báo giá)
05_surveyed → 06_quote_approved (customer duyệt)
05_surveyed → cancelled (customer từ chối)
06_quote_approved → 07_dispatched_execution (auto, nếu khác agent than survey)
06_quote_approved → 08_in_progress (auto, nếu cùng agent — Dual)
07_dispatched_execution → 08_in_progress (khi execution agent accept)
08_in_progress → 09_completed (khi agent submit completion)
09_completed → 10_rated (khi customer rate)
9_completed → 10_rated (auto sau 24h nếu không rate)

Any stage → cancelled (with reason, ngoại trừ 09, 10)
```

### BR-STAGE-002 · Cancel restrictions
- Stage 08 (đang sửa): chỉ ops admin cancel được, không phải customer/agent
- Stage 09, 10: KHÔNG cancel được — phải qua dispute flow

### BR-STAGE-003 · GPS check-in
Stage transit 03 → 04 yêu cầu agent GPS within **300m** của customer address. Nếu xa hơn → require justification + ops review.

### BR-STAGE-004 · Photo proof required
Stages requiring photo:
- 04: 1 ảnh thiết bị "before"
- 05: tổng ảnh ≥ 3 (before, during, các phần hư hỏng)
- 08: ảnh "after" per step (mỗi step ít nhất 1)
- 09: 1 ảnh "final" tổng quan

Không upload đủ → không submit được transition.

### BR-STAGE-005 · Stage 10 rating logic
- Customer rate 5: agent earnings × 1.05 bonus (5%)
- Customer rate 4: agent earnings × 1.00 standard
- Customer rate 3: agent earnings × 0.95
- Customer rate 1-2: agent earnings × 0.90 + auto-trigger dispute

### BR-STAGE-006 · Auto-cancel timers
- Stage 01: nếu không pay trong 30 phút → auto cancel
- Stage 05: nếu customer không duyệt báo giá trong 48h → auto cancel + refund 50% phí khảo sát

---

## F · Material Rules

### BR-MAT-001 · Stock check timing
- Trong quote (stage 05): real-time check wms
- Khi customer duyệt (stage 06): **reserve** trong wms (TTL 24h)
- Khi step start (stage 08): không cần check lại (đã reserved)
- Khi step done (stage 08 từng step): **commit out** trong wms (trừ stock thật)

### BR-MAT-002 · Reservation TTL
- Default 24h
- Auto-release nếu order cancelled hoặc stage không tiến trong 24h
- Có thể extend bởi ops admin (nếu khách báo lùi lịch)

### BR-MAT-003 · Free-form material
- Allowed khi agent không tìm được variant phù hợp trong catalog
- Required fields: name, quantity, unit, unit_price
- Tự động `verify_status = 'pending_verify'`
- Đơn vẫn complete được, nhưng task verify hiện cho ops
- Ops 3 lựa chọn: Approve off-catalog | Reject + agent compensate | Request clarification

### BR-MAT-004 · BOM variance detection
Sau khi step done, compare actual vs expected BOM:
- Variance ≤ 5%: auto-verified
- Variance 5-15%: flag for review (informational)
- Variance > 15%: blocked until ops approve

### BR-MAT-005 · Material from personal warehouse
Agent có thể chỉ định dùng material từ kho cá nhân (warehouse_type = 'personal'):
- Stock check: chỉ check warehouse cá nhân
- Commit: trừ warehouse cá nhân, không trừ kho chung
- Agent earnings: get full material margin (vì agent đã invest vào kho)

---

## G · Partner Platform Rules (v3.3)

### BR-PTN-001 · Partner types
- `business`: có GPKD, MST, rep_name (đại diện pháp luật) — invoice xuất tên công ty
- `individual_large`: cá nhân lớn, không GPKD — invoice xuất tên cá nhân

### BR-PTN-002 · Partner roles
1 partner có thể có 1 hoặc cả 2 roles:
- `customer`: đặt đơn (Type A)
- `supplier`: cung cấp đội thợ (Type B)
- Cả hai (Type AB): vd Hùng AC Service có đội thợ + còn đặt đơn outsource khi quá tải

### BR-PTN-003 · Supplier models
Partner supplier có 3 model trả phí:

| Model | SaaS fee | Commission | Thợ ăn |
|---|---|---|---|
| `commission_only` | 0 | 6% | 94% |
| `saas_only` | 5M/tháng | 0 | 100% |
| `hybrid` | 2M/tháng | 3% | 97% |

### BR-PTN-004 · Payment modes (customer-side)
- `prepaid_wallet`: nạp ví trước, trừ khi tạo đơn
- `monthly_invoice`: ghi nợ, NET 30 (default), xuất hoá đơn cuối tháng

### BR-PTN-005 · Payout modes (supplier-side)
- `passthrough`: SMP trả tiền công trực tiếp vào TK thợ
- `via_partner`: SMP trả tổng vào TK partner, partner tự trả thợ off-platform

### BR-PTN-006 · Partner user roles (RBAC)
| Role | Số người tối đa | Permissions |
|---|---|---|
| `partner_owner` | 1 (founder/CEO) | All within partner |
| `partner_manager` | Unlimited | Operations (orders, agents) |
| `partner_finance` | Unlimited | Finance only (wallet, invoices, reports) |
| `partner_dispatcher` | Unlimited | Operations + dispatch toggle |

Owner có thể demote/promote khác, nhưng không thể demote chính mình (cần SMP ops help).

### BR-PTN-007 · Wallet alert thresholds
- Balance < 7 ngày trung bình spending → alert email + in-app
- Balance < 3 ngày → block tạo đơn mới
- Negative balance: KHÔNG cho phép (atomic check)

### BR-PTN-008 · Partner-specific dispatch
Nếu partner có ≥ 5 active agents AND `customer_config.default_dispatch_visibility = 'private'` → default đặt đơn là private dispatch.

---

## H · Integration Rules (v3.2)

### BR-INT-001 · Customer source of truth
- `inside` là source of truth cho customer profile + payment methods + vouchers
- SMP **không** store customer name/phone permanent, chỉ cache 30s
- Snapshot per-order: name + address tại thời điểm tạo (lưu trong smp_order.orders)

### BR-INT-002 · Webhook signature
Mọi inbound webhook phải có HMAC SHA-256 signature trong header `X-Signature`:
```text
signature = HMAC-SHA256(webhook_secret, raw_body)
```

Outbound webhook (SMP → partner): tương tự, secret từ `partner.webhook_secret`.

### BR-INT-003 · Webhook retry
- Outbound: exponential backoff 1s, 10s, 60s, 5min, 30min
- Sau 5 lần fail → DLQ + alert ops
- Manual replay từ admin UI

### BR-INT-004 · Idempotency
- Mọi POST endpoint hỗ trợ `Idempotency-Key` header
- Server cache response 24h theo key
- Same key + same body → return cached response (idempotent)
- Same key + different body → 409 conflict

### BR-INT-005 · Circuit breaker
Per dependency (inside, wms):
- Trip after 5 consecutive failures OR error rate > 50% over 1 min window
- Half-open after 30s: try 1 request
- Close if success, else remain open

---

## I · Quality Rules

### BR-QUAL-001 · Auto-dispute trigger
Auto-create dispute task khi:
- Customer rate ≤ 2 sao
- Customer comment chứa keywords: "lừa", "không xong", "tệ" (config trong settings)
- Photo proof < 50% required count
- Stage 08 duration > 3× expected step duration

### BR-QUAL-002 · Dispute SLA
- Ops respond first contact: 4h
- Resolution target: 48h
- Escalate to manager: > 48h
- Customer satisfaction survey: sent 24h sau resolution

### BR-QUAL-003 · Agent suspension triggers
Auto-suspend nếu trong 30 ngày:
- avg_rating < 3.0 (min 5 đơn)
- no_show ≥ 3 lần
- 2 disputes resolved against agent
- Photo proof fraud detected (lặp ảnh, ảnh không khớp)

Suspend = `status = 'suspended'`, không thể nhận đơn cho đến khi ops review unsuspend.

### BR-QUAL-004 · Partner suspension triggers
Auto-suspend partner nếu:
- Outstanding invoice overdue > 60 ngày
- Multiple compliance violations (KYC docs giả, fraud reports)

---

## J · Notification Rules

### BR-NOTIF-001 · Customer notification channels
| Event | Push | SMS | Email |
|---|---|---|---|
| Order created | ✓ | | |
| Agent assigned | ✓ | | |
| Agent on the way | ✓ | | |
| Quote ready | ✓ | ✓ | |
| Order completed | ✓ | | ✓ (with invoice) |
| Refund processed | ✓ | | ✓ |

### BR-NOTIF-002 · Agent notification channels
| Event | Push | SMS |
|---|---|---|
| New order invitation | ✓ | |
| Order assigned | ✓ | |
| Customer message | ✓ | |
| Earnings paid | ✓ | ✓ |

### BR-NOTIF-003 · Partner notification channels
| Event | Push | Email |
|---|---|---|
| Order completed | | ✓ (daily digest, not per order) |
| Wallet low | ✓ | ✓ |
| Invoice generated | | ✓ |
| KYC status change | | ✓ |
| Agent KYC pending review | | ✓ |

### BR-NOTIF-004 · Quiet hours
- Customer: không gửi push 22:00 - 07:00 (trừ urgent: agent on the way, payment confirmation)
- Agent: không gửi push 22:00 - 06:00 (trừ scheduled order < 8h tới)

### BR-NOTIF-005 · Frequency cap
- Customer: max 3 push/day (chưa kể order updates)
- Agent: no cap on order invitations (revenue critical)

---

## K · Data Retention Rules

### BR-RET-001 · Order data
- Active + completed orders: keep indefinitely in MySQL (until contractual obligation expires)
- Order > 5 năm: archive to S3 (read-only), drop from MySQL
- Final delete only after 7 năm + legal review

### BR-RET-002 · Audit log
- Hot storage 90 ngày
- Warm storage 2 năm
- Cold (S3 Glacier) 7 năm
- Financial audit entries: 10 năm (tax compliance)

### BR-RET-003 · Photos
- Photo proof: 2 năm post-completion, then archive
- Customer-uploaded: customer can delete via account settings (deletion + 30d grace)
- KYC photos: 5 năm post-termination

### BR-RET-004 · Event log
- TTL 90 ngày (MongoDB TTL index)
- Đủ cho debug + recent analytics

### BR-RET-005 · Session/tokens
- Access token: 8h
- Refresh token: 30 ngày
- Session in Redis: TTL = token expiry

---

## L · Reconciliation & Fraud Detection Rules (v4.0)

> Bộ rules cho automated reconciliation (đối soát) và fraud detection. Daily jobs check inconsistency + flag suspicious patterns.

### BR-RECON-001 · Daily wallet reconciliation
- **Schedule**: Daily 02:00 UTC (sau khi end-of-day payment gateway closes)
- **Source A**: SMP `partner_wallet_transactions` cumulative balance per partner
- **Source B**: Payment gateway transaction export (VNPay/MoMo/ZaloPay) per partner_ref
- **Acceptable diff**: < 0.01% OR < 100k VND, whichever lower
- **Action if exceed**: Alert Finance + create reconciliation ticket
- **Owner**: Finance team reviews mỗi sáng

### BR-RECON-002 · Daily KYC status reconciliation
- **Schedule**: Daily 03:00 UTC
- **Check**: Partner/Agent với KYC docs uploaded > 7 ngày nhưng status vẫn `pending_kyc`
- **Action**: Page Ops Admin queue · max review time 48h SLA
- **Escalation**: nếu > 14 ngày, escalate Operations Manager

### BR-RECON-003 · Daily order-payment reconciliation
- **Schedule**: Daily 02:30 UTC
- **Check**:
  - Orders `status=completed` nhưng `payment_status=pending` > 24h → flag
  - Payments `status=succeeded` nhưng không có order linked → flag
  - Refunds requested nhưng không có payment gateway entry → flag
- **Threshold alert**: > 10 outstanding items
- **Owner**: Finance + Engineering joint review

### BR-RECON-004 · Monthly currency conversion audit (v4.0)
- **Schedule**: First day of month, 04:00 UTC
- **Check**: Every multi-currency transaction trong tháng trước có rate snapshot đúng (rate ngày txn vs rate stored)
- **Tolerance**: < 0.001% diff (currency conversion precision)
- **Output**: Monthly report email to Finance + CFO

### BR-FRAUD-001 · Wallet topup velocity check
- **Pattern**: > 5 topups trong 10 phút từ cùng partner
- **Severity**: Warning
- **Action**: Soft-flag, require email verification on 6th topup. No block.

### BR-FRAUD-002 · Geographic anomaly
- **Pattern**: Wallet topup từ IP country khác với partner registered country
- **Severity**: High
- **Action**: Block transaction · require manual review (Finance + Security)
- **Exception**: Partner mark "travel" flag trước 24h (Marketing white-list)

### BR-FRAUD-003 · Unusual amount
- **Pattern**: Single topup > 10x partner's running average + first time at this scale
- **Severity**: High
- **Action**: Hold txn for manual review (max 2h business hours)

### BR-FRAUD-004 · AML threshold (Anti-Money Laundering)
- **Pattern (VN)**: Single txn ≥ 300,000,000 VND OR cumulative ≥ 500,000,000 VND/day from 1 partner
- **Action**: Hold + report to SBV (State Bank of Vietnam) per Decree 03/2022/NĐ-CP
- **Audit log**: Required permanent record
- **Owner**: Compliance team handles SBV filing

### BR-FRAUD-005 · Multi-account device fingerprint
- **Pattern**: Same device fingerprint creates > 3 customer/agent accounts trong 24h
- **Severity**: Medium
- **Action**: Flag accounts for review · could be legitimate (shop owner) or fraud ring

### BR-FRAUD-006 · Refund abuse
- **Pattern**: Customer/Partner requests > 5 refunds trong tháng OR refund rate > 30%
- **Severity**: Medium
- **Action**: Review pattern, possible legitimate complaints vs fraud ring
- **Threshold for action**: > 8 refunds/month → temporary suspend pending investigation

### BR-RECON-005 · Cross-region data consistency (v4.0)
- **Schedule**: Every 6 hours
- **Check**: Master data (countries, currencies, currency_rates, tax_configs) consistent across all clusters (smp-asia, smp-china, smp-us)
- **Tolerance**: ZERO drift cho master data
- **Action nếu drift**: Auto-trigger MirrorMaker re-sync + alert DevOps
- **Owner**: DevOps + Data Engineering

### BR-RECON-006 · Audit log integrity check
- **Schedule**: Daily 04:30 UTC
- **Check**: Audit log có gap (missing audit_id sequence) hoặc hash chain broken (xem [Doc 12 · Audit Log section 10](../08-security/12-audit-log-spec.md) cho hash chain spec)
- **Severity**: SEV-1 (potential tamper)
- **Action**: Immediate page Security team + CTO

---

## Quy trình thay đổi business rule

1. Đề xuất change → tạo proposal doc (impact: customers, partners, agents, finance)
2. PM + Founder approve
3. Dev đánh giá technical effort
4. QC viết test cases cho new rule
5. Implement + deploy to staging
6. UAT với select users
7. Communicate to affected stakeholders 7 ngày trước GA
8. Deploy prod + monitor
9. Update doc này + relevant other docs

> **Quan trọng**: Rule nào active trong production → MUST đồng bộ với doc này. Nếu khác → bug hoặc doc outdated, phải fix.

---

## Appendix · Full `rules_engine.yaml` sample (v4.0 preview)

> Phần này là **preview** cho v4.0 rules engine. Convert sample chỉ cho 2 categories Dispatch + Pricing. Các categories còn lại (Payment, KYC, Stage, Material, Partner, Integration, Quality, Notification, Retention) sẽ convert tương tự khi v3.6.

### Cấu trúc file đầy đủ

```yaml
# rules_engine.yaml
# Single source of truth cho 80+ business rules
# Deploy: kubectl apply -f rules-configmap.yaml
# Audience: Edit by BA team, review by Tech Lead, audited via Git

version: "4.0.0"
last_updated: "2026-11-01T08:00:00Z"
maintainers:
  - "ba-team@smp.vn"
  - "tech-lead@smp.vn"

# ===========================================================================
# Context schema · Biến available trong expressions
# ===========================================================================
context_schema:
  country_code:   string    # 'VN', 'US', 'CN', 'TH', 'ID', 'SG', 'MY', 'PH'
  env:            string    # 'dev', 'staging', 'prod'
  now:            timestamp # UTC timestamp
  order:
    id:               string
    source:           string  # 'customer_direct', 'partner_customer', 'contract'
    partner_id:       string|null
    current_stage:    string
    country_code:     string
    currency:         string
    subtotal_amount:  int64   # minor units
    total_amount:     int64
  agent:
    id:                  int64
    level:               string  # 'junior', 'mid', 'senior'
    rating:              float
    completed_orders:    int
    skills:              array<string>
    home_district:       string
    home_city:           string
    is_online:           bool
    status:              string
  customer:
    id:                int64
    is_vip:            bool
    completed_orders:  int
  partner:
    id:                int64|null
    type:              string|null  # 'A', 'B', 'AB'
    payment_mode:      string|null  # 'prepaid_wallet', 'post_invoice'
    wallet_balance:    int64|null
    payout_mode:       string|null  # 'direct', 'via_partner'

# ===========================================================================
# A · DISPATCH RULES
# ===========================================================================
rules:

  - id: vn.dispatch.001
    name: "Round timeout · Vietnam"
    category: dispatch
    description: "Mỗi dispatch round có timeout 60s production, 30s dev/staging."
    enabled: true
    priority: 100
    when: "context.country_code == 'VN'"
    then:
      round_timeout_seconds: 60
    overrides:
      - when: "context.env in ['dev', 'staging']"
        then:
          round_timeout_seconds: 30
    legacy_id: "BR-DISP-001"
    changelog:
      - { version: "3.0", date: "2026-01-15", author: "BA team", change: "Initial" }

  - id: us.dispatch.001
    name: "Round timeout · US"
    category: dispatch
    description: "US thợ rải rác, cần 90s. Test nội bộ Q3 2026."
    enabled: false  # not yet launched
    priority: 100
    when: "context.country_code == 'US'"
    then:
      round_timeout_seconds: 90

  - id: vn.dispatch.002
    name: "Max rounds"
    category: dispatch
    enabled: true
    priority: 100
    when: "context.country_code == 'VN'"
    then:
      max_rounds: 3
      escalate_action: "ops_manual_dispatch"
    legacy_id: "BR-DISP-002"

  - id: vn.dispatch.003
    name: "Round radius expansion"
    category: dispatch
    description: "Round 1: same district. Round 2: +5km. Round 3: full city."
    enabled: true
    priority: 100
    when: "context.country_code == 'VN'"
    then:
      radius_per_round:
        - { round: 1, scope: "same_district" }
        - { round: 2, scope: "within_km", value: 5 }
        - { round: 3, scope: "same_city", include_offline: true }
    legacy_id: "BR-DISP-003"

  - id: "*.dispatch.004"
    name: "Qualified agent definition"
    category: dispatch
    description: "Áp dụng tất cả country. Filter agents qualified."
    enabled: true
    priority: 100
    when: "true"  # always
    then:
      qualified_filters:
        - "agent.status == 'active'"
        - "agent.is_online == true"
        - "agent.rating >= 3.5"
        - "agent.kyc_level in ['basic', 'advanced', 'premium']"
    legacy_id: "BR-DISP-004"

  - id: vn.dispatch.005
    name: "Private dispatch · partner-only agents"
    category: dispatch
    description: "Order partner-private chỉ gửi thợ thuộc partner đó."
    enabled: true
    priority: 200  # higher priority, overrides default
    when: "context.order.dispatch_visibility == 'private' && context.order.partner_id != null"
    then:
      filter_extra: "agent.partner_id == context.order.partner_id"
      fallback_action: "escalate_to_ops"  # nếu không có agent partner online
    legacy_id: "BR-DISP-005"

  - id: vn.dispatch.007
    name: "Surge multiplier per round"
    category: dispatch
    description: "Round 1 = 1.0x, round 2 = 1.2x, round 3 = 1.5x. Surge chỉ áp dụng labor."
    enabled: true
    priority: 100
    when: "context.country_code == 'VN'"
    then:
      surge_multipliers:
        round_1: 1.00
        round_2: 1.20
        round_3: 1.50
      applies_to: ["labor"]  # not material
    legacy_id: "BR-DISP-007"

# ===========================================================================
# B · PRICING RULES
# ===========================================================================

  - id: vn.pricing.001
    name: "Labor price multiplier by agent level"
    category: pricing
    description: "Junior 0.8x, Mid 1.0x, Senior 1.3x base price."
    enabled: true
    priority: 100
    when: "context.country_code == 'VN'"
    then:
      level_multipliers:
        junior: 0.8
        mid: 1.0
        senior: 1.3
    legacy_id: "BR-PRICE-001"

  - id: vn.pricing.002
    name: "Material sell price"
    category: pricing
    description: "Sell price từ catalog. SMP controlled, không tự ý discount."
    enabled: true
    priority: 100
    when: "context.country_code == 'VN'"
    then:
      source: "catalog.material_variants.sell_price_amount"
    legacy_id: "BR-PRICE-002"

  - id: vn.pricing.003
    name: "Quote subtotal calculation"
    category: pricing
    enabled: true
    priority: 100
    when: "context.country_code == 'VN'"
    then:
      formula: "labor_total + material_total + survey_fee - discount"
    legacy_id: "BR-PRICE-003"

  - id: vn.pricing.005
    name: "Voucher discount · pre-VAT"
    category: pricing
    description: "Voucher trừ vào subtotal trước khi tính VAT."
    enabled: true
    priority: 100
    when: "context.country_code == 'VN' && context.order.voucher_code != null"
    then:
      apply_order: ["subtotal", "discount", "vat", "total"]
      discount_cap_percent: 50  # max 50% subtotal
    legacy_id: "BR-PRICE-005"

  - id: vn.pricing.006
    name: "Surge applies to labor only"
    category: pricing
    description: "Multiplier surge chỉ nhân vào labor, không nhân material/fee."
    enabled: true
    priority: 100
    when: "context.country_code == 'VN'"
    then:
      surge_target: "labor"
    legacy_id: "BR-PRICE-006"

  - id: vn.pricing.007
    name: "VAT Vietnam"
    category: pricing
    enabled: true
    priority: 100
    when: "context.country_code == 'VN'"
    then:
      vat_rate: 0.10
      vat_name: "VAT"
      lookup_table: "tax_configs"  # actual rate fetched runtime để support changes
    legacy_id: "BR-PRICE-008"

  - id: sg.pricing.007
    name: "GST Singapore"
    category: pricing
    enabled: false  # not yet launched
    priority: 100
    when: "context.country_code == 'SG'"
    then:
      vat_rate: 0.09
      vat_name: "GST"

  - id: us.pricing.007
    name: "Sales Tax US · jurisdiction-dependent"
    category: pricing
    enabled: false
    priority: 100
    when: "context.country_code == 'US'"
    then:
      vat_rate_source: "tax_configs"  # query by state/county/city
      vat_name: "Sales Tax"

  - id: vn.pricing.009
    name: "Rounding rules · Vietnam"
    category: pricing
    description: "VND không decimal. Round to nearest 1000 VND cho display."
    enabled: true
    priority: 100
    when: "context.country_code == 'VN'"
    then:
      storage_minor_units: 1     # 1đ = 1 minor unit (no decimal)
      display_rounding: 1000     # round 47,523 → 47,500
      display_rounding_mode: "down"
    legacy_id: "BR-PRICE-009"

  - id: us.pricing.009
    name: "Rounding rules · US"
    category: pricing
    enabled: false
    priority: 100
    when: "context.country_code == 'US'"
    then:
      storage_minor_units: 100   # 1 USD = 100 cents
      display_rounding: 1        # cent precision
      display_rounding_mode: "nearest"

  - id: vn.pricing.010
    name: "Partner pricing override"
    category: pricing
    description: "Partner Type B có thể override labor price (contract negotiated)."
    enabled: true
    priority: 200
    when: |
      context.partner != null &&
      context.partner.type in ['B', 'AB'] &&
      context.partner.has_pricing_override
    then:
      use_partner_pricing: true
      source: "partner.pricing_config"
    legacy_id: "BR-PRICE-007"
```

### Đọc + apply rules trong code Go

```go
// dispatch-engine/internal/dispatch/round.go
package dispatch

import (
    "context"
    "smp/pkg/rules"
)

type RoundController struct {
    engine *rules.Engine
    log    *zap.Logger
}

func (r *RoundController) StartRound(ctx context.Context, order Order) error {
    // Build evaluation context
    ruleCtx := rules.Context{
        "country_code": order.CountryCode,
        "env":          os.Getenv("ENV"),
        "now":          time.Now().UTC(),
        "order":        toRuleMap(order),
    }

    // Eval dispatch rules
    decision := r.engine.Evaluate("dispatch", ruleCtx)

    timeout := decision.GetInt("round_timeout_seconds", 60)  // default 60
    maxRounds := decision.GetInt("max_rounds", 3)

    // Use values từ rules engine
    timer := time.NewTimer(time.Duration(timeout) * time.Second)
    // ...
    return nil
}
```

### Testing rules

```yaml
# tests/rules/dispatch_test.yaml
test_cases:
  - name: "VN production · 60s timeout"
    context:
      country_code: "VN"
      env: "prod"
    expect:
      round_timeout_seconds: 60
      max_rounds: 3

  - name: "VN dev · 30s timeout (override)"
    context:
      country_code: "VN"
      env: "dev"
    expect:
      round_timeout_seconds: 30

  - name: "Private dispatch with partner agents"
    context:
      country_code: "VN"
      env: "prod"
      order:
        dispatch_visibility: "private"
        partner_id: "P001"
    expect:
      filter_extra: "agent.partner_id == 'P001'"
```

Run với:
```bash
go test ./pkg/rules -fixtures=tests/rules/
```

