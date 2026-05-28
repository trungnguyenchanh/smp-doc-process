# SMP · System Functional Overview

**Version**: 3.3 · **Audience**: All roles (BA, Dev, QC, DevOps, Security, Stakeholder) · **Reading time**: 30–45 min

> Doc này là **executive summary** mô tả toàn bộ chức năng hệ thống SMP. Đọc trước khi đi sâu vào các docs kỹ thuật khác.

---

## Mục lục

1. [SMP là gì](#1-smp-là-gì)
2. [Personas · Ai dùng hệ thống](#2-personas--ai-dùng-hệ-thống)
3. [Module Map · 10 microservices](#3-module-map--10-microservices)
4. [Core Domain Concepts](#4-core-domain-concepts)
5. [Feature Catalog · 6 epics](#5-feature-catalog--6-epics)
6. [End-to-end Flows · 5 luồng chính](#6-end-to-end-flows--5-luồng-chính)
7. [Module ↔ Feature Matrix](#7-module--feature-matrix)
8. [Key Business Rules · Tóm tắt](#8-key-business-rules--tóm-tắt)
9. [KPIs · Đo lường thành công](#9-kpis--đo-lường-thành-công)
10. [Integration với hệ thống ngoài](#10-integration-với-hệ-thống-ngoài)
11. [Roadmap · v3.1 → v3.3 → tương lai](#11-roadmap--v31--v33--tương-lai)
12. [Quick links · Đào sâu](#12-quick-links--đào-sâu)

---

## 1. SMP là gì

**SMP** (Service Marketplace Platform) là nền tảng SaaS **3-sided marketplace** chuyên cho dịch vụ kỹ thuật tận nơi (sửa điều hòa, máy giặt, điện nước, vệ sinh, lắp đặt thiết bị...) tại thị trường Việt Nam.

### 3 bên tham gia

```text
       ┌─────────────┐
       │  CUSTOMER   │ Khách hàng cuối · đặt đơn qua mobile app
       │  (B2C)      │
       └──────┬──────┘
              │
              ▼
       ┌─────────────┐         ┌─────────────┐
       │   SMP CORE  │ ◄─────► │   PARTNER   │ Đối tác B2B (cty
       │  (3-sided)  │         │  (B2B2C)    │ bảo trì, chuỗi shop...)
       └──────┬──────┘         └─────────────┘
              │
              ▼
       ┌─────────────┐
       │ TECHNICIAN  │ Thợ kỹ thuật (freelance hoặc thuộc partner)
       │   (Agent)   │
       └─────────────┘
```

### Giá trị cốt lõi

- **Với customer**: đặt thợ trong 2 phút, biết giá trước, theo dõi real-time, bảo hành rõ ràng
- **Với technician**: nhận đơn gần nhà, thanh toán nhanh, không lo lừa đảo
- **Với partner**: outsource toàn bộ field operation, có dashboard quản lý, white-label được
- **Với SMP**: thu commission per order + subscription fee từ partner

### Tech stack tóm tắt

| Layer | Technology (v3.x) | Bổ sung v4.0 |
|---|---|---|
| Backend | Go 1.22 · microservices polyrepo | + `antonmedv/expr` (rules engine) |
| Primary DB | MySQL 8.0 (transactional) | + multi-currency schema, UTC, sharding |
| Cache + Queue | Redis 7 | + Redis Streams (light pub/sub) |
| Event Bus | (none, sync HTTP) | **Apache Kafka** (production), Debezium CDC |
| Read Model | (query MySQL) | **Elasticsearch** (dashboard, search, KPI) |
| Event store + audit | MongoDB 6 | (giữ nguyên) |
| Config | env files | + **ConfigMap** (rules_engine.yaml hot-reload) |
| Frontend Web | Vanilla HTML/CSS/JS (admin) | + i18n dictionary |
| Frontend Mobile | PWA (customer + technician + partner) | + locale switcher, currency formatter |
| Infrastructure | Kubernetes · GitHub Actions · ArgoCD | + region-based clusters (sovereignty) |
| Integration | Webhooks + REST · với inside + wms | + Kafka cross-region replication |

> **Note**: v3.4 (current) = docs phase, chưa code v4.0. Schema/architecture đang được design để forward-compatible.

---

## 2. Personas · Ai dùng hệ thống

| Persona | Channel | Mô tả | Quyền chính |
|---|---|---|---|
| 👤 **Customer** | Mobile PWA | Khách hàng cuối · đặt dịch vụ tại nhà | Tạo order, duyệt báo giá, rating, thanh toán |
| 🔧 **Technician (Agent)** | Mobile PWA | Thợ kỹ thuật · có thể là Survey (SUR) / Execution (EXC) / Dual | Nhận đơn, báo giá, upload photo proof, nhận tiền |
| 🏢 **Partner Owner** | Admin Web | Chủ công ty đối tác B2B | Đặt đơn cho khách cuối, quản lý wallet, xem báo cáo |
| 💼 **Partner Manager** | Admin Web | Quản lý cấp giữa của partner | Quản lý đội thợ partner, xem báo cáo team |
| 📊 **Partner Finance** | Admin Web | Kế toán partner | Topup wallet, xem invoice, export CSV |
| 📞 **Partner Dispatcher** | Admin Web | Điều phối viên partner | Tắt/bật agent, manual dispatch |
| 👥 **Partner End-user** | Mobile PWA | Khách cuối của partner (đôi khi partner đặt giúp họ) | Tracking + rating (limited) |
| 🛠️ **Ops Admin** | Admin Web | Nhân viên vận hành SMP | Approve KYC, manual dispatch, dispute, quản lý catalog |
| 🎯 **Ops Manager** | Admin Web | Quản lý vận hành SMP | Dashboard, surge config, partner approval |
| 💰 **Finance Admin** | Admin Web | Tài chính SMP | Payout thợ, reconcile, refund |
| 👑 **Super Admin** | Admin Web | Root cấp cao nhất | Full quyền + audit log access |

Chi tiết RBAC scopes: xem [Doc 08 · Auth Spec](08-security/08-auth-spec.md).

---

## 3. Module Map · 10 microservices

Hệ thống SMP chia thành **10 microservices**, mỗi service own 1 database, communicate qua REST + Event bus (Kafka).

```text
                    ┌──────────────────────┐
                    │   api-gateway        │ Routing, auth, rate limit
                    │   (Public entry)     │
                    └──────────┬───────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
┌───────▼───────┐    ┌─────────▼────────┐   ┌─────────▼────────┐
│  order-svc    │    │ dispatch-engine  │   │  catalog-svc     │
│  (đơn hàng)   │◄──►│  (match đơn↔thợ) │   │  (service, BOM)  │
└───────┬───────┘    └─────────┬────────┘   └──────────────────┘
        │                      │
        │              ┌───────▼────────┐   ┌──────────────────┐
        │              │   agent-svc    │   │  partner-svc     │
        │              │   (quản lý thợ)│   │  (Partner B2B)   │
        │              └────────────────┘   └──────────────────┘
        │
┌───────▼───────┐    ┌──────────────────┐   ┌──────────────────┐
│  finance-svc  │    │  quality-svc     │   │ integration-svc  │
│  (ví, payout) │    │  (rating, photo) │   │  (inside, wms)   │
└───────────────┘    └──────────────────┘   └──────────────────┘
                                                     │
                                            ┌────────▼─────────┐
                                            │ notification-svc │
                                            │ (SMS, push, mail)│
                                            └──────────────────┘
```

### Chi tiết từng module

#### 🟦 1. `api-gateway`
**Vai trò**: Cổng vào duy nhất từ public internet
**Chức năng**:
- Routing request đến các services nội bộ
- Authenticate JWT + verify scope
- Rate limiting (per IP, per user, per tier)
- Request logging + tracing header injection
- CORS, security headers

**Database**: không own DB · stateless

#### 🟦 2. `order-svc`
**Vai trò**: Quản lý toàn bộ vòng đời đơn hàng
**Chức năng**:
- Tạo order (customer direct, partner private, partner open)
- 10-stage lifecycle: created → searching → quoted → approved → in_progress → completed → ...
- Stage transitions với validation
- Quote management (báo giá Survey)
- Material consumption tracking (Execution)
- Order step assignment (link agent vào step)
- Cancel/reschedule logic

**Database**: `smp_order` (MySQL) · core tables: `orders`, `order_steps`, `order_step_materials`, `order_stage_log`

#### 🟦 3. `dispatch-engine`
**Vai trò**: Tim agent phù hợp gửi đơn
**Chức năng**:
- Round-based dispatch (60s/round, max 5 rounds)
- Filter qualified agents (skill, location, online, KYC, rating)
- Send concurrent invitations (max 3-5 per round)
- Surge multiplier calculation
- Private dispatch (chỉ thợ partner)
- Open dispatch with partner preference
- Manual override cho Ops

**Database**: `smp_dispatch` (Redis cho queue, MySQL cho audit) · queue `dispatch:queue:<round_id>`

#### 🟦 4. `catalog-svc`
**Vai trò**: Quản lý dữ liệu master
**Chức năng**:
- Service templates + steps + skills
- Material types + variants + BOMs
- Pricing rules per service/material
- Coverage zones (geo)

**Database**: `smp_catalog` (MySQL) · 7 tables (skills, steps, services, service_steps, material_types, material_variants, step_boms)

#### 🟦 5. `agent-svc`
**Vai trò**: Quản lý đời sống agent
**Chức năng**:
- Agent registration + KYC (basic, advanced, premium)
- Skill assignment
- Warehouse assignment (cho material stock)
- Online/offline tracking (Redis)
- Performance metrics (rating, completion rate)
- Suspend/terminate flow

**Database**: `smp_agent` (MySQL) · `agents`, `agent_skills`, `agent_warehouses`, `agent_kyc_docs`

#### 🟦 6. `partner-svc` (v3.3)
**Vai trò**: Quản lý Partner B2B (3-sided marketplace)
**Chức năng**:
- Partner onboarding (Type A/B/AB)
- Partner admin users + RBAC (Owner/Manager/Finance/Dispatcher)
- Wallet management (topup + auto-debit)
- Invoice generation
- Payout calculation (cho thợ thuộc partner)
- Partner-specific pricing override

**Database**: `smp_partner` (MySQL) · `partners`, `partner_admin_users`, `partner_wallet_transactions`, `partner_invoices`, `partner_payouts`

#### 🟦 7. `finance-svc`
**Vai trò**: Tiền bạc của SMP
**Chức năng**:
- Customer payment (qua inside)
- Agent payout (T+7 schedule)
- Refund processing
- Commission calculation
- Tax/VAT handling

**Database**: `smp_finance` (MySQL)

#### 🟦 8. `quality-svc`
**Vai trò**: Chất lượng dịch vụ
**Chức năng**:
- Rating 1-5 sao (customer rate agent)
- Photo proof storage + verification
- Dispute management (auto trigger nếu rating <3)
- KYC document review

**Database**: `smp_quality_media` (MongoDB) cho file paths + metadata

#### 🟦 9. `integration-svc`
**Vai trò**: Đồng bộ với hệ thống ngoài
**Chức năng**:
- Sync customer từ `inside` (HRM/CRM nội bộ)
- Sync payment status từ `inside`
- Sync stock từ `wms` (warehouse management)
- Circuit breaker pattern (nếu inside/wms down, không sập SMP)
- Webhook outbound (gửi event ra cho partner)

**Database**: dùng MongoDB `smp_events.event_log` (TTL 90 ngày)

#### 🟦 10. `notification-svc`
**Vai trò**: Gửi thông báo cho user
**Chức năng**:
- SMS OTP (qua Twilio/eSMS)
- Push notification (FCM cho mobile)
- Email (SendGrid)
- Template management
- Retry logic + dead letter queue

**Database**: dùng MongoDB cho log + Redis cho rate limit

Chi tiết kiến trúc + diagrams: xem [Doc 01 · Architecture](01-architecture/01-architecture.md).

---

## 4. Core Domain Concepts

Các khái niệm cốt lõi cần hiểu trước khi đọc các docs khác:

### Service Template & Step
**Service** = 1 gói dịch vụ end-to-end (vd "Sửa điều hòa tại nhà")
**Step** = 1 công đoạn trong service (vd "Khảo sát" + "Sửa chữa")
1 service có 1-N steps. Mỗi step thuộc về 1 **agent role**.

### Agent Role · 4 vai trò trong 1 order
| Role | Color | Chức năng |
|---|---|---|
| **SUR** (Survey) | 🟣 Violet | Khảo sát, báo giá |
| **EXC** (Execution) | 🟢 Teal | Thực thi (sửa, lắp) |
| **Dual** | Mixed | Làm cả 2 (đơn nhỏ) |
| **Pool** | Gray | Standby, chưa assign |

### Order Lifecycle · 10 stages
```text
01 created → 02 searching → 03 sur_assigned → 04 sur_arrived
→ 05 quoted → 06 approved → 07 exc_assigned → 08 exc_in_progress
→ 09 completed → 10 paid

[cancelled] có thể xảy ra ở bất kỳ stage nào ≤ 06
```

### Skill
Kỹ năng cụ thể (vd "điện lạnh", "điện nước"). 1 step yêu cầu 1-N skills. Agent phải có skill matching mới qualified.

### Material (BOM) · v3.2
- **Material Type**: nhóm vật tư (vd "Ống đồng")
- **Material Variant**: SKU cụ thể (vd "Ống đồng 6mm Daikin")
- **BOM**: định mức vật tư cho 1 step (vd "Step lắp đặt cần 2m ống đồng + 1 quả cảm biến")

### Dispatch · Round + Invitation
- **Round**: 1 lượt gửi đơn (timeout 60s)
- **Invitation**: 1 lời mời gửi đến 1 agent (max 3-5/round)
- **Escalate**: hết round vẫn ko ai nhận → expand bán kính / surge

### Partner Concepts · v3.3
- **Partner Type A**: Đặt đơn giúp khách cuối (không có thợ riêng)
- **Partner Type B**: Có thợ riêng, dùng SMP làm orchestration
- **Partner Type AB**: Cả 2
- **Private Dispatch**: chỉ gửi cho thợ thuộc partner
- **Open Dispatch**: gửi cho pool SMP (có thể prefer partner)
- **Wallet**: số dư trả trước cho partner để đặt đơn
- **Payout Mode**: Direct (SMP trả thợ) / Via Partner (partner trả thợ)

Chi tiết: xem [Doc 05 · Glossary](05-ba/05-glossary.md) (full bảng tra cứu A-M).

---

## 5. Feature Catalog · 6 epics

Toàn bộ chức năng hệ thống được tổ chức thành **6 epics**, mỗi epic gồm nhiều user stories.

### 🎯 Epic 1 · Customer Order Journey
**Mục đích**: Khách đặt đơn nhanh, biết giá trước, theo dõi real-time

| Story | Mô tả | Persona |
|---|---|---|
| US-1.1 | Customer đặt đơn dịch vụ (chọn service, slot, voucher) | Customer |
| US-1.2 | Customer xem tracking 2 thợ (SUR + EXC) trên map | Customer |
| US-1.3 | Customer duyệt báo giá (approve/reject/negotiate) | Customer |

### 🔧 Epic 2 · Technician Work Flow
**Mục đích**: Thợ nhận đơn → khảo sát → báo giá → thực thi → photo proof

| Story | Mô tả | Persona |
|---|---|---|
| US-2.1 | Thợ nhận đơn từ dispatch (window 60s) | Technician |
| US-2.2 | Thợ Survey nhập báo giá với material picker (có stock check) | Technician |
| US-2.3 | Thợ Execution upload photo proof bắt buộc | Technician |

### 🏢 Epic 3 · Partner Platform (v3.3)
**Mục đích**: 3-sided marketplace · partner B2B dùng SMP làm orchestration

| Story | Mô tả | Persona |
|---|---|---|
| US-3.1 | Partner Owner đặt đơn cho khách cuối + wallet check | Partner Owner |
| US-3.2 | Partner Finance xem báo cáo + export CSV | Partner Finance |
| US-3.3 | Partner Dispatcher tắt agent tạm thời | Partner Dispatcher |
| US-3.4 | Ops admin onboard partner mới (approve KYC) | Ops Admin |

### 🧰 Epic 4 · Material BOM (v3.2)
**Mục đích**: Quản lý vật tư + định mức theo step

| Story | Mô tả | Persona |
|---|---|---|
| US-4.1 | Ops admin tạo Material Variant mới (auto-calc margin) | Ops Admin |
| US-4.2 | Ops verify free-form material (thợ nhập tay, ko trong catalog) | Ops Admin |

### 🔗 Epic 5 · Integration với inside/wms (v3.2)
**Mục đích**: Đồng bộ với hệ thống ngoài

| Story | Mô tả | Persona |
|---|---|---|
| US-5.1 | System sync customer data từ inside (với circuit breaker) | System |
| US-5.2 | Webhook payment.succeeded từ inside (HMAC verify) | System |

### ⭐ Epic 6 · Quality & Rating
**Mục đích**: Đảm bảo chất lượng dịch vụ

| Story | Mô tả | Persona |
|---|---|---|
| US-6.1 | Customer đánh giá thợ (1-5 sao, auto trigger dispute nếu <3) | Customer |

Chi tiết Gherkin + Acceptance Criteria: xem [Doc 09 · User Stories](05-ba/09-user-stories.md).

---

## 6. End-to-end Flows · 5 luồng chính

### Flow 1 · Customer Direct Order

```text
[Customer mở app]
  ▼
[Chọn service + slot + địa chỉ]
  ▼
order-svc · POST /orders
  ▼ (event: OrderCreated)
dispatch-engine · tìm SUR agent
  ▼ (round 1, 60s)
[SUR agent nhận đơn] ──► agent-svc · update status
  ▼
[SUR đến tại nhà → khảo sát]
  ▼
[SUR nhập báo giá] ──► catalog-svc · query material price
  ▼
order-svc · POST /orders/:id/quote
  ▼ (notification gửi customer)
[Customer duyệt báo giá]
  ▼
order-svc · approve
  ▼ (event: QuoteApproved)
dispatch-engine · tìm EXC agent (có thể same SUR nếu Dual)
  ▼
[EXC thực thi → upload photo]
  ▼
order-svc · POST /orders/:id/complete
  ▼ (event: OrderCompleted)
finance-svc · trigger payment (qua inside)
  ▼
notification-svc · gửi receipt + rating prompt
  ▼
[Customer rating] ──► quality-svc · save rating
  ▼ (nếu <3: auto dispute)
[End]
```

### Flow 2 · Partner Private Order (v3.3)

Tương tự Flow 1 nhưng:
- Order tạo từ Partner admin với `source=partner` + `partner_id=P001`
- Trước khi tạo: check `partner.wallet_balance >= estimated_cost`
- Dispatch chỉ gửi cho agents có `partner_id=P001` (private dispatch)
- Sau khi complete: auto-debit từ wallet (không cần payment từ customer)
- Payout: tùy `payout_mode` của partner (Direct hoặc Via Partner)

### Flow 3 · Material Check trong Quote

Khi SUR đang nhập báo giá:
```text
SUR app · search material "Ống đồng"
  ▼
catalog-svc · GET /materials?q=...
  ▼
integration-svc · check stock từ wms (parallel)
  ▼
[Nếu wms down: circuit breaker → fallback "stock unknown"]
  ▼
Return material list với stock indicator
  ▼
SUR chọn → nhập số lượng
  ▼
[Validate vs BOM: nếu vượt định mức cảnh báo]
```

### Flow 4 · Dispatch Round với Surge

```text
order-svc emit OrderCreated event
  ▼
dispatch-engine · pick up event
  ▼
Round 1 (radius 3km, base price)
  ├─ Filter qualified agents (skill, KYC, online, rating)
  ├─ Send 3 concurrent invitations
  └─ Wait 60s
  ▼
[Nếu có ai accept: assign, done]
[Nếu ko ai accept]
  ▼
Round 2 (radius 5km, surge 1.2x)
  ▼
Round 3 (radius 8km, surge 1.5x)
  ▼
Round 4 (radius 12km, surge 1.8x)
  ▼
Round 5 (radius unlimited, surge 2.0x)
  ▼
[Hết 5 rounds: escalate to Ops · manual dispatch]
```

### Flow 5 · Partner Onboarding + KYC

```text
Sales SMP đưa link onboarding cho partner
  ▼
Partner Owner đăng ký · partner-svc · POST /partners
  ▼ (status: pending_kyc)
Upload tài liệu (ĐKKD, MST, hợp đồng)
  ▼
Ops admin nhận task review · approve/reject
  ▼
[Nếu approve] partner-svc · update status=active
  ▼ (event: PartnerApproved)
Partner Owner tạo admin users (Manager, Finance, Dispatcher)
  ▼
Partner Owner topup wallet (vd 10M VND)
  ▼ (event: PartnerWalletToppedUp)
[Partner ready để đặt đơn]
```

---

## 7. Module ↔ Feature Matrix

Bảng này map mỗi user story đến các modules tham gia:

| Story | api-gw | order | dispatch | catalog | agent | partner | finance | quality | integration | notification |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| US-1.1 Đặt đơn | ✅ | ✅ | · | ✅ | · | · | · | · | ✅ (customer sync) | ✅ |
| US-1.2 Tracking | ✅ | ✅ | · | · | ✅ | · | · | · | · | ✅ |
| US-1.3 Duyệt báo giá | ✅ | ✅ | · | · | · | · | · | · | · | ✅ |
| US-2.1 Nhận đơn | ✅ | ✅ | ✅ | · | ✅ | · | · | · | · | ✅ |
| US-2.2 Báo giá + material | ✅ | ✅ | · | ✅ | · | · | · | · | ✅ (wms stock) | · |
| US-2.3 Photo proof | ✅ | ✅ | · | · | · | · | · | ✅ | · | · |
| US-3.1 Partner Owner đặt đơn | ✅ | ✅ | ✅ | · | · | ✅ | · | · | · | ✅ |
| US-3.2 Finance báo cáo | ✅ | · | · | · | · | ✅ | ✅ | · | · | · |
| US-3.3 Tắt agent | ✅ | · | · | · | ✅ | ✅ | · | · | · | ✅ |
| US-3.4 KYC partner | ✅ | · | · | · | · | ✅ | · | ✅ | · | ✅ |
| US-4.1 Tạo material | ✅ | · | · | ✅ | · | · | · | · | · | · |
| US-4.2 Verify free-form | ✅ | ✅ | · | ✅ | · | · | · | ✅ | · | · |
| US-5.1 Sync customer | · | · | · | · | · | · | · | · | ✅ | · |
| US-5.2 Webhook payment | · | ✅ | · | · | · | · | ✅ | · | ✅ | · |
| US-6.1 Rating | ✅ | ✅ | · | · | ✅ | · | · | ✅ | · | ✅ |

**Đọc bảng**: vd US-1.1 đụng đến 5 modules (api-gw, order, catalog, integration, notification). Khi sửa US-1.1, cần test cả 5 modules.

---

## 8. Key Business Rules · Tóm tắt

Hệ thống có **~80 business rules** được code hóa thành ID `BR-<CATEGORY>-<NUM>`. Đây là 15 rule quan trọng nhất:

### Dispatch
- **BR-DISP-001**: Round timeout 60s (config được, đang là 60s)
- **BR-DISP-002**: Max 5 rounds trước khi escalate
- **BR-DISP-005**: Private dispatch (v3.3) — chỉ thợ thuộc partner_id
- **BR-DISP-007**: Surge multiplier theo round (1.0 / 1.2 / 1.5 / 1.8 / 2.0)

### Pricing
- **BR-PRICE-001**: Labor price theo level thợ (Junior 0.8x, Mid 1.0x, Senior 1.3x)
- **BR-PRICE-005**: Voucher discount áp dụng trước VAT
- **BR-PRICE-006**: Surge chỉ apply lên labor, không apply material
- **BR-PRICE-008**: VAT 10% mặc định, partner có thể override

### Payment
- **BR-PAY-001**: Customer thanh toán sau khi complete (qua inside)
- **BR-PAY-003**: Partner wallet topup tối thiểu 1M VND
- **BR-PAY-004**: Auto-debit wallet ngay khi order completed (partner private)

### KYC
- **BR-KYC-001**: 3 levels (basic, advanced, premium) với quyền tăng dần
- **BR-KYC-002**: Progression: basic → advanced sau 10 đơn rating ≥4

### Stages
- **BR-STAGE-001**: Stage transitions một chiều, không skip
- **BR-STAGE-005**: Cancel cho phép ≤ stage 06 (approved), sau đó cần dispute

### Quality
- **BR-QUALITY-001**: Rating <3 sao auto tạo dispute ticket
- **BR-QUALITY-003**: Photo proof bắt buộc trước khi mark completed

Chi tiết 80+ rules + formula: xem [Doc 15 · Business Rules](05-ba/15-business-rules.md).

---

## 9. KPIs · Đo lường thành công

Hệ thống track 5 nhóm metrics:

### 🎯 Business (cho PM, Founder)
- **GMV**: tổng giá trị đơn hàng (mục tiêu Y1: 50 tỷ VND)
- **Take rate**: SMP revenue / GMV (target 15%)
- **AOV**: giá trị trung bình/đơn (target 800k VND)
- **Repeat customer rate**: % khách quay lại (target 40% sau 6 tháng)

### ⚙️ Operational (cho Ops Manager)
- **Order completion rate**: % đơn complete / đơn create (target ≥85%)
- **Dispatch SLA**: % đơn được assign trong 5 phút (target ≥90%)
- **Stuck orders**: đơn kẹt > 24h (alert nếu > 10/ngày)

### 🔧 Agent (cho Agent Squad)
- **Active agents (DAU)**: thợ online ≥4h/ngày (target 60% pool)
- **Agent rating avg**: trung bình rating (target ≥4.5)
- **Acceptance rate**: % invitation được accept (target ≥40%)
- **No-show rate**: % đơn thợ ko đến (target ≤2%)

### 🏢 Partner (cho Partner Squad)
- **Partner GMV**: GMV từ partner orders (target 30% total Y2)
- **Wallet health**: % partner có balance ≥ 5M (target ≥80%)

### 🛠️ Technical (cho Engineering)
- API latency p95 < 500ms
- Dispatch latency p95 < 2s
- Uptime ≥99.9%
- Error rate < 0.1%

Chi tiết formula + dashboard mapping: xem [Doc 14 · KPI Metrics](05-ba/14-kpi-metrics-definition.md).

---

## 10. Integration với hệ thống ngoài

SMP không phải hệ thống isolated. Nó kết nối với:

### `inside` · HRM/CRM nội bộ
- **Direction**: Bidirectional
- **Data**: Customer profiles, payment status, sales pipeline
- **Method**: REST API + webhooks (HMAC SHA256)
- **Failover**: Circuit breaker (5 fails → open 30s → half-open retry)

### `wms` · Warehouse Management
- **Direction**: SMP → wms (read stock) + wms → SMP (stock updates)
- **Data**: Material stock per warehouse + reservation
- **Method**: REST API
- **Use case**: SUR app check stock khi nhập báo giá

### Payment Gateways (qua inside)
- VNPay, MoMo, ZaloPay
- SMP không handle trực tiếp, route qua inside

### SMS / Push / Email Providers
- SMS: Twilio (backup eSMS Vietnam)
- Push: Firebase Cloud Messaging
- Email: SendGrid

### Maps & Geocoding
- Google Maps API (geocoding, distance matrix)
- Mapbox (rendering ở mobile)

Chi tiết: xem [Doc 01 · Architecture section 5 (Data flow)](01-architecture/01-architecture.md) và [Doc 03 · API Contract](03-backend/03-api-contract.md).

---

## 11. Roadmap · v3.1 → v3.3 → tương lai

### v3.1 · 2-Agent Survey + Execution (đã release)
- Tách Survey và Execution thành 2 agent roles
- 10-stage order lifecycle
- Geo coverage zones
- 8-stage flow chính

### v3.2 · Material BOM + inside/wms Integration (đã release)
- Material types + variants + BOM
- WMS stock check trong Survey
- Customer sync từ inside
- Payment integration

### v3.3 · Partner Platform 3-sided (đã release · CURRENT)
- Partner B2B với 4 roles
- Private + Open dispatch
- Wallet system
- Partner pricing override
- Mobile triple-mode (Customer + Tech + Partner)

### v3.4 · Process Documentation + v4.0 Design (✅ COMPLETED · CURRENT)
- ✅ 16 docs chuẩn hóa process cho team 10+
- ✅ Doc 00 System Functional Overview
- ✅ Doc tổng hợp + onboarding paths cho 6 roles
- ✅ **v4.0 architecture preview**: Foundation terms, DB conventions, Rules Engine, Kafka, Masking, Compliance
- ✅ MIGRATION-PLAN-v4.md (xem root repo)
- 🔄 Deployed: GitHub Pages + Cloudflare (16 docs accessible)

### v3.5 · Database + Go foundations (📅 planned · 6 weeks)
- ⏳ Refactor DB: multi-currency columns (`amount` + `currency`)
- ⏳ Refactor DB: UTC timestamps (`*_utc` columns)
- ⏳ Add `country_code` to user-tenant tables
- ⏳ Create `smp_global` DB (countries, currencies, currency_rates, tax_configs, i18n_translations)
- ⏳ Build Go packages: `pkg/money`, `pkg/clock`, `pkg/timezone`, `pkg/i18n`
- ⏳ Linter rules forbid hardcoded VND + `time.Now()`
- 📖 Reference: [Doc 02 section 2.5](02-database/02-database-schema.md#25--v40-conventions--global-ready-schema-patterns), [Doc 04 section 1.15](03-backend/04-coding-standards-and-dev-setup.md)

### v3.6 · Rules Engine YAML (📅 planned · 4 weeks)
- ⏳ Build `pkg/rules` (antonmedv/expr + hot-reload via ConfigMap)
- ⏳ Migrate 80+ rules từ Go code sang `rules_engine.yaml`
- ⏳ Setup repo `smp-rules-config` riêng với BA-friendly PR workflow
- ⏳ Rule ID convention mới: `<country>.<category>.<num>` (multi-country ready)
- 📖 Reference: [Doc 01 section 7.5](01-architecture/01-architecture.md), [Doc 15 Appendix](05-ba/15-business-rules.md#appendix--full-rules_engineyaml-sample-v40-preview)

### v3.7 · Event-driven + CQRS + CDC (📅 planned · 8 weeks · HIGH RISK)
- ⏳ Apache Kafka 3.7 KRaft cluster (3 brokers production)
- ⏳ 9 topics + DLQ (orders, payments, agents, partners, dispatch, quality, integration, audit)
- ⏳ Debezium MySQL → Kafka CDC pipeline
- ⏳ Elasticsearch read model với ILM (hot 7d, warm 60d, cold 1y)
- ⏳ Dashboard queries migrate sang ES (10x faster aggregations)
- ⏳ Saga pattern cho long transactions
- ⏳ Outbox pattern cho atomic write+publish
- 📖 Reference: [Doc 01 section 7.6](01-architecture/01-architecture.md), [Doc 14 section 10.6](05-ba/14-kpi-metrics-definition.md), [Doc 11 INC-006 to INC-009](07-devops/11-runbook-incidents.md)

### v3.8 · Dynamic Data Masking + DSR (📅 planned · 6 weeks)
- ⏳ API Gateway middleware auto-masks PII (default-deny)
- ⏳ 9 PII unmask scopes (`pii.unmask.phone`, `pii.unmask.bank_account`...)
- ⏳ Endpoint `POST /pii/unmask` cho sensitive fields với MFA-fresh
- ⏳ Endpoint `GET /pii/audit-trail` cho DSR compliance
- ⏳ Automated DSR export workflow (7-day SLA cho PDPA)
- ⏳ Automated DSR deletion workflow (30-day grace + anonymize)
- 📖 Reference: [Doc 13 section 5.5](08-security/13-data-classification-encryption.md), [Doc 08 section 4.5](08-security/08-auth-spec.md), [Doc 03 section 11.5](03-backend/03-api-contract.md)

### v4.0 GA · Multi-region Global Ready (📅 planned · Q4 2026 · 4 weeks final)

> **Quan trọng**: v3.x = Vietnam-only. v4.0 = Global ready (Thailand, Indonesia, Philippines, US, China...).

**4 nhóm cải tiến lớn** (đã design đầy đủ trong v3.4 docs, sẽ implement Phase 1-4 above trước):

#### 4.0.1 · Chuẩn hoá Quốc tế (Global Scaling)
- **Đa tiền tệ**: Mọi cột tiền lưu kèm ISO 4217 currency code (`VND`, `USD`, `CNY`, `THB`...). Money type = `{ amount: BIGINT (minor units), currency_code: CHAR(3) }`.
- **Thuế suất linh hoạt**: Tax configuration per country + service category. VN VAT 10%, US Sales Tax theo bang, SG GST 9%.
- **Múi giờ UTC**: Tất cả timestamp DB lưu UTC. Convert sang local timezone ở API Gateway/Frontend theo `Asia/Ho_Chi_Minh`, `Asia/Bangkok`...
- **Đa ngôn ngữ động**: `i18n_translations` table (key, locale, value). Frontend lookup `(key, locale)` với fallback chain. Hỗ trợ Vietnamese, English, Thai, Indonesian, Chinese.

#### 4.0.2 · Rules Engine Config-as-Code
- **Single file**: `rules_engine.yaml` chứa 80+ business rules (dispatch, pricing, KYC, payment).
- **Storage**: Kubernetes ConfigMap, mount vào pod, hot-reload khi update (no restart).
- **Library**: `antonmedv/expr` (Go, ~20kb, sandbox eval expression).
- **Rule ID convention mới**: `<country>.<category>.<num>` (vd `vn.dispatch.001`, `us.pricing.005`). Multi-country ready.
- **Hot Reload**: file watcher monitor ConfigMap path, reload rules vào memory trong ~30s.

#### 4.0.3 · Event-Driven Architecture (Kafka + CQRS + CDC)
- **Apache Kafka**: replace HTTP sync calls cho dispatch + pricing. Topics: `orders.events`, `payments.events`, `agents.events`, `partners.events`.
- **Producers/Consumers**: order-svc, dispatch-engine, finance-svc, notification-svc.
- **Partitioning**: 12 partitions/topic, key = order_id để preserve order per entity.
- **CDC qua Debezium**: capture MySQL binlog → publish Kafka → sink Elasticsearch/ClickHouse.
- **CQRS**: Write side MySQL (transactional), Read side Elasticsearch (dashboard + search). Eventual consistency 1-5s.
- **Redis Streams**: fallback nhẹ cho notification, GPS tracking (giảm tải MySQL).

#### 4.0.4 · Security & Compliance Layer
- **Dynamic Data Masking**: API Gateway middleware tự che PII (phone, email, bank acc) dựa trên scope của caller. CSKH thấy `0912****890`, Finance thấy full.
- **Data Sovereignty**: Sharding by country. Cluster `smp-vn` (Singapore), `smp-cn` (Beijing), `smp-us` (Virginia). Routing theo `country_code`.
- **Automated Reconciliation**: Daily job đối soát partner wallet vs payment gateway, detect mismatch >0.01%, auto-alert Finance.
- **Compliance**: PDPA (VN), CPRA (US California), PIPL (China), GDPR (EU). Right to Erasure (30 ngày), Data Portability (7 ngày), Breach notification 72h.

**Migration strategy**: 5 phases tuần tự (v3.5 → v3.6 → v3.7 → v3.8 → v4.0 GA), ~28 weeks total. Each phase ships independently with rollback path. Xem chi tiết full plan ở [MIGRATION-PLAN-v4.md](MIGRATION-PLAN-v4.md).

---

## 12. Quick links · Đào sâu

Khi đọc xong overview này, đào sâu theo nhu cầu:

| Bạn muốn biết... | Đọc doc |
|---|---|
| Kiến trúc kỹ thuật + diagram chi tiết | [Doc 01 · Architecture](01-architecture/01-architecture.md) |
| Database schema, DDL, ERD | [Doc 02 · Database Schema](02-database/02-database-schema.md) |
| API endpoints, request/response | [Doc 03 · API Contract](03-backend/03-api-contract.md) |
| Code conventions + dev setup | [Doc 04 · Coding Standards](03-backend/04-coding-standards-and-dev-setup.md) |
| Thuật ngữ, định nghĩa | [Doc 05 · Glossary](05-ba/05-glossary.md) |
| Environment configs (dev/staging/prod) | [Doc 06 · Env Matrix](07-devops/06-environment-matrix.md) |
| CI/CD pipeline | [Doc 07 · CI/CD](07-devops/07-ci-cd-pipeline.md) |
| Auth + RBAC chi tiết | [Doc 08 · Auth Spec](08-security/08-auth-spec.md) |
| User stories + Gherkin acceptance | [Doc 09 · User Stories](05-ba/09-user-stories.md) |
| Test plan + cases + bug template | [Doc 10 · Test Plan](06-qa/10-test-plan-cases.md) |
| Incident playbook (top 5 sự cố) | [Doc 11 · Runbook](07-devops/11-runbook-incidents.md) |
| Audit log spec | [Doc 12 · Audit Log](08-security/12-audit-log-spec.md) |
| Data classification + encryption | [Doc 13 · Data Class](08-security/13-data-classification-encryption.md) |
| KPI formulas + dashboard | [Doc 14 · KPI Metrics](05-ba/14-kpi-metrics-definition.md) |
| Business rules chi tiết (80+ rules) | [Doc 15 · Business Rules](05-ba/15-business-rules.md) |

---

## Reading order cho onboarding

### Stakeholder (CEO, Investor, Partner mới)
**Đọc duy nhất doc này** + xem demo. Đủ 30-45 phút.

### Dev mới
1. Doc 00 (this) — 45 min
2. Doc 05 (Glossary) — 30 min
3. Doc 01 (Architecture) — 1h
4. Doc 04 (Setup) — 2h (vừa đọc vừa setup máy)
5. Doc 02 (Schema) — 1h
6. Doc 03 (API) — 1h
7. Doc 08 (Auth) — 30 min
**Total**: ~6h, có thể onboard trong 1 ngày

### BA mới
1. Doc 00 (this) — 45 min
2. Doc 05 (Glossary) — 30 min
3. SPEC v3.x (3 file product spec) — 2h
4. Doc 09 (User Stories) — 1h
5. Doc 15 (Business Rules) — 2h
6. Doc 14 (KPI) — 30 min
**Total**: ~6h

### QC mới
1. Doc 00 (this) — 45 min
2. Doc 05 (Glossary) — 30 min
3. Doc 09 (User Stories) — 1h
4. Doc 15 (Business Rules) — 2h
5. Doc 10 (Test Plan) — 1h
6. Doc 03 (API contract) — 1h (để hiểu endpoint test)
**Total**: ~6h

### DevOps mới
1. Doc 00 (this) — 45 min
2. Doc 01 (Architecture) — 1h
3. Doc 06 (Env Matrix) — 30 min
4. Doc 07 (CI/CD) — 1h
5. Doc 11 (Runbook) — 1h
6. Doc 13 (Encryption) — 30 min
**Total**: ~5h

### Security Reviewer
1. Doc 00 (this) — 45 min
2. Doc 08 (Auth Spec) — 1h
3. Doc 12 (Audit Log) — 30 min
4. Doc 13 (Data Class) — 1h
5. Doc 11 (Runbook) — 30 min
**Total**: ~4h

---

## Changelog

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026-05-27 | Docs team | Initial · gom từ docs 01, 05, 09, 14, 15 |

---

**Maintainers**: BA team + Tech lead
**Review cycle**: mỗi sprint (2 tuần) hoặc khi có release mới
**Slack**: `#engineering-docs`
