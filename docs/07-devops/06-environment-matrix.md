# SMP Environment Matrix · dev / staging / prod

**Audience**: DevOps, Backend, Security

---

## 1. Environment overview

| Env | Purpose | Data | Access | URL |
|---|---|---|---|---|
| `local` | Developer's machine | Seed only | localhost | http://localhost:8080 |
| `dev` | Shared dev integration | Pseudo data | All eng team | https://dev-api.smp.vn |
| `staging` | Pre-prod testing | Pseudo + sample | QC, BA, PM | https://staging-api.smp.vn |
| `prod` | Production | Real data | Customer-facing | https://api.smp.vn |

## 2. Infrastructure per env

| Component | local | dev | staging | prod |
|---|---|---|---|---|
| K8s nodes | docker-compose | 1 | 2 | 3+ |
| MySQL | Docker | 1 instance | Managed (replica) | Managed (HA + 2 replicas) |
| Redis | Docker | 1 node | Cluster 3 nodes | Cluster 6 nodes |
| MongoDB | Docker | 1 instance | Atlas M10 | Atlas M30 |
| Kafka | Docker | 1 broker | 3 brokers | 5 brokers |
| Object storage | Local FS | Cloudflare R2 dev bucket | R2 staging | R2 prod |
| CDN | None | Cloudflare | Cloudflare | Cloudflare |

## 3. URLs matrix

### Frontend
| App | dev | staging | prod |
|---|---|---|---|
| Mobile (web) | smp-mobile-dev.pages.dev | smp-mobile-staging.pages.dev | smp-mobile.pages.dev |
| Admin | smp-admin-dev.pages.dev | smp-admin-staging.pages.dev | smp-admin.pages.dev |

### Backend
| Service | dev | staging | prod |
|---|---|---|---|
| API Gateway | dev-api.smp.vn | staging-api.smp.vn | api.smp.vn |
| WebSocket | dev-ws.smp.vn | staging-ws.smp.vn | ws.smp.vn |
| Internal services | `<svc>.dev.smp.local` (k8s internal) | `<svc>.staging.smp.local` | `<svc>.prod.smp.local` |

### External integrations
| System | dev/staging | prod |
|---|---|---|
| inside | inside-staging.local | inside.local |
| wms | wms-staging.local | wms.local |

## 4. Resource sizing

### Per service replicas
| Service | dev | staging | prod | Auto-scale |
|---|---|---|---|---|
| api-gateway | 1 | 2 | 3-10 | CPU > 70% |
| order-svc | 1 | 2 | 3-8 | CPU > 70% |
| dispatch-engine | 1 | 2 | 2-5 | Custom (queue depth) |
| catalog-svc | 1 | 2 | 2-4 | CPU > 70% |
| agent-svc | 1 | 1 | 2-4 | CPU > 70% |
| partner-svc | 1 | 1 | 2-3 | CPU > 70% |
| finance-svc | 1 | 1 | 2 | None |
| quality-svc | 1 | 1 | 2 | CPU > 70% |
| integration-svc | 1 | 2 | 3-6 | Queue lag |
| notification-svc | 1 | 1 | 2-4 | Queue depth |

### Pod resources
| Service | CPU req | CPU lim | RAM req | RAM lim |
|---|---|---|---|---|
| api-gateway | 250m | 500m | 256Mi | 512Mi |
| order-svc | 500m | 1000m | 512Mi | 1Gi |
| dispatch-engine | 1000m | 2000m | 1Gi | 2Gi |
| Others | 250m | 500m | 256Mi | 512Mi |

## 5. Secrets management

### 5.1 Secret types

| Secret type | Storage | Rotation |
|---|---|---|
| JWT signing key (RS256) | HashiCorp Vault | Annual + on compromise |
| DB passwords | Vault | Quarterly |
| Redis password | Vault | Quarterly |
| MongoDB connection string | Vault | Quarterly |
| Kafka SASL credentials | Vault | Quarterly |
| inside/wms API tokens | Vault | Per partner agreement |
| Webhook signing secrets | Vault | Annual |
| SSL/TLS certs | Cloudflare Origin CA | Auto-renew |
| Cloudflare API tokens | Vault | Annual |
| OAuth client secrets (inside) | Vault | Annual |

### 5.2 Vault setup

```
Vault path structure:
  secret/smp/<env>/<service>/<key>

Examples:
  secret/smp/prod/order-svc/mysql_password
  secret/smp/prod/order-svc/redis_password
  secret/smp/prod/integration-svc/inside_api_token
  secret/smp/prod/api-gateway/jwt_signing_key
```

K8s pods access Vault via Vault Agent Injector (sidecar):
```yaml
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "order-svc"
    vault.hashicorp.com/agent-inject-secret-mysql: "secret/smp/prod/order-svc"
```

### 5.3 Local dev secrets

Developer dùng `.env` file (gitignored). NOT in Vault.

`.env.example` (committed) chỉ có placeholder:
```
MYSQL_PASSWORD=<dev_password>
INSIDE_API_TOKEN=<get_from_team_lead>
```

Onboarding new dev: team lead share `.env.dev` qua password manager (1Password vault "SMP Engineering").

### 5.4 NEVER commit secrets

Pre-commit hook check `gitleaks`:
```bash
brew install gitleaks
gitleaks install
```

CI cũng chạy gitleaks ở PR check.

## 6. Configuration matrix

### 6.1 Per-env config

Most config in env vars. Default trong `internal/config/config.go`, override per env.

```go
type Config struct {
    Server   ServerConfig
    MySQL    MySQLConfig
    Redis    RedisConfig
    Kafka    KafkaConfig
    Inside   IntegrationConfig
    WMS      IntegrationConfig
    Logging  LoggingConfig
    Tracing  TracingConfig
}
```

### 6.2 Feature flags

Per-env feature flags trong ConfigMap (k8s) hoặc dedicated service (Phase 2 dùng LaunchDarkly/Unleash).

Phase 1 dùng ConfigMap:
```yaml
# k8s/prod/configmap-flags.yaml
data:
  ENABLE_PARTNER_PRIVATE_DISPATCH: "true"
  ENABLE_AUTO_VOUCHER_SUGGEST: "true"
  ENABLE_MULTI_AGENT_FLOW: "true"
  MATERIAL_VERIFY_AUTO_APPROVE_THRESHOLD_PCT: "5"
```

### 6.3 Per-env values matrix

| Setting | dev | staging | prod |
|---|---|---|---|
| LOG_LEVEL | `debug` | `info` | `info` |
| DISPATCH_ROUND_TIMEOUT_SEC | 30 | 60 | 60 |
| DISPATCH_MAX_ROUNDS | 3 | 3 | 5 |
| ORDER_TIMEOUT_PAYMENT_MIN | 5 | 15 | 30 |
| WMS_RESERVATION_TTL_HOURS | 1 | 24 | 24 |
| CACHE_TTL_CUSTOMER_SEC | 10 | 30 | 30 |
| CACHE_TTL_WMS_STOCK_SEC | 10 | 30 | 30 |
| RATE_LIMIT_REQ_PER_MIN | 1000 | 200 | 60 |
| ENABLE_MOCK_PAYMENT | true | false | false |

## 6.5 Rules Engine deployment (v4.0)

> Quy trình deploy `rules_engine.yaml` qua Kubernetes ConfigMap, mount vào pods, hot-reload.

### Repo separation

```
github.com/trungnguyenchanh/smp-rules-config/   ← repo riêng
├── environments/
│   ├── dev/
│   │   └── rules_engine.yaml
│   ├── staging/
│   │   └── rules_engine.yaml
│   └── prod/
│       └── rules_engine.yaml
├── tests/
│   └── rules_test.yaml
├── .github/workflows/
│   ├── validate.yml     # YAML syntax + expression compile check
│   └── deploy.yml       # kubectl apply per env
└── README.md
```

### ConfigMap manifest

```yaml
# environments/prod/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: smp-rules-engine
  namespace: smp-prod
  labels:
    app: smp
    component: rules-engine
    version: "4.0.0"
data:
  rules_engine.yaml: |
    version: "4.0.0"
    # ... full content ...
```

### Pod volume mount

```yaml
# helm chart dispatch-engine/templates/deployment.yaml
spec:
  containers:
    - name: dispatch-engine
      volumeMounts:
        - name: rules-config
          mountPath: /etc/smp/rules
          readOnly: true
  volumes:
    - name: rules-config
      configMap:
        name: smp-rules-engine
        defaultMode: 0444   # read-only
```

### Per-env override matrix

| Setting | dev | staging | prod |
|---|---|---|---|
| File path in pod | `/etc/smp/rules/rules_engine.yaml` | same | same |
| ConfigMap source | local dev: file mount | smp-rules-config/staging | smp-rules-config/prod |
| Auto-reload latency | <5s (file watch on host) | ~60s (k8s ConfigMap sync) | ~60s |
| Approval to update | Anyone | Tech Lead | BA Lead + Tech Lead |
| Test required | None | Run `make rules-test` | Required + sign-off |
| Rollback | Edit file | `kubectl rollout undo` | Git revert + ArgoCD sync |

### Deploy flow

```
1. BA edit rules_engine.yaml trong smp-rules-config repo
   ▼
2. Create PR · CI runs:
   - YAML syntax validate (yamllint)
   - All expressions compile check (go run cmd/rule-lint)
   - Run rules_test.yaml fixtures (assert decisions)
   ▼
3. Reviewer approve (BA Lead + Tech Lead cho prod)
   ▼
4. Merge main
   ▼
5. GitHub Actions deploy.yml:
   - kubectl create configmap smp-rules-engine \
     --from-file=rules_engine.yaml=environments/${ENV}/rules_engine.yaml \
     --dry-run=client -o yaml | kubectl apply -f -
   ▼
6. ArgoCD detects ConfigMap change → reconcile
   ▼
7. Kubelet syncs ConfigMap to pods (~60s)
   ▼
8. Pods detect file change via fsnotify → reload rules
   ▼
9. Verify via metrics dashboard:
   - rules_loaded_total (gauge)
   - rules_last_reload_timestamp (gauge)
   - rules_reload_errors_total (counter)
```

### Rollback procedure

**Quick rollback** (last good version still in Git):
```bash
cd smp-rules-config
git revert HEAD            # or git checkout <good-sha> -- environments/prod/
git push origin main
# CD auto-deploys revert
```

**Emergency rollback** (skip Git, direct kubectl):
```bash
# Get previous ConfigMap revision
kubectl rollout history configmap/smp-rules-engine -n smp-prod
# Restore
kubectl rollout undo configmap/smp-rules-engine -n smp-prod
```

**Disaster fallback** (rules engine itself broken):
- Services có embedded `default-rules.yaml` shipped trong Docker image
- Nếu ConfigMap mount fail → fallback to default
- Default rules = conservative VN defaults

### Monitoring

Add Prometheus alerts:
```yaml
- alert: RulesEngineReloadFailed
  expr: rate(rules_reload_errors_total[5m]) > 0
  for: 1m
  annotations:
    summary: "Rules engine reload failing on {{ $labels.pod }}"
    runbook: "https://docs.smp.vn/runbooks/rules-engine-reload-failed"

- alert: RulesEngineStale
  expr: time() - rules_last_reload_timestamp > 86400
  for: 5m
  annotations:
    summary: "Rules engine on {{ $labels.pod }} hasn't reloaded in 24h"
```

## 7. Data seeding

### dev
Master data v3.3 từ JSON. Customer data fake (Vietnamese names library). 100 đơn random.

### staging
Master data v3.3. Customer + agent + partner: synthetic 1000 records. Order history: 10000 đơn last 90 days.

### prod
Master data only. Real customer/order data from launch.

### Refresh procedure
- dev: re-seedable any time `make seed`
- staging: monthly refresh from synthetic generator
- prod: NO seeding, real data only

## 8. Deployment pipeline summary

```
Developer push to feature branch
       ↓
GitHub Actions CI runs
       ↓
Open PR, peer review
       ↓
Merge to main
       ↓
Auto deploy to dev (within 5 min)
       ↓
Manual promote to staging (PM approval)
       ↓
QC regression test in staging
       ↓
Manual promote to prod (release manager approval)
       ↓
Deploy via blue/green to prod
       ↓
Smoke test + monitor 30 min
       ↓
Stable / rollback
```

Detail: xem `CI/CD pipeline doc`.

## 9. Access control

### dev
- All engineers: read + write (DB query, log access)
- No PII (chỉ pseudo data)

### staging
- Engineers: read-only DB
- QC: read-only DB
- Ops manager: read + manual UAT actions

### prod
- Engineers: NO direct DB access (except SRE on-call via bastion)
- Customer support: read-only via admin portal
- Ops manager: admin portal access only
- Finance: read-only finance domain
- All access logged in audit log

### Bastion access
```bash
# Engineer cần debug prod issue
ssh bastion.smp.vn       # MFA required
sudo -u smp-readonly mysql -h prod-mysql.local
# Session recorded, auto-terminate after 30 min idle
```

## 10. Environment promotion

| From | To | Trigger | Approval |
|---|---|---|---|
| main branch | dev | Auto on merge | None |
| dev | staging | Manual via UI | PM + tech lead |
| staging | prod | Manual via UI | Release manager + on-call SRE |

Rollback: 1-click revert to previous deployment image in K8s.

## 11. DNS & domains

| Domain | Owner | Purpose |
|---|---|---|
| smp.vn | Company | Marketing site |
| api.smp.vn | DevOps | Prod API |
| app.smp.vn | DevOps | Prod app shortcut |
| dev-api.smp.vn | DevOps | Dev API |
| staging-api.smp.vn | DevOps | Staging API |
| admin.smp.vn | DevOps | Admin portal |
| partner.smp.vn | DevOps | Partner portal landing (v3.3 redirect to admin) |
| docs.smp.vn | DevOps | Public API docs |
| status.smp.vn | DevOps | Status page (statuspage.io) |

All TLS via Cloudflare Origin CA + WAF rules.

## 12. Monitoring per env

| Tool | dev | staging | prod |
|---|---|---|---|
| Logs | stdout + Loki | Loki | Loki + S3 cold storage |
| Metrics | Prometheus | Prometheus | Prometheus + Mimir long-term |
| Traces | Jaeger local | Jaeger | Jaeger + Tempo |
| Alerts | None | Slack #dev-alerts | PagerDuty + Slack #prod-alerts |
| Uptime | None | Pingdom | Pingdom (1-min checks) |

## 13. Cost budgets (monthly)

| Env | Budget (USD) | Tracking |
|---|---|---|
| dev | $300 | Cloudflare + minimal cluster |
| staging | $800 | 2 nodes + small DBs |
| prod | $3000-5000 | Scale by traffic |

DevOps review monthly, alert if > 110% budget.

---

## 14. Multi-region deployment (v4.0 sovereignty)

> Khi launch global, mỗi region cần cluster riêng theo data sovereignty laws. Section này document deployment topology + routing + costs.

### 14.1 Cluster topology

```
                       ┌────────────────────────────────┐
                       │  Global Control Plane          │
                       │  (Observability + CI/CD only,  │
                       │   no PII data)                 │
                       │   Region: us-east-1 (Virginia) │
                       └────────────────────────────────┘
                                       │
        ┌──────────────────────────────┼─────────────────────────────┐
        │                              │                              │
        ▼                              ▼                              ▼
┌───────────────┐             ┌───────────────┐              ┌───────────────┐
│ smp-asia      │             │ smp-china     │              │ smp-us        │
│ cluster       │             │ cluster       │              │ cluster       │
│               │             │               │              │               │
│ Region: SG    │             │ Region: BJ    │              │ Region: VA    │
│ ap-southeast-1│             │ cn-beijing    │              │ us-east-1     │
│               │             │ (AliCloud)    │              │ (AWS)         │
│               │             │               │              │               │
│ Countries:    │             │ Countries:    │              │ Countries:    │
│  VN, TH, SG,  │             │  CN (only)    │              │  US (only)    │
│  ID, MY, PH   │             │               │              │               │
└───────────────┘             └───────────────┘              └───────────────┘
```

### 14.2 Cluster sizing per region

| Cluster | Region | Provider | Estimated DAU | K8s nodes | DB tier | Est. cost/month |
|---|---|---|---:|---|---|---:|
| smp-asia | ap-southeast-1 (SG) | AWS | 100k | 6× m5.large | RDS MySQL db.r6g.large | $4000 |
| smp-china | cn-beijing | Alibaba Cloud | TBD launch Q3 2026 | 3× ecs.c6.large | RDS MySQL 2-node | $1500 |
| smp-us | us-east-1 (Virginia) | AWS | TBD launch Q4 2026 | 3× m5.large | RDS MySQL db.r6g.large | $2500 |
| Global control plane | us-east-1 | AWS | n/a | 2× t3.large | none (just observability) | $400 |

**Total at full scale**: ~$8400/month (vs $3000-5000 single-region today).

### 14.3 Cross-region data flows

| Data type | Direction | Mechanism | Latency | Compliance |
|---|---|---|---|---|
| Master data (`smp_global`: countries, currencies, rates, tax, i18n) | Global → All regions | Kafka MirrorMaker pull | < 5min | OK (no PII) |
| Aggregated analytics (KPI metrics, anonymized) | All regions → Global | Kafka MirrorMaker push | < 15min | Anonymized only |
| Audit log (immutable, encrypted) | All regions → Global archive | Async batch (daily) | 24h | Encrypted + key rotation |
| PII data (orders, customers, agents, partners) | **NEVER cross region** | n/a | n/a | Strict |
| Application metrics (Prometheus) | All regions → Global | Mimir long-term | Real-time | No PII |
| Application logs (Loki) | Each region local + sanitized index → Global | Async filtered | < 1h | PII redacted before egress |

### 14.4 Latency map (typical)

| From → To | Latency p50 | Latency p99 |
|---|---:|---:|
| smp-asia (SG) ↔ VN users | 30-50ms | 80ms |
| smp-asia (SG) ↔ TH users | 25ms | 60ms |
| smp-asia (SG) ↔ ID users | 40ms | 90ms |
| smp-china (BJ) ↔ CN users | 20-40ms | 70ms |
| smp-us (VA) ↔ US East users | 20ms | 50ms |
| smp-us (VA) ↔ US West users | 65ms | 100ms |
| Inter-region (SG ↔ BJ) | 200ms | 400ms |
| Inter-region (SG ↔ VA) | 250ms | 500ms |

### 14.5 Routing & client resolution

API Gateway at edge (Cloudflare Workers) determines target cluster:

```javascript
// Pseudo-code
function resolveCluster(request) {
  // 1. Check JWT (authenticated user)
  const jwt = getJWT(request);
  if (jwt?.country_code) {
    return clusterFor(jwt.country_code);
  }
  
  // 2. Geolocation from Cloudflare CF-IPCountry header
  const country = request.headers['CF-IPCountry'];
  if (country) {
    return clusterFor(country);
  }
  
  // 3. Default to nearest cluster by datacenter
  return defaultClusterByDC(request.cf.colo);
}

function clusterFor(country) {
  const map = {
    'CN': 'smp-china',
    'US': 'smp-us',
    // VN, TH, SG, ID, MY, PH, etc → smp-asia
  };
  return map[country] || 'smp-asia';
}
```

**Critical**: routing decision is **sticky per session**. Once user logged in, all subsequent requests go to same cluster (avoid data hopping).

### 14.6 Cross-region failover

Each cluster is **independent** — failure of 1 cluster does NOT affect others.

| Cluster down | Impact | Mitigation |
|---|---|---|
| smp-asia | VN/TH/SG/ID/MY/PH users see error | Cloudflare maintenance page · cannot serve from other clusters (sovereignty) |
| smp-china | CN users see error | Same — cannot fallback to other regions |
| smp-us | US users see error | Same |
| Global control plane | Observability/CI/CD down, but apps OK | Apps continue running with last-known config |

**Implication**: each cluster needs full HA (3 AZ minimum). NO cross-region failover for user data.

### 14.7 Deployment sequence (per release)

```
1. Engineer merges PR to main
   ▼
2. CI builds Docker image (single image for all regions)
   ▼
3. Push image to:
   - ECR ap-southeast-1 (smp-asia)
   - ACR cn-beijing (smp-china) - via VPN/proxy
   - ECR us-east-1 (smp-us)
   ▼
4. ArgoCD deploys to each cluster sequentially:
   - dev (single region) → smoke test
   - staging (smp-asia only) → integration test
   - prod smp-asia → 10% traffic canary (24h soak)
   - prod smp-asia → 100% rollout
   - prod smp-us → canary (if applicable)
   - prod smp-china → canary (manual approval required)
```

**China deployment is manual**: due to firewall + ICP filing requirements, smp-china deploys are separate process owned by local DevOps in CN team.

### 14.8 Per-region cost monitoring

Tag every cloud resource với `region=<cluster-id>` for cost allocation:

```yaml
# Terraform tags
tags = {
  Environment   = "prod"
  Cluster       = "smp-asia"
  CountryGroup  = "VN-TH-SG-ID-MY-PH"
  CostCenter    = "engineering"
  Compliance    = "PDPA"
}
```

Cost dashboard split by `Cluster` tag. Alert if any cluster > 110% budget.
