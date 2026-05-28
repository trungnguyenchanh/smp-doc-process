# MIGRATION PLAN · v3.4 → v4.0

**Owner**: Tech Lead + BA Lead · **Timeline**: Q3 2026 → Q4 2026 (~6 months) · **Status**: Planning

---

## Tóm tắt

v3.4 (current) = **Vietnam-only** SaaS với MySQL transactional + sync HTTP. Hardcode VND, hardcode `Asia/Ho_Chi_Minh`, hardcode 80+ business rules trong Go.

v4.0 = **Global-ready** với multi-currency · UTC · i18n · Kafka event-driven · CQRS · Rules Engine YAML · Dynamic Data Masking · sovereignty deployment.

Migration chia thành **5 phases tuần tự**, mỗi phase tự deploy độc lập, có rollback path riêng.

---

## Roadmap visualization

```text
v3.4 ──► v3.5 ──► v3.6 ──► v3.7 ──► v3.8 ──► v4.0
 NOW    DB +Go   Rules    Kafka    Masking   Multi-
        global   engine   + CQRS   + DSR     region
                 YAML     + CDC               GA
```

| Phase | Version | Duration | Goal | Risk |
|---|---|---|---|---|
| **1** | v3.5 | 6 weeks | DB schema refactor + Go pkg/money + pkg/clock | 🟡 Medium |
| **2** | v3.6 | 4 weeks | Rules Engine YAML rollout | 🟢 Low |
| **3** | v3.7 | 8 weeks | Kafka + CQRS + Debezium CDC | 🔴 High |
| **4** | v3.8 | 6 weeks | Dynamic Data Masking + PII APIs | 🟡 Medium |
| **5** | v4.0 GA | 4 weeks | Multi-region sovereignty deployment | 🔴 High |

**Total**: ~28 weeks (~6 months active engineering) · không tính buffer cho testing/UAT/marketing prep.

---

## Phase 1 · v3.5 · Database & Go foundations (6 weeks)

### Goal
- DB schema ready cho multi-currency + UTC + i18n + country_code
- Go `pkg/money`, `pkg/clock`, `pkg/timezone`, `pkg/i18n` shipped
- ALL data still in Vietnam, default currency=VND, default tz=Asia/Ho_Chi_Minh
- **No user-facing changes**

### Tasks

| Week | Task | Owner | Doc reference |
|---|---|---|---|
| 1 | Create `smp_global` database với 5 tables (countries, currencies, currency_rates, tax_configs, i18n_translations) | DB | [Doc 02 · 7.5](./02-database/02-database-schema.md#75--v40-global-tables-db-smp_global--new) |
| 1 | Seed 8 countries + 10 currencies + tax_configs for VN | DB | Same |
| 2 | ALTER tables: add `country_code` + `currency` + `*_utc` columns to existing tables (orders, partner_wallet_transactions, partner_invoices, partner_payouts) | DB | [Doc 02 · 2.5](./02-database/02-database-schema.md#25--v40-conventions--global-ready-schema-patterns) |
| 2 | Backfill existing data: country_code='VN', currency='VND', convert timestamps to UTC | DB | Same |
| 3 | Build `pkg/money` Go package (Money struct, arithmetic, marshal) | Backend | [Doc 04 · 1.15](./03-backend/04-coding-standards-and-dev-setup.md#115-v40-patterns--money--time--i18n) |
| 3 | Build `pkg/clock` (Clock interface, UTC, testable mock) | Backend | Same |
| 4 | Build `pkg/timezone` (IANA tz conversion) | Backend | Same |
| 4 | Build `pkg/i18n` (translator với fallback chain) | Backend | Same |
| 5 | Refactor finance-svc to use `pkg/money` (drop INT amount) | Backend | Same |
| 5 | Refactor order-svc to use `*_utc` columns + `pkg/clock.NowUTC()` | Backend | Same |
| 6 | Add `forbidigo` linter rules to CI to prevent regression | DevOps | Same |
| 6 | UAT + Performance test + Deploy v3.5 production | All | n/a |

### Acceptance criteria
- [ ] All money fields in DB có pair `amount + currency` columns
- [ ] All timestamps in DB lưu UTC (verified via spot-check 100 records cross-tz)
- [ ] Linter blocks `time.Now()` direct + hardcoded currency strings
- [ ] Zero data loss in migration (recon job: count rows before/after match)
- [ ] All existing tests pass + new tests cover Money/Clock packages
- [ ] No user-facing change visible

### Rollback plan
- DDL migrations have `down` scripts (golang-migrate)
- Old columns kept for 30 days post-deploy as safety (`labor_price INT` + new `labor_price_amount BIGINT` both exist, code uses new, old shadow-updated)
- Rollback = revert deploy + drop new columns

---

## Phase 2 · v3.6 · Rules Engine (4 weeks)

### Goal
- All dispatch + pricing rules moved from Go code → `rules_engine.yaml`
- ConfigMap deployment với hot-reload
- BA team can edit rules independently
- Code rules path **deprecated** but kept as fallback

### Tasks

| Week | Task | Owner | Doc reference |
|---|---|---|---|
| 1 | Build `pkg/rules` Go package (Engine, types, loader, cache) | Backend | [Doc 04 · 1.16](./03-backend/04-coding-standards-and-dev-setup.md) |
| 1 | Create repo `smp-rules-config` with CI/CD pipeline | DevOps | [Doc 06 · 6.5](./07-devops/06-environment-matrix.md#65-rules-engine-deployment-v40) |
| 2 | Convert Dispatch rules (BR-DISP-*) to YAML format | BA + Backend | [Doc 15 · Appendix](./05-ba/15-business-rules.md#appendix--full-rules_engineyaml-sample-v40-preview) |
| 2 | Convert Pricing rules (BR-PRICE-*) to YAML | BA + Backend | Same |
| 3 | Convert Payment + KYC + Stage rules | BA + Backend | Same |
| 3 | Convert Material + Partner + Integration + Quality + Notification + Retention rules | BA + Backend | Same |
| 4 | Wire engine into dispatch-engine + finance-svc + catalog-svc | Backend | Same |
| 4 | Deploy ConfigMap + verify hot-reload in staging | DevOps | Same |
| 4 | Documentation review + BA training + Deploy v3.6 production | All | n/a |

### Acceptance criteria
- [ ] All 80+ business rules in `rules_engine.yaml` with `legacy_id` mapping (BR-* → vn.category.NNN)
- [ ] Hot reload works: change YAML → ConfigMap apply → pod reloads within 60s
- [ ] BA can edit YAML via PR (CODEOWNERS enforce review)
- [ ] Eval performance: p99 < 5µs per evaluation
- [ ] Failover: if reload fails, services keep old rules + log error (no crash)

### Rollback plan
- Code rules path kept (not deleted) as fallback
- Feature flag `USE_RULES_ENGINE=true` controls whether engine vs hardcode used
- Rollback = flip flag to false, redeploy

---

## Phase 3 · v3.7 · Event-driven + CQRS + CDC (8 weeks · HIGHEST RISK)

### Goal
- Kafka cluster deployed (KRaft mode, 3 brokers)
- Debezium MySQL → Kafka CDC operational
- Elasticsearch read model populated from Kafka
- All dashboards migrated to ES queries
- Selected services use event-driven (order created → dispatch picked up via Kafka)

### Tasks

| Week | Task | Owner | Doc reference |
|---|---|---|---|
| 1-2 | Deploy Kafka cluster (KRaft 3 brokers + Schema Registry) | DevOps | [Doc 01 · 7.6.2](./01-architecture/01-architecture.md) |
| 2 | Create topics (orders, payments, agents, partners, dispatch, quality, integration, audit + DLQ) | DevOps | [Doc 01 · 7.6.3](./01-architecture/01-architecture.md) |
| 3 | Build outbox table in smp_order DB + outbox publisher pattern | Backend | [Doc 01 · 7.6.5](./01-architecture/01-architecture.md) |
| 3-4 | Deploy Debezium connector, configure to read smp_order binlog → orders.events topic | DevOps | [Doc 01 · 7.6.7](./01-architecture/01-architecture.md) |
| 4 | Build consumer wrappers with idempotency (cache "processed:event_id") | Backend | [Doc 01 · 7.6.6](./01-architecture/01-architecture.md) |
| 5 | Migrate dispatch-engine to consume orders.events từ Kafka (instead of HTTP poll) | Backend | Same |
| 5 | Deploy Elasticsearch cluster (3 nodes, ILM hot/warm/cold/delete) | DevOps | [Doc 14 · 10.6.5](./05-ba/14-kpi-metrics-definition.md#1065-elasticsearch-index-design) |
| 6 | Deploy ES sink connector (Kafka Connect) | DevOps | [Doc 01 · 7.6.9](./01-architecture/01-architecture.md) |
| 6 | Build query routing layer: dashboard queries → ES first, MySQL fallback | Backend | [Doc 14 · 10.6.2](./05-ba/14-kpi-metrics-definition.md) |
| 7 | Migrate dashboard queries one by one (GMV, completion rate, agent utilization, partner GMV) | Backend + BA | Same |
| 7 | Build DLQ admin UI for poison message review | Backend | [Doc 01 · 7.6.10](./01-architecture/01-architecture.md) |
| 8 | Soak test + chaos engineering (kill broker, kill connector) + Deploy v3.7 prod | All | [Doc 11 · INC-006-009](./07-devops/11-runbook-incidents.md) |

### Acceptance criteria
- [ ] Order created → dispatch-engine pick up Kafka event < 100ms (vs HTTP polling 30s before)
- [ ] CDC: MySQL write → Kafka event < 1s lag
- [ ] ES read model: MySQL write → ES queryable < 5s
- [ ] Dashboard queries 10x faster (aggregate via ES)
- [ ] Failover: Kafka down → services fallback to direct HTTP (degraded but functional)
- [ ] Idempotency proven: replay event 3x = same final state

### Risks & mitigation
- **Risk**: Kafka complexity overwhelms team
  - Mitigation: 2-week training before phase, hire/contract Kafka expert
- **Risk**: ES sink drift causes wrong dashboard numbers
  - Mitigation: Daily reconciliation job MySQL vs ES, alert if drift > 0.1%
- **Risk**: Increased operational burden (more components to monitor)
  - Mitigation: Comprehensive runbook (Doc 11 already has INC-006 → INC-009)

### Rollback plan
- Feature flag per service `USE_KAFKA_EVENTS=true` (default false initially)
- Gradual rollout: dispatch-engine first, then notification, then others
- If issues: flip flag off, services revert to HTTP sync calls
- ES dashboards have MySQL fallback always wired

---

## Phase 4 · v3.8 · Dynamic Data Masking + DSR (6 weeks)

### Goal
- All API responses containing PII automatically masked at API Gateway
- Authorized roles can unmask with scope or `/pii/unmask` endpoint
- Audit trail captures every unmask
- Data Subject Rights workflows (export, delete) automated

### Tasks

| Week | Task | Owner | Doc reference |
|---|---|---|---|
| 1 | Define masking patterns + add struct tags to DTO types | Backend | [Doc 13 · 5.5.1](./08-security/13-data-classification-encryption.md) |
| 1-2 | Build `pkg/masking` middleware (tag-based, reflection-cached) | Backend | [Doc 13 · 5.5.2](./08-security/13-data-classification-encryption.md) |
| 2 | Add `pii.unmask.*` scopes to JWT claims + Auth service | Backend | [Doc 08 · 4.5](./08-security/08-auth-spec.md) |
| 3 | Build `/pii/unmask` endpoint với MFA-fresh requirement | Backend | [Doc 03 · 11.5](./03-backend/03-api-contract.md#115-pii-endpoints-v40) |
| 3 | Build `/pii/audit-trail` endpoint cho DSR | Backend | Same |
| 4 | Update Admin Web UI: show masked values + "Reveal" button + audit display | Frontend | [Doc 03 · 11.5](./03-backend/03-api-contract.md) |
| 4 | Update Mobile apps: masked customer info in support flows | Frontend | Same |
| 5 | Build DSR export workflow (queue request → batch generate ZIP → email link) | Backend | [Doc 12 · DSR events](./08-security/12-audit-log-spec.md) |
| 5 | Build DSR deletion workflow (mark deletion_pending → 30d grace → anonymize) | Backend | [Doc 13 · 6.4](./08-security/13-data-classification-encryption.md) |
| 6 | UAT with security + compliance team + deploy v3.8 | All | n/a |

### Acceptance criteria
- [ ] Default-deny: no PII visible without explicit scope
- [ ] Every unmask creates audit log entry with actor + reason + fields
- [ ] DSR export delivered to user within 7 days (PDPA SLA)
- [ ] DSR deletion completes within 30 days
- [ ] Performance: masking adds < 50µs per response
- [ ] Compliance review sign-off

### Rollback plan
- Feature flag `ENABLE_MASKING=true` per environment
- If issues: turn off masking, system returns raw data (PII risk but functional)
- DSR workflows can be done manually during rollback period

---

## Phase 5 · v4.0 GA · Multi-region sovereignty (4 weeks)

### Goal
- Spin up additional clusters for non-VN markets
- Routing logic at edge (Cloudflare Workers)
- Per-region data isolation enforced
- Compliance signed off per jurisdiction

### Tasks

| Week | Task | Owner | Doc reference |
|---|---|---|---|
| 1 | Provision smp-asia cluster in Singapore (AWS ap-southeast-1) — primary for VN expansion | DevOps | [Doc 06 · 14](./07-devops/06-environment-matrix.md#14-multi-region-deployment-v40-sovereignty) |
| 1 | Migrate VN production data từ current single-region cluster sang smp-asia | DevOps | Same |
| 2 | Cross-region replication setup: master data (countries, currencies, rates, tax, i18n) sync via MirrorMaker | DevOps | [Doc 06 · 14.3](./07-devops/06-environment-matrix.md) |
| 2 | Build Cloudflare Workers routing logic (country code → cluster) | DevOps | [Doc 06 · 14.5](./07-devops/06-environment-matrix.md) |
| 3 | (When launching country X) Provision X cluster + onboarding | DevOps + Country GM | Same |
| 3 | Per-country compliance review (PDPA, GDPR, CPRA, PIPL as applicable) | Legal + DPO | [Doc 13 · 8.5](./08-security/13-data-classification-encryption.md#85-multi-jurisdiction-compliance-v40) |
| 4 | DR drill: simulate cluster failure per region | DevOps + Security | [Doc 11 · INC-012](./07-devops/11-runbook-incidents.md) |
| 4 | v4.0 launch announcement + monitoring + on-call rotation expand | All | n/a |

### Acceptance criteria
- [ ] VN users routed to smp-asia, latency p95 < 200ms
- [ ] CN/US/EU users (when launched) routed to local cluster
- [ ] Cross-region master data sync < 5min
- [ ] PII data never crosses region (verified via traffic audit + sample log review)
- [ ] Compliance certifications obtained per region (DPO appointed)
- [ ] Cost per region ≤ projected budget ($4000/$1500/$2500/month)

### Risks & mitigation
- **Risk**: Cluster migration data loss
  - Mitigation: Multiple full backups + dry-run migration in staging x3
- **Risk**: Cross-region cost overrun
  - Mitigation: Strict cost monitoring per tag + alert > 110%
- **Risk**: CN cluster requires Alibaba Cloud + ICP filing (different ops model)
  - Mitigation: Hire local CN DevOps + start ICP process 3 months early

### Rollback plan
- Per-region rollback (cluster can be decommissioned independently)
- Traffic routing can be reverted via Cloudflare Workers config (1-click)
- Data per region is isolated, no cascade failure

---

## Cross-cutting concerns (apply to ALL phases)

### Communication plan
- Engineering all-hands at start of each phase (1h)
- Demo at end of each phase to PM/Founder
- Customer communication: ZERO breaking changes promised (backward compat enforced)
- Partner communication: 30-day advance notice for any API change

### Testing strategy
- Each phase: unit tests + integration tests + UAT in staging
- Performance test before prod: baseline vs new
- Chaos engineering tests for Phase 3 (Kafka)
- Security penetration test before Phase 4 deploy
- Full regression suite (Doc 10) run before each phase deploy

### Monitoring & observability
- Per-phase: add new dashboards for new components
- Phase 3: Kafka consumer lag, broker health, DLQ size
- Phase 4: masking middleware latency, unmask requests/hour
- Phase 5: per-region health, cross-region replication lag

### Documentation discipline
- Each phase MUST update relevant docs in this repo
- PR template requires "docs updated?" checkbox
- BA reviews all rule changes (Phase 2)
- Security reviews all masking changes (Phase 4)

---

## Decision points (need approval)

| Decision | When | Owner | Recommended |
|---|---|---|---|
| Approve full migration plan | NOW | Founder + Tech Lead | ✅ Approve |
| Hire Kafka expert (Phase 3) | Phase 3 start | Engineering Manager | Yes if no in-house expertise |
| Cloud vendor for smp-china | Phase 5 prep | Founder + Legal | Alibaba Cloud (Tencent secondary) |
| GDPR adequacy decision review | Phase 5 if EU expansion | Legal + DPO | Defer to actual EU launch decision |
| Tokenization for credit cards | Future | CTO + Security | Only when SMP takes cards directly |

---

## Success metrics post-migration

| Metric | Baseline (v3.4) | Target (v4.0) |
|---|---|---|
| API p99 latency | 800ms | 500ms (with ES read model) |
| Dispatch latency p99 | 30-60s (polling) | < 2s (Kafka) |
| Dashboard query time | 3-5s | 200-500ms |
| Rules change deploy time | 30min (code deploy) | 60s (ConfigMap apply) |
| Countries supported | 1 (VN) | 1+ ready (CN/US/etc on demand) |
| Currencies supported | 1 (VND) | 10 (ISO 4217 catalog) |
| PII compliance posture | Basic PDPA | PDPA + GDPR + CPRA + PIPL ready |
| Reconciliation coverage | Manual | Automated daily |

---

## References

- [Doc 00 · System Functional Overview](./00-system-functional-overview.md) — executive summary
- [Doc 01 · Architecture](./01-architecture/01-architecture.md) — components Rules Engine + Event Bus
- [Doc 02 · Database Schema](./02-database/02-database-schema.md) — v4.0 conventions + new tables + sharding
- [Doc 03 · API Contract](./03-backend/03-api-contract.md) — PII endpoints + masking responses
- [Doc 04 · Coding Standards](./03-backend/04-coding-standards-and-dev-setup.md) — pkg/money + pkg/clock + pkg/rules
- [Doc 05 · Glossary](./05-ba/05-glossary.md) — i18n + Rules + CQRS + Security terms
- [Doc 06 · Environment Matrix](./07-devops/06-environment-matrix.md) — Rules Engine deploy + Multi-region
- [Doc 08 · Auth Spec](./08-security/08-auth-spec.md) — PII unmask scopes
- [Doc 11 · Runbook](./07-devops/11-runbook-incidents.md) — INC-006 to INC-012
- [Doc 12 · Audit Log](./08-security/12-audit-log-spec.md) — PII + DSR + Compliance events
- [Doc 13 · Data Classification](./08-security/13-data-classification-encryption.md) — Masking + Multi-jurisdiction
- [Doc 14 · KPI Metrics](./05-ba/14-kpi-metrics-definition.md) — CQRS Read Model
- [Doc 15 · Business Rules](./05-ba/15-business-rules.md) — YAML migration + Reconciliation rules

---

## Changelog

| Date | Author | Change |
|---|---|---|
| 2026-05-28 | Docs team | Initial migration plan v3.4 → v4.0 |
