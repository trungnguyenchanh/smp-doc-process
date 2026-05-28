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

## Việc cần luật sư VN xác nhận
1. Ranh giới "service guarantee" vs "insurance" theo luật kinh doanh bảo hiểm.
2. Trách nhiệm thợ gốc khi đã nghỉ việc — có truy đòi được không?
3. Wording điều khoản loại trừ phù hợp với Luật Bảo vệ Quyền lợi Người tiêu dùng.

---

## Changelog
| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-05-28 | Tách từ trust-legal-policy-pack-v1.md (Section A). Service guarantee, không phải insurance. Quỹ reserve trích từ commission. |
