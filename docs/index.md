# SMP Process Documentation

Tài liệu chuẩn hoá process cho team SMP (10+ người · BA, Dev, QC, DevOps, Security, Finance, Legal) sử dụng **Go + MySQL + Redis + MongoDB**.

---

## Mục đích

Tài liệu này:

- Chuẩn hoá how-to-build cho 6 vai trò
- Reference cho onboarding member mới
- Single source of truth cho decisions kỹ thuật
- Tiền đề cho audit, compliance, scaling

---

## Quick start

### 🌟 Đọc đầu tiên (tất cả vai trò)
[**Doc 00 · System Functional Overview**](./00-system-functional-overview.md) — Executive summary 30-45 phút mô tả toàn bộ chức năng hệ thống SMP. Đọc trước khi đi sâu vào các docs kỹ thuật khác.

### Theo vai trò
| Role | Priority docs |
|---|---|
| **Dev mới onboard** | 00 → 05 (glossary) → 01 (architecture) → 04 (setup) → 03 (API) → 02 (DB) |
| **Backend** | 00 → 03 (API) → 02 (DB) → 08 (auth) → 15 (rules) → 16-18 (finance) |
| **Frontend** | 00 → 05 → 03 (API) → 08 (auth) → 09 (user stories) |
| **QA** | 00 → 09 (stories) → 15 (rules) → 10 (test plan) → 11 (runbook) |
| **DevOps / SRE** | 00 → 01 (arch) → 06 (env) → 07 (CI/CD) → 11 (runbook) → 13 (encryption) |
| **Security** | 00 → 01 → 08 (auth) → 12 (audit) → 13 (encryption) → 21 (PDPL) |
| **Finance** | 00 → 16 (ledger) → 17 (lifecycle) → 14 (KPI) → 20 (COD policy) |
| **BA / PM** | 00 → 05 (glossary) → 09 (stories) → 14 (KPI) → 15 (rules) |

---

## Doc map · 22 docs trong 10 categories

### 🌟 Start here
- [00 · System Functional Overview](./00-system-functional-overview.md)

### 🏛️ Architecture
- [01 · Architecture (C4 + Service Catalog)](./01-architecture/01-architecture.md)

### 🗄️ Database
- [02 · Database Schema (ERD + DDL)](./02-database/02-database-schema.md)

### ⚙️ Backend
- [03 · API Contract (OpenAPI 3.0)](./03-backend/03-api-contract.md)
- [04 · Go Coding Standards + Local Dev Setup](./03-backend/04-coding-standards-and-dev-setup.md)

### 📊 BA · Business Analyst
- [05 · Glossary](./05-ba/05-glossary.md)
- [09 · User Stories + Acceptance Criteria](./05-ba/09-user-stories.md)
- [14 · KPI / Metrics Definition](./05-ba/14-kpi-metrics-definition.md)
- [15 · Business Rules (80+ rules)](./05-ba/15-business-rules.md)

### 🧪 QA
- [10 · Test Plan + Test Cases](./06-qa/10-test-plan-cases.md)

### 🎨 Frontend (Mobile)
- [23 · Multi-agent Mobile Wireframes](./06-frontend/23-multi-agent-mobile-wireframes.md)
- [24 · Multi-agent Mobile Mockup](./06-frontend/24-multi-agent-mobile-mockup.html)
- [26 · Warranty Package Mobile Mockup](./06-frontend/26-warranty-package-mobile-mockup.html)
- [27 · Warranty Packages Comparison](./06-frontend/27-warranty-packages-comparison.html)

### 🚀 DevOps
- [06 · Environment Matrix (dev/staging/prod)](./07-devops/06-environment-matrix.md)
- [07 · CI/CD Pipeline](./07-devops/07-ci-cd-pipeline.md)
- [11 · Runbook · Top 5 Incidents](./07-devops/11-runbook-incidents.md)

### 🔒 Security
- [08 · Auth Spec (JWT + RBAC)](./08-security/08-auth-spec.md)
- [12 · Audit Log Spec](./08-security/12-audit-log-spec.md)
- [13 · Data Classification + Encryption Policy](./08-security/13-data-classification-encryption.md)

### 💰 Finance
- [16 · Finance Ledger Spec](./09-finance/16-finance-ledger-spec.md)
- [17 · Payment & Settlement Lifecycle](./09-finance/17-payment-settlement-lifecycle.md)
- [18 · Event Catalog](./09-finance/18-event-catalog.md)
- [22 · Multi-agent Sample Data](./09-finance/22-multi-agent-sample-data.md)
- [25 · Warranty Packages Sample Data](./09-finance/25-warranty-packages-sample-data.md)
- [28 · Ops Training: Warranty Claims](./09-finance/28-ops-training-warranty-claims.md)

### 📜 Legal & Compliance (DRAFT)
- [19 · Service Guarantee Policy](./10-legal/19-service-guarantee-policy.md)
- [20 · COD Payment Policy](./10-legal/20-cod-payment-policy.md)
- [21 · PDPL Data Policy](./10-legal/21-pdpl-data-policy.md)

> **⚠️ Legal disclaimer**: 3 docs Legal là bản thảo định hình chính sách, KHÔNG phải tư vấn pháp lý. Mọi điều khoản công bố cho KH/thợ/partner phải qua luật sư VN rà soát.

---

## Tech stack tham chiếu

| Layer | Technology |
|---|---|
| Backend language | **Go** (Gin/Echo · gRPC internal) |
| Primary DB | **MySQL 8.0** (transactional · 5 schemas: smp_order, smp_catalog, smp_agent, smp_partner, smp_geo) |
| Cache + queue | **Redis 7** (cache, distributed lock, BullMQ-style queue) |
| Analytics + logs | **MongoDB 6** (event logs, metrics, audit trail) |
| Auth | **OAuth 2.1 + JWT (RS256) + RBAC** |
| Deploy | **Kubernetes** (GitOps + ArgoCD + Helm) |
| Observability | Prometheus + Grafana + Loki + Tempo |
| Frontend | Mobile prototype (HTML/JS) → native iOS/Android (planned) |
| Admin | Vanilla HTML/JS/CSS (no framework) |

---

## Documentation conventions

### Doc numbering
Mỗi doc có 1 số duy nhất (00-28) trong tên file. Số không bao giờ tái sử dụng để dễ trace history.

### Audience tags
Mỗi doc khai báo target audience ở đầu file (All / Backend / Frontend / DevOps / Security / Finance / Legal / BA / QC).

### Status tags
- ✅ **STABLE** — đã review, sẵn sàng implement
- 📋 **DRAFT** — đang collect feedback
- ⚠️ **REVIEW PENDING** — chờ stakeholder review (đặc biệt Legal docs cần VN lawyer)

### Cross-references
Dùng relative path: `[Doc 01](./01-architecture/01-architecture.md)`. KHÔNG dùng tên file đầy đủ hoặc URL tuyệt đối.

---

## Đóng góp

Xem [Contributing Guide](./CONTRIBUTING.md) để biết:
- Markdown style rules
- Cách đặt tên doc mới
- PR review process

---

## License

Internal Use Only · © 2026 SMP Team
