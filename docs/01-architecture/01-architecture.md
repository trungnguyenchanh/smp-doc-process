# SMP Architecture · C4 Model + Service Catalog

**Version**: 3.3 · **Stack**: Go + MySQL + Redis + MongoDB · **Audience**: All engineering roles

---

## 1. Context (Level 1)

```text
                       ┌────────────────────┐
                       │   End Customer     │
                       │  (cá nhân · mobile)│
                       └─────────┬──────────┘
                                 │ HTTPS REST
                                 ▼
   ┌──────────┐         ┌────────────────────┐         ┌──────────┐
   │  inside  │ ◄──────►│        SMP         │ ◄──────►│   wms    │
   │  Cust ·  │ REST    │   Service          │  REST   │ Stock ·  │
   │  Pay ·   │ + Event │   Management       │ + Event │ Warehouse│
   │  Voucher │  Bus    │   Platform         │  Bus    │          │
   └──────────┘         └────────────────────┘         └──────────┘
                                 ▲
                    ┌────────────┼────────────┐
                    │            │            │
            ┌───────┴─────┐ ┌────┴────┐ ┌────┴─────────┐
            │ Technician  │ │ Partner │ │  Admin Ops   │
            │  (mobile)   │ │ Manager │ │  (web)       │
            └─────────────┘ └─────────┘ └──────────────┘
```

**External actors:**
- End Customer · request service
- Technician · execute job
- Partner Manager · B2B partner self-service
- Admin Ops · SMP internal ops manage system

**External systems:**
- `inside` — customer profile, payment gateway, voucher catalog (separate platform)
- `wms` — warehouse management, real-time stock (separate platform)

---

## 2. Container (Level 2)

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                            SMP Platform                                 │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │ Mobile App   │  │ Admin Portal │  │ Partner UI   │  │ Tech App    │ │
│  │ Vue/React    │  │ HTML/JS      │  │ (in Admin)   │  │ Vue/React   │ │
│  │ (Customer)   │  │              │  │              │  │             │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬──────┘ │
│         │                  │                 │                 │        │
│         └─────────┬────────┴─────────┬───────┴─────────────────┘        │
│                   │                  │                                  │
│                   ▼                  ▼                                  │
│        ┌──────────────────┐   ┌──────────────────┐                     │
│        │   API Gateway    │   │   WebSocket      │                     │
│        │   (Go · gin)     │   │   (Go · gorilla) │                     │
│        └────────┬─────────┘   └────────┬─────────┘                     │
│                 │                       │                               │
│   ┌─────────────┼───────────────────────┼─────────────────────┐         │
│   │             │                       │                     │         │
│   ▼             ▼                       ▼                     ▼         │
│ ┌────────┐ ┌────────────┐ ┌────────────────┐ ┌─────────────────┐       │
│ │ Order  │ │ Dispatch   │ │ Catalog        │ │ Partner         │       │
│ │ Svc    │ │ Engine     │ │ (Service/Step  │ │ Service         │       │
│ │ (Go)   │ │ (Go)       │ │  /Skill/BOM)   │ │ (Go)            │       │
│ │        │ │            │ │ (Go)           │ │                 │       │
│ └───┬────┘ └─────┬──────┘ └────────┬───────┘ └─────────┬───────┘       │
│     │            │                  │                   │               │
│ ┌───┴────┐ ┌─────┴──────┐ ┌────────┴───────┐ ┌─────────┴────────┐      │
│ │ Agent  │ │ Finance    │ │ Quality &      │ │ Integration      │      │
│ │ Svc    │ │ Svc        │ │ Dispute Svc    │ │ Svc (inside/wms) │      │
│ │ (Go)   │ │ (Go)       │ │ (Go)           │ │ (Go)             │      │
│ └────────┘ └────────────┘ └────────────────┘ └──────────────────┘      │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │              Shared Infrastructure                          │      │
│   │  ┌──────────┐  ┌────────────┐  ┌─────────┐  ┌────────────┐  │      │
│   │  │ MySQL    │  │ Redis      │  │ MongoDB │  │ Kafka      │  │      │
│   │  │ 8.0      │  │ 7 (cache,  │  │ 6       │  │ (event bus)│  │      │
│   │  │ (OLTP)   │  │  queue,    │  │ (logs,  │  │            │  │      │
│   │  │          │  │  session)  │  │  audit) │  │            │  │      │
│   │  └──────────┘  └────────────┘  └─────────┘  └────────────┘  │      │
│   └─────────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Service Catalog (Level 3 · Component)

| Service | Repo | Owner | Stack | Endpoint base | Datastores |
|---|---|---|---|---|---|
| `api-gateway` | `smp-api-gateway` | Platform | Go 1.22 + gin | `/api/v1/*` | Redis (rate limit) |
| `order-svc` | `smp-order-svc` | Order squad | Go + gin + sqlx | `/api/v1/orders/*` | MySQL `smp_order`, Redis (cache) |
| `dispatch-engine` | `smp-dispatch` | Order squad | Go + WebSocket | `/api/v1/dispatch/*`, ws `/ws/dispatch` | Redis (queue + state), MySQL |
| `catalog-svc` | `smp-catalog` | Platform | Go + gin + sqlx | `/api/v1/catalog/*` | MySQL `smp_catalog`, Redis (cache) |
| `agent-svc` | `smp-agent-svc` | Agent squad | Go + gin + sqlx | `/api/v1/agents/*` | MySQL `smp_agent`, Redis |
| `partner-svc` | `smp-partner-svc` | Partner squad | Go + gin + sqlx | `/api/v1/partners/*` | MySQL `smp_partner`, Redis |
| `finance-svc` | `smp-finance-svc` | Finance squad | Go + gin + sqlx | `/api/v1/finance/*` | MySQL `smp_finance`, MongoDB (txn log) |
| `quality-svc` | `smp-quality-svc` | Quality squad | Go + gin | `/api/v1/quality/*` | MySQL, MongoDB (photos meta) |
| `integration-svc` | `smp-integration` | Integration | Go + gin | `/api/v1/integrations/*` | MongoDB (event log), Redis |
| `notification-svc` | `smp-notification` | Platform | Go + gin | internal only | Redis (queue), MongoDB (log) |

**Service mesh**: Istio · mTLS internal · circuit breakers + retry

**Communication patterns:**
- Sync: REST/JSON for query, gRPC for high-throughput internal calls
- Async: Kafka events (order.created, payment.received, dispatch.assigned, etc.)
- Realtime: WebSocket for dispatch monitor + tech app live state

---

## 4. Database ownership

| Database | Type | Owner service | Schemas |
|---|---|---|---|
| `smp_order` | MySQL | order-svc | orders, order_steps, order_step_materials |
| `smp_catalog` | MySQL | catalog-svc | services, steps, skills, material_types, material_variants, step_boms |
| `smp_agent` | MySQL | agent-svc | agents, agent_skills, agent_kyc_docs, agent_warehouses |
| `smp_partner` | MySQL | partner-svc | partners, partner_admin_users, partner_wallet_transactions, partner_invoices, partner_payouts |
| `smp_finance` | MySQL | finance-svc | wallet_transactions, payouts, invoices |
| `smp_geo` | MySQL | platform | provinces, districts, wards, coverage_zones |
| `smp_events` | MongoDB | integration-svc | event_log, dlq, webhook_received |
| `smp_audit` | MongoDB | platform | audit_log (immutable, 7yr retention) |
| `smp_quality_media` | MongoDB | quality-svc | photo_meta, video_meta (URLs to S3) |

> No cross-database joins. Each service owns its DB exclusively.

---

## 5. Data flow examples

### 5.1 Order creation (Customer direct)

```text
Mobile (Customer)
   │ POST /api/v1/orders
   ▼
API Gateway ──► order-svc
                   │
                   │ INSERT INTO orders + order_steps
                   ▼
                MySQL
                   │
                   │ publish "order.created" event
                   ▼
                Kafka ──► dispatch-engine (consumer)
                              │
                              │ find candidate agents
                              ▼
                           Redis (online agents queue)
                              │
                              │ send WebSocket invitations
                              ▼
                           Technician apps (3 thợ nhận push)
```

### 5.2 Material check during quote (Technician)

```text
Tech app
   │ POST /api/v1/orders/{id}/materials/check
   ▼
order-svc ──► integration-svc
                   │
                   │ HTTP GET wms.local/api/v1/stock
                   ▼
                wms (external)
                   │
                   │ return { available: 3, cost: 50000 }
                   ▼
              integration-svc cache 30s in Redis
                   │
                   ▼
              order-svc respond UI
```

### 5.3 Partner private dispatch

```text
Partner UI (admin SMP với RBAC scoped)
   │ POST /api/v1/orders (source=partner_customer, visibility=private)
   ▼
order-svc ──► partner-svc.checkWalletBalance(partner_id)
              │
              ▼ OK (sufficient balance)
              │
              │ publish "order.created" with metadata.partner_id
              ▼
           Kafka ──► dispatch-engine
                        │
                        │ Stage 1: filter agents WHERE partner_id = X
                        ▼
                     send to partner's 8 agents only
```

---

## 6. Deployment topology

```text
                    ┌─────────────────┐
                    │  Cloudflare     │
                    │  CDN + WAF      │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
   ┌──────────┴───┐  ┌───────┴──────┐ ┌────┴────────┐
   │ Frontend     │  │ API Gateway  │ │ WebSocket   │
   │ (Pages)      │  │ Load Bal     │ │ Load Bal    │
   └──────────────┘  └──────┬───────┘ └────┬────────┘
                            │              │
                  ┌─────────┴──────────────┴─────────┐
                  │  Kubernetes Cluster              │
                  │  (3 nodes per env)               │
                  │                                  │
                  │  ┌──────────┐  ┌──────────┐     │
                  │  │ Pod:     │  │ Pod:     │ ... │
                  │  │ order-svc│  │ disp-eng │     │
                  │  └──────────┘  └──────────┘     │
                  └─────────┬────────────────────────┘
                            │
            ┌───────────────┼────────────────────────┐
            │               │                        │
    ┌───────┴─────┐  ┌──────┴──────┐  ┌─────────────┴──┐
    │ MySQL       │  │ Redis       │  │ MongoDB Atlas  │
    │ (managed)   │  │ Cluster     │  │                │
    └─────────────┘  └─────────────┘  └────────────────┘
```

**Environments:**
- `dev` · 1 node, shared DB · http://dev.smp.local
- `staging` · 2 nodes, isolated DB, pseudo data · https://staging.smp.vn
- `prod` · 3+ nodes, HA DB, real data · https://app.smp.vn

---

## 7. Cross-cutting concerns

| Concern | Solution |
|---|---|
| Auth | JWT (RS256) issued by `auth-svc` (separate service in inside) |
| Rate limit | Redis · token bucket per user/IP at API Gateway |
| Tracing | OpenTelemetry · Jaeger backend |
| Logging | Zap (Go) → stdout → Fluentbit → Elasticsearch |
| Metrics | Prometheus client_golang → Prometheus → Grafana |
| Secrets | Kubernetes secrets + Sealed Secrets in Git |
| Config | Viper · env vars + ConfigMap |
| DB migrations | golang-migrate · up/down scripts in repo |
| **Rules Engine** | **`antonmedv/expr` · YAML rules in ConfigMap · hot-reload via fsnotify** (v4.0) |
| **Event Bus** | **Apache Kafka · multi-partition · consumer groups** (v4.0) |
| **CDC** | **Debezium MySQL connector → Kafka → Elasticsearch sink** (v4.0) |

---

## 7.5 Rules Engine architecture · v4.0

> Hệ thống dispatch + pricing có 80+ business rules thay đổi theo quốc gia/policy. Thay vì hardcode vào Go code, dùng Rules Engine đọc YAML config + eval runtime.

### 7.5.1 Component diagram

```text
                       Kubernetes Cluster
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  ┌─────────────────────┐                                     │
  │  │   ConfigMap         │   kubectl apply rules_engine.yaml   │
  │  │ "smp-rules-engine"  │◄──────────── BA/Ops team            │
  │  │  rules_engine.yaml  │                                     │
  │  └──────────┬──────────┘                                     │
  │             │ mount as volume                                │
  │             ▼ /etc/smp/rules/rules_engine.yaml               │
  │  ┌─────────────────────────────────────┐                     │
  │  │  dispatch-engine Pod                │                     │
  │  │  ┌───────────────────────────────┐  │                     │
  │  │  │  pkg/rules/engine.go          │  │                     │
  │  │  │  ┌─────────────────────────┐  │  │                     │
  │  │  │  │ fsnotify file watcher   │  │  │                     │
  │  │  │  │ (reload on file change) │  │  │                     │
  │  │  │  └────────┬────────────────┘  │  │                     │
  │  │  │           ▼                   │  │                     │
  │  │  │  ┌─────────────────────────┐  │  │                     │
  │  │  │  │ YAML parser → Rule[]    │  │  │                     │
  │  │  │  └────────┬────────────────┘  │  │                     │
  │  │  │           ▼                   │  │                     │
  │  │  │  ┌─────────────────────────┐  │  │                     │
  │  │  │  │ expr.Compile() → cache  │  │  │                     │
  │  │  │  │ (pre-compiled programs) │  │  │                     │
  │  │  │  └────────┬────────────────┘  │  │                     │
  │  │  │           ▼                   │  │                     │
  │  │  │  ┌─────────────────────────┐  │  │                     │
  │  │  │  │ Evaluate(ctx) → result  │  │  │                     │
  │  │  │  └─────────────────────────┘  │  │                     │
  │  │  └───────────────────────────────┘  │                     │
  │  │                                     │                     │
  │  │  Business logic:                    │                     │
  │  │  timeout := engine.Get(             │                     │
  │  │    "dispatch", ctx)["round_timeout"]│                     │
  │  └─────────────────────────────────────┘                     │
  └──────────────────────────────────────────────────────────────┘
```

### 7.5.2 Hot reload mechanism

```text
1. BA/Ops update rules_engine.yaml trong Git repo
   ▼
2. PR review + approve → merge main
   ▼
3. GitHub Actions trigger `kubectl apply -f configmap.yaml`
   ▼
4. ConfigMap object update trong cluster
   ▼ (kubelet sync ~30-60s)
5. Mounted file ở pod tự update
   ▼ (fsnotify event)
6. engine.go reload + recompile expressions
   ▼ (atomic swap với mutex)
7. Next Evaluate() call dùng rules mới
```

**Latency**: từ merge PR đến rules hiệu lực ~1-2 phút (chủ yếu kubelet sync). Không cần restart pod.

### 7.5.3 Tại sao chọn `antonmedv/expr`?

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Drools** (Java) | Mature, rich features | Cần JVM, không native Go | ❌ |
| **OPA** (Rego) | Powerful policy language | Steep learning curve cho BA | ❌ |
| **CEL** (Google) | Standard, Kubernetes uses it | Limited Go integration ecosystem | ⚠️ |
| **Knetic/govaluate** | Pure Go, simple | Unmaintained since 2018 | ❌ |
| **antonmedv/expr** | Pure Go, type-safe, sandbox, ~1µs/eval, active maintenance | Less features than Drools (đủ dùng cho SMP) | ✅ |

Repo: https://github.com/expr-lang/expr · used by: Aviasales, Argo CD, OctoLinker.

### 7.5.4 Performance characteristics

- **Compile time**: ~50-100µs per expression. Done **once** at load/reload.
- **Evaluation time**: ~1µs per expression (cached compiled `*vm.Program`).
- **Memory**: ~80 rules × ~200 bytes/program = ~16KB per service replica. Negligible.
- **Hot reload**: file watcher debounce 500ms để tránh thrashing khi file ghi nhiều lần.

### 7.5.5 Failure modes

| Scenario | Behavior |
|---|---|
| YAML syntax error tại load | Service fails to start. CI/CD catch trước. |
| YAML syntax error tại hot-reload | Keep old rules in memory. Log error. Alert Slack. **Không crash service.** |
| Expression error tại eval (vd field nil) | Return default value cho rule đó. Log warning. Continue. |
| ConfigMap not mounted | Service start fail. Fall back to embedded `default-rules.yaml`. |
| Hot reload xung đột (race condition) | Mutex protect rule set swap. Atomic. |

### 7.5.6 Governance

- File `rules_engine.yaml` ở repo `smp-rules-config` (separate từ code repos).
- CODEOWNERS: BA team + Tech Lead approve.
- Every change PR: ai sửa, lý do, test coverage, rollback plan.
- Audit: Git log = single source of truth ai sửa gì khi nào.

---

## 7.6 Event-Driven + CQRS + CDC · v4.0

> v3.x dùng sync HTTP cho mọi inter-service communication. v4.0 chuyển sang **event-driven** cho dispatch + pricing + analytics, dùng Apache Kafka + Debezium CDC.

### 7.6.1 Architecture overview

```text
                  Write Path (Commands)              Read Path (Queries)
                  ─────────────────────              ──────────────────

    ┌───────┐    ┌─────────────┐                            ┌─────────────┐
    │ User  │───►│ api-gateway │                            │ Dashboard   │
    └───────┘    └──────┬──────┘                            │ Search UI   │
                        │                                   └──────┬──────┘
                        ▼                                          │
                  ┌──────────┐                                     │
                  │order-svc │                                     │
                  └────┬─────┘                                     │
                       │                                           │
                       ▼ INSERT                                    │
                  ┌──────────┐                                     │
                  │  MySQL   │  ◄──── primary write store          │
                  │ smp_order│                                     │
                  └────┬─────┘                                     │
                       │                                           │
                       ▼ binlog                                    │
                  ┌──────────┐                                     │
                  │ Debezium │  ◄──── CDC connector                │
                  └────┬─────┘                                     │
                       │                                           │
                       ▼                                           │
                  ┌──────────────────────────────────────┐         │
                  │       Apache Kafka                   │         │
                  │  ┌──────────────────┐                │         │
                  │  │ orders.events    │ 12 partitions  │         │
                  │  │ payments.events  │                │         │
                  │  │ agents.events    │                │         │
                  │  │ partners.events  │                │         │
                  │  │ dispatch.events  │                │         │
                  │  └──────┬───────────┘                │         │
                  └─────────┼────────────────────────────┘         │
              ┌─────────────┼─────────────┐                        │
              ▼             ▼             ▼                        │
        ┌──────────┐  ┌──────────┐  ┌──────────────┐               │
        │dispatch- │  │notify-svc│  │ ES sink      │───────────────┘
        │engine    │  │          │  │ (Kafka       │
        │          │  │          │  │ Connect)     │
        └──────────┘  └──────────┘  └──────┬───────┘
                                           │
                                           ▼
                                    ┌──────────────┐
                                    │Elasticsearch │  ◄── read model
                                    │ (denorm view)│
                                    └──────────────┘
```

### 7.6.2 Kafka cluster topology

| Setting | Value |
|---|---|
| Version | Apache Kafka 3.7+ |
| Mode | **KRaft** (no ZooKeeper, simpler ops) |
| Brokers | 3 (prod) · 1 (dev/staging) |
| Replication factor | 3 (prod) · 1 (dev/staging) |
| Min ISR | 2 (prod) |
| Partition count default | 12 per topic |
| Retention | 7 days (events), 30 days (audit), 90 days (compliance) |
| Compression | lz4 (good ratio + fast) |
| Schema Registry | Confluent Schema Registry · Avro format |
| Connect cluster | 2 nodes · for Debezium + ES sink |
| Storage per broker | 500GB SSD (prod), 100GB (staging) |

### 7.6.3 Topics catalog

| Topic | Partitions | Producer | Consumers | Retention |
|---|---:|---|---|---|
| `orders.events` | 12 | order-svc | dispatch-engine, notification-svc, finance-svc, ES sink | 7d |
| `payments.events` | 6 | integration-svc | order-svc, finance-svc, notification-svc | 30d |
| `agents.events` | 12 | agent-svc | dispatch-engine, partner-svc, ES sink | 7d |
| `partners.events` | 6 | partner-svc | finance-svc, ES sink, audit-svc | 30d |
| `dispatch.events` | 12 | dispatch-engine | order-svc, notification-svc, ES sink | 7d |
| `quality.events` | 6 | quality-svc | order-svc, agent-svc, ES sink | 30d |
| `integration.events` | 6 | integration-svc | order-svc, finance-svc | 7d |
| `audit.events` | 6 | * (all) | audit-svc | 90d |
| `*.dlq` (per topic) | 3 | retry-handler | manual review | 30d |

**Partition key strategy**:
- `orders.events`, `dispatch.events`: key = `order_id` (preserve order per entity)
- `agents.events`: key = `agent_id`
- `partners.events`: key = `partner_id`
- `payments.events`: key = `order_id` (correlate với order)

### 7.6.4 Event schema · Avro với Schema Registry

```json
// orders.events · OrderCreated v1
{
  "namespace": "vn.smp.events.orders",
  "type": "record",
  "name": "OrderCreated",
  "fields": [
    {"name": "event_id", "type": "string"},
    {"name": "event_version", "type": "string", "default": "1.0"},
    {"name": "occurred_at_utc", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "order_id", "type": "string"},
    {"name": "country_code", "type": "string"},
    {"name": "customer_id", "type": "string"},
    {"name": "partner_id", "type": ["null", "string"], "default": null},
    {"name": "source", "type": {
        "type": "enum",
        "name": "OrderSource",
        "symbols": ["customer_direct", "partner_customer", "contract"]
    }},
    {"name": "service_id", "type": "string"},
    {"name": "scheduled_at_utc", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "address", "type": {
        "type": "record",
        "name": "Address",
        "fields": [
            {"name": "line", "type": "string"},
            {"name": "district", "type": "string"},
            {"name": "city", "type": "string"},
            {"name": "lat", "type": "double"},
            {"name": "lng", "type": "double"}
        ]
    }},
    {"name": "estimated_total_amount", "type": "long"},
    {"name": "currency", "type": "string"},
    {"name": "trace_id", "type": "string"},
    {"name": "schema_version", "type": "string", "default": "1.0"}
  ]
}
```

**Schema evolution rules** (enforced by Schema Registry):
- **Forward compatible**: new field MUST have default value
- **Backward compatible**: KHÔNG được rename/remove fields (deprecate instead)
- **Breaking changes**: tạo topic v2 (`orders.events.v2`), migrate consumers, deprecate v1 sau 6 tháng

### 7.6.5 Producer pattern · Outbox

> Để đảm bảo "ghi DB + publish event" atomic, dùng Outbox table thay vì publish trực tiếp.

```text
1. Service ghi business data + outbox row trong cùng MySQL transaction
                        ▼
2. Transaction commit (atomic)
                        ▼
3. Debezium đọc MySQL binlog → publish vào Kafka
                        ▼
4. Outbox row marked processed (optional cleanup)
```

**Lý do**: nếu app crash sau ghi DB nhưng trước publish Kafka → mất event. Outbox đảm bảo Kafka event chỉ phát ra khi DB đã commit.

Schema outbox (xem doc 02 section 4 cho full DDL):
```sql
CREATE TABLE outbox_events (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  event_id CHAR(26) NOT NULL UNIQUE,           -- ULID
  aggregate_type VARCHAR(64) NOT NULL,         -- 'order', 'payment'
  aggregate_id VARCHAR(64) NOT NULL,
  event_type VARCHAR(128) NOT NULL,            -- 'OrderCreated'
  payload JSON NOT NULL,
  occurred_at_utc DATETIME(6) NOT NULL DEFAULT (UTC_TIMESTAMP(6)),
  INDEX idx_outbox_created (occurred_at_utc)
);
```

### 7.6.6 Consumer pattern · Idempotency

Kafka delivery = **at-least-once** → mỗi consumer MUST idempotent.

```go
// dispatch-engine/internal/consumer/orders.go
func (h *Handler) HandleOrderCreated(event OrderCreatedEvent) error {
    // Idempotency check: đã xử lý event này chưa?
    if h.cache.Exists("processed:" + event.EventID) {
        h.log.Info("event already processed", zap.String("event_id", event.EventID))
        return nil  // skip
    }

    // Business logic
    if err := h.dispatcher.StartRound(event.OrderID); err != nil {
        return err  // Kafka sẽ retry
    }

    // Mark processed (TTL 7 days, match topic retention)
    h.cache.Set("processed:"+event.EventID, "1", 7*24*time.Hour)
    return nil
}
```

### 7.6.7 CDC · Debezium MySQL connector

Debezium config sample (`debezium-connect-config.json`):
```json
{
  "name": "smp-mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql-primary.smp.internal",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "${file:/secrets/debezium-password}",
    "database.server.id": "184054",
    "topic.prefix": "smp.cdc",
    "database.include.list": "smp_order,smp_agent,smp_partner",
    "table.include.list": "smp_order.outbox_events,smp_order.orders,smp_agent.agents,smp_partner.partners",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "smp.cdc.schema-history",
    "include.schema.changes": "true",
    "snapshot.mode": "initial"
  }
}
```

**Topic output**: `smp.cdc.<database>.<table>`
- `smp.cdc.smp_order.outbox_events`
- `smp.cdc.smp_order.orders` (raw CDC, primarily for ES sink)

**Outbox event consumer** maps `smp.cdc.smp_order.outbox_events` → publish to proper business topic (`orders.events`, `payments.events`, etc.) based on `aggregate_type` field.

### 7.6.8 Saga pattern · Long-running transactions

Cho transactions span nhiều services (order creation + payment + dispatch), dùng Saga choreography (event-driven, no central orchestrator):

```text
[order-svc] OrderCreated
            ▼
   [finance-svc] reserves payment
            ▼ success
   PaymentReserved
            ▼
   [dispatch-engine] starts dispatch
            ▼ success
   AgentAssigned
            ▼
   [finance-svc] confirms payment
            ▼
   PaymentConfirmed
            ▼
   [order-svc] marks order ready
```

**Compensating actions** nếu fail:
- PaymentFailed → OrderCancelled (refund nếu đã charge)
- DispatchTimedOut → PaymentReleased + OrderCancelled
- AgentRejectedTooMany → OrderEscalated (manual ops)

Saga state lưu ở mỗi service (no central state DB). Mỗi service biết action của mình + emit event để service kế tiếp pick up.

### 7.6.9 CQRS · Read model in Elasticsearch

**Why Elasticsearch?**
- Dashboard queries: complex aggregations (group by district, time range, status) → ES `aggs` API faster than MySQL JOIN
- Search: full-text search trên order notes, customer names → ES native
- Decouple read traffic: dashboard không impact transactional MySQL

**Sink architecture**:
```text
Kafka topics
    ▼ (Kafka Connect: Elasticsearch Sink Connector)
    ▼
Elasticsearch indices:
  - orders-2026-05            (daily index, 30 days retention hot)
  - orders-2026-04            (warm tier)
  - agents
  - partners
  - dispatch-rounds
```

**Eventual consistency**: 1-5 seconds typical lag from MySQL write → ES queryable.

**Failure mode**: nếu ES sink down + lag tăng, dashboard hiển thị banner "Data lag X minutes". Critical reports fallback query MySQL replica (slower nhưng accurate).

### 7.6.10 Dead Letter Queue · DLQ

```text
Topic: orders.events
  ▼ consumer (dispatch-engine)
  ▼ retry 3x with exponential backoff (1s, 5s, 30s)
  ▼ still failing
  ▼
Topic: orders.events.dlq
  ▼ manual review qua admin UI
  ▼
Resolution:
  - Fix bug → reprocess from DLQ
  - Bad data → mark resolved
  - Critical → page on-call
```

Alert: `kafka_dlq_message_count > 10` → notify `#engineering-oncall`.

---

## 8. Decision records

ADRs (Architecture Decision Records) lưu tại `docs/adr/`. Mỗi quyết định có file riêng:
- ADR-001: Chọn Go thay Node.js · lý do performance + concurrency
- ADR-002: MySQL cho OLTP, MongoDB cho event log · lý do tách workload
- ADR-003: Kafka thay vì RabbitMQ · lý do throughput + replay
- ADR-004: Microservices monorepo vs polyrepo · chọn polyrepo per service
- ADR-005: Partner login in admin SMP vs sub-domain · chọn in admin với RBAC scoped
