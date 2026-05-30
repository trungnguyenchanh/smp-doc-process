# SMP Runbook · Top 5 Incidents
**Audience**: DevOps, SRE, On-call engineers · **Format**: Cookbook · **Updated**: v3.3

---

## Cách dùng runbook

1. Alert fires → on-call nhận PagerDuty
2. Tìm runbook khớp alert title (alert linkage)
3. Follow steps theo thứ tự: **Detect → Triage → Mitigate → Communicate → Resolve → Post-mortem**
4. Update doc nếu phát sinh kịch bản mới

---

## INC-001 · Dispatch engine down

### Symptoms
- Alert: `dispatch_engine_up == 0` for 2 min
- Alert: `order_stuck_in_stage_02_count > 10`
- Customer complaint: "đặt đơn không có thợ đến"
- Order count tăng nhưng stage không tiến

### Detect
```bash
# Kiểm tra pod status
kubectl -n smp-prod get pods -l app=dispatch-engine

# Kiểm tra logs gần đây
kubectl -n smp-prod logs -l app=dispatch-engine --tail=200 | grep -E "ERROR|FATAL|panic"

# Kiểm tra Redis (dispatch queue)
redis-cli -h prod-redis.smp.local LLEN dispatch:queue:HCMC
```

### Triage (5 phút)

**Nếu pod crashloop:**
- Lý do thường gặp: panic do nil ptr, OOM, Redis connection lost
- Check `kubectl describe pod` → `OOMKilled`?

**Nếu pod chạy nhưng không xử lý:**
- Check WebSocket connections: `curl http://dispatch-engine:8080/admin/stats`
- Check Kafka consumer lag: `kafka-consumer-groups --describe --group dispatch-engine`

**Nếu Redis queue đầy:**
- `LLEN dispatch:queue:HCMC` > 100 → backpressure

### Mitigate

**Option A · Restart pods (nhanh nhất, dùng khi panic/hang):**
```bash
kubectl -n smp-prod rollout restart deployment/dispatch-engine
# Đợi 30s, check pods up
kubectl -n smp-prod get pods -l app=dispatch-engine -w
```

**Option B · Scale up (nếu overload):**
```bash
kubectl -n smp-prod scale deployment/dispatch-engine --replicas=5
# Sau 5 phút đánh giá lại, scale down nếu OK
```

**Option C · Manual dispatch fallback:**
- Ops Lead vào admin portal → "Manual dispatch" tab
- Assign orders đang stuck thủ công cho thợ cụ thể
- Document trong audit log

**Option D · Disable dispatch tạm thời:**
- Feature flag `ENABLE_AUTO_DISPATCH = false`
- Customer thấy thông báo "Đang xử lý đơn của bạn, ops team sẽ liên hệ trong 15 phút"
- Ops handle hoàn toàn manual cho đến khi fix

### Communicate
- Slack #prod-alerts: "Dispatch engine down, investigating ETA 15 min"
- Status page (status.smp.vn): update component "Dispatch" → "Degraded"
- Customer-facing: in-app banner "Đang xử lý chậm, mời thông cảm"
- Nếu > 30 phút: email partners

### Resolve
- Verify pod healthy: `curl http://dispatch-engine:8080/health` → 200
- Verify queue draining: `LLEN dispatch:queue:HCMC` giảm về 0
- Verify orders moving from stage 02 → 03
- Update status page → "Operational"
- Slack: "Resolved, monitoring 30 min"

### Post-mortem (within 48h if customer impact)
- Root cause + timeline
- Detection gap (alerted in 2 min OK or delayed?)
- Action items: code fix, alert tuning, runbook update

---

## INC-002 · Payment webhook failed (inside down)

### Symptoms
- Alert: `webhook_processing_errors_5xx_rate > 5% for 5 min`
- Alert: `order_payment_pending_count > 50`
- Customer complaint: "đã trả tiền nhưng app báo chưa trả"

### Detect
```bash
# Kiểm tra integration-svc logs
kubectl -n smp-prod logs -l app=integration-svc --tail=200 | grep webhook

# Kiểm tra DLQ trong MongoDB
mongosh "$MONGO_URI" --eval 'db.event_log.countDocuments({status: "dlq"})'

# Test webhook endpoint
curl -X POST https://api.smp.vn/integrations/inside/payment-webhook -d '{"test":true}' -H "X-Signature: ..."
```

### Triage

**Case 1 · inside không gửi webhook:**
- Check inside status: liên hệ team inside
- Webhooks đang queue ở inside side (retry tự động)

**Case 2 · SMP nhận webhook nhưng fail processing:**
- Check `event_log.status = "dlq"` để xem error
- Common: invalid signature, order_id không tìm thấy, schema mismatch

**Case 3 · Network giữa SMP ↔ inside:**
- Curl test từ pod: `kubectl exec -it integration-svc-xxx -- curl http://inside.local/health`

### Mitigate

**Option A · Manual replay từ DLQ:**
```bash
# Get list events trong DLQ
mongosh "$MONGO_URI" --eval '
 db.event_log.find({status: "dlq", event_type: "inside.payment.succeeded"})
 .limit(100).forEach(e => print(e.event_id))
'

# Replay endpoint (admin only)
for eid in $(...); do
 curl -X POST https://api.smp.vn/admin/events/$eid/replay \
 -H "Authorization: Bearer $ADMIN_TOKEN"
done
```

**Option B · Manual reconcile from inside DB:**
- Ops Finance query inside → list payments completed today
- Cross-check với SMP orders.payment_status = "pending"
- Manual mark paid via admin UI

**Option C · Disable validation tạm (LAST RESORT):**
- KHÔNG dùng trong production. Có thể dùng ở staging để debug.

### Communicate
- Slack #prod-alerts ngay
- inside team liên hệ (chat group "SMP × inside")
- Status page: component "Payments" → "Degraded"
- Customer trong app: "Thanh toán đang xác nhận, mời đợi 5 phút"

### Resolve
- Webhook backlog drain về 0
- Orders với payment_status = "pending" được update đúng
- Post-mortem để tránh tái diễn

---

## INC-003 · MySQL replica lag > 30s

### Symptoms
- Alert: `mysql_replica_lag_seconds > 30`
- Reports inconsistency: list orders mới created không thấy ở report dashboard
- Slow queries trên replica

### Detect
```bash
# Check replica status
mysql -h prod-mysql-replica.smp.local -e "SHOW REPLICA STATUS\G" | grep -E "Seconds_Behind|Last_Error"

# Check binlog position
mysql -h prod-mysql-primary.smp.local -e "SHOW MASTER STATUS"

# Check replica thread
mysql -h prod-mysql-replica.smp.local -e "SHOW PROCESSLIST" | grep system
```

### Triage

**Case 1 · Long-running query block replica:**
- `SHOW PROCESSLIST` → kill query nếu cần

**Case 2 · Network/disk issue:**
- Check disk IOPS, network latency between primary ↔ replica
- Cloud provider status page

**Case 3 · Bulk insert đang chạy:**
- Migration script hoặc job ETL → đợi xong là replica catch up

### Mitigate

**Option A · Route reads to primary tạm thời:**
```yaml
# Update connection string in ConfigMap
MYSQL_READ_DSN=user:pass@tcp(prod-mysql-primary.smp.local:3306)/...
```
Sau đó restart pods affected.

**Option B · Skip non-critical event nếu blocked:**
```sql
STOP REPLICA;
SET GLOBAL sql_replica_skip_counter = 1;
START REPLICA;
```
**WARNING**: data inconsistency risk, cần justify.

**Option C · Re-init replica from backup (cuối cùng):**
- Stop replica
- Restore từ snapshot mới nhất
- CHANGE MASTER TO ... với binlog position mới
- START REPLICA

### Communicate
- Slack #infra-alerts
- Nếu read-heavy service ảnh hưởng (vd reports): notify business team

### Resolve
- `Seconds_Behind_Master = 0`
- Read replica routing restored

---

## INC-004 · Redis OOM (out of memory)

### Symptoms
- Alert: `redis_memory_used_percent > 90`
- Errors: `OOM command not allowed when used memory > 'maxmemory'`
- Cache miss rate tăng đột biến
- Session lookup fail → users bị logout

### Detect
```bash
redis-cli -h prod-redis.smp.local INFO memory | grep -E "used_memory|maxmemory"
redis-cli -h prod-redis.smp.local INFO stats | grep -E "evicted_keys|keyspace_misses"

# Largest keys
redis-cli -h prod-redis.smp.local --bigkeys
```

### Triage

**Identify culprit pattern:**
```bash
# Sample 1000 keys, group by prefix
redis-cli --scan --count 1000 | awk -F':' '{print $1}' | sort | uniq -c | sort -rn
```

Common offenders:
- `cache:wms:stock:*` — không expire (bug · phải có TTL 30s)
- `session:*` — quá nhiều active sessions
- `dispatch:invitations:*` — không cleanup sau order assigned

### Mitigate

**Option A · Flush specific pattern (cẩn thận, đọc note):**
```bash
# DANGER: chỉ flush cache, KHÔNG flush session/lock
redis-cli --scan --pattern "cache:wms:stock:*" | xargs redis-cli DEL
```
**Note**: dùng `--scan` không phải `KEYS` (KEYS block production).

**Option B · Scale up Redis (nếu managed service):**
- AWS ElastiCache: tăng node size
- Self-hosted: tăng `maxmemory` config + restart node (rolling nếu cluster)

**Option C · Fix eviction policy:**
```bash
# Set LRU eviction nếu chưa
redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

### Communicate
- Slack #infra-alerts
- Session-affected users: in-app "Vui lòng đăng nhập lại"

### Resolve
- `used_memory_percent < 70`
- Eviction rate < 1/s
- Fix root cause (TTL bug, leak)

---

## INC-005 · Integration circuit breaker triggered

### Symptoms
- Alert: `integration_circuit_breaker_state{service="inside"} == 1` (open)
- UI hiển thị "Đang cập nhật" thay vì customer info
- Logs: `circuit breaker open, fallback to default`

### Detect
```bash
# Check circuit breaker state
curl http://integration-svc:8080/admin/circuit-breaker

# Check error rate
kubectl logs -l app=integration-svc --tail=500 | grep -E "circuit|failed_to_call"
```

### Triage

**Verify dependency health:**
```bash
# From integration-svc pod
kubectl exec -it integration-svc-xxx -- curl http://inside.local/health
kubectl exec -it integration-svc-xxx -- curl http://wms.local/health
```

**Common causes:**
- inside/wms slow response > timeout (3s)
- inside/wms returning 5xx
- Auth token expired
- Network partition

### Mitigate

**Option A · Manual half-open (test recovery):**
```bash
curl -X POST http://integration-svc:8080/admin/circuit-breaker/half-open \
 -d '{"service":"inside"}'
```

**Option B · Bypass circuit (emergency):**
- Feature flag `INTEGRATION_BYPASS_CIRCUIT = true`
- Direct call without breaker → users see actual errors (better than stale)

**Option C · Use cached/stub data:**
- Integration-svc fallback to last successful response trong Redis
- Acceptable for read paths (customer info)
- NOT acceptable for write paths (payment, stock)

### Communicate
- Slack #infra-alerts
- Coordinate with inside/wms team
- Status page: affected component → "Degraded"

### Resolve
- Circuit breaker `state = closed`
- Error rate < 1%
- Verify with real user flow (test account)

---

## INC-006 · Kafka broker down 

**Trigger**: Prometheus alert `kafka_broker_down{cluster="prod"} > 0` hoặc producer/consumer error rate spike

### Detection
- Alert: `up{job="kafka"} == 0` for any broker
- Symptoms: producer write timeout, consumer lag tăng đột biến, services log `dial tcp: connection refused`

### Triage (5 phút)

```bash
# Check broker pods
kubectl get pods -n smp-prod -l app=kafka

# Check broker logs
kubectl logs -n smp-prod kafka-0 --tail=200

# Check cluster status
kubectl exec -n smp-prod kafka-0 -- kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# Check under-replicated partitions
kubectl exec -n smp-prod kafka-0 -- kafka-topics.sh --describe \
 --bootstrap-server localhost:9092 --under-replicated-partitions
```

### Mitigation

**Scenario 1: 1 broker down (out of 3)** · cluster still healthy
- Replication factor 3, min ISR 2 → cluster vẫn write được
- Không cần emergency action, fix broker bình thường
- ⚠️ Cảnh báo team không deploy mới cho đến khi broker recover

**Scenario 2: 2/3 brokers down** · cluster degraded
- Write blocked (min ISR = 2)
- Bring back ít nhất 1 broker ASAP:
```bash
kubectl rollout restart statefulset/kafka -n smp-prod
# Or scale up replacement
kubectl scale statefulset/kafka --replicas=4 -n smp-prod
```

**Scenario 3: All brokers down** · SEV-1 critical
- Producers buffer events trong memory (sẽ overflow sau ~30s)
- Consumers stall
- **Action**:
 1. Page on-call Tech Lead + CTO
 2. Activate fallback: services switch to **direct HTTP** mode (degraded mode, no eventual consistency)
 3. Restore Kafka cluster (xem DR runbook)
 4. Drain buffered events after recovery

### Prevention
- Multi-AZ deployment (brokers spread across 3 zones)
- Auto-restart unhealthy pods (k8s liveness probe)
- Monitoring: alert `under_replicated_partitions > 0` for > 5min

### Resolve
- All 3 brokers healthy
- `under_replicated_partitions == 0`
- Consumer lag back to normal baseline (< 1000 messages)

---

## INC-007 · Consumer lag spike 

**Trigger**: `kafka_consumergroup_lag{group="dispatch-engine"} > 10000`

### Detection
- Alert: lag > 10k messages for any consumer group
- Symptoms: events processed minutes-late, dispatch delays, notification delays

### Triage (5 phút)

```bash
# Check current lag per consumer group
kubectl exec kafka-0 -- kafka-consumer-groups.sh \
 --bootstrap-server localhost:9092 --describe --group dispatch-engine

# Output:
# TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET LAG CONSUMER-ID
# orders.events 0 15234 28500 13266 dispatch-engine-0
# orders.events 1 22100 28450 6350 dispatch-engine-1
# ...
```

### Mitigation

**Scenario 1: Slow consumer code** (common)
- Check consumer pod CPU/memory metrics
- Check downstream dependencies (DB slow query, external API slow)
```bash
kubectl top pod -n smp-prod -l app=dispatch-engine
kubectl logs -n smp-prod -l app=dispatch-engine --tail=100 | grep ERROR
```

**Scenario 2: Single partition hot** (skewed partitioning)
- Check if 1 partition has >>> messages than others
- Cause: bad partition key (vd cùng customer_id high-volume)
- Mitigation:
 - Short-term: scale up consumers
 - Long-term: review key strategy (xem doc 01 section 7.6.3)

**Scenario 3: Traffic spike**
- Scale up consumers:
```bash
kubectl scale deployment dispatch-engine --replicas=6 -n smp-prod
```
- Note: number of consumers ≤ number of partitions (extras idle). Topic có 12 partitions → max 12 consumers per group.

**Scenario 4: Poison message** (1 event repeatedly fails)
- Check DLQ:
```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
 --topic orders.events.dlq --from-beginning --max-messages 10
```
- Manually skip/fix the bad offset:
```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
 --group dispatch-engine --reset-offsets --to-offset <next_good_offset> \
 --topic orders.events:0 --execute
```

### Prevention
- Load test consumers regularly (chaos engineering)
- HPA (Horizontal Pod Autoscaler) on consumer pods based on lag metric

### Resolve
- Lag < 1000 messages
- Lag trend decreasing
- No DLQ messages accumulating

---

## INC-008 · Debezium connector crashed 

**Trigger**: `kafka_connect_connector_status{connector="smp-mysql-connector",state!="RUNNING"} > 0`

### Detection
- Alert: connector status != RUNNING
- Symptoms: no new CDC events in Kafka, MySQL writes not flowing to Elasticsearch, dashboard data freezes

### Triage (5 phút)

```bash
# Check connector status
curl -s http://kafka-connect:8083/connectors/smp-mysql-connector/status | jq

# Check task status (real worker)
curl -s http://kafka-connect:8083/connectors/smp-mysql-connector/tasks/0/status | jq

# Check Kafka Connect logs
kubectl logs -n smp-prod kafka-connect-0 --tail=300 | grep -i error
```

Common errors:
- `Lost connection to MySQL` → MySQL primary issue, check INC-related
- `Schema history topic not found` → infra issue
- `Snapshot taking too long` → initial sync, wait
- `Out of order events` → schema corruption, page on-call

### Mitigation

**Scenario 1: Connector paused/failed**
```bash
# Restart connector
curl -X POST http://kafka-connect:8083/connectors/smp-mysql-connector/restart

# Restart specific task
curl -X POST http://kafka-connect:8083/connectors/smp-mysql-connector/tasks/0/restart
```

**Scenario 2: MySQL binlog position lost** (connector trying to read beyond available history)
- Check MySQL `expire_logs_days` setting (should be >= 7d)
- If binlog purged, need full re-snapshot:
```bash
# Stop connector
curl -X PUT http://kafka-connect:8083/connectors/smp-mysql-connector/pause

# Delete offset (force re-snapshot)
# WARNING: this creates duplicate events in Kafka. Consumers must be idempotent.
kafka-delete-records.sh --bootstrap-server localhost:9092 \
 --offset-json-file offsets.json

# Resume
curl -X PUT http://kafka-connect:8083/connectors/smp-mysql-connector/resume
```

**Scenario 3: Schema change conflict** (rare)
- Manual fix schema history topic
- Page on-call engineer with Debezium expertise

### Prevention
- Monitor connector status every minute
- MySQL binlog retention >= 7 days
- Test connector restart in staging weekly

### Resolve
- Connector status = `RUNNING`
- Task status = `RUNNING`
- CDC events flowing (check `smp.cdc.smp_order.*` topic offset increasing)

---

## INC-009 · Elasticsearch read model drift 

**Trigger**: Dashboard shows lag > 60 seconds OR `es_sink_lag_seconds > 300`

### Detection
- Alert: ES sink lag > 5 minutes
- Symptoms: dashboard data old, search results stale, mismatch with MySQL

### Triage (5 phút)

```bash
# Check ES sink connector status
curl -s http://kafka-connect:8083/connectors/elasticsearch-sink/status

# Check Kafka consumer lag for ES sink group
kubectl exec kafka-0 -- kafka-consumer-groups.sh \
 --bootstrap-server localhost:9092 --describe --group connect-elasticsearch-sink

# Check ES cluster health
curl -s http://elasticsearch:9200/_cluster/health | jq

# Check index size
curl -s http://elasticsearch:9200/_cat/indices/orders-* | head -10
```

### Mitigation

**Scenario 1: ES cluster slow (red status)**
- Check disk space: `curl http://elasticsearch:9200/_cat/allocation`
- Free up old indices nếu cần:
```bash
curl -X DELETE "http://elasticsearch:9200/orders-2025-*"
```

**Scenario 2: Sink connector stalled**
- Restart sink:
```bash
curl -X POST http://kafka-connect:8083/connectors/elasticsearch-sink/restart
```

**Scenario 3: Schema mismatch** (field type changed in MySQL but ES mapping incompatible)
- Create new index version with corrected mapping
- Reindex: `curl -X POST http://elasticsearch:9200/_reindex -d '{...}'`
- Update connector to point new index
- Drop old index

**Scenario 4: Catastrophic divergence** (data clearly wrong)
- Full re-sync:
 1. Pause sink connector
 2. Delete ES indices
 3. Recreate from Kafka (Debezium will re-emit all events from binlog start)
 4. Resume sink

### Prevention
- Daily reconciliation job: count rows in MySQL vs documents in ES, alert if drift > 0.1%
- Test schema evolution in staging before prod

### Resolve
- Sink connector RUNNING
- Lag < 10s
- Spot check: query MySQL and ES for same order, verify match

---

## INC-010 · Wallet reconciliation mismatch 

**Trigger**: Daily reconciliation job alert · `wallet_recon_diff_pct > 0.01%` OR absolute diff > 100k VND

### Detection
- Alert from `wallet-reconciliation-cron` (runs daily 02:00 UTC)
- Symptoms: Finance dashboard shows mismatch between SMP wallet balance vs payment gateway records

### Triage (10 phút)

```bash
# Pull reconciliation report
kubectl exec -it finance-svc-0 -- /app/bin/recon-report --date=yesterday --format=csv > recon.csv

# Sample columns: partner_id, smp_balance, gateway_balance, diff_amount, diff_pct, last_txn_id
```

Common causes:
- Webhook from payment gateway missed/delayed → SMP not updated yet
- Manual adjustment in SMP without corresponding gateway entry
- Currency conversion rate mismatch (cron pulled stale rate)
- Bug in finance-svc calculation

### Mitigation

**Scenario 1: Webhook lag** (most common, ~80% cases)
- Wait 1h, re-run recon job
- If still off: replay webhooks from gateway:
```bash
kubectl exec finance-svc-0 -- /app/bin/replay-webhooks \
 --gateway=vnpay --from=$(date -d 'yesterday 00:00') --to=$(date -d 'today 00:00')
```

**Scenario 2: Manual adjustment** (no gateway txn)
- Check `partner_wallet_transactions WHERE type='adjustment'` cùng ngày
- Verify với Finance team về reason (vd compensation, error correction)
- If valid: update reconciliation expected_diff field, mark resolved
- If invalid: rollback adjustment, audit who made the change

**Scenario 3: Currency rate stale**
- Check `currency_rates` table latest date:
```sql
SELECT from_currency, to_currency, rate, rate_date
FROM currency_rates
WHERE rate_date >= CURDATE - INTERVAL 2 DAY
ORDER BY rate_date DESC LIMIT 20;
```
- If stale > 1 day: manually trigger rate refresh + recompute conversions
- Page Finance ASAP nếu involve large amount

**Scenario 4: Bug in calculation**
- Compare specific txn: pull both SMP record + gateway record
- Identify which side is wrong
- Patch + backfill correct value
- Required: post-mortem doc + add test case to prevent recurrence

### Prevention
- Daily recon job với auto-correction cho known patterns (lag < 24h)
- Alert escalation: > 1% diff = page on-call, > 5% = page CFO
- Monthly reconciliation report sent to Finance for review

### Resolve
- All wallets reconcile (diff < 0.01% và < 100k VND)
- Root cause documented
- If systemic bug: ticket created cho dev team fix

---

## INC-011 · Fraud detection triggered 

**Trigger**: `fraud_detection_signals_total{severity="high"} > 0` OR pattern alerts from Kafka stream

### Detection
- Stream processing job consumes `payments.events` + `partners.events` realtime
- Flags suspicious patterns matching fraud rules:
 - Multiple high-value topups from same IP across multiple partners
 - Velocity: > 5 topups within 10 min from same partner
 - Geography: topup IP country differs from partner registered country
 - Amount: topup > 10x partner average + first time
- Alerts go to `#security-alerts` Slack + page on-call

### Triage (5 phút) — TIME CRITICAL

```bash
# Pull recent suspicious txns
curl -s -H "Authorization: Bearer $JWT" \
 http://finance-svc:8080/internal/fraud-alerts/recent?hours=1 | jq

# Output: list of flagged partner_id, txn_ids, signals matched
```

### Mitigation

**Scenario 1: Confirmed fraud** (chargeback risk, money laundering signal)

```bash
# Immediate: freeze partner wallet (block further txns)
kubectl exec partner-svc-0 -- /app/bin/admin freeze \
 --partner=P00X --reason="fraud_alert_INC-011-2026-05-28"
```

- Notify partner via email + phone call (security team owns this comms)
- Coordinate with payment gateway to reverse pending settlements
- File report với:
 - Internal compliance (within 24h)
 - State Bank Vietnam (SBV) if > 300M VND single txn (luật phòng chống rửa tiền)
- Preserve evidence: export logs, IP records, device fingerprints

**Scenario 2: False positive** (legitimate, just unusual)
- Verify với partner via phone (use registered number)
- Unfreeze + add note in `partner_admin_users.notes`
- Tune fraud rule: increase threshold or exclude pattern

**Scenario 3: Suspicious but inconclusive**
- Soft block: limit wallet ops to 1M VND/day until investigation done (24-48h)
- Request additional documents (recent business invoices, ID re-verify)
- Internal review meeting

### Prevention
- Rules in `rules_engine.yaml` (category=fraud) reviewed monthly by Security
- IP allowlisting cho partner admin actions (configurable per partner)
- Velocity limits hard-coded as guard rails (no override even by Super Admin)
- Annual AML (anti-money laundering) training cho team

### Resolve
- Partner status restored or terminated (per investigation)
- Full audit trail filed
- Post-mortem ticket if false positive (tune rule)

---

## INC-012 · Cross-region replication lag (v4.0 sovereignty)

**Trigger**: `region_replication_lag_seconds{region="cn-to-global"} > 300` OR `> 60` for hot data

### Detection
- Cross-region replication monitor (deployed in central observability cluster)
- Symptoms: stale `currency_rates` / `tax_configs` / `i18n_translations` in remote clusters

### Triage (5 phút)

```bash
# Check replication status per region
kubectl --context=smp-asia get pods -n smp-prod -l app=replication-agent
kubectl --context=smp-china get pods -n smp-prod -l app=replication-agent
kubectl --context=smp-us get pods -n smp-prod -l app=replication-agent

# Check Kafka cross-region MirrorMaker status
curl -s http://mirrormaker-asia-to-cn:8080/metrics | grep lag
```

### Mitigation

**Scenario 1: Network blip** (common, transient)
- Wait 5-10 min, lag usually self-recovers
- If persistent: check ISP/cloud provider status page

**Scenario 2: Replication agent crashed**
```bash
# Restart MirrorMaker
kubectl --context=smp-asia rollout restart deployment/mirrormaker-to-cn -n smp-prod
```

**Scenario 3: Schema incompatibility** (rare, after deploy)
- Source emits new schema, target hasn't updated yet
- Pause replication, deploy target schema migration first, resume

**Scenario 4: Volume spike** (large batch insert at source)
- Lag will catch up over time
- Temporarily scale up replication workers:
```bash
kubectl --context=smp-asia scale deployment/mirrormaker-to-cn --replicas=3
```

**Compliance impact** (PIPL/GDPR concern):
- If CN cluster CAN'T pull updated global data → users see stale rates/tax → potentially wrong invoices
- If CN cluster CAN'T push out aggregated metrics → central analytics dashboard incomplete
- **CRITICAL**: Replication agent NEVER replicates PII outside CN. Only master data (countries, currencies, rates, i18n) is cross-region.

### Prevention
- 3 redundant replication paths between any 2 regions
- Auto-restart on crash (k8s)
- Monthly DR drill: simulate region isolation, verify behavior

### Resolve
- Lag < 30s for hot data, < 5min for warm data
- All regions verified consistent (spot check 10 records)
- If schema issue: deployment process updated to prevent recurrence

---

## General incident response process

### 1. Acknowledge (within 5 min)
- On-call ACK PagerDuty
- Post in #prod-alerts: "Investigating <alert>"

### 2. Assess severity
| Severity | Definition | Response |
|---|---|---|
| SEV-1 | Total outage, data loss | War room, all hands, status page red |
| SEV-2 | Major feature down | On-call + escalate, status page yellow |
| SEV-3 | Minor degradation | On-call alone, status page note |
| SEV-4 | Cosmetic | Ticket, normal flow |

### 3. War room (SEV-1, SEV-2)
- Slack channel `#incident-YYYY-MM-DD-<short>`
- Roles:
 - **Incident Commander** — coordinate, decide
 - **Communications Lead** — Slack + status page + customer comms
 - **Tech Lead** — debug + fix
 - **Scribe** — timeline log

### 4. Mitigate first, fix later
- Stop bleeding → users not affected
- Rollback if recent deploy suspicious
- Manual workaround if no quick fix

### 5. Resolution
- All metrics back to baseline
- No new errors for 30 min
- Status page green
- Close incident in tracker

### 6. Post-mortem (within 48h for SEV-1, SEV-2)
- Blameless format
- Sections: Summary, Timeline, Root cause, Impact, What went well, What went wrong, Action items
- Share with all engineering
- Track action items to completion

---

## Escalation matrix

| Time elapsed | Action |
|---|---|
| 0-15 min | On-call investigates |
| 15 min | Page secondary on-call |
| 30 min | Page Tech Lead |
| 1h | Page CTO + escalate |
| 2h | All-hands war room |

## On-call rotation

- Primary on-call: 1 SRE rotates weekly
- Secondary: backup SRE
- Schedule managed in PagerDuty
- Handoff Monday 10am: 15-min sync (active incidents, recent changes)

## Tools quick reference

| Tool | URL | Login |
|---|---|---|
| PagerDuty | smp.pagerduty.com | SSO |
| Grafana | grafana.smp.vn | SSO |
| Loki (logs) | grafana → "Explore" → Loki | SSO |
| Jaeger | jaeger.smp.vn | SSO |
| ArgoCD | argocd.smp.vn | SSO |
| K8s prod | `kubectl --context=prod` (after bastion + MFA) | k8s cert |
| Status page | status.smp.vn | StatusPage account |
