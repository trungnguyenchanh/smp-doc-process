<!-- 
  ────────────────────────────────────────────────────────────────
  📋 DOC STATUS NOTE
  
  Đây là tách từ trust-legal-policy-pack-v1.md (Section A).
  Tập trung vào chính sách Bảo hành / Đảm bảo dịch vụ.
  
  ⚠️ STATUS: DRAFT · bắt buộc legal review (luật sư VN) trước công bố.
  
  Related docs:
    - Doc 20 · COD Payment Policy
    - Doc 21 · PDPL Data Policy
    - Doc 15 · Business Rules (BR-WARRANTY rules sẽ chốt khi adopt)
    - Doc 16 · Finance Ledger (service_guarantee_reserve account)
  
  Author: SMP Tech Lead/CTO · Date: 2026-05-28
  Status: DRAFT v1.0
  ────────────────────────────────────────────────────────────────
-->

# Doc 19 · Service Guarantee Policy · v1.0 (DRAFT)

**Version**: 1.0 · **Date**: 2026-05-28 · **Status**: DRAFT — **bắt buộc legal review (luật sư VN) trước khi công bố/áp dụng** · **Audience**: Legal, Ops, BA, Founder

> ⚠️ **Disclaimer**: Đây là bản thảo do team soạn để định hình chính sách, **không phải tư vấn pháp lý**. Mọi điều khoản công bố cho KH/thợ/partner phải qua luật sư VN rà soát. Các viện dẫn luật trong doc này cần luật sư xác nhận áp dụng đúng tình huống SMP.

> **Lưu ý chữ dùng**: SMP dùng "**đảm bảo dịch vụ / bảo hành**" (service guarantee), **KHÔNG** dùng "**bảo hiểm**" trừ khi có hợp đồng bảo hiểm với đối tác được cấp phép. Quỹ nội bộ tên `service_guarantee_reserve`, không gọi là "insurance".

---

## 1. Phạm vi
- Áp dụng cho dịch vụ có `warranty_days > 0` (cấu hình theo từng service trong catalog).
- Chứng nhận bảo hành (warranty certificate) **tự động cấp khi đơn `completed`**, hiệu lực từ thời điểm đó (`warranty_until = completed_at + warranty_days`).

## 2. Được bảo hành (re-service miễn phí)
Trong thời hạn bảo hành, nếu lỗi **tái phát do tay nghề hoặc linh kiện** thuộc hạng mục đã thực hiện:
- SMP điều thợ quay lại xử lý **miễn phí** (ưu tiên thợ gốc; thợ off > 24h → thợ khác cùng skill).
- Hoặc, nếu không khắc phục được, hoàn tiền phần hạng mục lỗi.

## 3. KHÔNG được bảo hành (exclusions)
- Hư hỏng do **người dùng/bên thứ ba** tác động sau khi hoàn tất.
- **Hao mòn tự nhiên**, lỗi thiết bị không thuộc hạng mục SMP thực hiện.
- Thiệt hại **gián tiếp/hệ quả** (mất mát kinh doanh, dữ liệu…).
- Tự ý sửa chữa/can thiệp bởi bên ngoài trong thời hạn bảo hành.
- Sự kiện bất khả kháng.

## 4. Quy trình & SLA
1. KH mở yêu cầu bảo hành trong app + mô tả + ảnh.
2. Re-service resolution mục tiêu **< 48h**.
3. Tranh chấp lỗi tay nghề → Ops review.

## 5. Ai gánh chi phí
- Lỗi tay nghề thợ → **thợ gốc gánh** (trừ earnings/ảnh hưởng KPI).
- Thợ đã nghỉ việc → gánh bởi `service_guarantee_reserve`.

## 6. Quỹ đảm bảo dịch vụ (nội bộ)
- Trích **X%** commission mỗi đơn `SETTLED` vào `service_guarantee_reserve` (X chốt sau pilot 300–500 đơn — claim rate là biến cố low-frequency, 50 đơn không đủ).
- Có **trần chi** per claim + danh mục loại trừ (xem section 3).
- Accounting treatment: xem [Doc 16 · Finance Ledger Spec](../09-finance/16-finance-ledger-spec.md) — account `service_guarantee_reserve` (LIABILITY).

## 7. "Đền bù thiệt hại tài sản" (phased — CHƯA công bố)
- **P0–P1: KHÔNG** quảng bá "đền bù tới N tỷ" kiểu đối thủ.
- **P2**: chỉ triển khai sau khi có (a) policy + điều khoản loại trừ rõ, (b) legal review, (c) cân nhắc đối tác bảo hiểm được cấp phép. Khi đó mới được dùng chữ "bảo hiểm".

## 8. Điều khoản tóm tắt cho KH (snippet — chờ luật sư)
> "Dịch vụ có biểu tượng *Bảo hành N ngày* được SMP đảm bảo: nếu lỗi do tay nghề/linh kiện tái phát trong thời hạn, kỹ thuật viên sẽ xử lý lại miễn phí. Một số trường hợp loại trừ áp dụng (xem chi tiết). Đây là cam kết đảm bảo dịch vụ của SMP, không phải hợp đồng bảo hiểm."

---

## 9. Gói bảo hành mua riêng (Maintenance Subscription Package) · v3.5+

> Phân biệt rõ với "warranty embedded" ở Section 2-8 (tự động kèm theo order). Section này quy định **gói thuê bao** KH chủ động mua trước cho thiết bị có sẵn.

### 9.1 · Bản chất pháp lý

- **KHÔNG phải hợp đồng bảo hiểm** · không cần license của Bộ Tài chính
- Là **gói dịch vụ trả trước** (service prepaid package) · giống gói gym/spa subscription
- Chuẩn kế toán: VAS 14 (Revenue Recognition) · defer revenue, recognize over time
- Wording tránh: "bảo hiểm", "insurance", "đền bù". Wording dùng được: "gói bảo trì", "gói chăm sóc", "thuê bao dịch vụ"

### 9.2 · Cấu trúc gói

| Thành phần | Mô tả |
|---|---|
| **Thời hạn** | 6 tháng / 12 tháng / 24 tháng (configurable) |
| **Phạm vi thiết bị** | 1 gói = 1 thiết bị (xác định bằng serial number nếu có) |
| **Quota vệ sinh định kỳ** | Số lần vệ sinh trong period (vd 4 lần/năm cho AC) |
| **Quota sửa chữa** | Whitelist các issue được sửa free (vd capacitor, gas refill cho AC) |
| **Trần chi mỗi claim** | Cap value để tránh abuse (vd ≤ 500k/lần repair) |
| **Auto-suggest** | Hệ thống nhắc đặt lịch định kỳ |

### 9.3 · Quyền KH và nghĩa vụ SMP

KH có quyền:
- Được vệ sinh định kỳ theo quota (không cần lý do)
- Được sửa chữa free cho các issues whitelisted (cần Ops approve)
- Cooling-off period 7 ngày · refund 100% nếu không hài lòng
- Cancel bất kỳ lúc nào · refund proportional remaining months
- Renew với giá ưu đãi (10% off cho renewal on-time, configurable)

SMP nghĩa vụ:
- Cử thợ trong SLA (24h cho cleaning, 48h cho repair)
- Chỉ thợ KYC `advanced` mới được dispatch cho warranty claims (chất lượng đảm bảo)
- Refund đúng theo terms · không có "hidden fees"
- Notify trước 7 ngày khi gói sắp hết hạn

### 9.4 · Loại trừ (exclusions)

**KH KHÔNG được claim**:
- Hư hỏng do thiên tai, cháy nổ, ngập nước (force majeure)
- Hư hỏng do KH sử dụng sai cách (vd dùng AC quá công suất, không vệ sinh filter)
- Thiết bị có dấu hiệu tác động ngoại lực (vỡ, rơi)
- Thay block, thay máy mới (cần order riêng trả phí)
- Lỗi do thiết bị đã quá date sản xuất > 10 năm
- Linh kiện non-standard KH tự lắp trước đó

### 9.5 · Chống abuse

Risks + mitigations:
- **KH claim quá nhiều issues nhỏ** → cap repair quota count (vd max 6 lần/năm)
- **KH chuyển gói sang nhà khác** → linked với device serial + customer_id, không transferable
- **KH lạm dụng cooling-off** → log nếu cancel > 2 lần trong 6 tháng, flag account
- **Agent gian lận** (báo claim ảo) → audit photo bắt buộc + sample re-inspection 5% claims

### 9.6 · Pricing strategy (cho Founder + Finance decision)

Reference benchmarks (VN market):
- Vệ sinh AC định kỳ 1 lần: ~200k VND
- Gói "Bảo trì AC 1 năm" market range: 1.0M-1.8M VND (4 lần vệ sinh + repair minor)

Recommended pricing (DRAFT):
| Gói | Thời hạn | Vệ sinh | Repair quota | Giá đề xuất |
|---|---|---|---|---:|
| AC Basic | 12 tháng | 4 lần | 4 lần whitelist | 1,200,000 |
| AC Premium | 12 tháng | 4 lần | 8 lần whitelist | 1,800,000 |
| AC Family (3 devices) | 12 tháng | 4 × 3 | 4 × 3 | 3,200,000 (v3.7+) |
| Washer Basic | 12 tháng | 2 lần | 3 lần | 800,000 |
| Fridge Basic | 12 tháng | - | 4 lần | 900,000 |

> Pricing chốt sau pilot · Finance phân tích cost ~30% revenue (target gross margin 70%)

### 9.7 · Migration từ v3.4

- Khách hiện tại: hiển thị offer mua gói trên app
- Sales channel: in-app catalog + admin web "Bán gói cho KH"
- Pilot: 100 customers đầu tiên · 90-day window đo cancellation rate + claim rate
- Pricing tuning theo data pilot

---

## Việc cần luật sư VN xác nhận
1. Ranh giới "service guarantee" vs "insurance" theo luật kinh doanh bảo hiểm.
2. Trách nhiệm thợ gốc khi đã nghỉ việc — có truy đòi được không?
3. Wording điều khoản loại trừ phù hợp với Luật Bảo vệ Quyền lợi Người tiêu dùng.
4. **(v3.5 NEW)** Gói "Maintenance Subscription Package" có cần đăng ký với cơ quan nào không? Có nằm trong scope Luật Bảo vệ NTD 2023 không?
5. **(v3.5 NEW)** Cooling-off 7 days · phù hợp với Luật BVNTD VN không (article 9)?
6. **(v3.5 NEW)** Refund proportional · có phải tuân thủ thuế GTGT hoàn trả không?

---

## Changelog
| Version | Date | Changes |
|---|---|---|
| 2.0 | 2026-05-29 | Add Section 9 · Maintenance Subscription Package (v3.5+) · 5 packages draft pricing |
| 1.0 | 2026-05-28 | Tách từ trust-legal-policy-pack-v1.md (Section A). Service guarantee, không phải insurance. Quỹ reserve trích từ commission. |
