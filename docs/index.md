# SMP Process Documentation · v3.4

**Phiên bản**: 3.4 · **Ngày**: 2026-05-28 · **Scope**: 21 docs (15 baseline + 3 finance + 3 legal) + Migration Plan

Tài liệu chuẩn hoá process cho team SMP (10+ người · BA, Dev, QC, DevOps, Security, Finance, Legal) sử dụng **Go + MySQL + Redis + MongoDB**.

🚀 **Latest update**: Tích hợp Finance Ledger Spec (Pattern 2 double-entry) + Trust & Legal Policy Pack (PDPL VN compliance) từ Tech Lead review. Migration plan REVISED · 6 phases · 34 weeks. Sẵn sàng v3.5 kick-off.

---

## Mục đích

Sau khi product spec v3.1/v3.2/v3.3 đã ổn (UI/UX + business flow), team chuyển sang **execution phase**. Tài liệu này:
- Chuẩn hoá how-to-build cho 6 vai trò
- Reference cho onboarding member mới
- Single source of truth cho decisions kỹ thuật
- Tiền đề cho audit, compliance, scaling

---

## Danh sách 16 docs + 1 plan

### 🌟 Start here
| # | Doc | Audience | Purpose |
|---|---|---|---|
| **00** | [**System Functional Overview**](./00-system-functional-overview.md) | **All roles** | **Executive summary · gom module + feature + flows + KPIs · đọc đầu tiên trước khi đào sâu** |
| 🔮 | [**MIGRATION-PLAN-v4.md**](./MIGRATION-PLAN-v4.md) | **Tech Lead, BA Lead, DevOps** | **5-phase plan v3.4 → v4.0 · timeline · risks · rollback · success metrics** |

### 🏛️ Architecture
| # | Doc | Audience | Purpose |
|---|---|---|---|
| 01 | [Architecture · C4 + Service Catalog](./01-architecture/01-architecture.md) | All eng | Big picture: context, container, component diagrams · 10 microservices catalog · cross-cutting concerns |

### 🗄️ Database
| # | Doc | Audience | Purpose |
|---|---|---|---|
| 02 | [Database Schema · ERD + DDL](./02-database/02-database-schema.md) | DBA, Backend | Full MySQL DDL (smp_order, smp_catalog, smp_agent, smp_partner, smp_geo) + MongoDB collections + Redis patterns + naming + indexing + retention |

### ⚙️ Backend
| # | Doc | Audience | Purpose |
|---|---|---|---|
| 03 | [API Contract · OpenAPI 3.0](./03-backend/03-api-contract.md) | Backend, Frontend, Integration | REST conventions · auth · pagination · errors · all endpoints (orders, partners, dispatch, finance, quality) · rate limits |
| 04 | [Go Coding Standards + Local Dev Setup](./03-backend/04-coding-standards-and-dev-setup.md) | Backend | Go style guide · project structure · error handling · testing · DI · docker-compose for infra · Git workflow |

### 📊 BA · Business Analyst
| # | Doc | Audience | Purpose |
|---|---|---|---|
| 05 | [Glossary · Thuật ngữ thống nhất](./05-ba/05-glossary.md) | All | VN-EN dictionary cho domain (Service Template, Step, Agent, Partner, BOM, Stage, Dispatch...) |
| 09 | [User Stories + Acceptance Criteria](./05-ba/09-user-stories.md) | BA, Dev, QC | 6 epics: Customer, Technician, Partner v3.3, Material BOM, Integration, Quality · Gherkin format · DoD · Fibonacci sizing |
| 14 | [KPI / Metrics Definition](./05-ba/14-kpi-metrics-definition.md) | BA, PM, Data | Formulas cho 30+ metrics: GMV, completion rate, dispatch SLA, agent utilization, partner wallet health · dashboard mapping |
| 15 | [Business Rules](./05-ba/15-business-rules.md) | All | 80+ rules tập trung: dispatch, pricing, payment, KYC, stages, materials, partners, integration, quality, notifications, retention |

### 🧪 QA
| # | Doc | Audience | Purpose |
|---|---|---|---|
| 10 | [Test Plan + Test Cases](./06-qa/10-test-plan-cases.md) | QC | Test strategy · TC template · sample cases cho Order/Partner/Dispatch/BOM/Integration · performance criteria (k6) · bug template · severity matrix |

### 🚀 DevOps
| # | Doc | Audience | Purpose |
|---|---|---|---|
| 06 | [Environment Matrix · dev/staging/prod](./07-devops/06-environment-matrix.md) | DevOps, Backend | URLs · sizing · secrets (Vault) · per-env config · deployment promotion · access control · DNS · cost budgets |
| 07 | [CI/CD Pipeline](./07-devops/07-ci-cd-pipeline.md) | DevOps | GitHub Actions sample (lint/test/build/scan/deploy) · GitOps + ArgoCD · Helm · blue/green strategy · migration in pipeline · SemVer releases · rollback |
| 11 | [Runbook · Top 5 Incidents](./07-devops/11-runbook-incidents.md) | SRE, On-call | Detect → Triage → Mitigate → Communicate → Resolve · 5 scenarios: dispatch down, webhook fail, MySQL lag, Redis OOM, integration circuit · escalation matrix |

### 🔒 Security
| # | Doc | Audience | Purpose |
|---|---|---|---|
| 08 | [Auth Spec · JWT + RBAC](./08-security/08-auth-spec.md) | Backend, Frontend, Security | OAuth flows (customer phone OTP, ops email+2FA) · JWT structure (RS256) · scope catalog per role · resource-level authZ · password policy · session management · token revocation · CORS · rate limits |
| 12 | [Audit Log Spec](./08-security/12-audit-log-spec.md) | Security, Backend, Compliance | Event categories · schema · naming convention · 7-year retention · query API · tamper resistance · PII handling · compliance mapping (PDPA VN) |
| 13 | [Data Classification + Encryption Policy](./08-security/13-data-classification-encryption.md) | Security, Backend, DevOps | L0-L4 classification · field encryption (AES-256) · key management (Vault) · TLS · PII display masking · vendor risk · compliance checklist · breach response |

### 💰 Finance (v3.5+ specification · DRAFT)
| # | Doc | Audience | Purpose |
|---|---|---|---|
| 16 | [Finance Ledger Spec](./09-finance/16-finance-ledger-spec.md) | Finance, Backend, BA | Chart of Accounts · Pattern 2 (control account + subledger dimension) · double-entry bookkeeping · journal templates (gateway/wallet/COD/refund) · rounding rule (residual N=P-C-V) · VAT từ config · ledger↔settlement SoT |
| 17 | [Payment & Settlement Lifecycle](./09-finance/17-payment-settlement-lifecycle.md) | Backend, Finance, BA, QC | Settlement state machine · 9 states (INIT→AWAITING_GATEWAY→SETTLED→...) · transition matrix · invariants · COD lifecycle (collected→remitted) · 5 lifecycle questions chốt cứng |
| 18 | [Event Catalog](./09-finance/18-event-catalog.md) | Backend, BA, QC, DevOps | Outbox + DLQ pattern (KHÔNG cần Kafka cho pilot) · envelope schema · 16 events registry · payload contracts (SettlementSettled, CODCollected, RefundSettled) · consumer idempotency · versioning rules |

> **Note**: 3 docs trên là **DRAFT specification** cho v3.5+ implementation. Schema hiện tại (Doc 02) sẽ dual-write vào ledger mới ở Phase v3.5, cutover ở v3.6.

### 📜 Legal & Compliance (DRAFT · awaits VN lawyer review)
| # | Doc | Audience | Purpose |
|---|---|---|---|
| 19 | [Service Guarantee Policy](./10-legal/19-service-guarantee-policy.md) | Legal, Ops, BA, Founder | Phạm vi bảo hành · re-service miễn phí · exclusions · SLA 48h · ai gánh chi phí · quỹ `service_guarantee_reserve` (KHÔNG dùng "insurance") |
| 20 | [COD Payment Policy](./10-legal/20-cod-payment-policy.md) | Legal, Ops, Finance, BA | Điều kiện COD (trần đơn, KYC thợ) · 2 mốc UX vs kế toán · nghĩa vụ remit T+1 · refund-before-remit · e-invoice theo NĐ 70/2025 · viện dẫn luật chi tiết |
| 21 | [PDPL Data Policy](./10-legal/21-pdpl-data-policy.md) | Legal, DPO, Security, Ops | PDPL VN (Luật 91/2025/QH15 + NĐ 356/2025) · phân loại dữ liệu · retention table · ngoại lệ legal hold · quyền chủ thể · checklist tuân thủ · chat anti-off-platform |

> **⚠️ Disclaimer chung**: 3 docs Legal là **bản thảo định hình chính sách**, KHÔNG phải tư vấn pháp lý. Mọi điều khoản công bố cho KH/thợ/partner phải qua luật sư VN rà soát.

---

## Reading order theo vai trò

> 💡 **Mọi role bắt đầu với [Doc 00 · System Functional Overview](./00-system-functional-overview.md)** — đọc 30-45 phút để hiểu big picture trước khi đào sâu chi tiết.

### Dev mới onboard
1. [00 System Functional Overview](./00-system-functional-overview.md) — big picture (45 min)
2. [05 Glossary](./05-ba/05-glossary.md) — hiểu thuật ngữ
3. [01 Architecture](./01-architecture/01-architecture.md) — bức tranh tổng
4. [04 Coding Standards + Setup](./03-backend/04-coding-standards-and-dev-setup.md) — cài máy + chuẩn code
5. [02 Database Schema](./02-database/02-database-schema.md) — domain mình sẽ làm
6. [03 API Contract](./03-backend/03-api-contract.md) — contract với FE/other svc
7. [08 Auth Spec](./08-security/08-auth-spec.md) — security baseline
8. [15 Business Rules](./05-ba/15-business-rules.md) — không gì khác trừ doc này khi confused
9. [09 User Stories](./05-ba/09-user-stories.md) — what to build

### BA mới onboard
1. [00 System Functional Overview](./00-system-functional-overview.md) — big picture (45 min)
2. [05 Glossary](./05-ba/05-glossary.md)
3. SPEC-v3.md, SPEC-v3.2.md, SPEC-v3.3.md (product specs)
4. [09 User Stories](./05-ba/09-user-stories.md)
5. [15 Business Rules](./05-ba/15-business-rules.md)
6. [14 KPI Metrics](./05-ba/14-kpi-metrics-definition.md)

### QC mới onboard
1. [00 System Functional Overview](./00-system-functional-overview.md) — big picture (45 min)
2. [05 Glossary](./05-ba/05-glossary.md)
3. [09 User Stories](./05-ba/09-user-stories.md)
4. [15 Business Rules](./05-ba/15-business-rules.md)
5. [10 Test Plan + Cases](./06-qa/10-test-plan-cases.md)
6. [03 API Contract](./03-backend/03-api-contract.md) (cho API testing)

### DevOps mới onboard
1. [00 System Functional Overview](./00-system-functional-overview.md) — big picture (45 min)
2. [01 Architecture](./01-architecture/01-architecture.md)
3. [06 Environment Matrix](./07-devops/06-environment-matrix.md)
4. [07 CI/CD Pipeline](./07-devops/07-ci-cd-pipeline.md)
5. [11 Runbook Incidents](./07-devops/11-runbook-incidents.md)
6. [13 Data Classification](./08-security/13-data-classification-encryption.md) (encryption + Vault)

### Security review
1. [00 System Functional Overview](./00-system-functional-overview.md) — big picture (45 min)
2. [08 Auth Spec](./08-security/08-auth-spec.md)
3. [12 Audit Log](./08-security/12-audit-log-spec.md)
4. [13 Data Classification](./08-security/13-data-classification-encryption.md)
5. [11 Runbook](./07-devops/11-runbook-incidents.md) (incident response)

### Finance / Accounting (v3.5+ implementation team)
1. [00 System Functional Overview](./00-system-functional-overview.md) — big picture (45 min)
2. [05 Glossary](./05-ba/05-glossary.md) — financial terms first
3. [16 Finance Ledger Spec](./09-finance/16-finance-ledger-spec.md) — Pattern 2, journal templates, rounding
4. [17 Payment & Settlement Lifecycle](./09-finance/17-payment-settlement-lifecycle.md) — state machine
5. [18 Event Catalog](./09-finance/18-event-catalog.md) — events between services
6. [15 Business Rules](./05-ba/15-business-rules.md) — pricing + payment rules
7. [20 COD Payment Policy](./10-legal/20-cod-payment-policy.md) — legal context cho COD

### Legal / Compliance (DPO, Legal Officer)
1. [00 System Functional Overview](./00-system-functional-overview.md) — big picture (45 min)
2. [19 Service Guarantee Policy](./10-legal/19-service-guarantee-policy.md) — warranty/guarantee scope
3. [20 COD Payment Policy](./10-legal/20-cod-payment-policy.md) — COD legal obligations
4. [21 PDPL Data Policy](./10-legal/21-pdpl-data-policy.md) — PDPL VN compliance
5. [13 Data Classification](./08-security/13-data-classification-encryption.md) — technical encryption mapping
6. [12 Audit Log](./08-security/12-audit-log-spec.md) — PII access tracking

### Stakeholder (CEO, Investor, Partner mới)
**Đọc Doc 00 + Legal Pack**:
1. [00 System Functional Overview](./00-system-functional-overview.md) — 30-45 min
2. [19 Service Guarantee Policy](./10-legal/19-service-guarantee-policy.md) — hiểu commitment với KH
3. [20 COD Payment Policy](./10-legal/20-cod-payment-policy.md) — hiểu risk model COD
4. [21 PDPL Data Policy](./10-legal/21-pdpl-data-policy.md) — hiểu data compliance posture

---

## Cách maintain

Doc này là **living document**. Quy trình đóng góp đầy đủ xem [CONTRIBUTING.md](./CONTRIBUTING.md).

Tóm tắt:
1. PR vào repo `smp-doc-process`
2. Reviewer = 1 dev + 1 BA (cross-discipline)
3. Update version + changelog ở dưới
4. Notify team qua Slack `#engineering-docs`

## Deploy / Setup

Lần đầu push lên GitHub: xem [DEPLOY-GUIDE.md](./DEPLOY-GUIDE.md).

Repo URL: https://github.com/trungnguyenchanh/smp-doc-process

**Site đẹp** (sau khi setup MkDocs · xem DEPLOY-GUIDE): https://trungnguyenchanh.github.io/smp-doc-process/

## Phase 2 docs (chưa build · Batch C)

Khi đi GA cần thêm:
- Threat model (STRIDE per service)
- Performance test results + capacity planning
- Disaster recovery plan (RPO/RTO)
- Compliance checklist (đầy đủ PDPA + financial)
- Privacy policy + ToS (legal review)
- C4 Level 3 component diagrams chi tiết
- ADRs đầy đủ (architecture decision records)

## Changelog

| Version | Date | Change | Author |
|---|---|---|---|
| 3.4 | 2026-05-27 | Initial · 15 docs Batch A+B | Team SMP |
