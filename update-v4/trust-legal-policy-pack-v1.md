# SMP · Trust & Legal Policy Pack · v1.0 (DRAFT)

**Version**: 1.0 · **Date**: 2026-05-28 · **Status**: DRAFT — **bắt buộc legal review (luật sư VN) trước khi công bố/áp dụng** · **Audience**: Legal, Ops, Finance, BA, Founder

> ⚠️ **Disclaimer**: Đây là bản thảo do team soạn để định hình chính sách, **không phải tư vấn pháp lý**. Mọi điều khoản công bố cho KH/thợ/partner phải qua luật sư VN rà soát. Các viện dẫn luật dưới đây cần luật sư xác nhận áp dụng đúng tình huống SMP.

Gồm 3 chính sách: **A. Bảo hành / Đảm bảo dịch vụ** · **B. Thanh toán tiền mặt (COD)** · **C. Lưu trữ & Bảo vệ dữ liệu (PDPL)**.

---

# A. Chính sách Bảo hành / Đảm bảo dịch vụ

> **Lưu ý chữ dùng**: SMP dùng "**đảm bảo dịch vụ / bảo hành**" (service guarantee), **KHÔNG** dùng "**bảo hiểm**" trừ khi có hợp đồng bảo hiểm với đối tác được cấp phép. Quỹ nội bộ tên `service_guarantee_reserve`, không gọi là "insurance".

## A.1 Phạm vi
- Áp dụng cho dịch vụ có `warranty_days > 0` (cấu hình theo từng service trong catalog).
- Chứng nhận bảo hành (warranty certificate) **tự động cấp khi đơn `completed`**, hiệu lực từ thời điểm đó (`warranty_until = completed_at + warranty_days`).

## A.2 Được bảo hành (re-service miễn phí)
Trong thời hạn bảo hành, nếu lỗi **tái phát do tay nghề hoặc linh kiện** thuộc hạng mục đã thực hiện:
- SMP điều thợ quay lại xử lý **miễn phí** (ưu tiên thợ gốc; thợ off > 24h → thợ khác cùng skill).
- Hoặc, nếu không khắc phục được, hoàn tiền phần hạng mục lỗi.

## A.3 KHÔNG được bảo hành (exclusions)
- Hư hỏng do **người dùng/bên thứ ba** tác động sau khi hoàn tất.
- **Hao mòn tự nhiên**, lỗi thiết bị không thuộc hạng mục SMP thực hiện.
- Thiệt hại **gián tiếp/hệ quả** (mất mát kinh doanh, dữ liệu…).
- Tự ý sửa chữa/can thiệp bởi bên ngoài trong thời hạn bảo hành.
- Sự kiện bất khả kháng.

## A.4 Quy trình & SLA
1. KH mở yêu cầu bảo hành trong app + mô tả + ảnh.
2. Re-service resolution mục tiêu **< 48h**.
3. Tranh chấp lỗi tay nghề → Ops review.

## A.5 Ai gánh chi phí
- Lỗi tay nghề thợ → **thợ gốc gánh** (trừ earnings/ảnh hưởng KPI).
- Thợ đã nghỉ việc → gánh bởi `service_guarantee_reserve`.

## A.6 Quỹ đảm bảo dịch vụ (nội bộ)
- Trích **X%** commission mỗi đơn `SETTLED` vào `service_guarantee_reserve` (X chốt sau pilot 300–500 đơn — claim rate là biến cố low-frequency, 50 đơn không đủ).
- Có **trần chi** per claim + danh mục loại trừ (A.3).

## A.7 "Đền bù thiệt hại tài sản" (phased — CHƯA công bố)
- **P0–P1: KHÔNG** quảng bá "đền bù tới N tỷ" kiểu đối thủ.
- **P2**: chỉ triển khai sau khi có (a) policy + điều khoản loại trừ rõ, (b) legal review, (c) cân nhắc đối tác bảo hiểm được cấp phép. Khi đó mới được dùng chữ "bảo hiểm".

## A.8 Điều khoản tóm tắt cho KH (snippet — chờ luật sư)
> "Dịch vụ có biểu tượng *Bảo hành N ngày* được SMP đảm bảo: nếu lỗi do tay nghề/linh kiện tái phát trong thời hạn, kỹ thuật viên sẽ xử lý lại miễn phí. Một số trường hợp loại trừ áp dụng (xem chi tiết). Đây là cam kết đảm bảo dịch vụ của SMP, không phải hợp đồng bảo hiểm."

---

# B. Chính sách Thanh toán Tiền mặt (COD)

## B.1 Điều kiện áp dụng (limited beta)
- Đơn ≤ **trần X VND** (khởi điểm thấp, vd 500.000đ; nới theo dữ liệu).
- KH có **lịch sử tốt** (chống bom hàng) — BR-PAY-COD-003.
- Chỉ thợ **KYC advanced+** và rating ≥ ngưỡng được nhận đơn COD (whitelist).

## B.2 Thời điểm coi là "đã thanh toán"
| Góc nhìn | Mốc |
| --- | --- |
| Khách hàng (UX) | Khi thợ xác nhận đã thu cash (`COD_COLLECTED`) → "Đã thanh toán (tiền mặt)" |
| Kế toán SMP | Chỉ khi thợ **nộp tiền về SMP** (`cod_remitted` → `SETTLED`) |

→ Khoảng giữa: tiền ở tay thợ = `agent_cod_receivable` (thợ nợ SMP). Dashboard thợ hiển thị **"COD phải nộp"**.

## B.3 Nghĩa vụ nộp tiền của thợ
- Remit trong **cuối ngày làm việc / T+1**.
- Quá SLA → **freeze toàn bộ payout** của thợ + cảnh báo Ops.
- Nộp thiếu (variance) → ghi `agent_debt` = phần thiếu, freeze payout đến khi bù; lặp lại → review/đình chỉ COD.

## B.4 Hoàn tiền đơn COD
- Hoàn **trước khi thợ remit**: SMP hoàn về **ví KH** (đảo suspense), thợ vẫn phải remit, SMP thu hồi sau.
- Hoàn **sau remit**: như refund chuẩn (về ví ưu tiên).

## B.5 Hóa đơn & thuế (e-invoice)
> Viện dẫn: **Nghị định 123/2020/NĐ-CP**, sửa bởi **Nghị định 70/2025/NĐ-CP** (hiệu lực 01/06/2025); **Thông tư 32/2025/TT-BTC** (thay Thông tư 78/2021); chế tài **Nghị định 310/2025** (hiệu lực 16/01/2026). Luật sư/kế toán xác nhận áp dụng.
- **Thời điểm xuất hóa đơn dịch vụ**: khi **dịch vụ hoàn thành** (= `order.completed`), hoặc tại thời điểm thu tiền nếu thu trước (partner prepaid) — theo ĐN 70/2025 sửa Điều 9 NĐ123.
- **Hóa đơn điện tử khởi tạo từ máy tính tiền (POS)**: NĐ70/2025 mở rộng yêu cầu cho ngành bán lẻ/tiêu dùng → SMP consumer-facing **cần đánh giá nghĩa vụ POS e-invoice**.
- **Lưu trữ hóa đơn điện tử 10 năm**.
- **VAT**: đọc rate từ `tax_config`. Lưu ý mức **giảm tạm 10%→8%** cho hàng/dịch vụ đủ điều kiện tới **31/12/2026** (VAT Law 48/2024/QH15) — không hardcode.
- Phân biệt **receipt nội bộ** (xác nhận giao dịch trong app) vs **hóa đơn thuế (VAT e-invoice)**: subscription/đơn lẻ đều phải xác định rõ loại nào phát hành, ai là người mua trên hóa đơn.

## B.6 Điều khoản tóm tắt cho thợ (snippet — chờ luật sư)
> "Khi nhận đơn COD, bạn thu tiền mặt thay mặt SMP và có nghĩa vụ nộp lại đủ số tiền trong [thời hạn]. Số tiền chưa nộp được ghi nhận là khoản bạn đang giữ của SMP; nộp trễ/thiếu có thể tạm khóa khoản chi trả thu nhập của bạn cho đến khi hoàn tất đối soát."

---

# C. Chính sách Lưu trữ & Bảo vệ Dữ liệu (PDPL)

> Viện dẫn: **Luật Bảo vệ dữ liệu cá nhân — Luật số 91/2025/QH15 (PDPL)** + **Nghị định 356/2025/NĐ-CP**, cùng **hiệu lực 01/01/2026**, thay Nghị định 13/2023. Lưu ý chế tài có thể tới **5% doanh thu năm trước** cho vi phạm chuyển dữ liệu xuyên biên giới. Luật sư VN rà soát bắt buộc.

## C.1 Nguyên tắc
- Xử lý dữ liệu **đúng mục đích, trong phạm vi đã thông báo**.
- Lưu trữ **trong thời gian phù hợp mục đích**, không vô thời hạn.
- Có **căn cứ pháp lý/đồng ý** (consent) cho từng mục đích.

## C.2 Phân loại dữ liệu
Theo NĐ356: phân biệt **dữ liệu cá nhân cơ bản** (họ tên, SĐT, địa chỉ, email…) và **nhạy cảm** (sức khỏe, sinh trắc học, vị trí…). SMP rà lại classification (đặc biệt **Face ID/biometric** nếu dùng, **GPS live** trong SOS/tracking).

## C.3 Thời hạn lưu trữ mặc định (sửa từ Doc 16 v1.1 — KHÔNG ghi cứng)

| Loại dữ liệu | Default retention | Căn cứ |
| --- | --- | --- |
| Chat text (KH↔thợ) | 12 tháng | mục đích vận hành/dispute |
| Call recording (nếu bật, có consent) | 90 ngày | tối thiểu hóa dữ liệu |
| Hóa đơn / chứng từ thuế | **10 năm** | NĐ123/2020 + NĐ70/2025 |
| Ledger / audit tài chính | theo nghĩa vụ kế toán-thuế | luật kế toán |
| KYC thợ/partner | thời gian quan hệ + nghĩa vụ pháp lý | |
| GPS live (SOS/share-trip) | chỉ trong phiên active, xóa sau | tối thiểu hóa |
| SOS incident | giữ tới khi resolve + nghĩa vụ pháp lý | |

## C.4 Ngoại lệ / Legal hold (override retention default)
Không xóa dù hết hạn mặc định nếu rơi vào: **dispute/complaint đang mở · điều tra fraud · sự cố SOS/an toàn · legal hold · nghĩa vụ kế toán-thuế · yêu cầu cơ quan có thẩm quyền**. Hết ngoại lệ → áp lại lịch xóa.

## C.5 Quyền của chủ thể dữ liệu
KH/thợ có quyền truy cập, chỉnh sửa, xóa, rút đồng ý, phản đối xử lý… **Thời hạn phản hồi yêu cầu** theo NĐ356 (luật sư xác nhận mốc cụ thể). Xóa theo yêu cầu **trừ khi** vướng nghĩa vụ giữ lại khác (C.4).

## C.6 Yêu cầu tuân thủ tối thiểu (checklist)
- [ ] Privacy notice (thông báo xử lý dữ liệu) cho KH/thợ/partner.
- [ ] Cơ chế **consent**/căn cứ pháp lý theo mục đích (gồm consent ghi âm call — mặc định OFF).
- [ ] **DPA với vendor** (chat/call provider — ADR-001, payment gateway, cloud).
- [ ] **DPIA/TIA** nếu áp dụng (xử lý nhạy cảm, chuyển xuyên biên giới).
- [ ] **Access audit log** (ai xem PII, khi nào) — đặc biệt unmask SĐT/bank.
- [ ] Đánh giá **chuyển dữ liệu xuyên biên giới** (nếu dùng cloud/vendor ngoài VN) — rủi ro phạt 5% doanh thu.
- [ ] Bổ nhiệm nhân sự bảo vệ dữ liệu đạt yêu cầu (PDPL nâng yêu cầu năng lực so NĐ13).

## C.7 Chat anti-off-platform + privacy
- Detector PII không sanction tự động nếu message thuộc **whitelist context**: mã đơn, số căn hộ/tầng, serial thiết bị, mã bảo hành, hotline chính thức SMP.
- Vi phạm thật ≥3 lần → tạm khóa chat 24h + **cơ chế appeal** ("yêu cầu Ops mở lại").

---

## D. Việc cần luật sư VN xác nhận (gửi kèm khi review)
1. Áp dụng PDPL/NĐ356 cho mô hình marketplace 3-sided + biometric (Face ID) + GPS.
2. Mốc thời hạn phản hồi data-subject-request cụ thể theo NĐ356.
3. Nghĩa vụ POS e-invoice (NĐ70/2025) cho dịch vụ tận nơi consumer-facing.
4. Người mua trên hóa đơn VAT cho đơn partner-private (KH cuối vs partner).
5. Ranh giới "service guarantee" vs "insurance" theo luật kinh doanh bảo hiểm.
6. Điều khoản trách nhiệm thợ giữ tiền COD (quan hệ ủy quyền thu hộ).

## Changelog
| Version | Date | Changes |
| --- | --- | --- |
| 1.0 | 2026-05-28 | Khởi tạo: bảo hành/đảm bảo dịch vụ, COD + e-invoice, retention PDPL với ngoại lệ/legal hold; checklist tuân thủ; danh mục cần luật sư |
