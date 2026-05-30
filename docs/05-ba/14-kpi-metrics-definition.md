# SMP KPI & Metrics Definition
**Audience**: BA, PM, Ops, Data team · **Updated**: v3.3

---

## 1. Mục đích

Tài liệu này định nghĩa **chính xác** các metrics SMP track ở dashboard. Mỗi metric có:
- **Definition**: nói rõ đo cái gì
- **Formula**: công thức tính (SQL pseudo)
- **Dimensions**: filter/group by gì
- **Frequency**: tính real-time hay batch
- **Owner**: ai quan tâm metric này

> **Nguyên tắc**: nếu BA + Dev + Data đồng ý formula khác nhau → conflict. Phải resolve ở doc này trước khi build dashboard.

---

## 2. Business metrics (PM, Founder)

### 2.1 GMV (Gross Merchandise Value)
**Định nghĩa**: Tổng giá trị giao dịch thành công, gồm labor + material + VAT (trước bất kỳ discount/refund nào).

**Formula:**
```sql
SELECT SUM(total_amount + discount_amount) AS gmv
FROM orders
WHERE current_stage IN ('09_completed', '10_rated')
 AND created_at BETWEEN <from> AND <to>;
```

**Dimensions**: by period, by partner_id, by service_id, by city/district
**Frequency**: Daily snapshot (cron 00:05 hàng ngày)
**Unit**: VND

### 2.2 NMV (Net Merchandise Value)
**Định nghĩa**: GMV sau khi trừ discount và refund.

**Formula:**
```sql
SELECT SUM(total_amount) - SUM(refund_amount)
FROM orders
WHERE current_stage IN ('09_completed', '10_rated')
 AND created_at BETWEEN <from> AND <to>;
```

### 2.3 SMP Revenue
**Định nghĩa**: Phần SMP giữ lại (commission từ thợ + SaaS fee + material margin).

**Formula:**
```sql
-- Commission từ order labor
SELECT SUM(order_steps.labor_price * commission_rate)
FROM order_steps
JOIN orders ON ...
WHERE orders.current_stage IN ('09_completed', '10_rated')
 AND orders.created_at BETWEEN <from> AND <to>

UNION ALL

-- Material margin
SELECT SUM(material_variants.sell_price - material_variants.cost_price_avg) * quantity
FROM order_step_materials
WHERE source = 'reserved'
 AND order step is completed

UNION ALL

-- SaaS fee
SELECT SUM(partners.supplier_config.saas_fee_monthly)
FROM partners
WHERE supplier_config.model IN ('saas_only', 'hybrid')
 AND active in period
```

**Note**: tính 3 nguồn revenue tách biệt, sau đó cộng để có total SMP revenue.

### 2.4 Order count
**Định nghĩa**: Số đơn theo từng status trong period.

**Formula:**
```sql
SELECT current_stage, COUNT(*)
FROM orders
WHERE created_at BETWEEN <from> AND <to>
GROUP BY current_stage;
```

**Dimensions**: by source (customer_direct, partner_customer, contract), by city
**Frequency**: Real-time (Redis counter, sync to MySQL hourly)

### 2.5 Active customers
**Định nghĩa**: Khách có ≥ 1 đơn hoàn thành trong period.

**Formula:**
```sql
SELECT COUNT(DISTINCT customer_id)
FROM orders
WHERE current_stage IN ('09_completed', '10_rated')
 AND created_at BETWEEN <from> AND <to>;
```

### 2.6 Repeat customer rate
**Định nghĩa**: % khách có ≥ 2 đơn hoàn thành.

**Formula:**
```sql
WITH customer_orders AS (
 SELECT customer_id, COUNT(*) AS cnt
 FROM orders
 WHERE current_stage IN ('09_completed', '10_rated')
 GROUP BY customer_id
)
SELECT 
 SUM(CASE WHEN cnt >= 2 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS repeat_rate_pct
FROM customer_orders;
```

### 2.7 AOV (Average Order Value)
**Formula:**
```sql
SELECT AVG(total_amount)
FROM orders
WHERE current_stage IN ('09_completed', '10_rated')
 AND created_at BETWEEN <from> AND <to>;
```

## 3. Operational metrics (Ops Manager)

### 3.1 Order completion rate
**Định nghĩa**: % đơn hoàn thành (vs tổng tạo).

**Formula:**
```sql
SELECT 
 SUM(CASE WHEN current_stage IN ('09_completed', '10_rated') THEN 1 ELSE 0 END) * 100.0 / COUNT(*)
FROM orders
WHERE created_at BETWEEN <from> AND <to>;
```

**Target**: ≥ 90%

### 3.2 Cancellation rate
**Formula:**
```sql
SELECT 
 SUM(CASE WHEN current_stage = 'cancelled' THEN 1 ELSE 0 END) * 100.0 / COUNT(*)
FROM orders
WHERE created_at BETWEEN <from> AND <to>;
```

**Sub-dimension**: cancellation reason (customer_cancel, no_agent_available, quote_rejected, payment_failed)

**Target**: ≤ 5%

### 3.3 Dispatch SLA
**Định nghĩa**: % đơn assigned to agent trong N phút sau khi tạo.

**Formula:**
```sql
SELECT 
 SUM(CASE 
 WHEN TIMESTAMPDIFF(MINUTE, created_at, accept_at) <= 5 
 THEN 1 ELSE 0 END
 ) * 100.0 / COUNT(*)
FROM (
 SELECT o.created_at, MIN(log.created_at) AS accept_at
 FROM orders o
 JOIN order_stage_log log ON log.order_id = o.id
 WHERE log.to_stage = '03_survey_accepted'
 GROUP BY o.id
) t;
```

**Target**: 95% within 5 phút

### 3.4 Average time to assign
**Formula:**
```sql
SELECT AVG(TIMESTAMPDIFF(SECOND, created_at, accept_at)) AS avg_seconds
FROM ...
```

### 3.5 Average order duration (created → completed)
**Formula:**
```sql
SELECT AVG(TIMESTAMPDIFF(HOUR, created_at, completed_at))
FROM orders
WHERE current_stage IN ('09_completed', '10_rated');
```

### 3.6 Stuck orders count (alert)
**Định nghĩa**: Đơn ở 1 stage quá lâu so với threshold.

**Stages + threshold:**
| Stage | Max time | If exceeded |
|---|---|---|
| 01_created | 5 min | Dispatch issue |
| 02_dispatched_survey | 15 min | No agent accepting |
| 03_survey_accepted → 04_arrived_survey | 60 min | Agent late |
| 05_surveyed → 06_quote_approved | 24h | Customer not responding |
| 07_dispatched_execution | 15 min | No agent accepting |
| 08_in_progress | service duration × 2 | Step too long |

```sql
SELECT id, current_stage, TIMESTAMPDIFF(MINUTE, last_stage_change, NOW) AS stuck_min
FROM orders
WHERE current_stage NOT IN ('09_completed', '10_rated', 'cancelled')
 AND TIMESTAMPDIFF(MINUTE, last_stage_change, NOW) > <threshold>;
```

**Frequency**: Real-time alert (every 5 min cron)

### 3.7 Manual dispatch ratio
**Định nghĩa**: % đơn cần ops dispatch tay (sau 3 rounds fail).

**Formula:**
```sql
SELECT 
 SUM(CASE WHEN manual_dispatched = TRUE THEN 1 ELSE 0 END) * 100.0 / COUNT(*)
FROM orders
WHERE current_stage IN ('09_completed', '10_rated')
 AND created_at BETWEEN <from> AND <to>;
```

**Target**: ≤ 3%

## 4. Agent metrics (Agent Squad)

### 4.1 Agent utilization
**Định nghĩa**: % thời gian agent đang trong job (vs online time).

**Formula:**
```sql
WITH agent_time AS (
 SELECT 
 agent_id,
 SUM(TIMESTAMPDIFF(MINUTE, online_start, online_end)) AS online_minutes
 FROM agent_sessions
 WHERE date BETWEEN <from> AND <to>
 GROUP BY agent_id
),
agent_busy AS (
 SELECT 
 execution_agent_id AS agent_id,
 SUM(TIMESTAMPDIFF(MINUTE, started_at, completed_at)) AS busy_minutes
 FROM order_steps
 WHERE status = 'completed'
 AND started_at BETWEEN <from> AND <to>
 GROUP BY execution_agent_id
)
SELECT t.agent_id, b.busy_minutes * 100.0 / t.online_minutes AS utilization_pct
FROM agent_time t
LEFT JOIN agent_busy b ON t.agent_id = b.agent_id;
```

**Target**: 50-70% (cao quá → quá tải, thấp quá → lãng phí)

### 4.2 Active agents (DAU, MAU)
**Định nghĩa**: Số agent có ≥ 1 đơn completed trong period.

**Daily**: `WHERE completed_at >= today_start`
**Monthly**: `WHERE completed_at >= month_start`

### 4.3 Agent rating average
**Formula:**
```sql
SELECT agent_id, AVG(rating) AS avg_rating, COUNT(*) AS cnt
FROM order_ratings
WHERE rated_at BETWEEN <from> AND <to>
GROUP BY agent_id;
```

### 4.4 Agent earnings
**Per period:**
```sql
SELECT 
 os.execution_agent_id,
 SUM(os.labor_price * (1 - commission_rate)) AS earnings
FROM order_steps os
WHERE os.status = 'completed'
 AND os.completed_at BETWEEN <from> AND <to>
GROUP BY os.execution_agent_id;
```

### 4.5 Agent acceptance rate
**Định nghĩa**: % đơn agent accept khi nhận invitation (vs decline/timeout).

**Formula:**
```sql
SELECT 
 agent_id,
 SUM(CASE WHEN response = 'accept' THEN 1 ELSE 0 END) * 100.0 / COUNT(*)
FROM dispatch_invitations
WHERE created_at BETWEEN <from> AND <to>
GROUP BY agent_id;
```

**Note**: agent có acceptance rate < 30% → flag for ops review (low engagement).

### 4.6 No-show rate
**Định nghĩa**: % đơn agent accept nhưng không đến.

**Formula:**
```sql
SELECT 
 agent_id,
 SUM(CASE WHEN no_show = TRUE THEN 1 ELSE 0 END) * 100.0 / total_assigned
FROM agent_performance
WHERE date BETWEEN <from> AND <to>;
```

**Target**: ≤ 1%

## 5. Partner metrics (Partner Squad)

### 5.1 Partner GMV
**Định nghĩa**: GMV của các đơn có source = `partner_customer` cho partner cụ thể.

**Formula:**
```sql
SELECT SUM(total_amount)
FROM orders
WHERE source = 'partner_customer'
 AND partner_id = <partner>
 AND current_stage IN ('09_completed', '10_rated')
 AND created_at BETWEEN <from> AND <to>;
```

### 5.2 Partner active rate
**Định nghĩa**: % partner có ≥ 1 đơn trong period (vs tổng active partners).

```sql
SELECT 
 SUM(CASE WHEN order_count > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*)
FROM (
 SELECT p.id, COUNT(o.id) AS order_count
 FROM partners p
 LEFT JOIN orders o ON o.partner_id = p.id AND o.created_at BETWEEN <from> AND <to>
 WHERE p.status = 'active'
 GROUP BY p.id
) t;
```

### 5.3 Partner wallet health
**Định nghĩa**: % partner có balance < 7 ngày spending average (alert).

```sql
WITH partner_daily_spend AS (
 SELECT partner_id, AVG(daily_amount) AS avg_daily
 FROM (
 SELECT partner_id, DATE(created_at) AS d, SUM(total_amount) AS daily_amount
 FROM orders
 WHERE source = 'partner_customer'
 AND created_at >= NOW - INTERVAL 30 DAY
 GROUP BY partner_id, d
 ) t
 GROUP BY partner_id
)
SELECT 
 p.id, p.business_name, p.customer_config->>'$.wallet_balance' AS balance, 
 ds.avg_daily * 7 AS reserve_needed,
 CASE WHEN balance < ds.avg_daily * 7 THEN 'LOW' ELSE 'OK' END AS status
FROM partners p
JOIN partner_daily_spend ds ON ds.partner_id = p.id;
```

**Trigger**: nếu LOW → email partner_finance role với "Cần nạp ví"

### 5.4 Partner outstanding invoice
**Formula:**
```sql
SELECT partner_id, SUM(total_amount) AS outstanding
FROM partner_invoices
WHERE status IN ('sent', 'overdue')
GROUP BY partner_id;
```

### 5.5 Partner agent count
**Formula:**
```sql
SELECT 
 partner_id,
 SUM(CASE WHEN status = 'active' THEN 1 ELSE 0 END) AS active_count,
 COUNT(*) AS total_count
FROM agents
WHERE partner_id IS NOT NULL
GROUP BY partner_id;
```

## 6. Quality metrics (Quality Squad)

### 6.1 Average rating
**Formula:**
```sql
SELECT AVG(rating)
FROM order_ratings
WHERE rated_at BETWEEN <from> AND <to>;
```

**Target**: ≥ 4.5

### 6.2 Rating distribution
**Formula:**
```sql
SELECT rating, COUNT(*) 
FROM order_ratings
WHERE rated_at BETWEEN <from> AND <to>
GROUP BY rating;
```

### 6.3 NPS (Net Promoter Score)
**Định nghĩa**: % promoters (9-10) - % detractors (0-6) (giả định scale 0-10).

**Note**: SMP đang dùng scale 1-5 sao, chuyển đổi:
- Promoters: 5 sao
- Passives: 4 sao
- Detractors: 1-3 sao

```sql
WITH t AS (
 SELECT 
 SUM(CASE WHEN rating = 5 THEN 1 ELSE 0 END) AS promoters,
 SUM(CASE WHEN rating <= 3 THEN 1 ELSE 0 END) AS detractors,
 COUNT(*) AS total
 FROM order_ratings
)
SELECT (promoters - detractors) * 100.0 / total AS nps FROM t;
```

**Target**: ≥ 50

### 6.4 Dispute rate
**Formula:**
```sql
SELECT COUNT(disputes) * 100.0 / total_orders
FROM ...
```

**Target**: ≤ 2%

### 6.5 Average dispute resolution time
**Formula:**
```sql
SELECT AVG(TIMESTAMPDIFF(HOUR, opened_at, resolved_at))
FROM disputes
WHERE resolved_at IS NOT NULL
 AND resolved_at BETWEEN <from> AND <to>;
```

**Target**: ≤ 48h

### 6.6 Free-form material rate
**Định nghĩa**: % step có free-form material (vs reserved).

**Formula:**
```sql
SELECT 
 SUM(CASE WHEN source = 'free_form' THEN 1 ELSE 0 END) * 100.0 / COUNT(*)
FROM order_step_materials
WHERE created_at BETWEEN <from> AND <to>;
```

**Target**: ≤ 10% (cao quá → BOM coverage kém)

### 6.7 Variance review rate
**Định nghĩa**: % BOM có actual khác > 5% expected.

**Formula**: tính từ comparison BOM expected vs actual per step.

## 7. Financial metrics (Finance Squad)

### 7.1 Cash collected vs receivable
**Formula:**
```sql
SELECT 
 SUM(CASE WHEN payment_status = 'paid' THEN total_amount ELSE 0 END) AS collected,
 SUM(CASE WHEN payment_status = 'pending' THEN total_amount ELSE 0 END) AS receivable
FROM orders
WHERE created_at BETWEEN <from> AND <to>;
```

### 7.2 Refund rate
**Formula:**
```sql
SELECT SUM(refund_amount) / SUM(total_amount) * 100.0
FROM orders WHERE ...;
```

### 7.3 Pending payouts to agents/partners
**Formula:**
```sql
SELECT SUM(net_amount) FROM partner_payouts WHERE status = 'pending';
SELECT SUM(amount) FROM agent_payouts WHERE status = 'pending';
```

### 7.4 Material margin
**Formula:**
```sql
SELECT 
 SUM((mv.sell_price - mv.cost_price_avg) * osm.quantity) AS total_margin
FROM order_step_materials osm
JOIN material_variants mv ON osm.material_variant_id = mv.id
WHERE osm.source = 'reserved'
 AND osm.step is completed
 AND period;
```

## 8. Technical metrics (Engineering)

### 8.1 API latency (p50, p95, p99)
**Source**: Prometheus
**Dashboard**: Grafana

```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

**SLO targets**:
- Read endpoints: p95 < 300ms
- Write endpoints: p95 < 500ms
- Dispatch: p95 < 2s

### 8.2 Error rate (5xx)
```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

**SLO**: < 0.1%

### 8.3 Webhook delivery success rate
```sql
SELECT 
 SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END) * 100.0 / COUNT(*)
FROM webhook_delivery_log
WHERE sent_at BETWEEN <from> AND <to>;
```

### 8.4 Event log DLQ depth
**Source**: MongoDB count
**Alert**: > 100 entries

### 8.5 Cache hit ratio
**Source**: Redis INFO
**Target**: > 80% for cached endpoints

## 9. Dashboard mapping

### Dashboard 1 · Executive overview (PM, Founder)
- GMV (this month + last month + YoY if available)
- Order count
- Active customers
- Top services by revenue
- Top partners by GMV
- Rating average + NPS

### Dashboard 2 · Operations live (Ops Manager)
- Orders in progress (by stage breakdown)
- Stuck orders count + list
- Active agents (online now)
- Dispatch queue depth
- Manual dispatch tasks
- Pending material verifies

### Dashboard 3 · Agent performance (Agent Squad)
- Total agents · active vs pending KYC
- DAU / MAU
- Utilization distribution
- Top performers (orders + rating)
- Bottom performers (acceptance rate)

### Dashboard 4 · Partner overview (Partner Squad)
- Total partners · active
- Partner GMV ranking
- Wallet health alerts
- Outstanding invoices
- Recent KYC pending

### Dashboard 5 · Quality (Quality Squad)
- Rating average trend
- Dispute count + resolution time
- Free-form rate trend
- BOM variance trend
- Recent dispute list (open)

### Dashboard 6 · Finance (Finance Squad)
- Cash collected
- Pending payouts
- Material margin
- Refund rate
- Outstanding invoices aging

### Dashboard 7 · Tech health (Engineering)
- API latency
- Error rate
- Webhook delivery
- DLQ depth
- Pod CPU/RAM

## 10. Data pipeline

### 10.1 Real-time (Redis counters)
- Orders count by status (TTL 24h)
- Active agents (sets)
- Dispatch queue depth

### 10.2 Near real-time (5-min aggregations)
- Stuck orders alert
- Webhook delivery success
- Error rate

### 10.3 Hourly batch
- Order completion rate
- Average duration
- Cancellation reasons breakdown

### 10.4 Daily batch (00:05)
- GMV, NMV, SMP Revenue
- Partner GMV, wallet health
- Agent utilization, earnings
- Material margin

### 10.5 Tooling
- Prometheus for tech metrics
- Custom Go cron jobs for business metrics → write to `smp_analytics` MySQL DB
- Grafana queries `smp_analytics` for dashboards
- Phase 2: dbt + ClickHouse for heavy analytics

---

## 10.6 · v4.0 architecture · CQRS Read Model

> Quan trọng: từ v4.0, hầu hết dashboard queries chuyển sang **Elasticsearch read model** thay vì query MySQL primary trực tiếp. Lý do: scale + decouple workload analytic khỏi transactional.

### 10.6.1 Architecture diagram

```text
┌─────────────────────────────────────────────────────────────────┐
│ WRITE SIDE (Commands) │
│ │
│ API → service → MySQL primary │
│ │ │
│ │ binlog │
│ ▼ │
│ ┌──────────┐ │
│ │ Debezium │ CDC connector │
│ └────┬─────┘ │
│ │ │
│ ▼ │
│ ┌────────────────┐ │
│ │ Kafka topics │ │
│ └────────┬───────┘ │
└───────────────────────┼─────────────────────────────────────────┘
 │
┌───────────────────────┼─────────────────────────────────────────┐
│ ▼ READ SIDE (Queries) │
│ ┌────────────────┐ │
│ │ ES sink │ (Kafka Connect) │
│ │ Connector │ │
│ └────────┬───────┘ │
│ │ │
│ ▼ │
│ ┌────────────────┐ │
│ │ Elasticsearch │ ◄── read model │
│ │ indices: │ │
│ │ orders-YYYY-MM│ (daily, denormalized) │
│ │ agents │ │
│ │ partners │ │
│ │ dispatch-runs │ │
│ └────────┬───────┘ │
│ │ │
│ ┌──────────────┼──────────────┐ │
│ ▼ ▼ ▼ │
│ ┌─────────┐ ┌──────────┐ ┌──────────┐ │
│ │Grafana │ │Admin UI │ │Reports │ │
│ │dashboard│ │analytics │ │CSV/Excel │ │
│ └─────────┘ └──────────┘ └──────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 10.6.2 Query routing matrix

Cập nhật cách truy vấn từng metric:

| Metric category | v3.x source | v4.0 source | Latency | Fallback |
|---|---|---|---|---|
| Real-time counts (Section 3.6 Stuck orders) | MySQL primary | Redis (unchanged) | <100ms | n/a |
| Hourly aggs (Section 3.1 Completion rate) | MySQL replica | **Elasticsearch** | 1-5s lag | MySQL replica |
| Daily aggs (Section 2.1 GMV) | MySQL replica + cron | **Elasticsearch** + scheduled refresh | 1-5s lag | MySQL replica |
| Search queries (Customer support find order) | MySQL LIKE | **Elasticsearch full-text** | 50-200ms | MySQL LIKE (slow) |
| Map visualizations (heatmap orders by district) | MySQL geospatial | **Elasticsearch geo_point** | <500ms | n/a |
| Financial reports (Section 7.x) | MySQL primary (auditable) | **MySQL primary** (unchanged) | n/a | n/a |

**Note**: Financial reports KHÔNG migrate sang ES vì cần exact consistency cho audit. Eventual consistency 1-5s không acceptable cho accounting close.

### 10.6.3 Sample queries · before/after

**Example 1: GMV by district (Section 3.x)**

```sql
-- ❌ v3.x: SQL on MySQL replica
SELECT
 district,
 SUM(total_amount) AS gmv,
 COUNT(*) AS order_count
FROM orders
WHERE completed_at_utc >= DATE_SUB(NOW, INTERVAL 1 DAY)
 AND status = 'paid'
GROUP BY district
ORDER BY gmv DESC;
-- ~3-5 seconds with 1M orders
```

```json
// ✅ v4.0: Elasticsearch aggregation
POST /orders-*/_search
{
 "size": 0,
 "query": {
 "bool": {
 "filter": [
 {"range": {"completed_at_utc": {"gte": "now-1d/d"}}},
 {"term": {"status": "paid"}}
 ]
 }
 },
 "aggs": {
 "by_district": {
 "terms": {"field": "district", "size": 100, "order": {"gmv": "desc"}},
 "aggs": {
 "gmv": {"sum": {"field": "total_amount"}}
 }
 }
 }
}
// ~200-500ms với same data
```

**Example 2: Order full-text search (admin support)**

```sql
-- ❌ v3.x: MySQL LIKE on multiple columns
SELECT * FROM orders
WHERE customer_name LIKE '%nguyen%'
 OR address_line LIKE '%nguyen%'
 OR notes LIKE '%nguyen%'
ORDER BY created_at DESC LIMIT 50;
-- 2-5 seconds, no index usage
```

```json
// ✅ v4.0: Elasticsearch multi_match
POST /orders-*/_search
{
 "query": {
 "multi_match": {
 "query": "nguyen",
 "fields": ["customer_name^3", "address_line", "notes"],
 "fuzziness": "AUTO"
 }
 },
 "sort": [{"created_at_utc": "desc"}],
 "size": 50
}
// 50-100ms với analyzer + inverted index
```

### 10.6.4 Eventual consistency · UX guidance

Dashboard hiển thị data lag indicator:
- **Lag < 5s**: "Real-time" badge (green)
- **Lag 5-60s**: "Updated X seconds ago" (yellow)
- **Lag > 60s**: Banner "Data may be stale - last sync: HH:MM" + fallback to MySQL replica
- **Lag > 5min**: Critical alert, page on-call

Critical UX rule: **không hiển thị dashboard mà không có lag indicator** — user phải biết data tươi/cũ thế nào.

### 10.6.5 Elasticsearch index design

Daily rolling indices cho `orders`:
```text
orders-2026-05-28 ← hot (today, write + read)
orders-2026-05-27 ← hot (yesterday)
orders-2026-05-26 ← warm (older, read-only)
...
orders-2026-04-* ← warm → cold tier
```

**Index template** (Elasticsearch):
```json
PUT /_index_template/orders-template
{
 "index_patterns": ["orders-*"],
 "template": {
 "settings": {
 "number_of_shards": 3,
 "number_of_replicas": 1,
 "refresh_interval": "5s"
 },
 "mappings": {
 "properties": {
 "order_id": {"type": "keyword"},
 "customer_id": {"type": "keyword"},
 "customer_name": {
 "type": "text",
 "fields": {"keyword": {"type": "keyword"}}
 },
 "country_code": {"type": "keyword"},
 "currency": {"type": "keyword"},
 "total_amount": {"type": "long"},
 "status": {"type": "keyword"},
 "district": {"type": "keyword"},
 "city": {"type": "keyword"},
 "location": {"type": "geo_point"},
 "created_at_utc": {"type": "date"},
 "completed_at_utc": {"type": "date"}
 }
 }
 }
}
```

ILM (Index Lifecycle Management):
- **Hot phase**: 7 days, fast SSD, write enabled
- **Warm phase**: 60 days, slower disks, read-only, force_merge
- **Cold phase**: 1 year, archive, very slow access
- **Delete phase**: > 1 year, auto-delete

### 10.6.6 Failover

```text
Primary path: Service → ES query
 ▼ if ES down
Fallback path: Service → MySQL replica
 ▼ if also down
Emergency: Service → return cached snapshot (1h old) + alert
```

Circuit breaker pattern: nếu ES timeout > 3 lần trong 1 phút, mở circuit, route 100% traffic sang MySQL fallback trong 30s.

## 11. Validation

Cuối tháng, Finance đối chiếu:
- SMP Revenue formula (Section 2.3) vs accounting books
- Pending payouts (Section 7.3) vs bank statements
- GMV vs invoice totals

Bất kỳ discrepancy > 1% → investigate + fix formula or data pipeline.

## 12. Change management

Khi đổi formula 1 metric:
1. Tạo PR với new formula + old formula side-by-side comparison
2. Test trên historical data 30 ngày
3. PM + Finance approve
4. Deploy + recalculate historical metrics
5. Notify all dashboard users
6. Document change in this doc + changelog
