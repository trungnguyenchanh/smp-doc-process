# Doc 20 · COD Payment Policy · DRAFT
**Status**: DRAFT — **bắt buộc legal review (luật sư VN) trước khi công bố/áp dụng** · **Audience**: Legal, Ops, Finance, BA

> ⚠️ **Disclaimer**: Đây là bản thảo do team soạn để định hình chính sách, **không phải tư vấn pháp lý**. Mọi điều khoản công bố cho thợ/KH phải qua luật sư VN rà soát.

---

## 1. Điều kiện áp dụng (limited beta)
- Đơn ≤ **trần X VND** (khởi điểm thấp, vd 500.000đ; nới theo dữ liệu).
- KH có **lịch sử tốt** (chống bom hàng) — quy tắc BR-PAY-COD-003 (sẽ chốt khi adopt rules engine).
- Chỉ thợ **KYC advanced+** và rating ≥ ngưỡng được nhận đơn COD (whitelist).

## 2. Thời điểm coi là "đã thanh toán"

| Góc nhìn | Mốc |
|---|---|
| Khách hàng (UX) | Khi thợ xác nhận đã thu cash (`COD_COLLECTED`) → "Đã thanh toán (tiền mặt)" |
| Kế toán SMP | Chỉ khi thợ **nộp tiền về SMP** (`cod_remitted` → `SETTLED`) |

→ Khoảng giữa: tiền ở tay thợ = `agent_cod_receivable` (thợ nợ SMP). Dashboard thợ hiển thị **"COD phải nộp"**.

Chi tiết kỹ thuật state machine: xem [Doc 17 · Payment & Settlement Lifecycle](../09-finance/17-payment-settlement-lifecycle.md).

## 3. Nghĩa vụ nộp tiền của thợ
- Remit trong **cuối ngày làm việc / T+1**.
- Quá SLA → **freeze toàn bộ payout** của thợ + cảnh báo Ops.
- Nộp thiếu (variance) → ghi `agent_debt` = phần thiếu, freeze payout đến khi bù; lặp lại → review/đình chỉ COD.

## 4. Hoàn tiền đơn COD
- Hoàn **trước khi thợ remit**: SMP hoàn về **ví KH** (đảo suspense `cod_clearing`), thợ vẫn phải remit, SMP thu hồi sau.
- Hoàn **sau remit**: như refund chuẩn (về ví ưu tiên).

Chi tiết accounting: xem [Doc 16 · Finance Ledger Spec section 3.4 (COD edge & clawback)](../09-finance/16-finance-ledger-spec.md).

## 5. Hóa đơn & thuế (e-invoice)

> Viện dẫn: **Nghị định 123/2020/NĐ-CP**, sửa bởi **Nghị định 70/2025/NĐ-CP** (hiệu lực 01/06/2025); **Thông tư 32/2025/TT-BTC** (thay Thông tư 78/2021); chế tài **Nghị định 310/2025** (hiệu lực 16/01/2026). Luật sư/kế toán xác nhận áp dụng.

- **Thời điểm xuất hóa đơn dịch vụ**: khi **dịch vụ hoàn thành** (= `order.completed`), hoặc tại thời điểm thu tiền nếu thu trước (partner prepaid) — theo NĐ 70/2025 sửa Điều 9 NĐ 123.
- **Hóa đơn điện tử khởi tạo từ máy tính tiền (POS)**: NĐ 70/2025 mở rộng yêu cầu cho ngành bán lẻ/tiêu dùng → SMP consumer-facing **cần đánh giá nghĩa vụ POS e-invoice**.
- **Lưu trữ hóa đơn điện tử 10 năm**.
- **VAT**: đọc rate từ `tax_config` (xem [Doc 16 section 5](../09-finance/16-finance-ledger-spec.md)). Lưu ý mức **giảm tạm 10%→8%** cho hàng/dịch vụ đủ điều kiện tới **31/12/2026** (VAT Law 48/2024/QH15) — không hardcode.
- Phân biệt **receipt nội bộ** (xác nhận giao dịch trong app) vs **hóa đơn thuế (VAT e-invoice)**: subscription/đơn lẻ đều phải xác định rõ loại nào phát hành, ai là người mua trên hóa đơn.

## 6. Điều khoản tóm tắt cho thợ (snippet — chờ luật sư)
> "Khi nhận đơn COD, bạn thu tiền mặt thay mặt SMP và có nghĩa vụ nộp lại đủ số tiền trong [thời hạn]. Số tiền chưa nộp được ghi nhận là khoản bạn đang giữ của SMP; nộp trễ/thiếu có thể tạm khóa khoản chi trả thu nhập của bạn cho đến khi hoàn tất đối soát."

---

## Việc cần luật sư VN xác nhận
1. Ranh giới trách nhiệm thợ giữ tiền COD (quan hệ ủy quyền thu hộ).
2. Wording điều khoản nghĩa vụ thợ phải nộp tiền + chế tài.
3. Nghĩa vụ POS e-invoice (NĐ 70/2025) cho dịch vụ tận nơi consumer-facing.
4. Người mua trên hóa đơn VAT cho đơn partner-private (KH cuối vs partner).
5. Thời điểm xuất e-invoice cho đơn COD (collected vs remitted).

---

## Changelog
| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-05-28 | Tách từ trust-legal-policy-pack-v1.md (Section B). COD lifecycle, nghĩa vụ remit, refund-before-remit, e-invoice theo NĐ 70/2025. |
