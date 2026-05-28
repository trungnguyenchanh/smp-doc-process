# SMP User Stories + Acceptance Criteria

**Format**: Story + Gherkin · **Audience**: BA, Dev, QC · **Updated**: v3.3

---

## Cách viết user story

Mỗi story dùng template:
```
Là <role>, tôi muốn <action> để <benefit>.
```

Mỗi story có acceptance criteria viết bằng Gherkin (Given/When/Then) — QC viết test case directly từ đây.

---

## Epic 1 · Customer order journey

### US-1.1 · Customer đặt đơn dịch vụ
**Là** Customer, **tôi muốn** đặt đơn sửa AC qua app, **để** không phải gọi điện và biết được giá ước tính.

**Acceptance criteria:**

```gherkin
Feature: Customer create order

Scenario: Tạo đơn thành công với address mặc định
  Given tôi đã login với account "Nguyễn Văn A"
    And tôi có địa chỉ mặc định "15 Lê Lợi, Q.1"
  When tôi chọn service "Sửa điều hoà"
    And tôi confirm địa chỉ mặc định
    And tôi tap "Đặt ngay"
  Then đơn được tạo với order_id dạng "ord_..."
    And current_stage = "01_created"
    And dispatch tự động trigger trong 5 giây
    And tôi nhận push notification "Đang tìm thợ cho bạn"

Scenario: Lỗi không có thợ qualified trong zone
  Given tôi ở khu vực "Côn Đảo" không có thợ
  When tôi đặt đơn
  Then hệ thống trả về 422 với error "no_agent_available"
    And UI hiển thị "Hiện chưa có thợ ở khu vực này, mời thử khu vực khác"

Scenario: Voucher áp dụng đúng
  Given tôi có voucher HOTSUMMER20 (giảm 20%, max 200k)
    And tôi chọn service trị giá 750k
  When tôi nhập mã HOTSUMMER20
  Then discount = 150,000đ (20% × 750k)
    And total_amount = 600,000đ + VAT 60k = 660,000đ
```

### US-1.2 · Customer xem tracking 2 thợ
**Là** Customer, **tôi muốn** thấy được thợ khảo sát và thợ thực hiện đang ở đâu trong quy trình, **để** biết khi nào họ tới.

**Acceptance criteria:**

```gherkin
Scenario: Hiển thị Survey Agent vào stage 3
  Given đơn của tôi đang stage "03_survey_accepted"
    And Survey Agent là "Trương Minh K."
  When tôi mở màn tracking
  Then thấy section "Thợ khảo sát" với:
    - Tên "Trương Minh K."
    - Avatar
    - Pin trên map đang di chuyển
    - ETA "10 phút"
    - Badge SUR màu violet

Scenario: Stage 7 chuyển sang hiển thị Execution Agent
  Given đơn đang stage "07_dispatched_execution"
    And Execution Agent là "Phạm Quốc M."
  When tôi mở tracking
  Then thấy section "Thợ sửa chữa" với badge EXC màu teal
    And section "Thợ khảo sát" thu gọn (hoàn thành)
```

### US-1.3 · Customer duyệt báo giá
**Là** Customer, **tôi muốn** xem báo giá chi tiết từ thợ khảo sát và quyết định duyệt/từ chối, **để** kiểm soát chi phí.

```gherkin
Scenario: Báo giá có labor + materials
  Given Survey Agent đã hoàn thành khảo sát
    And đề xuất gồm:
      - Step "Thay tụ điện" · labor 100k + Sanyo CBB60-35 80k
      - Step "Nạp gas R32" · labor 120k + SubZero R32-800 400k
  When tôi mở màn báo giá
  Then thấy breakdown:
    - Labor: 220,000đ
    - Material: 480,000đ
    - Phí khảo sát: 80,000đ (đã trả)
    - VAT 10%: 70,000đ
    - Total: 770,000đ
    And có 2 button "Duyệt" và "Từ chối"

Scenario: Duyệt báo giá trigger dispatch Execution Agent
  Given tôi đang xem báo giá
  When tôi tap "Duyệt"
    And confirm payment method
  Then đơn chuyển sang stage "06_quote_approved"
    And dispatch Execution Agent bắt đầu
    And wms reserve materials đã chọn

Scenario: Từ chối báo giá cancel đơn
  When tôi tap "Từ chối"
    And nhập lý do
  Then đơn chuyển sang stage "cancelled"
    And tôi được refund phí khảo sát 50% (theo policy)
    And wms release any pending reservations
```

---

## Epic 2 · Technician work flow

### US-2.1 · Thợ nhận đơn từ dispatch
**Là** Technician, **tôi muốn** thấy đơn mới qua push notification và accept nhanh, **để** không bỏ lỡ cơ hội kiếm tiền.

```gherkin
Scenario: Push notification với 60s window
  Given tôi online và qualified cho đơn ord_X
  When dispatch invite tôi
  Then tôi nhận push trong 2 giây
    And mở app thấy bottom sheet "Đơn mới · 60s"
    And countdown timer hiển thị
    And có 2 button "Nhận" / "Bỏ qua"

Scenario: Accept thành công khi tôi là người đầu
  When tôi tap "Nhận" trong 60s
    And tôi là thợ đầu accept
  Then đơn assigned cho tôi
    And UI chuyển sang "Việc của tôi"
    And các thợ khác nhận thông báo "Đơn đã có thợ khác nhận"

Scenario: Accept nhưng đã có thợ khác accept trước
  When tôi tap "Nhận"
    And server detect tôi accept thứ 2
  Then API trả về 409 conflict
    And UI hiển thị "Đơn đã được thợ khác nhận, tiếp tục tìm việc"
```

### US-2.2 · Thợ Survey nhập báo giá với material picker
**Là** Survey Agent, **tôi muốn** chọn vật tư từ danh sách wms với stock real-time, **để** tránh hứa khách rồi không có vật tư.

```gherkin
Scenario: Material picker hiện variants có stock
  Given step "ac-replace-capacitor" cần material_type "mtyp_capacitor_35uf"
    And catalog có 3 variants: Sanyo (stock 3 cá nhân), Panasonic (stock 12 chung), TQ (hết)
  When tôi tap "Chọn vật tư" cho step này
  Then list hiển thị:
    - "Sanyo CBB60-35 · 80k · Kho cá nhân: 3 cái" (highlight, default selected)
    - "Panasonic AT-35F · 95k · Kho chung HCMC: 12 cái"
    - "TQ no-brand GEN-35 · 45k · Hết hàng" (disabled)

Scenario: Chọn variant xong cập nhật báo giá
  When tôi chọn "Panasonic AT-35F"
    And tap "Xác nhận"
  Then báo giá update:
    - Labor 100k + Material 95k = 195k cho step này

Scenario: Free-form material cho tình huống đặc biệt
  When tôi tap "Vật tư khác"
    And nhập tên "Dầu lạnh POE 100ml"
    And nhập qty 0.1 lit
    And nhập giá bán 80k
  Then thêm dòng material với verify_status = "pending_verify"
    And ops admin nhận task verify sau khi step done
```

### US-2.3 · Thợ Execution upload photo proof
**Là** Execution Agent, **tôi muốn** chụp ảnh trước/sau để chứng minh đã làm, **để** tránh tranh chấp với khách.

```gherkin
Scenario: Bắt buộc photo proof trước khi finish step
  Given tôi đang ở step "08_in_progress"
    And step yêu cầu photo proof
  When tôi tap "Hoàn thành step"
    And tôi chưa upload photo
  Then UI hiển thị error "Cần upload ít nhất 2 ảnh (before + after)"
    And button "Hoàn thành" disabled

Scenario: Upload ảnh thành công
  When tôi tap "Chụp ảnh trước"
    And chụp 1 ảnh
    And tap "Chụp ảnh sau"
    And chụp 1 ảnh
  Then 2 ảnh upload lên storage (S3/R2)
    And metadata lưu: timestamp, GPS, step_id, agent_id
    And button "Hoàn thành" enabled
```

---

## Epic 3 · Partner platform (v3.3)

### US-3.1 · Partner Owner đặt đơn cho khách cuối
**Là** Partner Owner (vd Hùng AC Service), **tôi muốn** đặt đơn nhân danh khách của tôi với dispatch chỉ trong đội thợ của tôi, **để** kiểm soát chất lượng.

```gherkin
Scenario: Tạo đơn private dispatch
  Given tôi login với role partner_owner cho "Hùng AC Service"
    And partner có 8 agents qualified
    And wallet balance = 8,500,000đ
  When tôi tạo đơn cho khách "Nguyễn Văn A"
    And chọn dispatch_visibility = "private"
    And tổng tiền dự kiến 750k
  Then đơn được tạo với source = "partner_customer"
    And dispatch chỉ gửi cho 8 agents của partner
    And wallet trừ 750k (balance = 7,750,000đ)

Scenario: Không đủ wallet balance
  Given wallet balance = 500,000đ
  When tôi tạo đơn 750k
  Then API trả về 402 insufficient_balance
    And UI hiển thị "Cần nạp thêm 250,000đ vào ví"
    And button "Nạp tiền ngay"

Scenario: Open dispatch fallback
  When tôi tạo đơn với dispatch_visibility = "open"
  Then dispatch round 1-2 gửi cho 8 agents của partner
    And nếu không ai accept sau 2 rounds → mở rộng cho mọi agents qualified
```

### US-3.2 · Partner Finance xem báo cáo
**Là** Partner Finance, **tôi muốn** xem báo cáo wallet transactions và payout pending, **để** đối chiếu sổ sách.

```gherkin
Scenario: Xem wallet transactions của tháng
  Given tôi login partner_finance
  When tôi mở "Tài chính" → "Wallet transactions"
    And filter month = "2026-05"
  Then thấy list 87 giao dịch (debit/credit/topup)
    And mỗi row có: time, amount, order_ref, balance_after
    And total debit = 12,800,000đ
    And total topup = 5,000,000đ

Scenario: Export CSV
  When tôi tap "Export CSV"
  Then file download với columns chuẩn:
    txn_id, created_at, type, amount, order_id, balance_after, notes
    And audit log entry created
```

### US-3.3 · Partner Dispatcher tắt agent tạm thời
**Là** Partner Dispatcher, **tôi muốn** toggle on/off agent khi họ nghỉ ốm/phép, **để** không nhận đơn họ không thể làm.

```gherkin
Scenario: Toggle off agent
  Given tôi có 8 agents
    And agent "Phạm Quốc M." đang online
  When tôi tap toggle off agent này
  Then agent.is_online = false ngay lập tức
    And không nhận đơn mới
    And đơn đang xử lý của agent này không bị ảnh hưởng

Scenario: Toggle on agent đang KYC pending
  Given agent "Phan Nam" có status = pending_kyc
  When tôi tap toggle on
  Then toggle disabled (không thực hiện được)
    And tooltip "Cần SMP duyệt KYC trước"
```

### US-3.4 · Ops admin onboard partner mới
**Là** Ops Admin, **tôi muốn** review KYC docs và approve partner mới, **để** đảm bảo chất lượng đối tác.

```gherkin
Scenario: Approve partner basic KYC
  Given partner "FreshAir Group" status = "pending_kyc"
    And đã upload GPKD + CCCD + STK
  When tôi mở partner detail
    And click "Approve KYC basic"
  Then partner status = "active"
    And kyc_level = "basic"
    And partner_owner nhận email + SMS thông báo
    And audit log entry

Scenario: Reject KYC vì doc không rõ
  When tôi review GPKD và thấy mờ
    And click "Request re-upload"
    And ghi note "GPKD scan không rõ, vui lòng chụp lại"
  Then partner nhận notification với note
    And status giữ nguyên "pending_kyc"
```

---

## Epic 4 · Material BOM (v3.2)

### US-4.1 · Ops admin tạo Material Variant mới
**Là** Ops Admin, **tôi muốn** thêm SKU mới vào catalog với giá bán, **để** thợ có nhiều lựa chọn cho khách.

```gherkin
Scenario: Tạo variant Sanyo CBB60-35 mới
  Given material_type "Tụ điện 35μF" đã tồn tại
  When tôi tap "Tạo variant" và nhập:
    - Brand: Sanyo
    - Model: CBB60-35
    - wms_sku: SKU-CAP-SAN-35
    - sell_price: 80,000đ
    - warranty_months: 6
  Then variant tạo thành công
    And system pull cost_price từ wms = 50,000đ
    And margin tự động tính = +60%

Scenario: wms SKU đã tồn tại (duplicate)
  When tôi nhập wms_sku đã có
  Then API trả về 409 conflict
    And error "SKU này đã được dùng cho variant khác"
```

### US-4.2 · Ops verify free-form material
**Là** Ops Admin, **tôi muốn** review vật tư thợ nhập free-form, **để** đảm bảo không gian lận.

```gherkin
Scenario: Review free-form material
  Given thợ nhập "Bơm xả Daikin TQ" qty 1, giá 280k
    And ghi chú "Khách yêu cầu dùng bơm TQ giá rẻ"
  When tôi mở material_verify page
    And click row này
  Then thấy panel "Expected (BOM)" và "Actual (thợ nhập)"
    And có 3 button: "Request clarification", "Reject + refund", "Approve off-catalog"

Scenario: Approve off-catalog
  When tôi click "Approve off-catalog"
  Then material entry status = "verified"
    And đơn này không bị flag
    And audit log entry
```

---

## Epic 5 · Integration với inside/wms (v3.2)

### US-5.1 · System sync customer data từ inside
**Là** System, **tôi muốn** lấy customer data real-time từ inside, **để** không cần đồng bộ DB duplicate.

```gherkin
Scenario: Get customer info success
  Given SMP có customer_id = "cus_01HX5K"
  When SMP gọi GET inside.local/api/v1/customers/cus_01HX5K
  Then nhận response với name, phone, addresses, tier
    And cache trong Redis 30 giây

Scenario: inside down → degraded mode
  Given inside service không response (timeout > 3s)
  When SMP cần customer data
  Then circuit breaker open
    And UI hiển thị "Đang cập nhật" thay vì tên khách
    And alert ops team
```

### US-5.2 · Webhook payment.succeeded từ inside
**Là** System, **tôi muốn** nhận webhook payment.succeeded để update order status.

```gherkin
Scenario: Valid webhook
  When inside gửi POST /integrations/inside/payment-webhook
    With body: {"event":"payment.succeeded","intent_id":"pi_X","amount_paid":750000,...}
    And HMAC signature valid
  Then SMP tìm order theo metadata.smp_order_id
    And update payment_status = "paid"
    And trigger dispatch survey agent
    And response 200 OK

Scenario: Invalid HMAC signature
  When webhook đến với signature wrong
  Then response 401 unauthorized
    And log security event
    And NOT update order
```

---

## Epic 6 · Quality & rating

### US-6.1 · Customer đánh giá thợ
**Là** Customer, **tôi muốn** đánh giá thợ 1-5 sao và viết review, **để** giúp các khách khác.

```gherkin
Scenario: Rate sau khi đơn hoàn thành
  Given đơn "ord_X" đang stage "09_completed"
  When tôi mở app sau khi thợ rời đi
  Then tự động hiển thị màn rating
    And bắt buộc chọn 1-5 sao
    And textarea review optional

Scenario: Submit rating
  When tôi chọn 5 sao
    And viết "Thợ K. làm rất nhanh và sạch"
    And tap submit
  Then đơn chuyển sang "10_rated"
    And agent.avg_rating recalculate
    And nếu rating &lt; 3 → tự động tạo dispute task cho ops
```

---

## Definition of Done

Mỗi story coi như done khi:

- [ ] Code implement đúng acceptance criteria
- [ ] Unit tests cover positive + negative scenarios (≥ 70% coverage)
- [ ] Integration test cho happy path
- [ ] API spec OpenAPI updated
- [ ] Database migration up + down
- [ ] Manual test với mock data ở dev
- [ ] PR review by 2 engineers
- [ ] QA acceptance test pass
- [ ] Doc updated nếu liên quan
- [ ] Deployed to staging và smoke test pass

## Story sizing (Fibonacci)

| Points | Description |
|---|---|
| 1 | Trivial · ≤2h · vd thêm field UI |
| 2 | Small · ≤4h · vd thêm 1 endpoint đơn giản |
| 3 | Medium · ≤1 day · vd CRUD 1 resource |
| 5 | Large · ≤3 days · vd flow đa bước có integration |
| 8 | Very large · 1 week · vd new feature complex |
| 13 | Epic-level · cần break down trước khi commit |

Sprint capacity: ~50 story points per team per 2-week sprint (10 dev).
