<!-- 
  ────────────────────────────────────────────────────────────────
  📋 DOC STATUS NOTE (added when integrating into smp-doc-process)
  
  Đây là DRAFT specification cho v3.5+ implementation.
  Hiện tại (v3.4) đang dùng 3 tables riêng: partner_wallet_transactions,
  agent_earnings, points_ledger (xem Doc 02 section 6 và 14).
  
  Adoption strategy (đã chốt):
    - Stage 1 (v3.4 · NOW): Doc này là SPEC, không deploy
    - Stage 2 (v3.5 · 6 weeks): Build PostJournal + dual-write shadow mode
    - Stage 3 (v3.6 · 4 weeks): Cutover read path, drop old tables
    - Stage 4 (v3.7+): Tiếp tục với Kafka, etc.
  
  Author: SMP Tech Lead/CTO · Date: 2026-05-28
  Status: DRAFT · awaiting finance + legal review
  Conflicts: thay thế các sections finance trong Doc 02 và Doc 15 khi cutover
  ────────────────────────────────────────────────────────────────
-->

# Doc 16 · Finance Ledger Spec · v1.2

**Version**: 1.2 · **Date**: 2026-05-28 · **Status**: DRAFT — *design-locked, finance build deferred tới sau pilot* · **Supersedes**: phần D4/ledger của v1.0 & v1.1
**Audience**: Backend (finance), Finance/Kế toán, BA, QC

> v1.2 chốt **mô hình ledger production-correct** sau accounting-review vòng 3. Sửa: Pattern 2 (control account vs subledger dimension), 3 CHECK constraint, journal template đúng chiều (clawback/refund/COD-clearing), **rounding rule**, và 3 bug bỏ sót (rounding, ledger↔settlement SoT, COD-refund-before-remit). Đọc cùng v1.1 (các feature non-finance giữ nguyên).

---

## 1. Chart of Accounts (v1.2)

| Code | Loại | Normal | Ý nghĩa | Δ so v1.1 |
| --- | --- | --- | --- | --- |
| `cash_gateway` | ASSET | Dr | Clearing tiền cổng đã confirm, chưa về bank | |
| `cash_bank` | ASSET | Dr | Tiền thật trong ngân hàng SMP | |
| `agent_cod_receivable` | ASSET | Dr | **Thợ giữ cash của SMP** (COD thu, chưa remit) | naming chốt |
| `agent_debt` | ASSET | Dr | Thợ nợ lại SMP (clawback / COD short) | |
| `customer_wallet` | LIABILITY | Cr | SMP nợ KH (subledger theo customer) | |
| `agent_payable` | LIABILITY | Cr | SMP nợ thợ — earnings (subledger theo agent) | |
| `partner_wallet` | LIABILITY | Cr | SMP nợ partner (prepaid) | |
| `cod_clearing` | LIABILITY | Cr | **Suspense giữa COD_COLLECTED → SETTLED** | **MỚI** |
| `refund_payable` | LIABILITY | Cr | Hoàn đã duyệt, chờ chi về nguồn gốc | |
| `loyalty_liability` | LIABILITY | Cr | Giá trị điểm đang lưu hành | |
| `service_guarantee_reserve` | LIABILITY | Cr | Quỹ bảo hành/đảm bảo dịch vụ | **RENAME** từ `insurance_fund` |
| `vat_payable` | LIABILITY | Cr | VAT phải nộp (rate đọc từ `tax_config`) | |
| `revenue_commission` | REVENUE | Cr | Doanh thu commission SMP | |
| `gateway_fee_expense` | EXPENSE | Dr | Phí cổng SMP gánh | **MỚI** |
| `marketing_expense` | EXPENSE | Dr | Chi phí điểm/referral | |
| `rounding_adjustment` | EXPENSE | Dr/Cr | Hấp thụ chênh làm tròn cross-entry | **MỚI** |

> **Rename `insurance_fund` → `service_guarantee_reserve`**: chữ "insurance" chỉ dùng khi có policy/đối tác bảo hiểm thật + legal review (ADR-003). Tránh rủi ro pháp lý/marketing.

---

## 2. Data model (Pattern 2 — control account + subledger dimension)

```sql
-- Control accounts: UNIQUE theo (code, currency). KHÔNG tạo 1 row/customer.
CREATE TABLE accounts (
  id BIGINT PK, code VARCHAR(32), currency CHAR(3) DEFAULT 'VND',
  type VARCHAR(12),            -- ASSET|LIABILITY|REVENUE|EXPENSE|EQUITY
  normal_side CHAR(2),         -- 'Dr'|'Cr'
  UNIQUE (code, currency)
);

CREATE TABLE journal_entries (   -- append-only + hash-chain
  id BIGINT PK,
  ref_type VARCHAR(24), ref_id VARCHAR(64),
  idempotency_key VARCHAR(80) UNIQUE NOT NULL,
  currency CHAR(3) DEFAULT 'VND',          -- 1 entry = 1 currency (FX riêng)
  memo VARCHAR(255),
  prev_hash CHAR(64), row_hash CHAR(64),
  created_at TIMESTAMP
);

CREATE TABLE journal_lines (
  id BIGINT PK, entry_id BIGINT, account_id BIGINT,
  -- SUBLEDGER DIMENSION (Pattern 2): owner sống ở line, không đẻ account row
  owner_type VARCHAR(12) NULL,  -- customer|agent|partner|NULL(control)
  owner_id   BIGINT NULL,
  debit_minor  BIGINT DEFAULT 0,
  credit_minor BIGINT DEFAULT 0,
  CONSTRAINT chk_nonneg   CHECK (debit_minor >= 0 AND credit_minor >= 0),
  CONSTRAINT chk_one_side CHECK ((debit_minor > 0) <> (credit_minor > 0)),
  INDEX idx_subledger (account_id, owner_type, owner_id)
);
```

**Invariant enforcement (3 lớp):**
1. **Single writer**: mọi posting đi qua đúng 1 hàm `PostJournal(entry, lines[])` trong transaction. Không service nào INSERT thẳng `journal_lines`.
2. **Deferred constraint trigger** validate `SUM(debit_minor) = SUM(credit_minor)` per `entry_id` **trước COMMIT** → entry không cân = rollback, không vào DB.
3. **Daily reconcile job**: trial balance toàn hệ thống `SUM(all debit) = SUM(all credit)` + so subledger balance vs control.

**Balance**:
- Control: `balance(code) = Σdebit − Σcredit` (×sign theo normal_side).
- Subledger: filter `journal_lines` theo `account_id + owner_type + owner_id`. → `customer_wallet` balance của KH X = subledger view, **không** query bảng ví riêng.
- `customer_wallet` **cấm âm** (overdraft): `PostJournal` reject nếu balance sau < 0 (trừ phi `allow_negative` flag cho nghiệp vụ đặc thù).

---

## 3. Journal Templates (v1.2 — đã sửa đúng chiều)

> Quy ước: `P` = tổng KH trả, `C` = commission (revenue), `V` = VAT, `N` = net thợ. **`P = N + C + V`**. VAT rate đọc từ `tax_config` (xem §5), KHÔNG hardcode.

### 3.1 Thu tiền

| Nghiệp vụ | Debit | Credit |
| --- | --- | --- |
| Gateway pay (có phí cổng `F`) | `cash_gateway` (P−F) + `gateway_fee_expense` (F) | `agent_payable` N + `revenue_commission` C + `vat_payable` V |
| Wallet topup (`T`) | `cash_gateway` (T−F) + `gateway_fee_expense` (F) | `customer_wallet` T |
| Wallet pay | `customer_wallet` P | `agent_payable` N + `revenue_commission` C + `vat_payable` V |
| **COD collected** (chưa revenue) | `agent_cod_receivable` P | `cod_clearing` P |
| **COD remit → SETTLED** (2 entry) | (a) `cash_bank` P · (b) `cod_clearing` P | (a) `agent_cod_receivable` P · (b) `agent_payable` N + `revenue_commission` C + `vat_payable` V |
| Gateway settle về bank (flush clearing, **không** đổi state) | `cash_bank` X | `cash_gateway` X |

> **COD recognize revenue tại SETTLED, không tại COLLECTED** — nhất quán Doc 17 (downstream đợi SETTLED). `cod_clearing` giữ suspense giữa hai mốc.

### 3.2 Quỹ & loyalty

| Nghiệp vụ | Debit | Credit |
| --- | --- | --- |
| Trích quỹ bảo hành (`R` = %·C) | `revenue_commission` R | `service_guarantee_reserve` R |
| Loyalty earn (confirmed, giá trị `L`) | `marketing_expense` L | `loyalty_liability` L |
| Loyalty redeem → ví | `loyalty_liability` L | `customer_wallet` L |

### 3.3 Refund — ma trận (đã sửa, có VAT reversal)

| Case | Debit | Credit |
| --- | --- | --- |
| **TRƯỚC payout · full · về ví** | `revenue_commission` C + `agent_payable` N + `vat_payable` V | `customer_wallet` P |
| **TRƯỚC payout · partial `p`** (split c,n,v; p=n+c+v) | `revenue_commission` c + `agent_payable` n + `vat_payable` v | `customer_wallet` p |
| **TRƯỚC payout · về nguồn gốc** | (như trên) | `refund_payable` p → sau đó `Dr refund_payable / Cr cash_gateway` |
| **SAU payout · lỗi THỢ** (clawback, 2 bước) | (1) `revenue_commission` C + `vat_payable` V + `agent_debt` N | (1) `customer_wallet` P |
| ↳ bước 2 (cấn earning tương lai) | (2) `agent_payable` | (2) `agent_debt` |
| **SAU payout · lỗi SMP / goodwill** (SMP gánh net) | `revenue_commission` C + `vat_payable` V + `marketing_expense`/`refund_expense` N | `customer_wallet` P |

> Phân biệt mấu chốt: **trước payout** net còn ở `agent_payable` → Dr để đảo. **Sau payout** net đã thành cash rời đi → dùng `agent_debt` (lỗi thợ) hoặc expense (lỗi SMP). KHÔNG bao giờ Cr `agent_payable` trong clawback (lỗi v1.1 đã sửa).

### 3.4 COD edge & clawback

| Nghiệp vụ | Debit | Credit |
| --- | --- | --- |
| **COD short** (remit thiếu `var`) | `cash_bank` (P−var) + `agent_debt` var | `agent_cod_receivable` P |
| **Refund đơn COD TRƯỚC remit** (bug bỏ sót #3) | `cod_clearing` P | `customer_wallet` P |
| ↳ thợ vẫn phải remit → khi remit: | `cash_bank` P | `agent_cod_receivable` P (SMP thu hồi cash đã ứng refund) |
| Thợ không remit sau khi SMP đã refund | `agent_debt` P | `agent_cod_receivable` P |

> **Bug #3 (COD-refund-before-remit)**: KH trả cash cho thợ rồi dispute → SMP hoàn về ví KH bằng cách đảo `cod_clearing` (chưa từng recognize revenue nên sạch), nhưng `agent_cod_receivable` **vẫn còn** → thợ vẫn nợ cash, SMP thu hồi sau. Có timing exposure (SMP credit ví trước khi cầm cash) → kiểm soát bằng COD cap + agent whitelist (ADR-004).

---

## 4. Rounding rule (bug bỏ sót #1)

Minor units + chia % (C, V) → làm tròn độc lập có thể lệch 1 VND → entry **không cân**.

**Rule:**
1. Tính `V` (VAT) và `C` (commission) trước, làm tròn về integer minor.
2. **`N` = `P − C − V`** (residual) — N hấp thụ toàn bộ phần dư → `Σdebit = Σcredit` luôn đúng trong cùng entry.
3. Cross-entry (hiếm, vd phân bổ nhiều đơn): chênh dồn vào `rounding_adjustment`, alert nếu > ngưỡng.

```text
P=99,999  rate_vat=8%  rate_comm=15%(trên P−V)
V = round(99,999 × 8/108) = 7,407
base = 92,592 ;  C = round(92,592 × 15%) = 13,889
N = 99,999 − 13,889 − 7,407 = 78,703   ← residual, entry cân tuyệt đối
```

---

## 5. VAT từ config (không hardcode 10%)

VAT Law 48/2024/QH15 + nghị quyết giảm tạm **10%→8%** cho hàng/dịch vụ đủ điều kiện tới **31/12/2026**. → `vat_payable` rate đọc từ:

```sql
CREATE TABLE tax_config (
  id BIGINT PK, country CHAR(2) DEFAULT 'VN',
  category VARCHAR(32), tax_type VARCHAR(8) DEFAULT 'VAT',
  rate_bps INT,            -- 800 = 8%, 1000 = 10%
  effective_from DATE, effective_to DATE NULL
);
-- BR-PRICE-008 REVISED: VAT rate = lookup(tax_config, category, order_date). KHÔNG hardcode.
```

---

## 6. Ledger ↔ Settlement: source of truth (bug bỏ sót #2)

| Khía cạnh | Source of truth |
| --- | --- |
| **Tiền** (số dư, đã thu/chi/nợ) | `journal_*` (ledger) |
| **Quy trình** (đang ở bước nào) | `settlements.state` (Doc 17) |

**Rule**: ledger và settlement state phải nhất quán qua event. Daily reconcile: mỗi `settlement` ở `SETTLED` phải có entry revenue tương ứng; lệch → **freeze payout liên quan + alert Finance**. Khi mâu thuẫn không giải được: **ledger thắng về tiền**, settlement re-derive.

---

## 7. Subledger thay 3 bảng tiền của v1.0

`points_ledger`, `agent_earnings`, `wallet_transactions` (v1.0) **bỏ** — thành **subledger view** trên `journal_lines`:
- Ví KH = filter `customer_wallet` + owner.
- Earnings thợ = filter `agent_payable` (available) / `agent_debt` (debt) / `agent_cod_receivable` (cod_liability) + owner → 4 bucket dashboard (Hit 7).
- Điểm = filter `loyalty_liability` + owner.

`agent_earnings_mv`, `agent_profile_mv` giữ (read model projection, refresh từ event).

## 8. Consumer-side idempotency (bug phụ)
Event không chỉ idempotent ở producer (outbox). Consumer phải có **dedup store** (Redis set `processed:{consumer}:{event_id}`, TTL 7d) → exactly-once xử lý. Chi tiết Event Catalog (Doc 18).

## 9. Changelog
| Version | Date | Changes |
| --- | --- | --- |
| 1.2 | 2026-05-28 | Pattern 2 ledger; 3 CHECK + single-writer + deferred trigger; journal templates sửa đúng chiều (clawback 2-bước, refund matrix + VAT reversal, COD clearing); rounding rule; VAT từ config; ledger↔settlement SoT; COD-refund-before-remit; wallet cấm âm; rename insurance→service_guarantee_reserve |
