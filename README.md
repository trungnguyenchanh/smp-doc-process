# SMP Process Documentation

Tài liệu chuẩn hoá process cho team SMP (10+ người · BA, Dev, QC, DevOps, Security, Finance, Legal) sử dụng **Go + MySQL + Redis + MongoDB**.

---

## Mục đích

- Chuẩn hoá how-to-build cho 6 vai trò
- Reference cho onboarding member mới
- Single source of truth cho decisions kỹ thuật
- Tiền đề cho audit, compliance, scaling

---

## Danh sách docs

### 🌟 Start here
| # | Doc | Audience | Purpose |
|---|---|---|---|
| **00** | [**System Functional Overview**](./00-system-functional-overview.md) | **All roles** | **Executive summary · gom module + feature + flows + KPIs · đọc đầu tiên trước khi đào sâu** |

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
| 09 | [User Stories + Acceptance Criteria](./05-ba/09-user-stories.md) | BA, Dev, QC | 6 epics: Customer, Technician, Partner, Material BOM, Integration, Quality · Gherkin format · DoD · Fibonacci sizing |
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

### 💰 Finance
| # | Doc | Audience | Purpose |
|---|---|---|---|
| 16 | [Finance Ledger Spec](./09-finance/16-finance-ledger-spec.md) | Finance, Backend, BA | Chart of Accounts · Pattern 2 (control account + subledger dimension) · double-entry bookkeeping · journal templates (gateway/wallet/COD/refund) · rounding rule (residual N=P-C-V) · VAT từ config · ledger↔settlement SoT |
| 17 | [Payment & Settlement Lifecycle](./09-finance/17-payment-settlement-lifecycle.md) | Backend, Finance, BA, QC | Settlement state machine · 9 states (INIT→AWAITING_GATEWAY→SETTLED→...) · transition matrix · invariants · COD lifecycle (collected→remitted) · 5 lifecycle questions chốt cứng |
| 18 | [Event Catalog](./09-finance/18-event-catalog.md) | Backend, BA, QC, DevOps | Outbox + DLQ pattern · envelope schema · 16 events registry · payload contracts (SettlementSettled, CODCollected, RefundSettled) · consumer idempotency · versioning rules |

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
5. [03 API Contract](./03-backend/03-api-contract.md) — endpoints
6. [02 Database Schema](./02-database/02-database-schema.md) — data model

### Backend Engineer
1. Onboarding flow trên
2. [08 Auth](./08-security/08-auth-spec.md) — JWT + RBAC
3. [15 Business Rules](./05-ba/15-business-rules.md) — 80+ rules
4. [16-18 Finance docs](./09-finance/) — ledger + events
5. [12 Audit Log](./08-security/12-audit-log-spec.md) — observability

### Frontend Engineer
1. [00 System Overview](./00-system-functional-overview.md)
2. [05 Glossary](./05-ba/05-glossary.md)
3. [03 API Contract](./03-backend/03-api-contract.md)
4. [08 Auth](./08-security/08-auth-spec.md)
5. [09 User Stories](./05-ba/09-user-stories.md)

### QA Engineer
1. [00 System Overview](./00-system-functional-overview.md)
2. [09 User Stories](./05-ba/09-user-stories.md) — acceptance criteria
3. [15 Business Rules](./05-ba/15-business-rules.md) — edge cases
4. [10 Test Plan](./06-qa/10-test-plan-cases.md) — test strategy
5. [11 Runbook](./07-devops/11-runbook-incidents.md) — incident scenarios

### DevOps / SRE
1. [00 System Overview](./00-system-functional-overview.md)
2. [01 Architecture](./01-architecture/01-architecture.md)
3. [06 Environment Matrix](./07-devops/06-environment-matrix.md)
4. [07 CI/CD](./07-devops/07-ci-cd-pipeline.md)
5. [11 Runbook](./07-devops/11-runbook-incidents.md)
6. [13 Data Classification](./08-security/13-data-classification-encryption.md)

### Security Engineer
1. [00 System Overview](./00-system-functional-overview.md)
2. [01 Architecture](./01-architecture/01-architecture.md)
3. [08 Auth Spec](./08-security/08-auth-spec.md)
4. [12 Audit Log](./08-security/12-audit-log-spec.md)
5. [13 Data Classification](./08-security/13-data-classification-encryption.md)
6. [21 PDPL Data Policy](./10-legal/21-pdpl-data-policy.md)

### Finance / Accounting
1. [00 System Overview](./00-system-functional-overview.md)
2. [16 Finance Ledger](./09-finance/16-finance-ledger-spec.md)
3. [17 Payment Lifecycle](./09-finance/17-payment-settlement-lifecycle.md)
4. [14 KPI Metrics](./05-ba/14-kpi-metrics-definition.md)
5. [20 COD Policy](./10-legal/20-cod-payment-policy.md)

### BA / Product Manager
1. [00 System Overview](./00-system-functional-overview.md)
2. [05 Glossary](./05-ba/05-glossary.md)
3. [09 User Stories](./05-ba/09-user-stories.md)
4. [14 KPI Metrics](./05-ba/14-kpi-metrics-definition.md)
5. [15 Business Rules](./05-ba/15-business-rules.md)

---

## Cấu trúc thư mục

```
smp-doc-process/
├── README.md                                 # File này
├── CONTRIBUTING.md                           # Quy ước đóng góp
├── DEPLOY-GUIDE.md                           # Hướng dẫn deploy docs site
├── mkdocs.yml                                # MkDocs Material config
├── requirements.txt                          # Python deps
├── requirements-docs.txt                     # Docs build deps
└── docs/
    ├── index.md                              # Landing page
    ├── 00-system-functional-overview.md      # Executive summary
    ├── 01-architecture/                      # C4 + service catalog
    ├── 02-database/                          # ERD + DDL + Mongo + Redis
    ├── 03-backend/                           # API + coding standards
    ├── 05-ba/                                # Glossary, user stories, KPI, rules
    ├── 06-qa/                                # Test plan + cases
    ├── 06-frontend/                          # Mobile wireframes + mockups
    ├── 07-devops/                            # Env matrix, CI/CD, runbook
    ├── 08-security/                          # Auth, audit log, data classification
    ├── 09-finance/                           # Ledger, lifecycle, events
    └── 10-legal/                             # Service guarantee, COD, PDPL
```

---

## Documentation site

Docs được build bằng **MkDocs Material** và deploy tự động tại:

🌐 **https://smp-doc-process.pages.dev/**

Build local:
```bash
pip install -r requirements-docs.txt
mkdocs serve
# → http://localhost:8000
```

Xem chi tiết deployment tại [DEPLOY-GUIDE.md](./DEPLOY-GUIDE.md).

---

## Đóng góp

Xem [CONTRIBUTING.md](./CONTRIBUTING.md) cho quy ước:
- Markdown style
- Naming convention cho doc mới
- PR review process

---

## License

Internal Use Only · © 2026 SMP Team
