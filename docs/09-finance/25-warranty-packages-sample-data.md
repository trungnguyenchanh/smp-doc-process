# Sample Data · 5 Warranty Packages · Maintenance Subscription

> **Status**: DRAFT for review by Finance + Ops + Founder · v3.5+ 
> **Date**: 2026-05-29 
> **Action required**: Finance review pricing + Ops review quotas + Legal review wording

> ⚠️ **Disclaimer**: Đây là draft proposal. Pricing chốt sau pilot 100 customers. Wording điều khoản chờ luật sư VN sign-off.

---

## Tóm tắt 5 packages

| # | Package code | Device | Duration | Vệ sinh | Repair quota | Giá | Target gross margin |
|---|---|---|---:|---:|---:|---:|---:|
| 1 | `wpkg_ac_basic_1y` | AC | 12 tháng | 4 lần | 4 lần | 1,200,000 | 70% |
| 2 | `wpkg_ac_premium_1y` | AC | 12 tháng | 4 lần | 8 lần | 1,800,000 | 65% |
| 3 | `wpkg_washer_basic_1y` | Washer | 12 tháng | 2 lần | 3 lần | 800,000 | 70% |
| 4 | `wpkg_fridge_basic_1y` | Fridge | 12 tháng | 0 | 4 lần | 900,000 | 70% |
| 5 | `wpkg_water_heater_basic_1y` | Water heater | 12 tháng | 1 lần | 3 lần | 700,000 | 70% |

**Pricing rationale**:
- AC packages priced highest (most common · biggest market)
- "Premium" tier có 2x repair quota (target heavy users)
- Washer/Fridge cheaper (less frequent issues)
- Water heater small package (simple device)

---

## 1. wpkg_ac_basic_1y · Bảo trì máy lạnh cơ bản 1 năm

**Giá**: 1,200,000 VND (đã bao gồm VAT 8%)

**Quotas**:
- Vệ sinh định kỳ: **4 lần/năm** (1 lần/3 tháng max, auto-suggest)
- Sửa chữa cơ bản: **4 lần/năm** (1 lần/tháng max)

**Covered repair issues** (whitelist):
| Issue category | Mô tả | Max value per claim |
|---|---|---:|
| `capacitor` | Tụ điện máy lạnh | 300,000 |
| `gas_refill_partial` | Nạp gas bổ sung (không quá 0.5kg) | 500,000 |
| `fan_motor` | Quạt dàn lạnh/nóng | 400,000 |
| `drainage` | Đường thoát nước | 200,000 |
| `thermostat` | Cảm biến nhiệt | 250,000 |
| `remote_control` | Remote điều khiển | 150,000 |

**Exclusions** (KH phải đặt order trả phí):
- Thay block/compressor
- Thay máy mới
- Thay coil dàn nóng/lạnh
- Sửa máy đã > 10 năm tuổi
- Hư do force majeure

**Target customer**: Hộ gia đình có 1 máy lạnh sử dụng thường xuyên.

---

## 2. wpkg_ac_premium_1y · Bảo trì máy lạnh Premium 1 năm

**Giá**: 1,800,000 VND (đã bao gồm VAT 8%)

**Quotas**:
- Vệ sinh định kỳ: **4 lần/năm** (như basic)
- Sửa chữa cơ bản: **8 lần/năm** (1 lần/2 tuần max)
- Inspection: **1 lần/năm** (kiểm tra tổng thể)

**Covered repair issues**: Tất cả của Basic + thêm:
| Issue category | Mô tả | Max value per claim |
|---|---|---:|
| `gas_refill_full` | Nạp gas full (đến 1kg) | 800,000 |
| `circuit_board_minor` | Mạch điều khiển (sửa nhỏ) | 600,000 |
| `inverter_module_check` | Kiểm tra module inverter | 500,000 |

**Bonus**:
- Priority dispatch (ưu tiên giờ vàng)
- 24/7 phone support
- Free replacement filter định kỳ

**Target customer**: Hộ gia đình cao cấp, văn phòng nhỏ, máy lạnh inverter.

---

## 3. wpkg_washer_basic_1y · Bảo trì máy giặt cơ bản 1 năm

**Giá**: 800,000 VND

**Quotas**:
- Vệ sinh định kỳ: **2 lần/năm** (1 lần/6 tháng, auto-suggest)
- Sửa chữa cơ bản: **3 lần/năm**

**Covered issues**:
| Issue | Mô tả | Max value |
|---|---|---:|
| `belt_replacement` | Dây curoa | 300,000 |
| `drainage_pump` | Bơm thoát nước | 400,000 |
| `door_seal` | Roan cửa | 350,000 |
| `inlet_valve` | Van cấp nước | 250,000 |
| `vibration_dampener` | Bộ giảm chấn | 300,000 |

**Exclusions**:
- Thay motor giặt/vắt
- Thay bo mạch chính
- Thay lồng giặt

**Target customer**: Hộ gia đình dùng máy giặt thường xuyên (gia đình 4+ người).

---

## 4. wpkg_fridge_basic_1y · Bảo trì tủ lạnh cơ bản 1 năm

**Giá**: 900,000 VND

**Quotas**:
- Vệ sinh: **KHÔNG** (tủ lạnh không cần vệ sinh định kỳ)
- Sửa chữa: **4 lần/năm**
- Inspection: **1 lần/năm** (kiểm tra gas + thermostat)

**Covered issues**:
| Issue | Mô tả | Max value |
|---|---|---:|
| `gasket` | Roan cửa | 350,000 |
| `thermostat` | Cảm biến nhiệt | 400,000 |
| `defrost_heater` | Bộ xả tuyết | 450,000 |
| `fan_motor` | Quạt làm lạnh | 400,000 |
| `light_assembly` | Đèn tủ lạnh | 100,000 |
| `door_hinge` | Bản lề cửa | 250,000 |

**Exclusions**:
- Thay compressor
- Thay gas full
- Hư board nguồn chính

**Target customer**: Mọi hộ gia đình có tủ lạnh > 3 tuổi.

---

## 5. wpkg_water_heater_basic_1y · Bảo trì máy nước nóng 1 năm

**Giá**: 700,000 VND

**Quotas**:
- Vệ sinh + súc rửa: **1 lần/năm**
- Sửa chữa: **3 lần/năm**

**Covered issues**:
| Issue | Mô tả | Max value |
|---|---|---:|
| `heating_element` | Điện trở | 500,000 |
| `thermostat_safety` | Rơ-le an toàn | 350,000 |
| `pressure_valve` | Van áp suất | 300,000 |
| `anode_rod` | Thanh từ chống ăn mòn | 250,000 |

**Exclusions**:
- Thay binh chứa
- Thay máy mới
- Thay đường ống nước nhà

**Target customer**: Hộ gia đình có máy nước nóng > 2 năm.

---

## SQL seed script (DRAFT)

```sql
USE smp_catalog;

-- Package 1: AC Basic
INSERT INTO warranty_packages (package_code, name, device_category, duration_months, price, marketing_tag) 
VALUES ('wpkg_ac_basic_1y', 'Bảo trì máy lạnh cơ bản 1 năm', 'ac', 12, 1200000, 'best_seller');

SET @pkg_id = LAST_INSERT_ID;

-- Quotas
INSERT INTO warranty_package_quotas (package_id, quota_type, service_code, count_total, count_per_period, period_days, auto_suggest, suggest_interval_days) VALUES
 (@pkg_id, 'cleaning', 'svc_ac_periodic_clean', 4, 1, 90, TRUE, 90),
 (@pkg_id, 'repair_basic', NULL, 4, 1, 30, FALSE, NULL);

-- Covered issues
INSERT INTO warranty_package_covered_issues (package_id, issue_category, issue_name, max_value) VALUES
 (@pkg_id, 'capacitor', 'Tụ điện máy lạnh', 300000),
 (@pkg_id, 'gas_refill_partial', 'Nạp gas bổ sung (< 0.5kg)', 500000),
 (@pkg_id, 'fan_motor', 'Quạt dàn lạnh/nóng', 400000),
 (@pkg_id, 'drainage', 'Đường thoát nước', 200000),
 (@pkg_id, 'thermostat', 'Cảm biến nhiệt', 250000),
 (@pkg_id, 'remote_control', 'Remote điều khiển', 150000);

-- Package 2: AC Premium
INSERT INTO warranty_packages (package_code, name, device_category, duration_months, price, marketing_tag) 
VALUES ('wpkg_ac_premium_1y', 'Bảo trì máy lạnh Premium 1 năm', 'ac', 12, 1800000, 'premium');
SET @pkg_id = LAST_INSERT_ID;

INSERT INTO warranty_package_quotas (package_id, quota_type, service_code, count_total, count_per_period, period_days, auto_suggest, suggest_interval_days) VALUES
 (@pkg_id, 'cleaning', 'svc_ac_periodic_clean', 4, 1, 90, TRUE, 90),
 (@pkg_id, 'repair_basic', NULL, 8, 1, 14, FALSE, NULL),
 (@pkg_id, 'inspection', NULL, 1, 1, 365, TRUE, 180);

-- (covered issues: insert basic 6 + thêm 3 premium issues · skipped for brevity)

-- Packages 3-5: similar pattern (see tables above for full data)

-- Verification queries
SELECT 
 p.package_code, p.name, p.price,
 (SELECT COUNT(*) FROM warranty_package_quotas q WHERE q.package_id = p.id) AS quota_count,
 (SELECT COUNT(*) FROM warranty_package_covered_issues ci WHERE ci.package_id = p.id) AS covered_count
FROM warranty_packages p;
-- Expect 5 rows
```

---

## Cost analysis (cho Finance review)

### Package 1 (AC Basic 1,200,000 VND) cost breakdown

| Item | Cost per claim | Expected claims/year | Total cost |
|---|---:|---:|---:|
| Vệ sinh (per claim) | 200,000 | 4 (assume 100% used) | 800,000 |
| Repair basic (per claim) | 350,000 avg | 2 (assume 50% used) | 700,000 |
| **Total expected cost** | | | **1,500,000** |
| **Net (after VAT 8%)** | | | 1,111,111 |
| **Margin** | | | -388,889 → **LOSS!** |

⚠️ **Cost analysis cho thấy nếu KH dùng 100% quota → SMP LỖ.**

### Hai giả định để có margin 70%:

**Option A**: Quota utilization ~50% (KH chỉ dùng nửa số quotas)
- Expected cost: 750k VND
- Margin = (1.1M - 750k) / 1.1M = 33% ⚠️ vẫn thấp

**Option B**: Tăng giá hoặc giảm quota
- Giá 1.8M VND: margin 58%
- Quota 2 cleaning + 2 repair: cost ~600k → margin 46%

### Recommendations cho Finance

1. **Pricing tests**: A/B test 3 mức giá (1.0M / 1.2M / 1.5M) với 100 customers đầu
2. **Quota tuning**: bắt đầu với quota cao (như hiện tại) để attract, monitor utilization, adjust v3.7+
3. **Underwriting**: hạn chế bán cho thiết bị > 7 năm tuổi (high claim rate)
4. **Caps**: max_value per claim đã có, nhưng có thể thêm "total annual repair value cap" = 1.5M VND

### Sample customer scenarios

**Power user** (dùng hết quota):
- 4 cleaning × 200k = 800k cost cho SMP
- 4 repair × 350k = 1.4M cost
- Total cost: 2.2M vs revenue 1.2M → SMP loss 1M
- → Need careful underwriting

**Casual user** (typical):
- 2 cleaning × 200k = 400k
- 1 repair × 350k = 350k
- Total cost: 750k vs 1.2M → margin 38%

**Light user** (under-utilize):
- 1 cleaning × 200k = 200k
- 0 repair = 0
- Total cost: 200k vs 1.2M → margin 83% (HIGH)

→ Critical: monitor distribution + adjust pricing/quotas post-pilot.

---

## Open questions cho review

| # | Question | Owner | When |
|---|---|---|---|
| 1 | Tăng giá lên 1.5M VND để có margin 50%+? | Founder + Finance | Trước launch |
| 2 | "Underwriting" tự động: refuse sale cho device > 7 năm tuổi? | BA + Ops | Sprint 1 |
| 3 | "Total annual repair cap" 1.5M VND cho gói Basic? | Finance | Sprint 1 |
| 4 | Cooling-off 7 ngày có rủi ro abuse không (KH mua → claim ngay → cancel)? | Legal + Ops | Sprint 1 |
| 5 | Reset quota khi renew vs carry-over 50%? | Founder | Trước renewal flow |
| 6 | Có nên có gói "lifetime" cho device không? | Founder | v3.7+ |
| 7 | Discount cho mua bundle multi-device (vd 2 AC + 1 washer)? | Marketing | v3.6 |

---

## Marketing copy templates (DRAFT · chờ luật sư review)

### In-app catalog card (AC Basic):

```text
🏠 Bảo trì máy lạnh cơ bản 1 năm

Giá: 1,200,000 ₫ (thay vì 1,400,000 ₫ - tiết kiệm 14%)

✅ 4 lần vệ sinh định kỳ (cứ 3 tháng 1 lần)
✅ Sửa free 4 lần cho lỗi cơ bản
✅ Miễn phí thay tụ, gas bổ sung, quạt
✅ Bảo hành tay nghề thợ
⏱️ Đặt lịch trong 24h

[Mua ngay] [Xem chi tiết]
```

### Disclaimer footer:
```text
Đây là gói thuê bao dịch vụ bảo trì + sửa chữa, không phải hợp đồng bảo hiểm. 
Các trường hợp được sửa miễn phí + loại trừ chi tiết tại Điều khoản dịch vụ.
SMP không chịu trách nhiệm cho hư hỏng do thiên tai, sử dụng sai cách, 
thiết bị > 10 năm tuổi, và một số trường hợp khác (xem chi tiết).
```

---

## Implementation checklist (cho Sprint 1-3)

- [ ] Sprint 1: Schema deploy (Doc 02 §7.8)
- [ ] Sprint 1: Seed 5 packages trong staging
- [ ] Sprint 1: Finance review pricing + adjust
- [ ] Sprint 2: Build `pkg/warranty` (Doc 04 §1.15.4)
- [ ] Sprint 2: Deferred revenue cron + reconcile
- [ ] Sprint 3: Build APIs (Doc 03)
- [ ] Sprint 3: Mobile UI (browse + purchase + claim)
- [ ] Sprint 4: Admin Web (package management + claim approval queue)
- [ ] Sprint 4: Pilot launch with 100 customers
- [ ] Sprint 5+: Renewal flow + auto-suggest cron
