# Ops Training · Warranty Claim Approval Workflow

> **Audience**: Ops Admin team (Customer Success + Quality)  
> **Duration**: 2 giờ training + 1 giờ shadow + ongoing weekly review  
> **Prerequisites**: Đã hoàn tất Admin Web 101 training  
> **Launch**: Sprint 4 (mid-July 2026) cho pilot 100 customers  
> **Owner**: Ops Manager · QC Lead  
> **Last updated**: 2026-05-29

---

## 0. Tại sao training quan trọng?

Warranty package là **business model mới** của SMP. Khác với pay-per-order trước đây:

- KH đã trả tiền trước (1.2M-1.8M VND)
- Mỗi claim approve = SMP gánh cost (agent payable + materials)
- **Approve sai → SMP lỗ tiền + agent tốn công**
- **Reject sai → KH mất niềm tin + complaints + churn**

→ Ops team là "gatekeeper" giữa KH và quỹ warranty. Cân bằng giữa:
- **Customer satisfaction** (approve nhanh + đúng case)
- **Cost control** (reject các case không cover hoặc fraud)
- **SLA** (5 phút review · BR-WPKG-006)

---

## 1. Knowledge prerequisites (phải đọc trước training)

| Doc | Section | Mục đích |
|---|---|---|
| [Doc 19](../10-legal/19-service-guarantee-policy.md) §9 | Maintenance Subscription Package | Hiểu bản chất gói |
| [Doc 15](../05-ba/15-business-rules.md) Section R | BR-WPKG-001 → 012 | Rules chi tiết |
| [Doc 25](../09-finance/25-warranty-packages-sample-data.md) | 5 packages + covered_issues | Memorize whitelist |
| [Doc 16](../09-finance/16-finance-ledger-spec.md) §3.6 | Accounting impact | Hiểu tại sao approve = cost |
| [HTML mockup](27-warranty-packages-comparison.html) | Marketing material | Hiểu KH expectation |

**Quiz pass-rate required**: 8/10 câu trước khi được approve solo (xem Section 9).

---

## 2. Two-tier approval flow

```text
┌─────────────────────────────────────────────────────┐
│            Customer opens claim                      │
└────────────────┬────────────────────────────────────┘
                 ↓
        ┌────────────────┐
        │ System auto    │
        │ validation     │
        └────────┬───────┘
                 ↓
   ┌─────────────┴──────────────┐
   ↓                            ↓
[CLEANING claim]          [REPAIR claim]
   ↓                            ↓
Auto-approve ✓            Wait Ops review
(no human needed)          (you here!)
   ↓                            ↓
Create free order         Decision: approve OR reject
                              ↓
                       Notify customer + agent
```

**You only see REPAIR claims trong queue**. Cleaning auto-approve.

---

## 3. Admin Web · Warranty Claims Queue

### Access:
- URL: `https://admin.smp.vn/warranty/claims`
- Role required: `ops_admin` (talk to Admin nếu chưa có)

### Queue layout:

```text
┌────────────────────────────────────────────────────────┐
│ Warranty Claims · Pending (5)                          │
├────────────────────────────────────────────────────────┤
│ Filter: [All ▾] [Pending ▾] [Today ▾]   Sort by: SLA ⬆ │
├────────────────────────────────────────────────────────┤
│ ⚠️ WC-001234 · 4 min ago · Anh Nguyễn A · AC capacitor │
│ ⚠️ WC-001233 · 7 min ago · Chị Trần B · Washer belt    │ ← SLA breach soon
│ 🔴 WC-001232 · 12 min ago · Bác Lê D · Fridge gasket   │ ← SLA breached
│ 🟢 WC-001231 · 2 min ago · ...                          │
│ ...                                                     │
└────────────────────────────────────────────────────────┘
```

**Priority sort**: SLA expiring nhất lên đầu. Red ⚠️ > 5 phút.

---

## 4. Decision framework · 6-step checklist

Khi mở 1 claim, đi qua 6 bước theo thứ tự:

### Step 1 · Verify customer + warranty active

✅ Kiểm tra:
- `customer_warranty.status = 'active'` (system tự check)
- `end_date_utc > NOW()` (còn hạn)
- Quota count > 0 cho `repair_basic`

❌ Nếu fail bất kỳ → **REJECT** với reason cụ thể:
- "Gói đã hết hạn ngày X. Vui lòng renew."
- "Đã dùng hết quota sửa trong gói. Đặt order trả phí?"

### Step 2 · Verify issue_category trong whitelist

Mở thẻ `Covered issues` của gói KH đang dùng (sample data Doc 25).

Vd KH gói `wpkg_ac_basic_1y`:

| Whitelist (cover) | Out of scope (reject) |
|---|---|
| capacitor (tụ điện) | compressor (thay block) |
| gas_refill_partial | coil replacement |
| fan_motor | full unit replacement |
| drainage | sửa do force majeure |
| thermostat | máy > 10 năm tuổi |
| remote_control | hư do KH sai cách dùng |

❌ Nếu issue **không** trong whitelist:

```text
Quyết định: REJECT
Reason template:
"Lỗi '<tên lỗi>' không nằm trong scope gói <tên gói>. 
Chúng tôi đề xuất bạn đặt order trả phí thông thường.
Giá ước tính: <range>. Đặt order: <deep link>"
```

### Step 3 · Verify photos + description

KH đã upload ảnh + mô tả? Check:

- **Có ảnh**: tốt · ✓
- **Không ảnh**: chấp nhận được nhưng tăng risk fraud. Nếu issue value > 300k → request thêm ảnh trước khi approve

❌ Red flags trong description:
- "Tôi tự thay tụ trước rồi nhưng vẫn không chạy" → có thể KH tự can thiệp gây hỏng thêm
- "Máy bị rơi/ngập nước/sét đánh" → force majeure exclusion
- "Đã mua máy 12 năm" → quá tuổi cover

**Action**: REJECT với reason đúng exclusion category.

### Step 4 · Check claim history (anti-abuse)

Click "Customer history" tab. Xem:

```text
History claims · Anh Nguyễn A (cust_123):
- WC-001230 · 5 ngày trước · capacitor · approved → completed
- WC-001215 · 12 ngày trước · capacitor · approved → completed  ← cùng issue!
- WC-001200 · 18 ngày trước · gas_refill · approved → completed
```

🚨 **Red flags**:
- **3+ claims trong 7 ngày**: KH có vấn đề thật hoặc abuse → escalate Ops Manager
- **Cùng issue category lặp lại < 30 ngày**: nghi ngờ thợ cũ làm không kỹ → escalate quality team review
- **5+ claims trong 1 tháng**: throttle · pause approval + investigate

→ Khi escalate: chuyển claim sang "Manager review" tab + add notes.

### Step 5 · Estimate cost vs warranty fund value

Mental math nhanh:

- Issue X có `max_value` = vd 300k (cap per claim, từ whitelist)
- Estimate actual cost = labor (~150k) + part (~100-200k)
- So với `customer_warranty.remaining_quota_value` (còn lại)

⚠️ Nếu estimated cost > 80% remaining quota value → **flag** Ops Manager review trước.

Vd: gói còn 1 claim, max budget 500k, KH yêu cầu nạp gas full (cost ~600k). 
→ Reject hoặc partial approve (chỉ nạp 1 phần trong cap).

### Step 6 · Decide & action

3 options:

#### Option A: APPROVE
```text
Click [✅ Approve]
Add note (optional): "Issue confirmed. Authorized free repair."
Submit
```

**System action**:
- Emit `WarrantyClaimApproved` event
- Create order với `is_warranty_order=TRUE, amount_charged=0`
- Decrement quota count (immediately)
- Push notification cho KH: "Yêu cầu đã duyệt. Tìm thợ..."
- Dispatch agent qua normal flow

#### Option B: REJECT
```text
Click [❌ Reject]
Reason (required dropdown):
- not_covered (issue not in whitelist)
- excluded_force_majeure (thiên tai, rơi)
- excluded_age (thiết bị > 10 năm)
- excluded_user_fault (KH dùng sai)
- quota_exceeded (heads up nhưng system thường catch trước)
- duplicate_claim (cùng issue gần đây)
- suspected_fraud (escalate)
- other (free text)

Customer message (template auto-fill, edit được):
"Yêu cầu sửa <issue> không được cover bởi gói <pkg>.
Lý do: <reason explanation>.
Bạn có thể đặt order trả phí: <link>."

Submit
```

**System action**:
- Emit `WarrantyClaimRejected` event
- Restore quota count (nếu đã decrement)
- Push notification cho KH với reason
- Suggest paid order

#### Option C: ESCALATE
```text
Click [🔼 Escalate to Manager]
Reason: free text · vd "Suspected abuse · 3 claims in 5 days"
Submit
```

**System action**:
- Move to `manager_review` status
- Notify Ops Manager
- Pause SLA clock (won't count against you)

---

## 5. Common scenarios · Practice cases

### Case 1: Easy approve

```text
Customer: Anh Lê (cust_456)
Package: AC Basic 1y · 6 months remaining
Quota: cleaning 2/4, repair 3/4
Claim: capacitor failure
Photos: 2 photos clear (capacitor blown, AC label visible)
History: 0 prior claims
```

**Decision**: ✅ APPROVE
**Reason**: Whitelist ✓ · Quota OK · No fraud signals · Photos clear

---

### Case 2: Reject - out of scope

```text
Customer: Chị Phạm (cust_789)
Package: AC Basic 1y
Claim type: repair_basic
Issue selected: capacitor
BUT description says: "Máy lạnh không lạnh dù tụ thì ok, 
                       compressor không chạy"
```

**Decision**: ❌ REJECT
**Reason code**: `not_covered`
**Message**:
> Hệ thống ghi nhận yêu cầu của bạn liên quan đến compressor (block) 
> không khởi động, không phải tụ điện. "Thay/sửa compressor" KHÔNG nằm 
> trong scope gói AC Basic. Bạn có thể đặt order trả phí với mức 
> 2.5M-4M VND để được thợ khảo sát + báo giá chính xác.
> [Đặt order trả phí →]

---

### Case 3: Reject - force majeure

```text
Customer: Bác Hoàng (cust_321)
Package: Fridge Basic 1y
Claim: thermostat
Description: "Tủ lạnh ngập nước do mưa lớn tuần trước. 
              Giờ thermostat không hoạt động đúng"
```

**Decision**: ❌ REJECT
**Reason code**: `excluded_force_majeure`
**Message**:
> Hư hỏng do ngập nước / thiên tai thuộc trường hợp loại trừ của gói 
> bảo trì (xem Điều khoản dịch vụ §9.4). Đề xuất bạn:
> 1. Đặt order trả phí · thợ sẽ khảo sát toàn bộ thiết bị
> 2. Kiểm tra bảo hiểm gia đình (nếu có)
> [Đặt order →]

---

### Case 4: Escalate - suspected abuse

```text
Customer: Anh Trần (cust_555)
Package: AC Premium 1y · 2 months remaining
History (last 30 days):
- 5 claims · 5 approved · all completed
- Pattern: claim every 5-6 days · alternating capacitor/gas
- Total cost to SMP: 2.8M VND vs gói giá 1.8M VND
```

**Decision**: 🔼 ESCALATE
**Notes to Manager**:
> "Customer claim rate abnormal: 5 in 30 days, alternating same issues.
> Cost-to-revenue ratio 155%. Suggest investigation.
> Possible scenarios:
> (a) Genuine multiple device failures - need re-inspection
> (b) Agent quality issue - prior repairs ineffective
> (c) Abuse - fake symptoms
> Recommend: re-inspection visit by senior tech before next approval."

---

### Case 5: Partial approve / negotiate

```text
Customer: Chị Vũ (cust_888)
Package: AC Basic 1y · 8 months left
Quota: 3/4 repairs used
Claim: gas_refill_partial
Description: "Máy lạnh không lạnh, hết gas. Nạp full 1kg luôn nhé."
```

**Issue**: Gói chỉ cover `gas_refill_partial` (max 0.5kg, max_value 500k).
Full refill ~1kg cost ~800k VND, không trong scope.

**Decision**: ✅ APPROVE (partial) + ❌ explain
**Action**:
1. Approve cho partial refill (cover by gói)
2. Add note cho thợ: "Customer requested full refill (1kg). 
   Package covers only partial (0.5kg, max 500k). 
   Charge customer for excess (additional 0.5kg ≈ 300-400k VND)."

**Message to customer**:
> Gói AC Basic của bạn cover nạp gas bổ sung (< 0.5kg, miễn phí). 
> Nếu thợ kiểm tra cần nạp nhiều hơn, phần thêm sẽ tính phí thường 
> (~300-400k VND cho 0.5kg). Thợ sẽ tư vấn cụ thể tại nhà.

---

## 6. SLA management

### Target SLA per BR-WPKG-006:
- **5 phút** review từ khi claim opened
- **10 phút** maximum acceptable (after 10 min, complaint risk)

### How to maintain SLA:
- **Mở claim** ngay khi vào queue (don't batch-process)
- **Standard cases** (whitelist + photos OK + no red flags): < 2 phút decision
- **Edge cases** (need investigation): use ESCALATE thay vì hold

### Dashboard tracking:
Admin Web → "My performance" tab:
- Claims processed today
- Average decision time
- Approval rate (target: 75-85% based on whitelist accuracy)
- SLA breach count

### Performance review:
- **Daily standup**: Ops Manager reviews previous day's claims
- **Weekly QC**: 10% random sample audit by QC Lead
- **Monthly**: full review · adjust whitelist nếu nhiều case grey area

---

## 7. Anti-fraud playbook

### Pattern detection rules

| Pattern | Threshold | Action |
|---|---|---|
| Same issue, same customer | 2+ in 14 days | Investigate quality of prior repair |
| Multiple devices, same address | 3+ different devices/warranties | Verify legitimacy |
| Customer claim/cancel cycle | 2+ in 6 months | Flag account · review T&C compliance |
| Photos appear reused | Image hash matches prior claim | Reject + warning |
| Description copy-pasted | Verbatim from prior claim | Reject + warning |
| Claim immediately after purchase | < 7 days from start_date | Verify legitimate (esp if heavy claim) |

### Escalation channels

| Severity | Channel | Owner |
|---|---|---|
| Single suspicious claim | Escalate trong Admin Web | Ops Manager |
| Repeat offender (3+ flags) | Slack #warranty-fraud | Ops Manager + Legal |
| Suspected criminal fraud | Phone immediate | Legal + CEO |

### Customer education (proactive)

Nếu phát hiện patterns nhẹ (vd KH chưa quen với quota), ưu tiên **educate** trước reject:
- "Anh/chị có biết quota gói còn 1 lần thôi, có cần dùng lần này không?"
- "Lần trước thợ A đã sửa cùng vấn đề. Anh có thể cho biết lý do tái phát?"

---

## 8. Customer communication templates

### Approval message (Vietnamese)

```text
🎉 Yêu cầu sửa đã được duyệt!

Cảm ơn anh/chị đã sử dụng gói {package_name}. 
Yêu cầu sửa {issue_name} của anh/chị đã được duyệt miễn phí.

Đang tìm thợ phù hợp. Dự kiến thợ sẽ liên hệ trong 30-60 phút.

Order: #{order_id}
Phí KH: 0₫ (đã cover bởi gói)
Quota còn lại: {remaining}/{total} lần

Cần hỗ trợ? Gọi 1900-XXX-XXX
```

### Rejection - not covered

```text
Xin lỗi anh/chị, yêu cầu sửa không được duyệt 😔

Lý do: Lỗi "{issue}" không nằm trong scope gói {package_name}.

Gói {package_name} cover các lỗi:
{covered_issues_list}

Đề xuất:
✓ Đặt order trả phí: giá ước tính {price_range}
✓ Hoặc thử lại nếu bạn nghĩ lỗi thực sự khác

[Đặt order trả phí →]

Có thắc mắc? Phản hồi trong app hoặc gọi 1900-XXX-XXX.
```

### Rejection - exclusion (force majeure)

```text
Xin lỗi anh/chị, yêu cầu sửa không được duyệt 😔

Lý do: Hư hỏng do {force_majeure_type} thuộc trường hợp loại trừ.

Tham khảo Điều khoản gói §9.4 để xem danh sách loại trừ đầy đủ.

Đề xuất: Đặt order trả phí · thợ sẽ khảo sát toàn bộ thiết bị 
và báo giá chính xác. [Đặt order →]

Chúng tôi rất tiếc vì điều này. Nếu có khiếu nại, vui lòng phản hồi.
```

### Escalation hold message

```text
🔄 Yêu cầu của anh/chị đang được Manager xem xét

Một số yêu cầu cần được review thêm để đảm bảo quyền lợi của bạn.
Manager sẽ phản hồi trong vòng 30 phút.

Xin lỗi vì thời gian chờ và cảm ơn sự kiên nhẫn của anh/chị.
```

---

## 9. Quiz · Knowledge check (cần pass trước khi solo)

10 câu hỏi (8/10 to pass):

1. **Cooling-off period là bao nhiêu ngày?**  
   a) 3 b) 5 c) **7** d) 14

2. **KH có 2 máy lạnh, mua 1 gói AC Basic, gói cover bao nhiêu máy?**  
   a) **1** b) 2 c) Tất cả máy lạnh trong nhà d) Tùy thuộc

3. **Cleaning claim cần Ops approve không?**  
   a) Yes b) **No (auto-approve)** c) Chỉ khi cao tần suất d) Chỉ Premium gói

4. **SLA review claim repair là bao nhiêu phút?**  
   a) 2 b) **5** c) 15 d) 30

5. **Customer claim "thay block máy lạnh" trong gói AC Basic → quyết định?**  
   a) Approve nếu còn quota b) **Reject (not in whitelist)** c) Escalate d) Approve partial

6. **KH cancel gói sau 5 ngày dùng → refund?**  
   a) 0% b) Proportional c) **100% (cooling-off)** d) 50%

7. **Gói AC Basic cover bao nhiêu issue categories?**  
   a) 4 b) **6** c) 8 d) Unlimited

8. **Approve sai 1 claim "thay block" → consequences?**  
   a) Không sao b) Reduce performance score c) Pause approval rights + retrain d) **All of the above (b+c)**

9. **Customer có 5 claims trong 30 ngày, cùng issue → action?**  
   a) Approve · KH dùng quota của họ  
   b) Reject all  
   c) **Escalate to Ops Manager**  
   d) Suspend warranty

10. **Issue "không tìm thấy trong dropdown" → KH chọn "other" và mô tả. Action?**  
    a) Auto-reject  
    b) Approve nếu plausible  
    c) **Read description carefully, match với whitelist, decide based on actual content**  
    d) Ask customer to re-submit

**Answer key**: 1c · 2a · 3b · 4b · 5b · 6c · 7b · 8d · 9c · 10c

---

## 10. Shadow training plan

### Week 1: Observation (2 days)
- Sit with senior Ops admin
- Watch 20+ real claims being processed
- Take notes on edge cases

### Week 2: Buddy review (3 days)
- Process claims with buddy present
- Buddy reviews every decision before submit
- Discuss why approve/reject

### Week 3: Solo with safety net
- Solo decisions
- Manager reviews 100% trong tuần đầu
- Random sample drops xuống 50%, then 25%, then 10%

### Ongoing
- 10% random audit weekly
- Monthly performance review
- Quarterly retraining (especially on new whitelist additions)

---

## 11. Resources

### Quick reference cards (in-Admin)

Ops có hotkey trong Admin Web:
- `?` → mở help menu
- `W` → Whitelist lookup (search issue → see which gói covers)
- `H` → History lookup (search customer → see all claims)
- `T` → Templates (paste rejection messages)

### Escalation contacts

| Issue | Contact | Channel |
|---|---|---|
| Edge case decision | Ops Manager | Slack DM |
| Fraud suspect | Ops Manager + Legal | Slack #warranty-fraud |
| System bug (Admin Web) | Backend on-call | PagerDuty |
| Customer complaint after rejection | Customer Success | Slack #cs |
| Policy ambiguity | Founder + Legal | Slack #leadership |

### Internal docs

- [Doc 19 §9](../10-legal/19-service-guarantee-policy.md) - Policy full text
- [Doc 15 §R](../05-ba/15-business-rules.md) - 12 rules cụ thể
- [Doc 25](../09-finance/25-warranty-packages-sample-data.md) - Whitelist 5 packages

---

## 12. Continuous improvement

### Weekly review meeting (every Friday 4pm)

Agenda:
1. Review last week metrics:
   - Total claims processed
   - Approval rate
   - SLA performance
   - Escalation rate
2. Audit sample of 5 random claims (each Ops admin)
3. Edge cases discussion · update playbook nếu cần
4. New whitelist requests (based on customer feedback)

### Monthly retro
- What worked well?
- What's confusing?
- Policy updates needed?
- Training gaps?

### Quarterly review
- Adjust whitelist based on data
- Review pricing/packages with Finance
- Customer satisfaction metrics review

---

## Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-05-29 | Initial training script for v3.5 launch |

---

## Sign-off

Trainer (Ops Manager): _________________  Date: ___________

Trainee: _________________  Date: ___________

Quiz score: ___/10 · Pass/Fail: _______
