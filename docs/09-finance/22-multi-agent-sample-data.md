# Sample Data · Top 10 Services · Step Weights + Role Splits

> **Status**: DRAFT for review by BA + Finance + Ops teams · v3.5+  
> **Date**: 2026-05-29  
> **Owner**: Documentation team (Claude AI draft based on Vietnam home services context)  
> **Action required**: Anh + Finance + Ops review từng service, chỉnh số liệu theo thực tế trước khi seed production.

> ⚠️ **Disclaimer**: Đây là **draft proposal** dựa trên hiểu biết chung về home services tại VN. Các số liệu cụ thể (weights + splits) cần Finance + Ops review xác nhận. Không seed thẳng vào production trước khi sign-off.

---

## Bảng tóm tắt 10 services

| # | Service code | Service name | Category | N steps | Base price (VND) | Lead-only ratio |
|---|---|---|---:|---:|---:|---:|
| 1 | `svc_ac_repair` | Sửa máy lạnh | ac | 4 | 350,000 | 35% steps |
| 2 | `svc_ac_install` | Lắp đặt máy lạnh | ac | 5 | 600,000 | 0% steps |
| 3 | `svc_camera_install` | Lắp đặt camera an ninh | electric | 4 | 800,000 | 25% steps |
| 4 | `svc_washer_repair` | Sửa máy giặt | washer | 3 | 280,000 | 67% steps |
| 5 | `svc_fridge_repair` | Sửa tủ lạnh | fridge | 3 | 320,000 | 67% steps |
| 6 | `svc_water_heater_install` | Lắp máy nước nóng | electric | 4 | 450,000 | 25% steps |
| 7 | `svc_plumbing_general` | Sửa ống nước tổng quát | plumbing | 3 | 250,000 | 100% steps |
| 8 | `svc_electric_general` | Sửa điện tổng quát | electric | 3 | 220,000 | 100% steps |
| 9 | `svc_house_cleaning_deep` | Vệ sinh nhà tổng quát | cleaner | 4 | 500,000 | 0% steps |
| 10 | `svc_ac_periodic_clean` | Vệ sinh máy lạnh định kỳ | ac | 3 | 200,000 | 67% steps |

**Lead-only ratio** = số steps chỉ cần lead làm 1 mình / tổng steps. High ratio = service đơn giản hơn, ít cần team.

---

## 1. Sửa máy lạnh (`svc_ac_repair`) · 4 steps

> Đơn phổ biến nhất. Cần thợ chính có chuyên môn máy lạnh; helper hỗ trợ tháo/lắp.

| Step | Name | Weight (bps) | Weight % | Roles | Defaults |
|---|---|---:|---:|---|---|
| 1 | Khảo sát & chẩn đoán | 1500 | 15% | Lead 100% | Solo work |
| 2 | Tháo dỡ máy | 2000 | 20% | Lead 60% + Helper 40% | Cần 2 người để khiêng |
| 3 | Sửa chữa & thay linh kiện | 4000 | 40% | Lead 100% | Chỉ thợ chính làm |
| 4 | Lắp lại & test | 2500 | 25% | Lead 70% + Helper 30% | Cần 2 người lắp + test |
| | **Total** | **10000** | **100%** | | |

**Lead chịu warranty** (BR-MA-006): khắc phục lỗi máy hư sau sửa.

---

## 2. Lắp đặt máy lạnh (`svc_ac_install`) · 5 steps

> Service phức tạp · thường cần team 2-3 thợ · có specialist (thợ điện).

| Step | Name | Weight (bps) | Weight % | Roles | Defaults |
|---|---|---:|---:|---|---|
| 1 | Khảo sát vị trí | 800 | 8% | Lead 100% | Solo |
| 2 | Đục tường + đi ống | 2500 | 25% | Lead 50% + Helper 50% | Lao động chân tay |
| 3 | Lắp dàn nóng (ngoài) | 2500 | 25% | Lead 50% + Helper 50% | Cần 2 người treo dàn |
| 4 | Lắp dàn lạnh (trong) + đấu điện | 2700 | 27% | Lead 40% + Specialist electrician 40% + Helper 20% | Cần thợ điện đấu nguồn |
| 5 | Test + nghiệm thu | 1500 | 15% | Lead 60% + Helper 40% | Test + dọn dẹp |
| | **Total** | **10000** | **100%** | | |

**Specialty `electrician`**: bắt buộc cho step 4 (đấu điện 3 pha cho máy công suất lớn). Pricing thường +50k VND cho specialist.

---

## 3. Lắp đặt camera an ninh (`svc_camera_install`) · 4 steps

> Service cao cấp · cần specialist (thợ điện + thợ mạng).

| Step | Name | Weight (bps) | Weight % | Roles | Defaults |
|---|---|---:|---:|---|---|
| 1 | Khảo sát vị trí + tư vấn | 1500 | 15% | Lead 100% | Solo (chuyên môn cao) |
| 2 | Đi dây + lắp camera (vị trí) | 4000 | 40% | Lead 50% + Helper 50% | Cần thang + 2 người |
| 3 | Đấu điện + cấu hình NVR | 3000 | 30% | Lead 40% + Specialist electrician 30% + Specialist network 30% | Cần thợ điện + thợ mạng |
| 4 | Test xem app + bàn giao KH | 1500 | 15% | Lead 100% | Solo |
| | **Total** | **10000** | **100%** | | |

---

## 4. Sửa máy giặt (`svc_washer_repair`) · 3 steps

> Service đơn giản · thường 1 thợ làm hết, helper có thể có khi máy nặng.

| Step | Name | Weight (bps) | Weight % | Roles | Defaults |
|---|---|---:|---:|---|---|
| 1 | Chẩn đoán | 2000 | 20% | Lead 100% | Solo |
| 2 | Sửa chữa & thay linh kiện | 5500 | 55% | Lead 80% + Helper 20% | Helper khiêng máy giúp |
| 3 | Test + dọn dẹp | 2500 | 25% | Lead 100% | Solo |
| | **Total** | **10000** | **100%** | | |

---

## 5. Sửa tủ lạnh (`svc_fridge_repair`) · 3 steps

> Tương tự sửa máy giặt · single agent đa số case.

| Step | Name | Weight (bps) | Weight % | Roles | Defaults |
|---|---|---:|---:|---|---|
| 1 | Chẩn đoán + nạp gas test | 2500 | 25% | Lead 100% | Solo |
| 2 | Thay linh kiện / nạp gas | 5000 | 50% | Lead 80% + Helper 20% | Helper di chuyển tủ |
| 3 | Test + hoàn thiện | 2500 | 25% | Lead 100% | Solo |
| | **Total** | **10000** | **100%** | | |

---

## 6. Lắp máy nước nóng (`svc_water_heater_install`) · 4 steps

> Cần specialist plumber + electrician.

| Step | Name | Weight (bps) | Weight % | Roles | Defaults |
|---|---|---:|---:|---|---|
| 1 | Khảo sát | 1000 | 10% | Lead 100% | Solo |
| 2 | Lắp đặt + đấu nước | 4000 | 40% | Lead 50% + Specialist plumber 50% | Cần thợ ống nước |
| 3 | Đấu điện | 3000 | 30% | Lead 40% + Specialist electrician 60% | Cần thợ điện chính |
| 4 | Test + bàn giao | 2000 | 20% | Lead 100% | Solo |
| | **Total** | **10000** | **100%** | | |

---

## 7. Sửa ống nước tổng quát (`svc_plumbing_general`) · 3 steps

> Single agent · plumber chuyên môn cao.

| Step | Name | Weight (bps) | Weight % | Roles | Defaults |
|---|---|---:|---:|---|---|
| 1 | Khảo sát + báo giá | 1500 | 15% | Lead 100% | Solo |
| 2 | Sửa chữa | 6500 | 65% | Lead 100% | Solo (chuyên môn) |
| 3 | Test + dọn dẹp | 2000 | 20% | Lead 100% | Solo |
| | **Total** | **10000** | **100%** | | |

---

## 8. Sửa điện tổng quát (`svc_electric_general`) · 3 steps

> Tương tự plumbing · single agent electrician.

| Step | Name | Weight (bps) | Weight % | Roles | Defaults |
|---|---|---:|---:|---|---|
| 1 | Chẩn đoán | 2000 | 20% | Lead 100% | Solo |
| 2 | Sửa chữa | 6500 | 65% | Lead 100% | Solo |
| 3 | Test + bàn giao | 1500 | 15% | Lead 100% | Solo |
| | **Total** | **10000** | **100%** | | |

---

## 9. Vệ sinh nhà tổng quát (`svc_house_cleaning_deep`) · 4 steps

> Service cần team đông · ít chuyên môn, nhiều lao động.

| Step | Name | Weight (bps) | Weight % | Roles | Defaults |
|---|---|---:|---:|---|---|
| 1 | Chuẩn bị + di chuyển đồ | 1500 | 15% | Lead 40% + Helper 30% + Helper 30% | Cần 3 người |
| 2 | Vệ sinh phòng khách + bếp | 3500 | 35% | Lead 40% + Helper 30% + Helper 30% | 3 người cùng làm |
| 3 | Vệ sinh phòng ngủ + WC | 3500 | 35% | Lead 40% + Helper 30% + Helper 30% | 3 người |
| 4 | Hoàn thiện + nghiệm thu | 1500 | 15% | Lead 60% + Helper 40% | Lead chính kiểm tra |
| | **Total** | **10000** | **100%** | | |

**Note**: 2 helpers cho mỗi step · lead có thể điều chỉnh chỉ 1 helper nếu nhà nhỏ. Default 3 người vì service "deep cleaning" usually cho nhà 60-100m².

---

## 10. Vệ sinh máy lạnh định kỳ (`svc_ac_periodic_clean`) · 3 steps

> Service đơn giản · thường 1 thợ làm.

| Step | Name | Weight (bps) | Weight % | Roles | Defaults |
|---|---|---:|---:|---|---|
| 1 | Tháo lưới + bộ lọc | 2000 | 20% | Lead 100% | Solo |
| 2 | Vệ sinh + xịt rửa | 6000 | 60% | Lead 80% + Helper 20% | Helper hỗ trợ khi treo cao |
| 3 | Lắp lại + test | 2000 | 20% | Lead 100% | Solo |
| | **Total** | **10000** | **100%** | | |

---

## SQL seed script (DRAFT)

```sql
-- Cho v3.5 Sprint 1 task T199: BA + Ops + Finance sign-off trước khi run

USE smp_catalog;

-- Update service_steps with weights
-- (Assume service IDs 1-10 mapping to services above)

-- 1. svc_ac_repair (service_id=1)
UPDATE service_steps SET default_step_weight_bps = 1500 WHERE service_id = 1 AND sequence_no = 1;
UPDATE service_steps SET default_step_weight_bps = 2000 WHERE service_id = 1 AND sequence_no = 2;
UPDATE service_steps SET default_step_weight_bps = 4000 WHERE service_id = 1 AND sequence_no = 3;
UPDATE service_steps SET default_step_weight_bps = 2500 WHERE service_id = 1 AND sequence_no = 4;

-- 2. svc_ac_install (service_id=2)
UPDATE service_steps SET default_step_weight_bps =  800 WHERE service_id = 2 AND sequence_no = 1;
UPDATE service_steps SET default_step_weight_bps = 2500 WHERE service_id = 2 AND sequence_no = 2;
UPDATE service_steps SET default_step_weight_bps = 2500 WHERE service_id = 2 AND sequence_no = 3;
UPDATE service_steps SET default_step_weight_bps = 2700 WHERE service_id = 2 AND sequence_no = 4;
UPDATE service_steps SET default_step_weight_bps = 1500 WHERE service_id = 2 AND sequence_no = 5;

-- 3-10: ... (similar pattern, see tables above)

-- Verify: each service SUM = 10000
SELECT 
  s.service_code, 
  s.name, 
  SUM(ss.default_step_weight_bps) AS total_weight_bps
FROM services s
JOIN service_steps ss ON s.id = ss.service_id
GROUP BY s.id
HAVING total_weight_bps != 10000;
-- Expect: 0 rows · all services should sum to 10000

-- Seed service_step_role_splits
-- Example for svc_ac_repair step 2 "Tháo dỡ" (service_step_id assumed 5)
INSERT INTO service_step_role_splits 
  (service_step_id, role, specialty, default_split_bps, min_required, max_allowed)
VALUES
  (5, 'lead', NULL, 6000, 1, 1),
  (5, 'helper', NULL, 4000, 1, 2);

-- svc_ac_install step 4 "Lắp dàn lạnh + đấu điện" with electrician
-- service_step_id assumed 9
INSERT INTO service_step_role_splits VALUES
  (9, 'lead', NULL, 4000, 1, 1),
  (9, 'specialist', 'electrician', 4000, 1, 1),
  (9, 'helper', NULL, 2000, 0, 1);

-- ... (full seed for all 10 services in implementation)

-- Verify per step SUM = 10000
SELECT 
  ss.service_step_id,
  SUM(ss.default_split_bps) AS total_split_bps,
  COUNT(*) FILTER (WHERE role = 'lead') AS lead_count
FROM service_step_role_splits ss
GROUP BY ss.service_step_id
HAVING total_split_bps != 10000 OR lead_count != 1;
-- Expect: 0 rows
```

---

## Open questions cho review meeting

| # | Question | Owner | When |
|---|---|---|---|
| 1 | Pricing premium cho specialist (electrician/plumber): +50k VND? Calc thế nào? | Finance Lead | Sprint 1 |
| 2 | Service "Sửa ống nước phức tạp" có cần riêng template không (vd thay đường ống chính, 2 steps cần 2 thợ)? | Ops Manager | Sprint 1 |
| 3 | Service vệ sinh nhà · default 3 người · có quá nhiều không? Test feedback từ pilot | Ops Manager | Sprint 4 (pilot end) |
| 4 | Camera install · separate `network` specialty hay merge vào `electrician`? | Tech Lead | Sprint 1 |
| 5 | Weights có nên dynamic by service tier (gia đình vs commercial)? Hay giữ flat? | Founder + Finance | Phase 4+ |
| 6 | Trường hợp lead muốn giữ 100% (không invite helper) cho service có helper required → cho phép không? | BA Lead | Sprint 1 |

---

## Migration notes

Khi v3.5 deploy:

1. **Existing services** không có weight config → run backfill: equal weights `10000 / N` per step
2. **Existing orders** không có multi-agent: legacy `order_steps.agent_id` → migrate thành 1 row `order_step_agents` với `role='lead', split_bps=10000`
3. **Test data**: seed 10 services trên trong dev/staging trước khi prod
4. **Production cutover**: feature flag `ENABLE_MULTI_AGENT=false` ban đầu → migrate data → enable cho 10% traffic test → ramp up

Xem [MIGRATION-PLAN-v4 §Phase 1](MIGRATION-PLAN-v4.md) for detailed timeline.
