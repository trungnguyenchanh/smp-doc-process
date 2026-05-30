# Doc 21 · PDPL Data Policy · DRAFT
**Status**: DRAFT — **bắt buộc legal review (luật sư VN) trước khi công bố/áp dụng** · **Audience**: Legal, DPO, Security, Backend, Ops

> ⚠️ **Disclaimer**: Đây là bản thảo do team soạn để định hình chính sách, **không phải tư vấn pháp lý**. Mọi điều khoản công bố cho KH/thợ/partner phải qua luật sư VN rà soát.

> Viện dẫn: **Luật Bảo vệ dữ liệu cá nhân — Luật số 91/2025/QH15 (PDPL)** + **Nghị định 356/2025/NĐ-CP**, cùng **hiệu lực 01/01/2026**, thay Nghị định 13/2023. Lưu ý chế tài có thể tới **5% doanh thu năm trước** cho vi phạm chuyển dữ liệu xuyên biên giới. Luật sư VN rà soát bắt buộc.

---

## 1. Nguyên tắc
- Xử lý dữ liệu **đúng mục đích, trong phạm vi đã thông báo**.
- Lưu trữ **trong thời gian phù hợp mục đích**, không vô thời hạn.
- Có **căn cứ pháp lý/đồng ý** (consent) cho từng mục đích.

## 2. Phân loại dữ liệu
Theo NĐ 356: phân biệt **dữ liệu cá nhân cơ bản** (họ tên, SĐT, địa chỉ, email…) và **nhạy cảm** (sức khỏe, sinh trắc học, vị trí…). SMP rà lại classification (đặc biệt **Face ID/biometric** nếu dùng, **GPS live** trong SOS/tracking).

Chi tiết phân loại + encryption: xem [Doc 13 · Data Classification + Encryption Policy](../08-security/13-data-classification-encryption.md).

## 3. Thời hạn lưu trữ mặc định

| Loại dữ liệu | Default retention | Căn cứ |
|---|---|---|
| Chat text (KH↔thợ) | 12 tháng | mục đích vận hành/dispute |
| Call recording (nếu bật, có consent) | 90 ngày | tối thiểu hóa dữ liệu |
| Hóa đơn / chứng từ thuế | **10 năm** | NĐ 123/2020 + NĐ 70/2025 |
| Ledger / audit tài chính | theo nghĩa vụ kế toán-thuế | luật kế toán |
| KYC thợ/partner | thời gian quan hệ + nghĩa vụ pháp lý | — |
| GPS live (SOS/share-trip) | chỉ trong phiên active, xóa sau | tối thiểu hóa |
| SOS incident | giữ tới khi resolve + nghĩa vụ pháp lý | — |

## 4. Ngoại lệ / Legal hold (override retention default)
Không xóa dù hết hạn mặc định nếu rơi vào: **dispute/complaint đang mở · điều tra fraud · sự cố SOS/an toàn · legal hold · nghĩa vụ kế toán-thuế · yêu cầu cơ quan có thẩm quyền**. Hết ngoại lệ → áp lại lịch xóa.

## 5. Quyền của chủ thể dữ liệu
KH/thợ có quyền truy cập, chỉnh sửa, xóa, rút đồng ý, phản đối xử lý… **Thời hạn phản hồi yêu cầu** theo NĐ 356 (luật sư xác nhận mốc cụ thể). Xóa theo yêu cầu **trừ khi** vướng nghĩa vụ giữ lại khác (section 4).

## 6. Yêu cầu tuân thủ tối thiểu (checklist)
- [ ] Privacy notice (thông báo xử lý dữ liệu) cho KH/thợ/partner.
- [ ] Cơ chế **consent**/căn cứ pháp lý theo mục đích (gồm consent ghi âm call — mặc định OFF).
- [ ] **DPA với vendor** (chat/call provider, payment gateway, cloud).
- [ ] **DPIA/TIA** nếu áp dụng (xử lý nhạy cảm, chuyển xuyên biên giới).
- [ ] **Access audit log** (ai xem PII, khi nào) — đặc biệt unmask SĐT/bank.
 - Implementation: xem [Doc 12 · Audit Log Spec](../08-security/12-audit-log-spec.md).
- [ ] Đánh giá **chuyển dữ liệu xuyên biên giới** (nếu dùng cloud/vendor ngoài VN) — rủi ro phạt 5% doanh thu.
- [ ] Bổ nhiệm nhân sự bảo vệ dữ liệu đạt yêu cầu (PDPL nâng yêu cầu năng lực so NĐ 13).

## 7. Chat anti-off-platform + privacy
- Detector PII không sanction tự động nếu message thuộc **whitelist context**: mã đơn, số căn hộ/tầng, serial thiết bị, mã bảo hành, hotline chính thức SMP.
- Vi phạm thật ≥ 3 lần → tạm khóa chat 24h + **cơ chế appeal** ("yêu cầu Ops mở lại").

---

## Việc cần luật sư VN xác nhận
1. Áp dụng PDPL/NĐ 356 cho mô hình marketplace 3-sided + biometric (Face ID) + GPS.
2. Mốc thời hạn phản hồi data-subject-request cụ thể theo NĐ 356.
3. Đánh giá DPIA/TIA cho xử lý nhạy cảm + chuyển dữ liệu xuyên biên giới.
4. Yêu cầu năng lực bổ nhiệm DPO theo PDPL mới.
5. Wording privacy notice + consent flow phù hợp PDPL.

---

## Changelog
| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-05-28 | Tách từ trust-legal-policy-pack-v1.md (Section C + D). PDPL 2025 (Luật 91/2025/QH15 + NĐ 356/2025). |
