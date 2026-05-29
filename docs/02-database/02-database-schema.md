# SMP Database · ERD + Schema DDL

**Engine**: MySQL 8.0 · InnoDB · utf8mb4 · **Version**: 3.3 schema · **Audience**: DB, Dev

---

## 🗺️ Schema roadmap (READ FIRST)

Doc này chứa **3 generations** schema, mỗi cái có status riêng:

| Generation | Status | Sections | Notes |
|---|---|---|---|
| **v3.x current** | ✅ Production | 1 – 7 + 8 – 14 | Đang chạy production, mọi service đọc từ đây |
| **v4.0 Global** | 📅 Planned v4.0 | 2.5, 7.5, 11.5 | Multi-currency + UTC + i18n + Sharding |
| **v3.5+ Finance Ledger** | 🚨 DRAFT spec | **7.6** | Pattern 2 (control + subledger), dual-write planned v3.5, cutover v3.6 |

**Adoption strategy cho Finance Ledger (section 7.6)**:
1. **v3.4 NOW**: Section 7.6 là DRAFT spec, không deploy
2. **v3.5 (6 weeks)**: Add new tables + build `PostJournal` Go function + dual-write shadow mode
3. **v3.6 (4 weeks)**: Switch read path sang subledger view → drop legacy tables ở section 6 (partner_wallet_transactions etc.)
4. **v3.7+**: Continue với Kafka, etc.

Đọc chi tiết: [Doc 16 · Finance Ledger Spec](../09-finance/16-finance-ledger-spec.md) + [MIGRATION-PLAN-v4](../MIGRATION-PLAN-v4.md).

---

## 1. ERD overview · Core domain

```text
            ┌──────────────┐
            │  partners    │  v3.3
            └──────┬───────┘
                   │ 1:N
       ┌───────────┼────────────┐
       │           │            │
       ▼           ▼            ▼
  ┌────────┐  ┌────────┐  ┌──────────────┐
  │ orders │  │ agents │  │ partner_*    │
  │        │  │        │  │ (wallet,inv) │
  └───┬────┘  └────┬───┘  └──────────────┘
      │ 1:N        │
      ▼            │ N:M
  ┌──────────────┐ │   ┌──────────┐
  │ order_steps  │ │   │  skills  │
  └──────┬───────┘ ├──►│          │
         │ 1:N     │   └──────────┘
         ▼         │
  ┌──────────────┐ │   ┌────────────────┐
  │ order_step_  │ │   │ material_      │
  │  materials   │◄┴──►│   variants     │
  └──────────────┘     └────────────────┘
                              │ N:1
                              ▼
                       ┌────────────────┐
                       │ material_types │
                       └────────────────┘
                              │ N:M
                              ▼
                       ┌────────────────┐
                       │   step_boms    │
                       └────────────────┘
                              │ N:1
                              ▼
                       ┌────────────────┐
                       │     steps      │
                       └────────────────┘
                              │ N:M
                              ▼
                       ┌────────────────┐
                       │   services     │
                       └────────────────┘
```

---

## 2. Naming conventions

- Table: `snake_case`, plural (e.g., `orders`, `order_steps`)
- Column: `snake_case`, singular (e.g., `customer_id`)
- PK: `id` (BIGINT UNSIGNED AUTO_INCREMENT) hoặc business ID dạng `ord_01HX7K2M`
- FK: `<entity>_id` (e.g., `order_id`, `partner_id`)
- Timestamps: `created_at`, `updated_at`, `deleted_at` (soft delete)
- Booleans: `is_<adj>` (e.g., `is_active`, `is_required`)
- Enum: dùng VARCHAR + CHECK constraint thay vì MySQL ENUM (dễ migrate)
- Index: `idx_<table>_<columns>`
- Unique: `uniq_<table>_<columns>`

---

## 2.5 · v4.0 Conventions · Global-ready schema patterns

> **Quan trọng**: Section này định nghĩa **convention** cho v4.0 (Global). Mọi table CREATE/ALTER từ v3.5 onwards phải tuân theo. Existing v3.x tables sẽ migrate dần (xem [Doc 15 · Migration Plan](../05-ba/15-business-rules.md#-migration-plan-v40--markdown-rules--rules_engineyaml)).

### Pattern 1 · Money columns (Multi-currency)

**Rule**: Mọi cột tiền PHẢI lưu kèm currency code (ISO 4217).

```sql
-- ❌ AVOID (v3.x style — hardcode VND)
labor_price INT UNSIGNED NOT NULL COMMENT 'VND',

-- ✅ PREFER (v4.0 style — multi-currency)
labor_price_amount BIGINT NOT NULL COMMENT 'minor units (đồng VND, cent USD)',
labor_price_currency CHAR(3) NOT NULL DEFAULT 'VND' COMMENT 'ISO 4217',
```

**Rationale**:
- `BIGINT` (8 bytes, signed) thay vì `INT` (4 bytes) vì 1 USD = 100 cents, dễ tràn INT. Max BIGINT signed = 9.2 quintillion.
- `minor units` (không decimal) để tránh sai số float. VND: 1đ = 1 minor unit. USD: 1 cent = 1 minor unit. JPY: 1 yên = 1 minor unit (JPY không có decimal).
- `CHAR(3)` fixed length, dùng UPPERCASE: `VND`, `USD`, `CNY`, `THB`, `IDR`, `PHP`, `SGD`, `MYR`, `EUR`, `JPY`.
- Default `VND` cho backward compat trong giai đoạn migration.

**Naming convention cho money columns**:
- Pattern: `<purpose>_amount` + `<purpose>_currency`
- Cặp **luôn đi với nhau**. Không tách rời (vd không được có amount nhưng thiếu currency).
- Examples: `subtotal_amount` + `subtotal_currency`, `vat_amount` + `vat_currency`, `total_amount` + `total_currency`.

**Constraint**: trong cùng 1 order, các money fields phải cùng currency. Validate ở app layer:
```go
if order.SubtotalCurrency != order.VATCurrency || order.SubtotalCurrency != order.TotalCurrency {
    return ErrCurrencyMismatch
}
```

### Pattern 2 · Timestamps (UTC always)

**Rule**: Mọi timestamp lưu UTC. Có thể lưu thêm origin timezone cho audit.

```sql
-- ✅ v4.0 standard
created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
updated_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6))
                              ON UPDATE UTC_TIMESTAMP(6),
created_at_tz VARCHAR(64) DEFAULT 'Asia/Ho_Chi_Minh'
              COMMENT 'IANA timezone của user khi tạo (audit only)',
```

**Rationale**:
- `DATETIME(6)`: microsecond precision (6 digits), không có timezone implicit. Lưu trữ chính xác UTC.
- KHÔNG dùng `TIMESTAMP`: MySQL TIMESTAMP convert dựa trên `time_zone` session — gây bugs cross-region.
- `UTC_TIMESTAMP(6)`: function MySQL trả về UTC, KHÔNG phụ thuộc `time_zone` setting.
- `created_at_tz`: optional, lưu IANA tz name (`Asia/Ho_Chi_Minh`, `America/Los_Angeles`). Để audit "user ở múi giờ nào lúc đặt đơn", phục vụ analytics.

**Migration từ v3.x**:
```sql
-- v3.x dùng: created_at DATETIME(3) DEFAULT CURRENT_TIMESTAMP(3)
-- CURRENT_TIMESTAMP() phụ thuộc session timezone → KHÔNG SAFE cho global.
-- Migration: ALTER COLUMN + backfill từ existing data, assume Asia/Ho_Chi_Minh.

ALTER TABLE orders
  ADD COLUMN created_at_utc DATETIME(6) NULL AFTER created_at,
  ADD COLUMN created_at_tz VARCHAR(64) DEFAULT 'Asia/Ho_Chi_Minh' AFTER created_at_utc;

-- Backfill: assume tất cả timestamp v3.x là Asia/Ho_Chi_Minh (UTC+7)
UPDATE orders
SET created_at_utc = CONVERT_TZ(created_at, '+07:00', '+00:00')
WHERE created_at_utc IS NULL;

ALTER TABLE orders MODIFY COLUMN created_at_utc DATETIME(6) NOT NULL;
ALTER TABLE orders DROP COLUMN created_at;  -- after verification period
```

### Pattern 3 · Country code column

**Rule**: Tables chứa data per-user/per-tenant PHẢI có `country_code`.

```sql
country_code CHAR(2) NOT NULL DEFAULT 'VN'
             COMMENT 'ISO 3166-1 alpha-2 (uppercase): VN, US, CN, TH, ID, SG, MY',

-- Index for query patterns + sharding
INDEX idx_country_created (country_code, created_at_utc),
```

**Tables cần có**: `orders`, `customers`, `agents`, `partners`, `services`, `materials`, `tax_configs`, `currency_rates`.
**Tables KHÔNG cần**: `i18n_translations` (per locale), `audit_log` (cross-country logs), pure config tables.

**Why CHAR(2) instead of FK to `countries` table?**:
- Hot field, query liên tục → muốn inline, tránh JOIN
- ISO 3166 codes là standard, không đổi
- Validate ở app layer hoặc CHECK constraint:
```sql
CHECK (country_code REGEXP '^[A-Z]{2}$')
```

### Pattern 4 · Locale-aware text fields

**Rule**: Cột text user-facing (nhãn UI, mô tả) phải hỗ trợ đa ngôn ngữ qua `i18n_translations`. KHÔNG hardcode trong table.

```sql
-- ❌ AVOID (v3.x style)
CREATE TABLE services (
  ...
  name_vi VARCHAR(255),     -- hardcoded language
  description_vi TEXT,
);

-- ✅ PREFER (v4.0 style)
CREATE TABLE services (
  ...
  name_key VARCHAR(100),    -- vd "service.aircon_repair.name"
                           -- → lookup `i18n_translations(name_key, locale)`
  -- KHÔNG có description column trong table chính
);

-- 1 row = 1 (key, locale) pair
CREATE TABLE i18n_translations (
  key_path VARCHAR(255) NOT NULL,
  locale VARCHAR(10) NOT NULL,
  value TEXT NOT NULL,
  ...
);
```

**Exception**: data nhập tự do bởi user (vd `customer_note`, `address_text`) lưu trực tiếp, không qua i18n. Chỉ master data có translation.

### Pattern 5 · Sharding hint columns

Cho tương lai sharding by country: thêm `country_code` vào primary key composite hoặc dùng nó làm sharding key.

```sql
-- Logical partition by country
CREATE TABLE orders (
  ...
  PRIMARY KEY (id, country_code),
  ...
) PARTITION BY LIST COLUMNS (country_code) (
  PARTITION p_vn VALUES IN ('VN'),
  PARTITION p_us VALUES IN ('US'),
  PARTITION p_cn VALUES IN ('CN'),
  PARTITION p_other VALUES IN (DEFAULT)
);
```

Physical sharding (separate clusters) details: xem section [11.5 · Sharding strategy](#115--sharding-strategy-v40).

---

## 3. Catalog domain (DB: `smp_catalog`)

```sql
CREATE DATABASE smp_catalog CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE smp_catalog;

CREATE TABLE skills (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  skill_code VARCHAR(64) NOT NULL UNIQUE COMMENT 'ac-mechanic, washer-mechanic, etc.',
  name VARCHAR(128) NOT NULL,
  description TEXT,
  category VARCHAR(64) NOT NULL COMMENT 'mechanic, cleaner, electrician',
  status VARCHAR(16) NOT NULL DEFAULT 'active',
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  CHECK (status IN ('active','deprecated'))
) ENGINE=InnoDB;

CREATE TABLE steps (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  step_code VARCHAR(64) NOT NULL UNIQUE COMMENT 'ac-replace-capacitor, washer-clean-drum',
  name VARCHAR(255) NOT NULL,
  description TEXT,
  skill_id BIGINT UNSIGNED NOT NULL,
  min_level TINYINT UNSIGNED NOT NULL COMMENT 'L1=1, L5=5',
  duration_min SMALLINT UNSIGNED NOT NULL,
  requires_quote BOOLEAN NOT NULL DEFAULT FALSE,
  status VARCHAR(16) NOT NULL DEFAULT 'active',
  labor_price_l1 INT UNSIGNED COMMENT 'VND',
  labor_price_l2 INT UNSIGNED,
  labor_price_l3 INT UNSIGNED,
  labor_price_l4 INT UNSIGNED,
  labor_price_l5 INT UNSIGNED,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  CONSTRAINT fk_steps_skill FOREIGN KEY (skill_id) REFERENCES skills(id),
  CHECK (min_level BETWEEN 1 AND 5),
  CHECK (status IN ('active','deprecated')),
  INDEX idx_steps_skill (skill_id),
  INDEX idx_steps_status (status)
) ENGINE=InnoDB;

CREATE TABLE services (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  service_code VARCHAR(64) NOT NULL UNIQUE,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  category VARCHAR(64) NOT NULL COMMENT 'ac, washer, fridge, heater, electric, plumbing',
  base_price INT UNSIGNED NOT NULL DEFAULT 0,
  status VARCHAR(16) NOT NULL DEFAULT 'active',
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3)
) ENGINE=InnoDB;

CREATE TABLE service_steps (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  service_id BIGINT UNSIGNED NOT NULL,
  step_id BIGINT UNSIGNED NOT NULL,
  sequence_no SMALLINT UNSIGNED NOT NULL,
  is_required BOOLEAN NOT NULL DEFAULT TRUE,
  is_conditional BOOLEAN NOT NULL DEFAULT FALSE,
  condition_expr VARCHAR(255) NULL,
  CONSTRAINT fk_ss_service FOREIGN KEY (service_id) REFERENCES services(id) ON DELETE CASCADE,
  CONSTRAINT fk_ss_step FOREIGN KEY (step_id) REFERENCES steps(id),
  UNIQUE KEY uniq_service_step_seq (service_id, sequence_no)
) ENGINE=InnoDB;

-- Material BOM v3.2

CREATE TABLE material_types (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  type_code VARCHAR(64) NOT NULL UNIQUE COMMENT 'mtyp_capacitor_35uf',
  name VARCHAR(255) NOT NULL,
  category VARCHAR(64) NOT NULL COMMENT 'gas, capacitor, pump, etc.',
  unit VARCHAR(16) NOT NULL COMMENT 'cái, kg, lit, m',
  description TEXT,
  status VARCHAR(16) NOT NULL DEFAULT 'active',
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  INDEX idx_mtype_category (category)
) ENGINE=InnoDB;

CREATE TABLE material_variants (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  variant_code VARCHAR(64) NOT NULL UNIQUE,
  type_id BIGINT UNSIGNED NOT NULL,
  brand VARCHAR(128),
  model VARCHAR(128),
  wms_sku VARCHAR(64) NOT NULL UNIQUE COMMENT 'mapping to wms',
  sell_price INT UNSIGNED NOT NULL COMMENT 'VND - SMP controlled',
  cost_price_avg INT UNSIGNED COMMENT 'reference from wms',
  warranty_months SMALLINT UNSIGNED DEFAULT 0,
  spec_json JSON,
  status VARCHAR(16) NOT NULL DEFAULT 'active',
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  CONSTRAINT fk_mvar_type FOREIGN KEY (type_id) REFERENCES material_types(id),
  INDEX idx_mvar_type (type_id),
  INDEX idx_mvar_status (status)
) ENGINE=InnoDB;

CREATE TABLE step_boms (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  step_id BIGINT UNSIGNED NOT NULL,
  material_type_id BIGINT UNSIGNED NOT NULL,
  default_quantity DECIMAL(10,3) NOT NULL,
  is_required BOOLEAN NOT NULL DEFAULT TRUE,
  notes TEXT,
  CONSTRAINT fk_bom_step FOREIGN KEY (step_id) REFERENCES steps(id) ON DELETE CASCADE,
  CONSTRAINT fk_bom_type FOREIGN KEY (material_type_id) REFERENCES material_types(id),
  UNIQUE KEY uniq_step_material (step_id, material_type_id)
) ENGINE=InnoDB;
```

## 4. Order domain (DB: `smp_order`)

```sql
CREATE DATABASE smp_order CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE smp_order;

CREATE TABLE orders (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  order_code VARCHAR(32) NOT NULL UNIQUE COMMENT 'ord_01HX7K2M',
  customer_id VARCHAR(32) NOT NULL COMMENT 'ref to inside',
  service_id BIGINT UNSIGNED NOT NULL COMMENT 'ref to smp_catalog.services',
  
  -- v3.3 partner fields
  source VARCHAR(32) NOT NULL DEFAULT 'customer_direct' COMMENT 'customer_direct|partner_customer|contract',
  partner_id VARCHAR(32) NULL COMMENT 'ref to smp_partner.partners',
  partner_order_ref VARCHAR(64) NULL COMMENT 'partner internal reference',
  dispatch_visibility VARCHAR(16) NOT NULL DEFAULT 'open' COMMENT 'private|open',
  journey_mode VARCHAR(16) NOT NULL DEFAULT 'full_10steps' COMMENT 'full_10steps|minimal',
  end_customer_id VARCHAR(32) NULL COMMENT 'cus cuối, có thể trùng customer_id',
  
  -- v4.0 global fields
  country_code CHAR(2) NOT NULL DEFAULT 'VN' COMMENT 'ISO 3166-1 alpha-2',
  locale VARCHAR(10) NOT NULL DEFAULT 'vi-VN' COMMENT 'BCP 47 locale, vd vi-VN, en-US',
  
  current_stage VARCHAR(32) NOT NULL DEFAULT '01_created',
  current_agent_id BIGINT UNSIGNED NULL COMMENT 'current handling agent',
  survey_agent_id BIGINT UNSIGNED NULL,
  execution_agent_id BIGINT UNSIGNED NULL,
  
  -- Address
  address_line VARCHAR(255) NOT NULL,
  ward VARCHAR(64),
  district VARCHAR(64) NOT NULL,
  city VARCHAR(64) NOT NULL,
  lat DECIMAL(10,7),
  lng DECIMAL(10,7),
  
  -- Pricing (v4.0 multi-currency)
  -- ❌ v3.x: subtotal_labor INT UNSIGNED NOT NULL DEFAULT 0,
  -- ❌ v3.x: subtotal_material INT UNSIGNED NOT NULL DEFAULT 0,
  -- ✅ v4.0: cặp amount + currency
  subtotal_labor_amount BIGINT NOT NULL DEFAULT 0 COMMENT 'minor units',
  subtotal_material_amount BIGINT NOT NULL DEFAULT 0 COMMENT 'minor units',
  survey_fee_amount BIGINT NOT NULL DEFAULT 0,
  discount_amount BIGINT NOT NULL DEFAULT 0,
  voucher_id VARCHAR(32) NULL COMMENT 'ref to inside.vouchers',
  voucher_code VARCHAR(64) NULL,
  vat_amount BIGINT NOT NULL DEFAULT 0,
  total_amount BIGINT NOT NULL DEFAULT 0,
  currency CHAR(3) NOT NULL DEFAULT 'VND' COMMENT 'ISO 4217. All amounts in this order use this currency',
  vat_rate DECIMAL(5,4) NOT NULL DEFAULT 0.1000 COMMENT 'snapshot tại thời điểm tạo, vd 0.1000 = 10%',
  
  -- Payment
  payment_intent_id VARCHAR(64) NULL COMMENT 'ref to inside.payments',
  payment_status VARCHAR(16) NOT NULL DEFAULT 'pending',
  paid_at_utc DATETIME(6) NULL,
  
  -- Timestamps (v4.0 UTC)
  -- ❌ v3.x: scheduled_at DATETIME(3), CURRENT_TIMESTAMP(3) phụ thuộc session tz
  -- ✅ v4.0: *_utc với UTC_TIMESTAMP(6), không phụ thuộc session tz
  scheduled_at_utc DATETIME(6) NULL COMMENT 'UTC',
  arrived_at_utc DATETIME(6) NULL,
  completed_at_utc DATETIME(6) NULL,
  rated_at_utc DATETIME(6) NULL,
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  updated_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)) ON UPDATE UTC_TIMESTAMP(6),
  created_at_tz VARCHAR(64) DEFAULT 'Asia/Ho_Chi_Minh' COMMENT 'IANA tz tại thời điểm tạo (audit only)',
  
  CHECK (source IN ('customer_direct','partner_customer','contract')),
  CHECK (dispatch_visibility IN ('private','open')),
  CHECK (journey_mode IN ('full_10steps','minimal')),
  CHECK (payment_status IN ('pending','paid','refunded','failed')),
  CHECK (country_code REGEXP '^[A-Z]{2}$'),
  CHECK (currency REGEXP '^[A-Z]{3}$'),
  
  INDEX idx_orders_customer (customer_id),
  INDEX idx_orders_partner (partner_id),
  INDEX idx_orders_stage (current_stage),
  INDEX idx_orders_agent_exec (execution_agent_id),
  INDEX idx_orders_country_created (country_code, created_at_utc) COMMENT 'v4.0 sharding hint',
  INDEX idx_orders_currency (currency),
  INDEX idx_orders_district_city (district, city)
) ENGINE=InnoDB;

CREATE TABLE order_steps (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  order_id BIGINT UNSIGNED NOT NULL,
  step_id BIGINT UNSIGNED NOT NULL COMMENT 'ref to smp_catalog.steps',
  sequence_no SMALLINT UNSIGNED NOT NULL,
  agent_level TINYINT UNSIGNED COMMENT 'level of agent assigned, determines labor_price',
  labor_price INT UNSIGNED NOT NULL,
  status VARCHAR(16) NOT NULL DEFAULT 'pending',
  started_at DATETIME(3) NULL,
  completed_at DATETIME(3) NULL,
  CONSTRAINT fk_ostep_order FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
  CHECK (status IN ('pending','in_progress','completed','skipped')),
  INDEX idx_ostep_order (order_id),
  INDEX idx_ostep_status (status)
) ENGINE=InnoDB;

CREATE TABLE order_step_materials (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  order_step_id BIGINT UNSIGNED NOT NULL,
  material_variant_id BIGINT UNSIGNED NULL COMMENT 'NULL if free_form',
  free_form_name VARCHAR(255) NULL,
  quantity DECIMAL(10,3) NOT NULL,
  unit_price INT UNSIGNED NOT NULL,
  source VARCHAR(16) NOT NULL DEFAULT 'reserved' COMMENT 'reserved|free_form',
  reservation_id VARCHAR(64) NULL COMMENT 'wms reservation ref',
  verify_status VARCHAR(32) NOT NULL DEFAULT 'auto_verified',
  verified_by BIGINT UNSIGNED NULL,
  verified_at DATETIME(3) NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  CONSTRAINT fk_osmat_step FOREIGN KEY (order_step_id) REFERENCES order_steps(id) ON DELETE CASCADE,
  CHECK (source IN ('reserved','free_form')),
  CHECK (verify_status IN ('auto_verified','pending_verify','verified','rejected')),
  INDEX idx_osmat_step (order_step_id),
  INDEX idx_osmat_verify (verify_status)
) ENGINE=InnoDB;

CREATE TABLE order_stage_log (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  order_id BIGINT UNSIGNED NOT NULL,
  from_stage VARCHAR(32),
  to_stage VARCHAR(32) NOT NULL,
  triggered_by_type VARCHAR(16) NOT NULL COMMENT 'system|agent|customer|partner|ops',
  triggered_by_id VARCHAR(32),
  notes TEXT,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  CONSTRAINT fk_oslog_order FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
  INDEX idx_oslog_order_time (order_id, created_at)
) ENGINE=InnoDB;
```

## 5. Agent domain (DB: `smp_agent`)

```sql
CREATE DATABASE smp_agent CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE smp_agent;

CREATE TABLE agents (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  agent_code VARCHAR(32) NOT NULL UNIQUE COMMENT 'agt_T4K9',
  full_name VARCHAR(128) NOT NULL,
  phone VARCHAR(20) NOT NULL UNIQUE,
  email VARCHAR(128),
  cccd_number VARCHAR(20) NOT NULL UNIQUE COMMENT 'masked in app',
  cccd_issued_date DATE,
  cccd_issued_place VARCHAR(128),
  date_of_birth DATE,
  
  -- v3.3 partner link
  partner_id VARCHAR(32) NULL COMMENT 'ref to smp_partner.partners, NULL=direct SMP',
  partner_relation VARCHAR(16) NULL COMMENT 'employed|affiliated',
  payout_mode VARCHAR(16) NOT NULL DEFAULT 'passthrough' COMMENT 'passthrough|via_partner',
  
  -- Address
  home_address VARCHAR(255),
  home_district VARCHAR(64),
  home_city VARCHAR(64),
  home_lat DECIMAL(10,7),
  home_lng DECIMAL(10,7),
  
  -- Status
  status VARCHAR(16) NOT NULL DEFAULT 'pending_kyc',
  is_online BOOLEAN NOT NULL DEFAULT FALSE,
  last_online_at DATETIME(3),
  
  -- Stats (denormalized for fast read)
  total_orders_completed INT UNSIGNED NOT NULL DEFAULT 0,
  total_orders_cancelled INT UNSIGNED NOT NULL DEFAULT 0,
  avg_rating DECIMAL(3,2),
  
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  
  CHECK (status IN ('pending_kyc','active','suspended','terminated')),
  CHECK (payout_mode IN ('passthrough','via_partner')),
  CHECK (partner_relation IS NULL OR partner_relation IN ('employed','affiliated')),
  
  INDEX idx_agents_partner (partner_id),
  INDEX idx_agents_status_online (status, is_online),
  INDEX idx_agents_location (home_city, home_district)
) ENGINE=InnoDB;

CREATE TABLE agent_skills (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  agent_id BIGINT UNSIGNED NOT NULL,
  skill_code VARCHAR(64) NOT NULL COMMENT 'ref to smp_catalog.skills',
  level TINYINT UNSIGNED NOT NULL,
  verified_by BIGINT UNSIGNED NULL COMMENT 'ops user who verified',
  verified_at DATETIME(3),
  certificate_url VARCHAR(512),
  CONSTRAINT fk_askill_agent FOREIGN KEY (agent_id) REFERENCES agents(id) ON DELETE CASCADE,
  UNIQUE KEY uniq_agent_skill (agent_id, skill_code),
  CHECK (level BETWEEN 1 AND 5),
  INDEX idx_askill_skill_level (skill_code, level)
) ENGINE=InnoDB;

CREATE TABLE agent_warehouses (
  agent_id BIGINT UNSIGNED NOT NULL,
  warehouse_id VARCHAR(32) NOT NULL COMMENT 'ref to wms',
  warehouse_type VARCHAR(16) NOT NULL COMMENT 'personal|shared',
  PRIMARY KEY (agent_id, warehouse_id),
  CONSTRAINT fk_awh_agent FOREIGN KEY (agent_id) REFERENCES agents(id) ON DELETE CASCADE,
  CHECK (warehouse_type IN ('personal','shared'))
) ENGINE=InnoDB;

CREATE TABLE agent_kyc_docs (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  agent_id BIGINT UNSIGNED NOT NULL,
  doc_type VARCHAR(32) NOT NULL COMMENT 'cccd_front|cccd_back|portrait|bank_statement|certificate',
  file_url VARCHAR(512) NOT NULL,
  verified BOOLEAN NOT NULL DEFAULT FALSE,
  verified_at DATETIME(3),
  CONSTRAINT fk_akyc_agent FOREIGN KEY (agent_id) REFERENCES agents(id) ON DELETE CASCADE,
  INDEX idx_akyc_agent (agent_id)
) ENGINE=InnoDB;
```

## 6. Partner domain (DB: `smp_partner`) — v3.3

> ℹ️ **Migration note v3.5+**: Table `partner_wallet_transactions` sẽ bị **deprecated ở v3.6** khi cutover sang ledger (section 7.6). Subledger view trên `journal_lines` với `account=partner_wallet`, `owner_type=partner` thay thế. Tables `partner_invoices` và `partner_payouts` **giữ nguyên** vì là legal records riêng.

```sql
CREATE DATABASE smp_partner CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE smp_partner;

CREATE TABLE partners (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  partner_code VARCHAR(32) NOT NULL UNIQUE COMMENT 'ptn_hung_acservice',
  partner_type VARCHAR(32) NOT NULL COMMENT 'business|individual_large',
  roles JSON NOT NULL COMMENT '["customer","supplier"]',
  
  business_name VARCHAR(255) NOT NULL,
  legal_name VARCHAR(255),
  tax_code VARCHAR(32),
  rep_name VARCHAR(128) NOT NULL,
  rep_cccd VARCHAR(20),
  rep_phone VARCHAR(20) NOT NULL,
  rep_email VARCHAR(128),
  
  primary_address_line VARCHAR(255),
  primary_address_ward VARCHAR(64),
  primary_address_district VARCHAR(64),
  primary_address_city VARCHAR(64),
  
  -- Customer-side config (JSON)
  customer_config JSON COMMENT '{payment_mode, wallet_balance, credit_limit, ...}',
  
  -- Supplier-side config (JSON)
  supplier_config JSON COMMENT '{model, saas_fee_monthly, commission_percent, ...}',
  
  -- KYC
  kyc_level VARCHAR(16) NOT NULL DEFAULT 'pending' COMMENT 'pending|basic|full',
  
  status VARCHAR(32) NOT NULL DEFAULT 'pending_kyc',
  onboarded_at DATETIME(3),
  notes TEXT,
  
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  
  CHECK (partner_type IN ('business','individual_large')),
  CHECK (kyc_level IN ('pending','basic','full')),
  CHECK (status IN ('pending_kyc','active','suspended','terminated')),
  
  INDEX idx_partners_type (partner_type),
  INDEX idx_partners_status (status)
) ENGINE=InnoDB;

CREATE TABLE partner_admin_users (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  user_code VARCHAR(32) NOT NULL UNIQUE,
  partner_id BIGINT UNSIGNED NOT NULL,
  name VARCHAR(128) NOT NULL,
  email VARCHAR(128) NOT NULL UNIQUE,
  phone VARCHAR(20),
  role VARCHAR(32) NOT NULL,
  scopes JSON NOT NULL,
  last_login_at DATETIME(3),
  status VARCHAR(16) NOT NULL DEFAULT 'active',
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  CONSTRAINT fk_pau_partner FOREIGN KEY (partner_id) REFERENCES partners(id) ON DELETE CASCADE,
  CHECK (role IN ('partner_owner','partner_manager','partner_finance','partner_dispatcher')),
  CHECK (status IN ('active','suspended')),
  INDEX idx_pau_partner (partner_id)
) ENGINE=InnoDB;

CREATE TABLE partner_wallet_transactions (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  txn_code VARCHAR(32) NOT NULL UNIQUE,
  partner_id BIGINT UNSIGNED NOT NULL,
  country_code CHAR(2) NOT NULL DEFAULT 'VN',
  -- ❌ v3.x: amount INT NOT NULL COMMENT 'VND, có thể âm'
  -- ❌ v3.x: balance_after INT UNSIGNED NOT NULL
  -- ✅ v4.0: amount + currency
  amount BIGINT NOT NULL COMMENT 'minor units, có thể âm',
  currency CHAR(3) NOT NULL DEFAULT 'VND' COMMENT 'ISO 4217',
  balance_after BIGINT NOT NULL COMMENT 'minor units, snapshot sau giao dịch',
  type VARCHAR(32) NOT NULL,
  order_id VARCHAR(32) NULL COMMENT 'ref to smp_order',
  -- Currency conversion audit trail (nếu transaction là multi-currency)
  source_amount BIGINT NULL COMMENT 'amount ban đầu nếu đã convert',
  source_currency CHAR(3) NULL,
  exchange_rate DECIMAL(20,10) NULL COMMENT 'tỷ giá apply lúc convert',
  notes TEXT,
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  created_at_tz VARCHAR(64) DEFAULT 'Asia/Ho_Chi_Minh',
  CONSTRAINT fk_pwt_partner FOREIGN KEY (partner_id) REFERENCES partners(id),
  CHECK (type IN ('topup','order_debit','refund','adjustment','currency_conversion')),
  CHECK (country_code REGEXP '^[A-Z]{2}$'),
  CHECK (currency REGEXP '^[A-Z]{3}$'),
  INDEX idx_pwt_partner_time (partner_id, created_at_utc),
  INDEX idx_pwt_country_currency (country_code, currency)
) ENGINE=InnoDB;

CREATE TABLE partner_invoices (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  invoice_code VARCHAR(32) NOT NULL UNIQUE,
  partner_id BIGINT UNSIGNED NOT NULL,
  country_code CHAR(2) NOT NULL DEFAULT 'VN',
  period_start DATE NOT NULL,
  period_end DATE NOT NULL,
  -- v4.0 multi-currency
  subtotal_amount BIGINT NOT NULL DEFAULT 0 COMMENT 'minor units (pre-tax)',
  tax_amount BIGINT NOT NULL DEFAULT 0 COMMENT 'VAT/GST/Sales Tax',
  tax_rate DECIMAL(5,4) NOT NULL DEFAULT 0.1000 COMMENT 'snapshot, 0.1000 = 10%',
  tax_name VARCHAR(20) NOT NULL DEFAULT 'VAT' COMMENT 'VAT|GST|Sales Tax',
  total_amount BIGINT NOT NULL COMMENT 'subtotal + tax',
  currency CHAR(3) NOT NULL DEFAULT 'VND',
  orders_count INT UNSIGNED NOT NULL,
  status VARCHAR(16) NOT NULL DEFAULT 'draft',
  due_date DATE,
  paid_at_utc DATETIME(6),
  file_url VARCHAR(512),
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  updated_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)) ON UPDATE UTC_TIMESTAMP(6),
  CONSTRAINT fk_pinv_partner FOREIGN KEY (partner_id) REFERENCES partners(id),
  CHECK (status IN ('draft','sent','paid','overdue','cancelled')),
  CHECK (country_code REGEXP '^[A-Z]{2}$'),
  CHECK (currency REGEXP '^[A-Z]{3}$'),
  INDEX idx_pinv_partner_status (partner_id, status),
  INDEX idx_pinv_country_period (country_code, period_start)
) ENGINE=InnoDB;

CREATE TABLE partner_payouts (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  payout_code VARCHAR(32) NOT NULL UNIQUE,
  partner_id BIGINT UNSIGNED NOT NULL,
  country_code CHAR(2) NOT NULL DEFAULT 'VN',
  period_start DATE NOT NULL,
  period_end DATE NOT NULL,
  agents_count INT UNSIGNED NOT NULL,
  orders_count INT UNSIGNED NOT NULL,
  -- v4.0 multi-currency
  gross_amount BIGINT NOT NULL DEFAULT 0,
  saas_fee_deducted BIGINT NOT NULL DEFAULT 0,
  commission_deducted BIGINT NOT NULL DEFAULT 0,
  net_amount BIGINT NOT NULL,
  currency CHAR(3) NOT NULL DEFAULT 'VND',
  status VARCHAR(16) NOT NULL DEFAULT 'pending',
  paid_at_utc DATETIME(6),
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  CONSTRAINT fk_pp_partner FOREIGN KEY (partner_id) REFERENCES partners(id),
  CHECK (status IN ('pending','paid','cancelled')),
  CHECK (country_code REGEXP '^[A-Z]{2}$'),
  CHECK (currency REGEXP '^[A-Z]{3}$'),
  INDEX idx_pp_partner_status (partner_id, status),
  INDEX idx_pp_country_period (country_code, period_start)
) ENGINE=InnoDB;
```

## 7. Geo domain (DB: `smp_geo`)

```sql
CREATE DATABASE smp_geo CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE smp_geo;

CREATE TABLE provinces (
  code VARCHAR(8) PRIMARY KEY COMMENT 'VN-79=HCMC',
  name VARCHAR(128) NOT NULL,
  region VARCHAR(32) COMMENT 'Bắc/Trung/Nam'
) ENGINE=InnoDB;

CREATE TABLE districts (
  code VARCHAR(16) PRIMARY KEY,
  province_code VARCHAR(8) NOT NULL,
  name VARCHAR(128) NOT NULL,
  CONSTRAINT fk_dist_prov FOREIGN KEY (province_code) REFERENCES provinces(code),
  INDEX idx_dist_prov (province_code)
) ENGINE=InnoDB;

CREATE TABLE wards (
  code VARCHAR(24) PRIMARY KEY,
  district_code VARCHAR(16) NOT NULL,
  name VARCHAR(128) NOT NULL,
  CONSTRAINT fk_ward_dist FOREIGN KEY (district_code) REFERENCES districts(code),
  INDEX idx_ward_dist (district_code)
) ENGINE=InnoDB;

CREATE TABLE coverage_zones (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  zone_code VARCHAR(32) NOT NULL UNIQUE,
  name VARCHAR(128) NOT NULL,
  district_code VARCHAR(16) NOT NULL,
  agents_count INT UNSIGNED NOT NULL DEFAULT 0,
  surge_multiplier DECIMAL(3,2) NOT NULL DEFAULT 1.00,
  status VARCHAR(16) NOT NULL DEFAULT 'active',
  CONSTRAINT fk_cz_dist FOREIGN KEY (district_code) REFERENCES districts(code)
) ENGINE=InnoDB;
```

---

## 7.5 · v4.0 Global tables (DB: `smp_global`) · NEW

> Database mới riêng cho data global/cross-country. Không partition theo country (cần access từ mọi region). Master data tập trung.

```sql
CREATE DATABASE smp_global CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE smp_global;

-- ===========================================================================
-- 7.5.1 · countries · Master list các nước SMP operate
-- ===========================================================================
CREATE TABLE countries (
  code CHAR(2) PRIMARY KEY COMMENT 'ISO 3166-1 alpha-2',
  code3 CHAR(3) NOT NULL UNIQUE COMMENT 'ISO 3166-1 alpha-3',
  name_key VARCHAR(100) NOT NULL COMMENT 'i18n key, vd country.vn.name',
  default_currency CHAR(3) NOT NULL COMMENT 'ISO 4217',
  default_locale VARCHAR(10) NOT NULL COMMENT 'BCP 47, vd vi-VN',
  default_timezone VARCHAR(64) NOT NULL COMMENT 'IANA tz, vd Asia/Ho_Chi_Minh',
  phone_prefix VARCHAR(8) NOT NULL COMMENT 'vd +84',
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  sovereignty_region VARCHAR(32) NOT NULL DEFAULT 'global'
                     COMMENT 'global|china|eu|us · routing cluster',
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  CHECK (code REGEXP '^[A-Z]{2}$'),
  CHECK (default_currency REGEXP '^[A-Z]{3}$')
) ENGINE=InnoDB;

-- Seed
INSERT INTO countries (code, code3, name_key, default_currency, default_locale, default_timezone, phone_prefix, sovereignty_region) VALUES
('VN', 'VNM', 'country.vn.name', 'VND', 'vi-VN', 'Asia/Ho_Chi_Minh', '+84', 'global'),
('US', 'USA', 'country.us.name', 'USD', 'en-US', 'America/Los_Angeles', '+1', 'us'),
('CN', 'CHN', 'country.cn.name', 'CNY', 'zh-CN', 'Asia/Shanghai', '+86', 'china'),
('TH', 'THA', 'country.th.name', 'THB', 'th-TH', 'Asia/Bangkok', '+66', 'global'),
('ID', 'IDN', 'country.id.name', 'IDR', 'id-ID', 'Asia/Jakarta', '+62', 'global'),
('SG', 'SGP', 'country.sg.name', 'SGD', 'en-SG', 'Asia/Singapore', '+65', 'global'),
('MY', 'MYS', 'country.my.name', 'MYR', 'ms-MY', 'Asia/Kuala_Lumpur', '+60', 'global'),
('PH', 'PHL', 'country.ph.name', 'PHP', 'en-PH', 'Asia/Manila', '+63', 'global');

-- ===========================================================================
-- 7.5.2 · currencies · Master list ISO 4217 currencies SMP support
-- ===========================================================================
CREATE TABLE currencies (
  code CHAR(3) PRIMARY KEY COMMENT 'ISO 4217',
  numeric_code CHAR(3) NOT NULL COMMENT 'ISO 4217 numeric',
  name_key VARCHAR(100) NOT NULL COMMENT 'i18n key',
  symbol VARCHAR(8) NOT NULL COMMENT 'vd ₫, $, ¥',
  decimal_places TINYINT UNSIGNED NOT NULL DEFAULT 2
                COMMENT '0=VND/JPY/KRW (no decimal), 2=USD/EUR/SGD, 3=BHD/JOD',
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  CHECK (code REGEXP '^[A-Z]{3}$'),
  CHECK (decimal_places <= 4)
) ENGINE=InnoDB;

-- Seed
INSERT INTO currencies (code, numeric_code, name_key, symbol, decimal_places) VALUES
('VND', '704', 'currency.vnd.name', '₫', 0),
('USD', '840', 'currency.usd.name', '$', 2),
('CNY', '156', 'currency.cny.name', '¥', 2),
('THB', '764', 'currency.thb.name', '฿', 2),
('IDR', '360', 'currency.idr.name', 'Rp', 2),
('SGD', '702', 'currency.sgd.name', 'S$', 2),
('MYR', '458', 'currency.myr.name', 'RM', 2),
('PHP', '608', 'currency.php.name', '₱', 2),
('EUR', '978', 'currency.eur.name', '€', 2),
('JPY', '392', 'currency.jpy.name', '¥', 0);

-- ===========================================================================
-- 7.5.3 · currency_rates · Tỷ giá quy đổi (historical)
-- ===========================================================================
CREATE TABLE currency_rates (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  from_currency CHAR(3) NOT NULL,
  to_currency CHAR(3) NOT NULL,
  rate DECIMAL(20,10) NOT NULL COMMENT '1 from_currency = X to_currency',
  rate_date DATE NOT NULL,
  source VARCHAR(50) NOT NULL DEFAULT 'manual'
         COMMENT 'manual|vcb|exchangerate-api|fixer|sbv',
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  CONSTRAINT fk_cr_from FOREIGN KEY (from_currency) REFERENCES currencies(code),
  CONSTRAINT fk_cr_to FOREIGN KEY (to_currency) REFERENCES currencies(code),
  UNIQUE KEY uniq_pair_date_source (from_currency, to_currency, rate_date, source),
  INDEX idx_cr_pair_recent (from_currency, to_currency, rate_date DESC)
) ENGINE=InnoDB;

-- Note: rates được fetch hàng ngày qua cron job. Lưu lịch sử để báo cáo nhất quán.
-- Truy vấn rate latest: SELECT rate FROM currency_rates WHERE from_currency='USD' AND to_currency='VND' ORDER BY rate_date DESC LIMIT 1;

-- ===========================================================================
-- 7.5.4 · tax_configs · Cấu hình thuế suất per country + category
-- ===========================================================================
CREATE TABLE tax_configs (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  country_code CHAR(2) NOT NULL,
  category VARCHAR(50) NOT NULL DEFAULT 'default'
           COMMENT 'default|service|material|delivery|partner_invoice',
  tax_name VARCHAR(50) NOT NULL COMMENT 'VAT|GST|Sales Tax|Service Tax',
  tax_rate DECIMAL(5,4) NOT NULL COMMENT '0.1000 = 10%',
  jurisdiction VARCHAR(64) NULL
              COMMENT 'sub-region cho US state tax, NULL nếu country-wide',
  effective_from DATE NOT NULL,
  effective_to DATE NULL COMMENT 'NULL = vẫn còn hiệu lực',
  notes TEXT,
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  CONSTRAINT fk_tax_country FOREIGN KEY (country_code) REFERENCES countries(code),
  UNIQUE KEY uniq_country_cat_juri_from (country_code, category, jurisdiction, effective_from),
  INDEX idx_tax_lookup (country_code, category, effective_from)
) ENGINE=InnoDB;

-- Seed
INSERT INTO tax_configs (country_code, category, tax_name, tax_rate, effective_from) VALUES
('VN', 'default', 'VAT', 0.1000, '2024-01-01'),
('VN', 'service', 'VAT', 0.0800, '2024-07-01'),  -- VN giảm VAT 2% cho dịch vụ
('SG', 'default', 'GST', 0.0900, '2024-01-01'),
('TH', 'default', 'VAT', 0.0700, '2024-01-01'),
('US', 'default', 'Sales Tax', 0.0725, '2024-01-01'),  -- California base rate, jurisdiction granular sau
('CN', 'default', 'VAT', 0.0600, '2024-01-01');

-- ===========================================================================
-- 7.5.5 · i18n_translations · Đa ngôn ngữ động (open dictionary)
-- ===========================================================================
CREATE TABLE i18n_translations (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  key_path VARCHAR(255) NOT NULL
           COMMENT 'dot notation, vd order.status.in_progress, service.aircon.name',
  locale VARCHAR(10) NOT NULL COMMENT 'BCP 47, vd vi-VN, en-US',
  value TEXT NOT NULL,
  context VARCHAR(50) NULL
          COMMENT 'ui|notification|email|sms|invoice — nullable cho generic',
  is_html BOOLEAN NOT NULL DEFAULT FALSE COMMENT 'TRUE nếu value có HTML tags',
  notes TEXT COMMENT 'description cho translator',
  updated_by VARCHAR(64) NULL,
  updated_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)) ON UPDATE UTC_TIMESTAMP(6),
  UNIQUE KEY uniq_key_locale_ctx (key_path, locale, context),
  INDEX idx_i18n_locale (locale),
  INDEX idx_i18n_key (key_path)
) ENGINE=InnoDB;

-- Seed sample
INSERT INTO i18n_translations (key_path, locale, value) VALUES
('order.status.created', 'vi-VN', 'Đã tạo'),
('order.status.created', 'en-US', 'Created'),
('order.status.created', 'th-TH', 'สร้างแล้ว'),
('order.status.in_progress', 'vi-VN', 'Đang thực hiện'),
('order.status.in_progress', 'en-US', 'In progress'),
('order.status.completed', 'vi-VN', 'Hoàn thành'),
('order.status.completed', 'en-US', 'Completed'),
('country.vn.name', 'vi-VN', 'Việt Nam'),
('country.vn.name', 'en-US', 'Vietnam'),
('currency.vnd.name', 'vi-VN', 'Đồng Việt Nam'),
('currency.vnd.name', 'en-US', 'Vietnamese Dong');

-- Lookup pattern (with fallback):
-- 1. Try exact locale: WHERE key_path='X' AND locale='zh-HK'
-- 2. Try language only: WHERE key_path='X' AND locale LIKE 'zh-%' LIMIT 1
-- 3. Fallback en-US: WHERE key_path='X' AND locale='en-US'
-- 4. Return key_path itself nếu không có gì (dev debug)
```

### Cách query với fallback chain (Go pseudo-code)

```go
func Translate(key, locale string) string {
    // Try exact match
    if v := cache.Get(key, locale); v != "" {
        return v
    }
    // Try language only (vd zh-HK → zh)
    if lang := strings.Split(locale, "-")[0]; lang != locale {
        if v := cache.GetByLangPrefix(key, lang); v != "" {
            return v
        }
    }
    // Fallback default
    if v := cache.Get(key, "en-US"); v != "" {
        return v
    }
    // Last resort: return key for debugging
    return "[" + key + "]"
}
```

---

## 7.6 · v3.5+ Finance Ledger schema (DB: `smp_finance`) · DRAFT

> **🚨 IMPORTANT · v3.5+ specification, NOT YET DEPLOYED**
>
> Section này document schema **Pattern 2 (control account + subledger dimension)** từ [Doc 16 · Finance Ledger Spec](../09-finance/16-finance-ledger-spec.md). Adoption strategy:
> - **v3.4 (NOW)**: Schema cũ ở section 6 (`partner_wallet_transactions`, `partner_invoices`, `partner_payouts`) đang production
> - **v3.5 (planned 6 weeks)**: Build new tables below + dual-write shadow mode
> - **v3.6 (planned 4 weeks)**: Switch read path sang subledger view, drop old tables
>
> Schema này thay thế design 3-bảng-ví/điểm/earnings hiện tại bằng **single source of truth = `journal_lines`** với subledger filter.

```sql
CREATE DATABASE smp_finance CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE smp_finance;
```

### 7.6.1 · `accounts` · Chart of Accounts

Control accounts — UNIQUE theo `(code, currency)`. **KHÔNG tạo 1 row per customer/agent** (đó là anti-pattern).

```sql
CREATE TABLE accounts (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  code VARCHAR(32) NOT NULL COMMENT 'cash_gateway, customer_wallet, agent_payable...',
  currency CHAR(3) NOT NULL DEFAULT 'VND' COMMENT 'ISO 4217',
  type VARCHAR(12) NOT NULL COMMENT 'ASSET|LIABILITY|REVENUE|EXPENSE|EQUITY',
  normal_side CHAR(2) NOT NULL COMMENT 'Dr|Cr',
  description VARCHAR(255),
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  UNIQUE KEY uniq_code_currency (code, currency),
  CHECK (type IN ('ASSET','LIABILITY','REVENUE','EXPENSE','EQUITY')),
  CHECK (normal_side IN ('Dr','Cr')),
  CHECK (currency REGEXP '^[A-Z]{3}$')
) ENGINE=InnoDB;

-- Seed chuẩn theo Doc 16 §1
INSERT INTO accounts (code, currency, type, normal_side, description) VALUES
-- ASSETS (Dr normal)
('cash_gateway',           'VND', 'ASSET',     'Dr', 'Clearing tiền cổng đã confirm, chưa về bank'),
('cash_bank',              'VND', 'ASSET',     'Dr', 'Tiền thật trong ngân hàng SMP'),
('agent_cod_receivable',   'VND', 'ASSET',     'Dr', 'Thợ giữ cash của SMP (COD thu, chưa remit)'),
('agent_debt',             'VND', 'ASSET',     'Dr', 'Thợ nợ lại SMP (clawback / COD short)'),
-- LIABILITIES (Cr normal)
('customer_wallet',        'VND', 'LIABILITY', 'Cr', 'SMP nợ KH (subledger theo customer)'),
('agent_payable',          'VND', 'LIABILITY', 'Cr', 'SMP nợ thợ - earnings (subledger theo agent)'),
('partner_wallet',         'VND', 'LIABILITY', 'Cr', 'SMP nợ partner (prepaid)'),
('cod_clearing',           'VND', 'LIABILITY', 'Cr', 'Suspense giữa COD_COLLECTED → SETTLED'),
('refund_payable',         'VND', 'LIABILITY', 'Cr', 'Hoàn đã duyệt, chờ chi về nguồn gốc'),
('loyalty_liability',      'VND', 'LIABILITY', 'Cr', 'Giá trị điểm đang lưu hành'),
('service_guarantee_reserve','VND','LIABILITY','Cr', 'Quỹ bảo hành/đảm bảo dịch vụ (KHÔNG dùng "insurance")'),
('vat_payable',            'VND', 'LIABILITY', 'Cr', 'VAT phải nộp (rate từ tax_configs)'),
-- REVENUE (Cr normal)
('revenue_commission',     'VND', 'REVENUE',   'Cr', 'Doanh thu commission SMP'),
-- EXPENSES (Dr normal)
('gateway_fee_expense',    'VND', 'EXPENSE',   'Dr', 'Phí cổng SMP gánh'),
('marketing_expense',      'VND', 'EXPENSE',   'Dr', 'Chi phí điểm/referral'),
('rounding_adjustment',    'VND', 'EXPENSE',   'Dr', 'Hấp thụ chênh làm tròn cross-entry');
```

### 7.6.2 · `journal_entries` · Header (append-only + hash chain)

```sql
CREATE TABLE journal_entries (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  ref_type VARCHAR(24) NOT NULL COMMENT 'settlement|refund|payout|topup|adjustment...',
  ref_id VARCHAR(64) NOT NULL COMMENT 'ID của object reference',
  idempotency_key VARCHAR(80) NOT NULL UNIQUE COMMENT 'producer-side dedup',
  currency CHAR(3) NOT NULL DEFAULT 'VND' COMMENT '1 entry = 1 currency. FX riêng entry',
  memo VARCHAR(255),
  
  -- Tamper resistance: hash chain
  prev_hash CHAR(64) NULL COMMENT 'SHA-256 of previous entry row_hash',
  row_hash  CHAR(64) NOT NULL COMMENT 'SHA-256(prev_hash || canonical_payload)',
  
  posted_by VARCHAR(64) NULL COMMENT 'user_id of admin manual posting, NULL nếu system',
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  
  INDEX idx_je_ref (ref_type, ref_id),
  INDEX idx_je_created (created_at_utc),
  CHECK (currency REGEXP '^[A-Z]{3}$')
) ENGINE=InnoDB;
```

**Append-only enforcement**: trigger BEFORE UPDATE/DELETE → REJECT:
```sql
CREATE TRIGGER trg_je_no_update BEFORE UPDATE ON journal_entries
FOR EACH ROW SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'journal_entries is append-only';
CREATE TRIGGER trg_je_no_delete BEFORE DELETE ON journal_entries
FOR EACH ROW SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'journal_entries is append-only';
```

### 7.6.3 · `journal_lines` · Subledger dimension (Pattern 2)

```sql
CREATE TABLE journal_lines (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  entry_id BIGINT UNSIGNED NOT NULL,
  account_id BIGINT UNSIGNED NOT NULL,
  
  -- SUBLEDGER DIMENSION: owner sống ở line, không đẻ account row
  owner_type VARCHAR(12) NULL COMMENT 'customer|agent|partner|NULL(control account only)',
  owner_id   BIGINT UNSIGNED NULL,
  
  -- Money (CHÍNH XÁC một bên Dr hoặc Cr, không cả 2)
  debit_minor  BIGINT NOT NULL DEFAULT 0 COMMENT 'minor units',
  credit_minor BIGINT NOT NULL DEFAULT 0,
  
  description VARCHAR(255),
  
  CONSTRAINT fk_jl_entry FOREIGN KEY (entry_id) REFERENCES journal_entries(id),
  CONSTRAINT fk_jl_account FOREIGN KEY (account_id) REFERENCES accounts(id),
  CONSTRAINT chk_nonneg   CHECK (debit_minor >= 0 AND credit_minor >= 0),
  CONSTRAINT chk_one_side CHECK ((debit_minor > 0) <> (credit_minor > 0)),
  
  INDEX idx_jl_entry (entry_id),
  INDEX idx_jl_account_owner (account_id, owner_type, owner_id) COMMENT 'subledger query',
  INDEX idx_jl_owner (owner_type, owner_id) COMMENT 'all activity per owner'
) ENGINE=InnoDB;
```

**Balance enforcement · 3 lớp (xem Doc 16 §2)**:

1. **Single writer**: mọi posting đi qua hàm `PostJournal(entry, lines[])` trong Go (pkg/finance). Không INSERT thẳng từ service nào khác.

2. **Deferred constraint trigger**: validate `Σdebit = Σcredit` per `entry_id` trước COMMIT.

```sql
DELIMITER //
CREATE TRIGGER trg_jl_balanced_check
AFTER INSERT ON journal_lines
FOR EACH ROW
BEGIN
  DECLARE total_dr, total_cr BIGINT;
  SELECT SUM(debit_minor), SUM(credit_minor)
    INTO total_dr, total_cr
    FROM journal_lines
    WHERE entry_id = NEW.entry_id;
  -- Note: kiểm tra final check sẽ chạy ở application layer (PostJournal)
  -- trigger này log warning nếu detect imbalance trong batch insert
  -- (final commit check ở app layer transaction)
END //
DELIMITER ;
```

3. **Daily reconcile job**: trial balance toàn hệ thống `SUM(all debit) = SUM(all credit)` + so subledger balance vs control.

**Subledger view** (helpful view, không phải table):

```sql
CREATE VIEW v_subledger_balance AS
SELECT
  a.code AS account_code,
  jl.owner_type,
  jl.owner_id,
  SUM(jl.debit_minor) - SUM(jl.credit_minor) AS balance_minor,
  jl.entry_id  -- max(entry_id) cho latest activity
FROM journal_lines jl
JOIN accounts a ON a.id = jl.account_id
GROUP BY a.code, jl.owner_type, jl.owner_id;

-- Example query: customer X có ví bao nhiêu?
-- SELECT -balance_minor AS wallet_balance  -- credit-normal → flip sign
-- FROM v_subledger_balance
-- WHERE account_code = 'customer_wallet' AND owner_type = 'customer' AND owner_id = 12345;
```

### 7.6.4 · `outbox` · Transactional outbox pattern

```sql
CREATE TABLE outbox (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  event_id CHAR(36) NOT NULL UNIQUE COMMENT 'UUID, idempotency key',
  event_name VARCHAR(48) NOT NULL COMMENT 'SettlementSettled, CODCollected...',
  event_version VARCHAR(8) NOT NULL DEFAULT '1.0',
  
  aggregate_type VARCHAR(24) NOT NULL COMMENT 'settlement, order, customer',
  aggregate_id VARCHAR(64) NOT NULL,
  ordering_key VARCHAR(64) NULL COMMENT 'preserve order per key (vd order_id)',
  
  payload JSON NOT NULL,
  
  status VARCHAR(12) NOT NULL DEFAULT 'pending' COMMENT 'pending|dispatched|failed',
  attempts INT NOT NULL DEFAULT 0,
  next_retry_at_utc DATETIME(6) NULL,
  last_error TEXT NULL,
  
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  dispatched_at_utc DATETIME(6) NULL,
  
  CHECK (status IN ('pending','dispatched','failed')),
  INDEX idx_outbox_status_retry (status, next_retry_at_utc) COMMENT 'dispatcher polling',
  INDEX idx_outbox_aggregate (aggregate_type, aggregate_id),
  INDEX idx_outbox_created (created_at_utc)
) ENGINE=InnoDB;
```

**Producer pattern**: ghi `outbox` row trong cùng transaction với business data:

```sql
START TRANSACTION;

-- Business state change (vd settlement state SETTLED)
UPDATE settlements SET state = 'SETTLED' WHERE id = ?;

-- Journal entries (ledger)
INSERT INTO journal_entries (...) VALUES (...);
INSERT INTO journal_lines (...) VALUES (...), (...);

-- Outbox event (cùng transaction → atomic guarantee)
INSERT INTO outbox (event_id, event_name, aggregate_type, aggregate_id, ordering_key, payload)
VALUES (UUID(), 'SettlementSettled', 'settlement', ?, ?, JSON_OBJECT(...));

COMMIT;
```

**Dispatcher loop** (separate worker):
```sql
SELECT * FROM outbox
WHERE status = 'pending' AND (next_retry_at_utc IS NULL OR next_retry_at_utc <= NOW())
ORDER BY id
LIMIT 100;
-- → gọi consumer in-process → success: UPDATE status='dispatched'
-- → fail: INCREMENT attempts, SET next_retry_at_utc với exponential backoff
-- → > 5 attempts: INSERT vào dead_letters + UPDATE outbox status='failed'
```

Chi tiết: xem [Doc 18 · Event Catalog](../09-finance/18-event-catalog.md).

### 7.6.5 · `dead_letters` · DLQ

```sql
CREATE TABLE dead_letters (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  event_id CHAR(36) NOT NULL COMMENT 'từ outbox.event_id',
  consumer VARCHAR(32) NOT NULL COMMENT 'service name nào fail xử lý',
  attempts INT NOT NULL,
  error TEXT NOT NULL,
  payload JSON NOT NULL,
  
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  replayed_at_utc DATETIME(6) NULL COMMENT 'NULL = pending manual review',
  replayed_by VARCHAR(64) NULL,
  
  INDEX idx_dl_event (event_id),
  INDEX idx_dl_pending (replayed_at_utc) COMMENT 'find unresolved'
) ENGINE=InnoDB;
```

Admin UI sẽ list `dead_letters WHERE replayed_at_utc IS NULL` cho manual review.

### 7.6.6 · `tax_configs` REVISED · Use basis points (bps)

> ⚠️ **Conflict với section 7.5.4 hiện tại**: section đó dùng `tax_rate DECIMAL(5,4)`. Doc 16 đề xuất dùng **basis points** (integer) để avoid float precision issues.
>
> **Migration**: v3.5 add column `rate_bps`, dual-write, v3.6 drop `tax_rate` DECIMAL column.

```sql
-- v3.5 ALTER (additive, backward compatible)
ALTER TABLE tax_configs
  ADD COLUMN rate_bps INT NULL COMMENT 'basis points: 800=8%, 1000=10%' AFTER tax_rate;

-- Backfill from existing
UPDATE tax_configs SET rate_bps = ROUND(tax_rate * 10000) WHERE rate_bps IS NULL;

-- v3.6 ALTER (after verification)
ALTER tax_configs MODIFY COLUMN rate_bps INT NOT NULL;
-- Eventually DROP COLUMN tax_rate after grace period
```

**Final shape (v3.6+):**

```sql
CREATE TABLE tax_configs (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  country_code CHAR(2) NOT NULL,
  category VARCHAR(50) NOT NULL DEFAULT 'default',
  tax_name VARCHAR(50) NOT NULL COMMENT 'VAT|GST|Sales Tax|Service Tax',
  rate_bps INT NOT NULL COMMENT '800=8%, 1000=10%, 900=9%',  -- NEW
  jurisdiction VARCHAR(64) NULL,
  effective_from DATE NOT NULL,
  effective_to DATE NULL,
  notes TEXT,
  created_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  CONSTRAINT fk_tax_country FOREIGN KEY (country_code) REFERENCES countries(code),
  UNIQUE KEY uniq_country_cat_juri_from (country_code, category, jurisdiction, effective_from),
  INDEX idx_tax_lookup (country_code, category, effective_from),
  CHECK (rate_bps >= 0 AND rate_bps <= 10000)
) ENGINE=InnoDB;

-- Seed
INSERT INTO tax_configs (country_code, category, tax_name, rate_bps, effective_from) VALUES
('VN', 'default', 'VAT', 1000, '2024-01-01'),
('VN', 'service', 'VAT', 800,  '2024-07-01'),  -- VN giảm VAT 2%
('VN', 'service', 'VAT', 1000, '2027-01-01'),  -- back to 10% sau khi hết giảm 2026
('SG', 'default', 'GST', 900,  '2024-01-01'),
('TH', 'default', 'VAT', 700,  '2024-01-01'),
('CN', 'default', 'VAT', 600,  '2024-01-01');
```

### 7.6.7 · Migration path từ schema cũ

Trong v3.5 (dual-write phase), schema **CŨ và MỚI cùng tồn tại**. Code mới ghi vào cả 2:

| v3.4 cũ (production) | v3.5 mới (shadow) | v3.6 (cutover) |
|---|---|---|
| `partner_wallet_transactions` (in `smp_partner`) | journal_lines với `account=partner_wallet`, `owner_type=partner` | Drop old |
| `partner_invoices` (giữ - đây là legal record) | (giữ — không thay thế) | (giữ) |
| `partner_payouts` (giữ - đây là payment record) | (giữ — không thay thế) | (giữ) |
| (chưa có points/loyalty) | journal_lines với `account=loyalty_liability` | (mới) |
| (chưa có agent_earnings) | journal_lines với `account=agent_payable`, `owner_type=agent` | (mới) |

**Daily reconcile job** (v3.5 phase):
```python
# Pseudo
for partner in active_partners:
    old_balance = sum(partner_wallet_transactions.amount where partner_id=partner.id)
    new_balance = subledger_view(account='partner_wallet', owner_type='partner', owner_id=partner.id)
    
    if abs(old_balance - new_balance) > 1:  # 1 VND tolerance
        alert_finance(f"Drift detected: partner {partner.id}")
        log_to_reconcile_table(...)
```

Nếu chạy 30 ngày zero drift → confident cutover ở v3.6.

### 7.6.8 · Operational notes

- **Performance**: 1M orders × ~5 lines/order = 5M `journal_lines` rows/year. Index `idx_jl_account_owner` cover query subledger balance. Materialized balance table optional nếu cần (hourly refresh).
- **Backup**: `journal_*` tables = critical financial data → daily backup + 10-year retention (theo NĐ 123/2020 + NĐ 70/2025).
- **Audit trail**: hash chain ở `journal_entries.row_hash` cho tamper detection. Cron job verify hash chain integrity daily.
- **DR**: cross-region backup encrypted (xem [Doc 11 INC-012](../07-devops/11-runbook-incidents.md) for cross-region replication).

---

## 7.7 · v3.5+ Multi-agent step assignment · DRAFT

> 🆕 **New in v3.5+**: Hệ thống hiện tại assign 1 agent cho cả order. Yêu cầu mới: mỗi **step** trong order có thể có **nhiều agents** với role khác nhau (lead/helper/specialist) và **tỉ lệ chia tiền riêng**. Section này định nghĩa schema extension cần thiết.

### 7.7.1 · Design principles

1. **1 service template = N steps cố định** (định nghĩa trước trong `service_steps`)
2. **1 step có exactly 1 lead** + 0..N helpers/specialists
3. **Step weight + role split = hybrid**: service-level default + lead có thể override per order
4. **Warranty liability**: chỉ lead chịu, helpers không liable
5. **Earnings calc per step**: khi step completed → post journal entries riêng cho từng agent

### 7.7.2 · ALTER existing tables

#### `smp_catalog.service_steps` (add weight column)

```sql
ALTER TABLE smp_catalog.service_steps
  ADD COLUMN default_step_weight_bps INT UNSIGNED NOT NULL DEFAULT 0
    COMMENT 'Default weight of this step in basis points · sum per service = 10000 (100%)',
  ADD CONSTRAINT chk_weight_bps CHECK (default_step_weight_bps BETWEEN 0 AND 10000);

-- App-level validation: SUM(default_step_weight_bps) per service_id MUST = 10000
-- Migration: backfill existing services with EQUAL weights (10000 / N steps)
```

#### `smp_order.order_steps` (add multi-agent fields)

```sql
ALTER TABLE smp_order.order_steps
  ADD COLUMN step_weight_bps INT UNSIGNED NOT NULL DEFAULT 0
    COMMENT 'Effective weight for this order_step (default or overridden)',
  ADD COLUMN step_revenue BIGINT UNSIGNED NULL
    COMMENT 'Computed: order.total × step_weight_bps / 10000 · set when step starts',
  ADD COLUMN lead_agent_id BIGINT UNSIGNED NULL
    COMMENT 'Denormalized FK to order_step_agents WHERE role=lead · for fast lookup',
  ADD COLUMN weight_overridden BOOLEAN DEFAULT FALSE,
  ADD COLUMN weight_override_by BIGINT UNSIGNED NULL,
  ADD COLUMN weight_override_at_utc DATETIME(3) NULL,
  ADD INDEX idx_ostep_lead (lead_agent_id);
```

### 7.7.3 · NEW table · `service_step_role_splits` (template)

```sql
CREATE TABLE smp_catalog.service_step_role_splits (
  id                  BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  service_step_id     BIGINT UNSIGNED NOT NULL COMMENT 'FK to service_steps.id',
  role                VARCHAR(16) NOT NULL COMMENT 'lead | helper | specialist',
  specialty           VARCHAR(64) NULL COMMENT 'electrician, plumber, NULL for generic helper',
  default_split_bps   INT UNSIGNED NOT NULL COMMENT '7000 = 70%',
  min_required        TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT 'min agents for this role · lead always 1',
  max_allowed         TINYINT UNSIGNED NOT NULL DEFAULT 1,
  created_at_utc      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  CONSTRAINT fk_ssrs_step FOREIGN KEY (service_step_id) REFERENCES service_steps(id) ON DELETE CASCADE,
  CONSTRAINT chk_split_bps CHECK (default_split_bps BETWEEN 0 AND 10000),
  CONSTRAINT chk_role CHECK (role IN ('lead','helper','specialist')),
  UNIQUE KEY uniq_step_role_specialty (service_step_id, role, specialty),
  INDEX idx_ssrs_step (service_step_id)
) ENGINE=InnoDB
  COMMENT='Template: role splits per service_step · SUM(default_split_bps) per service_step_id MUST = 10000';

-- App-level validation:
--   - Exactly 1 row WHERE role='lead' per service_step_id
--   - SUM(default_split_bps) per service_step_id = 10000
--   - role='lead' MUST have specialty IS NULL
```

### 7.7.4 · NEW table · `order_step_agents` (effective assignment)

```sql
CREATE TABLE smp_order.order_step_agents (
  id                  BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  order_step_id       BIGINT UNSIGNED NOT NULL,
  agent_id            BIGINT UNSIGNED NOT NULL,
  role                VARCHAR(16) NOT NULL,
  specialty           VARCHAR(64) NULL,
  split_bps           INT UNSIGNED NOT NULL COMMENT 'Effective split (default or override)',
  is_override         BOOLEAN NOT NULL DEFAULT FALSE,
  override_by         BIGINT UNSIGNED NULL COMMENT 'agent_id của lead who overrode',
  override_at_utc     DATETIME(3) NULL,
  status              VARCHAR(16) NOT NULL DEFAULT 'assigned',
  assigned_at_utc     DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  accepted_at_utc     DATETIME(3) NULL,
  completed_at_utc    DATETIME(3) NULL,
  amount_earned       BIGINT UNSIGNED NULL COMMENT 'Computed when step.status=completed',
  amount_currency     CHAR(3) NOT NULL DEFAULT 'VND',
  CONSTRAINT fk_osa_step FOREIGN KEY (order_step_id) REFERENCES order_steps(id) ON DELETE CASCADE,
  CONSTRAINT chk_osa_role CHECK (role IN ('lead','helper','specialist')),
  CONSTRAINT chk_osa_split CHECK (split_bps BETWEEN 0 AND 10000),
  CONSTRAINT chk_osa_status CHECK (status IN ('assigned','accepted','rejected','in_progress','completed','cancelled')),
  UNIQUE KEY uniq_step_agent (order_step_id, agent_id),
  INDEX idx_osa_agent_status (agent_id, status),
  INDEX idx_osa_role (order_step_id, role)
) ENGINE=InnoDB
  COMMENT='N-to-N · multiple agents per step · SUM(split_bps) per order_step_id MUST = 10000';

-- App-level validation (Doc 15 BR-MA-002, BR-MA-003):
--   - Exactly 1 row WHERE role='lead' per order_step_id (before step can transition to in_progress)
--   - SUM(split_bps) per order_step_id MUST = 10000 (before step can transition to completed)
--   - role='lead' MUST have specialty IS NULL
--   - status transitions: assigned → accepted → in_progress → completed (or cancelled/rejected)
```

### 7.7.5 · Earnings query view

```sql
CREATE VIEW v_agent_step_earnings AS
SELECT
  osa.agent_id,
  osa.order_step_id,
  os.order_id,
  osa.role,
  osa.split_bps,
  os.step_revenue,
  osa.amount_earned,
  osa.amount_currency,
  os.completed_at AS step_completed_at,
  os.status AS step_status
FROM smp_order.order_step_agents osa
JOIN smp_order.order_steps os ON osa.order_step_id = os.id
WHERE osa.status = 'completed' AND os.status = 'completed';
```

### 7.7.6 · Example scenario

Service "Sửa máy lạnh", order total = 1,000,000 VND, 4 steps:

```text
service_steps (template):
  step 1 "Tháo dỡ"      default_step_weight_bps = 3000 (30%)
  step 2 "Vận chuyển"   default_step_weight_bps = 2000 (20%)
  step 3 "Lắp đặt"      default_step_weight_bps = 3000 (30%)
  step 4 "Hoàn thiện"   default_step_weight_bps = 2000 (20%)
  SUM = 10000 ✓

service_step_role_splits (template for step 3 "Lắp đặt"):
  role=lead,        specialty=NULL,         default_split_bps=5000 (50%)
  role=specialist,  specialty=electrician,  default_split_bps=5000 (50%)
  SUM = 10000 ✓

order_steps (actual for this order):
  order_step_id=101, step_no=3, step_weight_bps=3000, step_revenue=300000, lead_agent_id=A

order_step_agents (3 agents on step 3):
  agent_id=A, role=lead,       split_bps=4000 (overridden 50→40), is_override=TRUE
  agent_id=C, role=specialist, split_bps=4000 (overridden 50→40), is_override=TRUE
  agent_id=D, role=helper,     split_bps=2000 (new helper added by lead)
  SUM = 10000 ✓

Earnings:
  A: 300000 × 4000/10000 = 120,000 VND
  C: 300000 × 4000/10000 = 120,000 VND
  D: 300000 × 2000/10000 =  60,000 VND
```

### 7.7.7 · Migration plan

| Phase | Action |
|---|---|
| v3.5 Sprint 1 | Add `default_step_weight_bps` to `service_steps` + backfill equal weights (10000/N) |
| v3.5 Sprint 1 | Create `service_step_role_splits` table + seed default rows (1 lead 100% for all existing service_steps) |
| v3.5 Sprint 2 | ALTER `order_steps` add multi-agent columns + backfill from existing single-agent data |
| v3.5 Sprint 2 | Create `order_step_agents` + backfill from `order_steps.agent_id` → 1 row role='lead' split_bps=10000 |
| v3.5 Sprint 3 | Build APIs assign/remove agent + override split (xem [Doc 03 §multi-agent](../03-backend/03-api-contract.md)) |
| v3.6 | Service templates UI cho BA team config weights + role splits |
| v3.6.5 | Drop `order_steps.agent_id` (replaced by `lead_agent_id` + `order_step_agents` table) |

### 7.7.8 · Operational notes

- **Performance**: with N agents per step + M steps per order → cardinality ~3-5× `order_steps`. Acceptable for pilot (~300 orders/day × 4 steps × 2 agents = 2,400 rows/day).
- **Concurrent writes**: lead override + helper accept can race → use optimistic locking with `version` column OR row-level lock on `order_steps.id` before mutating.
- **Validation**: SUM check enforced trong app layer (Go function `ValidateOrderStepSplits`), NOT in DB constraint (DB cannot easily enforce SUM across rows).
- **Reconciliation**: daily job verify `SUM(order_step_agents.amount_earned) per order_step_id ≈ step_revenue` (small drift OK do rounding).

---

## 7.8 · v3.5+ Warranty packages (maintenance subscriptions) · DRAFT

> 🆕 **New in v3.5+**: 2 loại warranty:
> 1. **Embedded warranty** · cấp tự động khi order completed (1 tháng default) · đã có ở v3.4
> 2. **Maintenance subscription packages** · KH mua gói trước (vd 12 tháng) cho 1 thiết bị, được vệ sinh định kỳ + sửa các issues whitelisted
>
> Schema này định nghĩa loại 2. Loại 1 vẫn dùng cơ chế cũ ở [Doc 19](../10-legal/19-service-guarantee-policy.md) + [Doc 15 Section L](../05-ba/15-business-rules.md).

### 7.8.1 · Design principles

1. **1 gói = 1 thiết bị** (customer_devices.id), KH có 2 máy lạnh → mua 2 gói
2. **Quotas có giới hạn**: vd 4 lần vệ sinh/năm, max 1 lần/tháng
3. **Covered repairs = whitelist**: liệt kê rõ issue_category được cover
4. **Auto-suggest scheduling**: cron tạo suggestion mỗi N tháng cho cleaning quota chưa dùng
5. **Deferred revenue accounting**: revenue recognized dần theo thời gian, không recognized full khi sale
6. **No-charge orders**: claim → tạo order với `amount_charged = 0`, agent vẫn earn từ warranty fund

### 7.8.2 · Catalog tables (DB: `smp_catalog`)

#### `warranty_packages` · Gói bán được

```sql
CREATE TABLE smp_catalog.warranty_packages (
  id                BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  package_code      VARCHAR(64) NOT NULL UNIQUE COMMENT 'wpkg_ac_basic_1y',
  name              VARCHAR(255) NOT NULL COMMENT 'Bảo trì máy lạnh cơ bản 1 năm',
  description       TEXT,
  device_category   VARCHAR(64) NOT NULL COMMENT 'ac, washer, fridge, water_heater · 1 gói cho 1 category',
  duration_months   SMALLINT UNSIGNED NOT NULL COMMENT '12, 24, 36...',
  price             BIGINT UNSIGNED NOT NULL COMMENT 'Giá bán cho KH (VND minor)',
  currency          CHAR(3) NOT NULL DEFAULT 'VND',
  status            VARCHAR(16) NOT NULL DEFAULT 'active',
  -- Display metadata
  marketing_tag     VARCHAR(64) COMMENT 'best_seller, new, premium',
  display_order     SMALLINT UNSIGNED DEFAULT 0,
  -- Audit
  created_at_utc    DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at_utc    DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  CONSTRAINT chk_wpkg_status CHECK (status IN ('active','deprecated','draft')),
  CONSTRAINT chk_wpkg_duration CHECK (duration_months BETWEEN 1 AND 60),
  INDEX idx_wpkg_category_status (device_category, status)
) ENGINE=InnoDB COMMENT='Catalog of maintenance subscription packages';
```

#### `warranty_package_quotas` · Quotas trong gói

```sql
CREATE TABLE smp_catalog.warranty_package_quotas (
  id                  BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  package_id          BIGINT UNSIGNED NOT NULL,
  quota_type          VARCHAR(32) NOT NULL COMMENT 'cleaning | repair_basic | repair_full',
  service_code        VARCHAR(64) NULL COMMENT 'FK to services.service_code · NULL for repair quotas',
  count_total         SMALLINT UNSIGNED NOT NULL COMMENT 'Tổng số lần trong duration_months',
  count_per_period    SMALLINT UNSIGNED NULL COMMENT 'Max per period (vd 1/month)',
  period_days         SMALLINT UNSIGNED NULL COMMENT 'Period for count_per_period limit',
  is_unlimited        BOOLEAN NOT NULL DEFAULT FALSE COMMENT 'If TRUE, count_total ignored',
  auto_suggest        BOOLEAN NOT NULL DEFAULT FALSE COMMENT 'Auto suggest scheduling',
  suggest_interval_days SMALLINT UNSIGNED NULL COMMENT 'Vd 90 = mỗi 3 tháng',
  notes               TEXT,
  CONSTRAINT fk_wpq_pkg FOREIGN KEY (package_id) REFERENCES warranty_packages(id) ON DELETE CASCADE,
  CONSTRAINT chk_wpq_type CHECK (quota_type IN ('cleaning','repair_basic','repair_full','inspection')),
  INDEX idx_wpq_pkg (package_id)
) ENGINE=InnoDB COMMENT='Quotas defined per package (cleaning count, repair count, etc.)';
```

#### `warranty_package_covered_issues` · Whitelist các issues được sửa free

```sql
CREATE TABLE smp_catalog.warranty_package_covered_issues (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  package_id      BIGINT UNSIGNED NOT NULL,
  issue_category  VARCHAR(64) NOT NULL COMMENT 'capacitor, gas_refill, fan_motor, drainage',
  issue_name      VARCHAR(255) COMMENT 'Display name to customer',
  max_value       BIGINT UNSIGNED NULL COMMENT 'Trần chi per claim · NULL = no cap',
  currency        CHAR(3) NOT NULL DEFAULT 'VND',
  notes           TEXT,
  CONSTRAINT fk_wpci_pkg FOREIGN KEY (package_id) REFERENCES warranty_packages(id) ON DELETE CASCADE,
  UNIQUE KEY uniq_pkg_issue (package_id, issue_category),
  INDEX idx_wpci_pkg (package_id)
) ENGINE=InnoDB COMMENT='Whitelist of covered repair issues per package';
```

### 7.8.3 · Customer tables (DB: `smp_customer`)

#### `customer_devices` · Thiết bị của KH (NEW)

> Cần thiết để link 1 gói = 1 thiết bị. Nếu chưa có table này → tạo mới.

```sql
CREATE TABLE smp_customer.customer_devices (
  id                BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  customer_id       BIGINT UNSIGNED NOT NULL COMMENT 'FK to customers (in inside)',
  device_category   VARCHAR(64) NOT NULL COMMENT 'ac, washer, fridge, water_heater',
  brand             VARCHAR(128) COMMENT 'Daikin, Panasonic, LG',
  model             VARCHAR(128),
  serial_no         VARCHAR(128),
  install_date      DATE NULL COMMENT 'Ngày KH lắp đặt (declared by customer)',
  install_location  VARCHAR(255) COMMENT 'Phòng khách, phòng ngủ chính, etc.',
  notes             TEXT,
  status            VARCHAR(16) NOT NULL DEFAULT 'active',
  created_at_utc    DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at_utc    DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  CONSTRAINT chk_cd_status CHECK (status IN ('active','retired','transferred')),
  INDEX idx_cd_customer (customer_id),
  INDEX idx_cd_category (device_category, status)
) ENGINE=InnoDB COMMENT='Customer-owned devices for warranty subscription tracking';
```

### 7.8.4 · Warranty execution tables (DB: `smp_warranty` · NEW DB)

```sql
CREATE DATABASE smp_warranty CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE smp_warranty;
```

#### `customer_warranties` · Gói KH đã mua

```sql
CREATE TABLE smp_warranty.customer_warranties (
  id                  BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  customer_id         BIGINT UNSIGNED NOT NULL,
  package_id          BIGINT UNSIGNED NOT NULL COMMENT 'FK to smp_catalog.warranty_packages',
  device_id           BIGINT UNSIGNED NOT NULL COMMENT 'FK to smp_customer.customer_devices · 1 gói = 1 thiết bị',
  purchase_order_id   BIGINT UNSIGNED NOT NULL COMMENT 'FK to smp_order.orders · order KH mua gói',
  start_date_utc      DATETIME(3) NOT NULL,
  end_date_utc        DATETIME(3) NOT NULL,
  price_paid          BIGINT UNSIGNED NOT NULL COMMENT 'Snapshot of price (in case package price changes later)',
  currency            CHAR(3) NOT NULL DEFAULT 'VND',
  status              VARCHAR(16) NOT NULL DEFAULT 'active',
  -- Denormalized quotas for fast read (source of truth = warranty_quota_balance)
  remaining_quotas    JSON COMMENT '{"cleaning": 3, "repair_basic": 5}',
  -- Deferred revenue accounting
  total_amount_recognized  BIGINT UNSIGNED DEFAULT 0 COMMENT 'Revenue đã recognize tới thời điểm hiện tại',
  next_recognition_date_utc DATETIME(3) COMMENT 'Cron tiếp theo ghi revenue',
  -- Lifecycle
  cancelled_at_utc    DATETIME(3) NULL,
  cancelled_reason    VARCHAR(255) NULL,
  refunded_amount     BIGINT UNSIGNED DEFAULT 0,
  -- Audit
  created_at_utc      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at_utc      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  CONSTRAINT chk_cw_status CHECK (status IN ('active','expired','cancelled','used_up','suspended')),
  CONSTRAINT chk_cw_dates CHECK (end_date_utc > start_date_utc),
  INDEX idx_cw_customer_status (customer_id, status),
  INDEX idx_cw_device (device_id),
  INDEX idx_cw_end_date (end_date_utc) COMMENT 'For expiry cron'
) ENGINE=InnoDB COMMENT='Active maintenance subscriptions purchased by customers';
```

#### `warranty_quota_balance` · Source of truth cho quotas còn lại

```sql
CREATE TABLE smp_warranty.warranty_quota_balance (
  id                BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  customer_warranty_id BIGINT UNSIGNED NOT NULL,
  quota_type        VARCHAR(32) NOT NULL,
  service_code      VARCHAR(64) NULL,
  count_remaining   SMALLINT NOT NULL COMMENT 'Có thể negative nếu over-claim (audit case)',
  count_used        SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  last_used_at_utc  DATETIME(3) NULL,
  next_eligible_at_utc DATETIME(3) NULL COMMENT 'Per count_per_period rule',
  updated_at_utc    DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  CONSTRAINT fk_wqb_warranty FOREIGN KEY (customer_warranty_id) REFERENCES customer_warranties(id) ON DELETE CASCADE,
  UNIQUE KEY uniq_warranty_quota (customer_warranty_id, quota_type, service_code),
  INDEX idx_wqb_warranty (customer_warranty_id)
) ENGINE=InnoDB COMMENT='Per-warranty quota tracking · authoritative count';
```

#### `warranty_claims` · Mỗi lần KH dùng gói

```sql
CREATE TABLE smp_warranty.warranty_claims (
  id                  BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  customer_warranty_id BIGINT UNSIGNED NOT NULL,
  order_id            BIGINT UNSIGNED NULL COMMENT 'Order generated from this claim · NULL until approved',
  claim_type          VARCHAR(32) NOT NULL COMMENT 'cleaning | repair_basic | repair_full',
  service_code        VARCHAR(64) NULL COMMENT 'For cleaning quotas',
  issue_category      VARCHAR(64) NULL COMMENT 'For repair claims · must match covered_issues',
  issue_description   TEXT,
  claimed_at_utc      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  -- Approval
  status              VARCHAR(16) NOT NULL DEFAULT 'pending',
  approved_by         BIGINT UNSIGNED NULL,
  approved_at_utc     DATETIME(3) NULL,
  rejection_reason    VARCHAR(255) NULL,
  -- Linked to actual service order (free-of-charge order)
  completed_at_utc    DATETIME(3) NULL,
  -- Cost tracking (for accounting · cost to SMP for this free service)
  service_cost        BIGINT UNSIGNED NULL COMMENT 'Internal cost · agent payable + materials',
  cost_currency       CHAR(3) DEFAULT 'VND',
  CONSTRAINT fk_wc_warranty FOREIGN KEY (customer_warranty_id) REFERENCES customer_warranties(id),
  CONSTRAINT chk_wc_status CHECK (status IN ('pending','approved','rejected','in_progress','completed','cancelled')),
  CONSTRAINT chk_wc_type CHECK (claim_type IN ('cleaning','repair_basic','repair_full','inspection')),
  INDEX idx_wc_warranty (customer_warranty_id),
  INDEX idx_wc_order (order_id),
  INDEX idx_wc_status_claimed (status, claimed_at_utc)
) ENGINE=InnoDB COMMENT='Customer claims against active warranties';
```

#### `warranty_revenue_recognition` · Lịch ghi nhận doanh thu hàng tháng

```sql
CREATE TABLE smp_warranty.warranty_revenue_recognition (
  id                  BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  customer_warranty_id BIGINT UNSIGNED NOT NULL,
  recognition_date_utc DATETIME(3) NOT NULL COMMENT 'Date this revenue chunk recognized',
  amount              BIGINT UNSIGNED NOT NULL COMMENT 'Revenue chunk · usually price/duration_months',
  currency            CHAR(3) NOT NULL DEFAULT 'VND',
  journal_entry_id    BIGINT UNSIGNED NULL COMMENT 'FK to journal_entries · NULL until posted',
  posted_at_utc       DATETIME(3) NULL,
  CONSTRAINT fk_wrr_warranty FOREIGN KEY (customer_warranty_id) REFERENCES customer_warranties(id),
  INDEX idx_wrr_date_posted (recognition_date_utc, posted_at_utc) COMMENT 'For cron'
) ENGINE=InnoDB COMMENT='Monthly revenue recognition schedule (deferred revenue → revenue)';
```

### 7.8.5 · ALTER existing tables

```sql
-- Link order to warranty claim (if free-of-charge order)
ALTER TABLE smp_order.orders
  ADD COLUMN warranty_claim_id BIGINT UNSIGNED NULL COMMENT 'NULL = normal paid order; not NULL = warranty claim order (free)',
  ADD COLUMN is_warranty_order BOOLEAN NOT NULL DEFAULT FALSE,
  ADD INDEX idx_orders_warranty_claim (warranty_claim_id);

-- Add accounts to Chart of Accounts (Doc 16 §1)
INSERT INTO smp_finance.accounts (account_code, name, account_type, is_balance_sheet) VALUES
  ('deferred_revenue', 'Doanh thu chưa thực hiện (gói BH)', 'liability', TRUE),
  ('revenue_warranty_subscription', 'Doanh thu thuê bao BH', 'revenue', FALSE),
  ('warranty_service_cost', 'Chi phí dịch vụ BH', 'expense', FALSE);
```

### 7.8.6 · Example scenario

KH "Anh Nguyễn A" có 2 máy lạnh, mua 2 gói "Bảo trì AC cơ bản 1 năm" giá 1,200,000 VND/gói:

```text
1. Tạo customer_devices:
   device_1: AC1 phòng khách (Daikin) → device_id=101
   device_2: AC2 phòng ngủ (Panasonic) → device_id=102

2. KH mua 2 gói qua order O-001 = 2,400,000 VND
   warranty_1: package=wpkg_ac_basic_1y, device_id=101, start=2026-06-01, end=2027-06-01
   warranty_2: package=wpkg_ac_basic_1y, device_id=102, start=2026-06-01, end=2027-06-01

3. Mỗi gói init quotas:
   warranty_quota_balance:
     - warranty_1: cleaning x4, repair_basic x6
     - warranty_2: cleaning x4, repair_basic x6

4. Tháng 7 (1 tháng sau mua), KH yêu cầu vệ sinh AC1:
   - Tạo warranty_claim: claim_type='cleaning', service_code='svc_ac_periodic_clean'
   - Quota check: warranty_1.cleaning > 0 → approve
   - Tạo order O-050 với amount_charged=0, is_warranty_order=TRUE
   - Sau khi step completed: warranty_quota_balance.count_remaining --
   - Cost: 200k VND (agent payable) → ghi vào warranty_service_cost
   
5. Cron auto-suggest:
   - Tháng 10, warranty_1.cleaning vẫn còn 3 → suggest KH đặt lịch
   - Push notification: "Đã 3 tháng từ lần vệ sinh trước, đặt lịch ngay?"
```

### 7.8.7 · Migration plan

| Phase | Action |
|---|---|
| v3.5 Sprint 1 | Create `smp_catalog.warranty_packages` + quotas + covered_issues tables |
| v3.5 Sprint 1 | Create `smp_customer.customer_devices` (if not exists) |
| v3.5 Sprint 1 | Create `smp_warranty` DB + 4 tables |
| v3.5 Sprint 1 | Insert 3 new accounts vào Chart of Accounts |
| v3.5 Sprint 2 | Implement `pkg/warranty` package (quota check, claim approval) |
| v3.5 Sprint 2 | Implement deferred revenue recognition cron job |
| v3.5 Sprint 3 | Build APIs (purchase, claim, quota check) |
| v3.5 Sprint 3 | Customer mobile UI · gói catalog + my warranties |
| v3.5 Sprint 4 | Admin Web · package management page |
| v3.6 | Auto-suggest scheduling cron |
| v3.6 | Renewal flow · expire 7d before notify |

### 7.8.8 · Operational notes

- **Cardinality**: 1000 customers × 2 devices × 1 active warranty = 2000 warranties. Quota balance ~5 rows per warranty = 10k rows. Acceptable.
- **Concurrent quota updates**: claim approval + cancellation can race → use row-level lock on `warranty_quota_balance.id` + optimistic version.
- **Revenue recognition cron**: nightly job · process `next_recognition_date_utc <= NOW()` · idempotent via `journal_entry_id` link.
- **Refund handling**: if KH cancel gói (vd trong 7 days cooling off period) → refund proportional remaining duration, reverse deferred_revenue, log `cancelled_reason`.
- **Backup**: same as financial data · 10-year retention (`smp_warranty` is financial-adjacent).

---

## 8. MongoDB collections

### `smp_events` (integration-svc)

```javascript
// event_log collection
{
  _id: ObjectId,
  event_id: "evt_01HX7K",
  event_type: "inside.payment.succeeded",
  occurred_at: ISODate,
  schema_version: "1.0",
  data: { ... },
  consumed_by: ["order-svc", "finance-svc"],
  consumed_at: { "order-svc": ISODate, ... },
  status: "consumed | dlq | retry",
  retry_count: 0,
  ttl_at: ISODate  // TTL index, auto delete after 90 days
}
// Indexes:
// { event_id: 1 } unique
// { event_type: 1, occurred_at: -1 }
// { status: 1, retry_count: 1 }
// { ttl_at: 1 } expireAfterSeconds=0
```

### `smp_audit` (immutable audit log)

```javascript
// audit_log collection
{
  _id: ObjectId,
  audit_id: "aud_01HX7K",
  timestamp: ISODate,
  actor_type: "user | system | partner | api",
  actor_id: "user_123",
  action: "order.created | partner.kyc_approved | etc.",
  resource_type: "order | partner | agent",
  resource_id: "ord_01HX7K2M",
  before: { ... },
  after: { ... },
  ip_address: "...",
  user_agent: "...",
  request_id: "..."
  // No TTL — retain 7 years for compliance
}
// Indexes:
// { audit_id: 1 } unique
// { timestamp: -1 }
// { actor_id: 1, timestamp: -1 }
// { resource_type: 1, resource_id: 1, timestamp: -1 }
```

## 9. Redis key patterns

```text
agent:online:<agent_id>           SET (TTL 60s, heartbeat refresh)
agent:location:<agent_id>         HASH { lat, lng, updated_at } (TTL 60s)
dispatch:queue:<zone_code>        LIST (FIFO queue of pending orders)
dispatch:invitations:<order_id>   SET (agent_ids invited)
session:<token>                   HASH { user_id, role, expires_at } (TTL 8h)
ratelimit:<user_id>:<endpoint>    INT (token bucket, TTL 60s)
cache:customer:<cus_id>           STRING (JSON, TTL 30s)
cache:wms:stock:<sku>:<wh_id>     STRING (JSON, TTL 30s)
lock:order:<order_id>             STRING (distributed lock, TTL 30s)
```

## 10. Cross-database FK strategy

MySQL không support cross-database FK reference. Quy ước:
- Khi reference cross-DB, dùng `<entity>_code` VARCHAR (vd `partner_code`, không phải `partner_id` BIGINT)
- Service layer validate trước khi insert
- Eventual consistency cho denormalized fields (vd `agent.total_orders_completed` cập nhật qua event consumer)

## 11. Indexes & performance

**High-frequency queries:**
1. Dispatch: `SELECT * FROM agents WHERE is_online=TRUE AND status='active' AND home_city=? AND home_district IN (?, ?, ?)` → covered by `idx_agents_status_online` + `idx_agents_location`
2. Order list cho customer: `SELECT * FROM orders WHERE customer_id=? ORDER BY created_at DESC LIMIT 20` → covered by `idx_orders_customer` + `idx_orders_created`
3. Order list cho partner: `WHERE partner_id=?` → covered by `idx_orders_partner`

**Partitioning** (Phase 2 khi >10M rows):
- `orders` partition by RANGE(YEAR(created_at))
- `order_stage_log` partition by RANGE(MONTH(created_at))

---

## 11.5 · Sharding strategy (v4.0)

> Khi expand global, có 2 strategies sharding theo `country_code`. SMP dùng **hybrid**: logical partition cho hầu hết countries, physical separation cho China + US (compliance).

### Strategy A · Logical sharding (default)

Cùng MySQL cluster, partition tables by `country_code` column. Áp dụng cho VN, TH, SG, ID, MY, PH (data có thể co-locate).

```sql
-- Pre-requisite: country_code phải có trong PRIMARY KEY composite
ALTER TABLE orders
  DROP PRIMARY KEY,
  ADD PRIMARY KEY (id, country_code);

ALTER TABLE orders
PARTITION BY LIST COLUMNS (country_code) (
  PARTITION p_vn VALUES IN ('VN'),
  PARTITION p_th VALUES IN ('TH'),
  PARTITION p_sg VALUES IN ('SG'),
  PARTITION p_id VALUES IN ('ID'),
  PARTITION p_my VALUES IN ('MY'),
  PARTITION p_ph VALUES IN ('PH'),
  PARTITION p_other VALUES IN (DEFAULT)
);
```

**Pros**: Đơn giản, 1 cluster quản lý, cross-country query dễ (cho reporting).
**Cons**: Không meet sovereignty cho CN/EU/US strict regulations.

### Strategy B · Physical sharding (compliance-driven)

Mỗi country/region 1 cluster MySQL độc lập. Áp dụng cho:
- **China (CN)**: PIPL yêu cầu data localization. Cluster đặt tại datacenter TQ (Alibaba Cloud Beijing/Shanghai).
- **US (US)**: CPRA + state-level laws. Cluster đặt tại Virginia (AWS us-east-1).
- **EU (DE, FR, ...)**: GDPR cross-border restrictions. Cluster đặt tại Frankfurt (AWS eu-central-1).

```text
┌──────────────────────┐    ┌──────────────────────┐    ┌──────────────────────┐
│  smp-asia cluster    │    │  smp-china cluster   │    │  smp-us cluster      │
│  (Singapore region)  │    │  (Beijing region)    │    │  (Virginia region)   │
│                      │    │                      │    │                      │
│  VN, TH, SG, ID,     │    │  CN only             │    │  US only             │
│  MY, PH (logical     │    │  (full isolation)    │    │  (full isolation)    │
│  partitions)         │    │                      │    │                      │
└──────────────────────┘    └──────────────────────┘    └──────────────────────┘
```

**Routing layer**: API Gateway xác định cluster dựa trên:
1. User session country_code (đã đăng nhập)
2. Geolocation IP (lần đầu)
3. Default = nearest cluster theo geo

```go
// api-gateway routing logic
func ResolveCluster(req Request) string {
    country := req.User.CountryCode
    switch country {
    case "CN":
        return "smp-china"
    case "US":
        return "smp-us"
    default:
        return "smp-asia"  // VN, TH, SG, ID, MY, PH, ...
    }
}
```

### Cross-cluster data

Một số data PHẢI replicate cross-cluster để hệ thống hoạt động:
- `smp_global` (countries, currencies, currency_rates, tax_configs, i18n_translations) → **read-only replica** ở mọi cluster
- `audit_log` (MongoDB) → write local cluster, async aggregate sang central archive (encrypted)

Một số data **không bao giờ** rời cluster nguồn:
- `orders`, `customers`, `agents`, `partners` của country đó
- `kyc_documents` (PII nhạy cảm)

### Migration approach

Phase 1 (v3.x → v4.0): Hiện tại 1 cluster, all data Vietnam. Add `country_code` column với DEFAULT 'VN'.
Phase 2 (v4.0 GA): Spin up `smp-asia` cluster với tất cả VN data. Logical partitioning.
Phase 3 (khi launch CN): Spin up `smp-china` cluster riêng. Onboard CN customers → routed sang đó.
Phase 4 (khi launch US): Spin up `smp-us` cluster.

### Disaster recovery considerations

Mỗi cluster có:
- **Primary** (read-write) ở region chính
- **Replica** (read-only) trong cùng region để load balance
- **Cross-region backup** (encrypted, S3 cross-region replication) cho DR

RPO < 5 minutes, RTO < 1 hour.

## 12. Migration tooling

Dùng [golang-migrate/migrate](https://github.com/golang-migrate/migrate):

```text
migrations/
  smp_order/
    000001_init.up.sql
    000001_init.down.sql
    000002_add_partner_fields.up.sql
    000002_add_partner_fields.down.sql
  smp_partner/
    000001_init.up.sql
    ...
```

Mỗi service có folder migrations riêng. Run:
```bash
migrate -database "mysql://user:pass@tcp(host:3306)/smp_order" -path migrations/smp_order up
```

## 13. Backup strategy

| DB | Method | Frequency | Retention |
|---|---|---|---|
| MySQL | mysqldump + binlog | Daily full + hourly binlog | 30 days |
| MongoDB | mongodump | Daily | 30 days |
| Redis | RDB snapshot + AOF | RDB every 6h, AOF always | 7 days |

Restore drill: monthly · staging env · verify RTO < 1h.

## 14. Data retention policy

| Data | Retention | Notes |
|---|---|---|
| Order data (orders, order_steps) | 5 năm sau completed_at | Compliance VN |
| Audit log | 7 năm | Tax + financial regulation |
| Event log (smp_events) | 90 ngày | TTL index auto-cleanup |
| Customer PII via inside | Per inside policy | SMP không own |
| Photo proof | 2 năm sau order completed | Then archive to cold storage |
