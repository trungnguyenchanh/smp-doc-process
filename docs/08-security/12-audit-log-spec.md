# SMP Audit Log Specification

**Storage**: MongoDB `smp_audit.audit_log` · **Retention**: 7 years · **Audience**: Security, Backend, Compliance

---

## 1. Purpose

Audit log ghi nhận **mọi hành động có ảnh hưởng** đến data, tài chính, security, compliance trên hệ thống SMP. Mục đích:

- **Forensics**: trace ai làm gì khi xảy ra incident
- **Compliance**: PDPA VN, tax regulation (giữ 7 năm)
- **Trust**: partner/customer query "ai sửa data của tôi"
- **Operations**: debug "đơn này tại sao stage chuyển bất ngờ"

## 2. Event categories

| Category | Examples | Required |
|---|---|---|
| **Auth** | login, logout, password_change, 2fa_enable, token_revoke | ✅ |
| **Account** | account_create, account_update, account_suspend, account_delete | ✅ |
| **Permission** | role_assign, scope_grant, scope_revoke | ✅ |
| **Order** | order_create, order_cancel, order_stage_transit (only manual), order_refund | ✅ |
| **Financial** | wallet_topup, wallet_debit, payout_create, payout_pay, invoice_generate | ✅ |
| **Partner** | partner_create, partner_kyc_approve, partner_suspend, partner_config_change | ✅ |
| **Agent** | agent_create, agent_kyc_approve, agent_skill_verify, agent_suspend | ✅ |
| **Catalog** | service_create, service_update, material_create, material_price_change | ✅ |
| **Material verify** | free_form_approve, free_form_reject, variance_review | ✅ |
| **Data access (sensitive)** | pii_view, financial_data_export, audit_log_query | ✅ |
| **System** | config_change, feature_flag_toggle, deploy_event | ✅ |
| **Security** | failed_login (5+), suspicious_access, ip_whitelist_change | ✅ |
| **PII Access ** | pii.unmask, pii.bulk_export, pii.search | ✅ critical |
| **Compliance ** | dsr.export_request, dsr.deletion_request, consent.update, breach.notify | ✅ critical |
| **Rules Engine ** | rules.config_apply, rules.reload, rules.evaluation_anomaly | ✅ |

**KHÔNG audit**:
- Read endpoints thông thường (vd `GET /orders`) — quá nhiều, dùng access log thay
- Dispatch state changes tự động (vd round_1_expired) — có order_stage_log riêng
- Cache misses, internal retries

### Cross-cutting · PII access events 

Đây là **category quan trọng nhất** từ compliance perspective. Mọi lần PII được unmask MUST có audit entry.

#### Event types

| action | When triggered | Severity |
|---|---|---|
| `pii.unmask.viewed` | Caller có scope `pii.unmask.<field>` xem được unmask data | Info |
| `pii.unmask.requested` | Caller call `POST /pii/unmask` cho sensitive field (CCCD, bank) | Notice |
| `pii.bulk_export.initiated` | Caller export CSV/Excel với PII unmask | **Warning** |
| `pii.search.by_phone` | Caller search customer/agent by exact phone number | Notice |
| `pii.search.by_email` | Tương tự với email | Notice |
| `pii.search.fuzzy` | Search PII với fuzzy/partial match (potential reconnaissance) | **Warning** |
| `pii.access.cross_country` | Caller country X access PII data của user country Y | **Critical** — block default |

#### Required fields cho PII events

```javascript
{
 // Standard audit fields (xem section 3)
 audit_id: "aud_01HX...",
 timestamp_utc: ISODate,
 actor_id: "user_123",
 actor_country: "VN",
 
 // PII-specific
 action: "pii.unmask.viewed",
 data_subject_type: "customer | agent | partner_admin",
 data_subject_id: "cus_01HX2M",
 data_subject_country: "VN",
 fields_unmasked: ["phone", "address_full"], // chính xác field nào
 unmask_session_id: "ums_01H..." (nếu qua /pii/unmask),
 reason: "Customer called for support, ticket ZD-1234" (required cho sensitive fields),
 
 // Context
 scope_used: ["pii.unmask.phone"],
 ip_address: "...",
 user_agent: "...",
 ticket_id: "ZD-1234" (nếu có),
 request_id: "req_..."
}
```

### Cross-cutting · Compliance events 

#### Data Subject Rights (DSR)

| action | Trigger | SLA |
|---|---|---|
| `dsr.export_request` | User request export data của họ | 7 ngày (PDPA), 30 ngày (GDPR) |
| `dsr.export_completed` | Hệ thống đã generate + gửi data | tracking |
| `dsr.deletion_request` | User request "right to be forgotten" | 30 ngày (PDPA), 30 ngày (GDPR), 45 ngày (CPRA) |
| `dsr.deletion_completed` | Data đã xóa across systems | tracking |
| `dsr.access_request` | User request danh sách "ai đã access data của tôi" | 7 ngày |
| `dsr.rectification_request` | User request sửa data sai | tracking |

#### Consent management

| action | Trigger |
|---|---|
| `consent.grant` | User opt-in cho category (marketing, analytics, share with partner) |
| `consent.withdraw` | User opt-out |
| `consent.expired` | Consent hết hạn (vd marketing 1 năm phải renew) |

#### Data breach

| action | Trigger | SLA |
|---|---|---|
| `breach.detected` | System/manual detect potential breach | Immediate page |
| `breach.confirmed` | Investigation confirmed breach | 24h |
| `breach.notified.authority` | Notified DPA (PDPA/GDPR authority) | 72h (PDPA, GDPR) |
| `breach.notified.users` | Notified affected users | 72h cho high-risk |
| `breach.resolved` | Containment + remediation done | tracking |

### Cross-cutting · Rules Engine events 

| action | Trigger |
|---|---|
| `rules.config_apply` | New `rules_engine.yaml` applied to ConfigMap | Notice |
| `rules.reload.success` | Service successfully hot-reloaded rules | Info (sample) |
| `rules.reload.failure` | Reload failed, kept old rules | **Warning** |
| `rules.evaluation_anomaly` | Rule eval returned unexpected default | **Warning** |
| `rules.override_used` | Production rule override invoked (vd surge override) | Notice |


## 3. Schema

```javascript
{
 // Identification
 _id: ObjectId,
 audit_id: "aud_01HX7K2M9QWRA1B2", // ULID
 timestamp: ISODate("2026-05-27T08:14:32.123Z"),
 schema_version: "1.0",
 
 // Actor (who did it)
 actor_type: "user" | "system" | "partner_api" | "external_webhook",
 actor_id: "usr_01HX7K2M" | "system" | "ptn_01HX",
 actor_role: "ops_admin" | "partner_owner" | "customer" | "technician" | null,
 actor_name: "Nguyễn Hùng",
 actor_ip: "210.245.32.18",
 user_agent: "Mozilla/5.0 ...",
 
 // Action
 action: "agent.kyc.approve", // <category>.<resource>.<verb>
 action_category: "agent",
 outcome: "success" | "failure" | "denied",
 
 // Resource (what was acted upon)
 resource_type: "agent" | "order" | "partner" | ...,
 resource_id: "agt_T4K9",
 resource_name: "Trương Minh K.",
 
 // Change details
 before: { ... }, // state before
 after: { ... }, // state after
 changes: ["kyc_level: pending → full", "verified_by: → usr_01HX"],
 
 // Context
 request_id: "req_01HX7K", // correlate with access log
 session_id: "ses_01HX",
 partner_id: "ptn_hung_acservice", // if relevant
 notes: "Approved after manual review",
 
 // Compliance flags
 contains_pii: true | false,
 legal_basis: "consent" | "contract" | "legitimate_interest" | "legal_obligation"
}
```

## 4. Action naming convention

Format: `<category>.<resource>.<verb>[.<qualifier>]`

Examples:
```text
auth.user.login
auth.user.login.failed
auth.user.logout
auth.user.password.changed
auth.user.2fa.enabled
auth.user.2fa.disabled
auth.user.token.revoked

account.partner_user.created
account.partner_user.updated
account.partner_user.suspended

order.order.created
order.order.cancelled
order.order.stage_transit.manual
order.order.refunded

finance.wallet.topup
finance.wallet.debit
finance.payout.created
finance.payout.paid
finance.invoice.generated
finance.invoice.sent

partner.partner.created
partner.partner.kyc.approved
partner.partner.kyc.rejected
partner.partner.suspended
partner.partner.config.updated

agent.agent.created
agent.agent.kyc.approved
agent.agent.skill.verified
agent.agent.suspended

catalog.service.created
catalog.service.price.updated
catalog.material_variant.created
catalog.material_variant.price.updated

quality.material_verify.approved
quality.material_verify.rejected
quality.dispute.opened
quality.dispute.resolved

data.pii.viewed
data.financial.exported
data.audit_log.queried

system.config.changed
system.feature_flag.toggled
system.deploy.completed

security.login.brute_force_detected
security.access.unauthorized_attempted
security.ip_whitelist.modified
```

## 5. Required fields per action category

Minimum required (must be present):
- `audit_id`, `timestamp`, `schema_version`
- `actor_type`, `actor_id`, `actor_ip`
- `action`, `outcome`
- `resource_type`, `resource_id`
- `request_id`

Optional per category:
- Financial: `before.balance`, `after.balance`, `amount`
- Permission change: `before.scopes`, `after.scopes`
- Config: `before.value`, `after.value`

## 6. Examples

### Example 1: Ops approves agent KYC
```json
{
 "audit_id": "aud_01HX7K2M",
 "timestamp": "2026-05-27T08:14:32.123Z",
 "schema_version": "1.0",
 "actor_type": "user",
 "actor_id": "usr_ops_001",
 "actor_role": "ops_admin",
 "actor_name": "Lê Quỳnh Mai",
 "actor_ip": "210.245.32.18",
 "action": "agent.agent.kyc.approved",
 "action_category": "agent",
 "outcome": "success",
 "resource_type": "agent",
 "resource_id": "agt_T4K9",
 "resource_name": "Trương Minh K.",
 "before": { "kyc_level": "pending", "verified_by": null },
 "after": { "kyc_level": "full", "verified_by": "usr_ops_001" },
 "changes": ["kyc_level: pending → full"],
 "request_id": "req_01HX7K",
 "notes": "All docs verified manually"
}
```

### Example 2: Partner Owner change config
```json
{
 "audit_id": "aud_01HX7K2N",
 "timestamp": "2026-05-27T09:00:01.456Z",
 "actor_type": "user",
 "actor_id": "usr_partner_hung",
 "actor_role": "partner_owner",
 "actor_name": "Nguyễn Hùng",
 "actor_ip": "118.69.x.x",
 "action": "partner.partner.config.updated",
 "outcome": "success",
 "resource_type": "partner",
 "resource_id": "ptn_hung_acservice",
 "partner_id": "ptn_hung_acservice",
 "before": { "default_dispatch_visibility": "open" },
 "after": { "default_dispatch_visibility": "private" },
 "changes": ["default_dispatch_visibility: open → private"],
 "request_id": "req_01HX9P"
}
```

### Example 3: Failed login (security)
```json
{
 "audit_id": "aud_01HX7K2O",
 "timestamp": "2026-05-27T09:15:22.789Z",
 "actor_type": "user",
 "actor_id": "usr_partner_finance_002",
 "actor_name": "(login attempt)",
 "actor_ip": "203.205.x.x",
 "action": "auth.user.login.failed",
 "outcome": "failure",
 "resource_type": "user",
 "resource_id": "usr_partner_finance_002",
 "notes": "Wrong password, attempt 3/5"
}
```

### Example 4: System trigger payout
```json
{
 "audit_id": "aud_01HX7K2P",
 "timestamp": "2026-05-27T23:59:59.000Z",
 "actor_type": "system",
 "actor_id": "cronjob:payout-monthly",
 "actor_name": "system",
 "actor_ip": "10.0.x.x",
 "action": "finance.payout.created",
 "outcome": "success",
 "resource_type": "payout",
 "resource_id": "pyt_01HX7K2P",
 "partner_id": "ptn_hung_acservice",
 "after": {
 "amount": 4500000,
 "period_start": "2026-05-01",
 "period_end": "2026-05-31",
 "orders_count": 23
 }
}
```

## 7. Implementation

### 7.1 Middleware approach (Go)

```go
package audit

type Event struct {
 AuditID string
 Timestamp time.Time
 ActorType string
 ActorID string
 ActorRole string
 ActorIP string
 Action string
 Outcome string
 ResourceType string
 ResourceID string
 Before interface{}
 After interface{}
 RequestID string
 Notes string
}

type Logger interface {
 Log(ctx context.Context, e Event) error
}

// Usage in service layer
func (s *AgentService) ApproveKYC(ctx context.Context, agentID string) error {
 agent, err := s.repo.Get(ctx, agentID)
 if err != nil { return err }
 
 before := agent.Snapshot
 
 agent.KYCLevel = "full"
 agent.VerifiedBy = auth.UserIDFromCtx(ctx)
 
 if err := s.repo.Save(ctx, agent); err != nil {
 return err
 }
 
 s.audit.Log(ctx, audit.Event{
 Action: "agent.agent.kyc.approved",
 Outcome: "success",
 ResourceType: "agent",
 ResourceID: agentID,
 Before: before,
 After: agent.Snapshot,
 })
 
 return nil
}
```

### 7.2 Common context extraction

Audit logger extract from ctx:
- `actor_type`, `actor_id`, `actor_role` từ JWT claims
- `actor_ip` từ X-Forwarded-For header
- `request_id` từ X-Request-ID header
- `session_id` từ JWT jti

### 7.3 Async write
- Use buffered channel + worker pool to write to MongoDB
- If write fails → fallback to local file → batch ship later
- Audit log loss is HIGH severity (compliance impact)

## 8. Storage & retention

### 8.1 MongoDB config
- Database: `smp_audit`
- Collection: `audit_log`
- Sharding: by `timestamp` (range)
- Compression: zstd
- WiredTiger storage

### 8.2 Indexes
```javascript
db.audit_log.createIndex({ audit_id: 1 }, { unique: true })
db.audit_log.createIndex({ timestamp: -1 })
db.audit_log.createIndex({ actor_id: 1, timestamp: -1 })
db.audit_log.createIndex({ resource_type: 1, resource_id: 1, timestamp: -1 })
db.audit_log.createIndex({ action: 1, timestamp: -1 })
db.audit_log.createIndex({ partner_id: 1, timestamp: -1 })
```

### 8.3 Retention policy
| Age | Storage |
|---|---|
| 0-90 days | Hot (MongoDB primary) |
| 90 days - 2 years | Warm (MongoDB secondary, slower disk) |
| 2 years - 7 years | Cold (archived to S3 Glacier, gzip JSONL) |
| > 7 years | Delete (after legal review) |

### 8.4 Archive process
- Monthly cron job
- Export logs older than 90 days → JSONL files
- Upload to S3 with lifecycle policy
- Delete from hot storage AFTER verify successful upload

## 9. Query API

Internal API cho ops, security investigation.

### 9.1 Endpoints

`GET /api/v1/audit/logs`

Query params:
- `from` ISO8601 (required, max 90 days range without archive search)
- `to` ISO8601 (required)
- `actor_id`
- `resource_type`
- `resource_id`
- `action` (or pattern `agent.*`)
- `outcome` (`success` / `failure`)
- `partner_id`
- `cursor`, `limit` (max 100)

Response: paginated list.

### 9.2 Access control

| Role | Access |
|---|---|
| `ops_admin` (with `audit.read` scope) | All logs |
| `partner_owner` | Only logs where `partner_id = own` |
| `security_team` | All logs + sensitive |
| Others | DENIED |

### 9.3 Query audit (meta-audit)

**Important**: Every audit log query itself MUST audit (self-referential):
```text
data.audit_log.queried
```

This prevents abuse where someone queries audit history without trace.

## 10. Tamper resistance

### 10.1 Append-only
- MongoDB collection NEVER updated/deleted (application level)
- No DELETE permission for app users
- Only `audit_archiver` system user has DELETE (after archive)

### 10.2 Hash chaining (v3.5+ REQUIRED · upgraded from optional)

> 🔄 **Update từ Tech Lead review**: Hash chain **MUST** for compliance audit (PDPL Article 16). Consistent với `journal_entries` hash chain ở [Doc 02 section 7.6.2](../02-database/02-database-schema.md).

Each audit entry includes hash of previous entry → tamper detection:

```javascript
{
 audit_id: "aud_X",
 prev_hash: "sha256:abc...", // hash của entry trước
 row_hash: "sha256:def..." // sha256(prev_hash || canonical_payload)
}
```

**Implementation**:
- Canonical payload = JSON.stringify với keys sorted alphabetically (deterministic)
- `prev_hash` của entry đầu tiên trong genesis batch = 64 chars `0`
- Daily cron job verify chain integrity từ genesis → latest
- Broken chain → SEV-1 incident · page Security + CTO immediately
- Audit `audit.chain_verification_failed` event với details + diff

**Reconcile schedule** (BR-RECON-006 trong Doc 15):
- Daily 04:30 UTC: verify hash chain integrity
- Output report: số entries verified, chain status (intact/broken at position N)
- Retention: chain verification logs giữ permanent (cùng audit log)

### 10.3 Off-site backup
- Daily snapshot to separate S3 bucket (different region + account)
- Versioning enabled
- MFA delete required

## 11. PII handling

Some audit entries contain PII (vd customer name in `actor_name`).

Rules:
- **DO log**: structured PII fields (name, email, phone) in known locations
- **DO NOT log**: free-text user input (could contain unstructured PII)
- **DO mask**: in API responses to non-privileged viewers
- **DO encrypt**: at-rest with MongoDB Encryption at Rest

When customer requests data deletion (PDPL right to erasure):
- DO NOT delete audit entries
- DO anonymize PII fields: `actor_name` → "deleted_user_XXX"
- Keep audit_id, action, timestamp (operational integrity)

## 11.5 DSR (Data Subject Request) Lifecycle Events 

> Theo [Doc 21 · PDPL Data Policy](./13-data-classification-encryption.md cross-link → 10-legal/21-pdpl-data-policy.md), KH/thợ có quyền truy cập, xóa, chỉnh sửa dữ liệu cá nhân. Mọi bước trong lifecycle MUST có audit trail.

### 11.5.1 DSR event types

| action | Trigger | Severity |
|---|---|---|
| `dsr.request_received` | User submit DSR via app / API / email | Info |
| `dsr.identity_verified` | Identity check passed (OTP / KYC) | Info |
| `dsr.access_export_started` | Backend bắt đầu generate ZIP export | Notice |
| `dsr.access_export_delivered` | Export ZIP sent tới user (email link / app download) | Notice |
| `dsr.deletion_marked` | User account marked `pending_deletion`, 30-day grace started | **Warning** |
| `dsr.deletion_cancelled` | User cancel deletion trong 30 days | Notice |
| `dsr.deletion_executed` | Grace expired → actual anonymization | **Critical** |
| `dsr.correction_applied` | User corrected wrong PII data | Info |
| `dsr.consent_withdrawn` | User rút consent cho specific purpose (marketing, etc.) | Notice |
| `dsr.consent_granted` | User cấp lại consent | Info |
| `dsr.objection_raised` | User phản đối xử lý cụ thể | **Warning** |
| `dsr.legal_hold_applied` | Admin override deletion vì legal/compliance reason | **Critical** |

### 11.5.2 Required fields cho DSR events

```javascript
{
 "audit_id": "aud_01HX8K...",
 "action": "dsr.access_export_started",
 "category": "compliance",
 "severity": "notice",
 
 // Subject (whose data)
 "subject_type": "customer" | "agent" | "partner",
 "subject_id": "cust_12345",
 "subject_country": "VN", // ← determines applicable law
 
 // Requestor (often same as subject, but admin can act on behalf)
 "actor_type": "self" | "admin_proxy",
 "actor_id": "cust_12345",
 
 // Request details
 "request_id": "dsr_2026_05_28_001",
 "request_type": "access" | "deletion" | "correction" | "consent_update" | "objection",
 "request_channel": "app" | "api" | "email" | "phone" | "ops_form",
 
 // Lifecycle tracking
 "sla_deadline_utc": "2026-06-04T03:00:00Z", // 7 days for access per PDPL
 "previous_step_audit_id": "aud_01HX8J...", // chain DSR events together
 
 // Context
 "purposes_affected": ["marketing", "analytics"], // for consent_withdrawn
 "fields_exported": ["name", "phone", "address"], // for access_export
 "records_count": 1247, // for export delivery
 
 // Compliance evidence
 "legal_basis": "user_request_PDPL_article_17",
 "legal_hold_reason": null, // chỉ populated cho legal_hold_applied
 
 "occurred_at_utc": "2026-05-28T03:00:00Z",
 "ip": "203.0.113.45",
 "trace_id": "..."
}
```

### 11.5.3 DSR SLA tracking

Mỗi `dsr.request_received` MUST trigger SLA tracker:

| Request type | SLA (PDPL VN) | Audit alert nếu vượt |
|---|---|---|
| Access (export) | 7 days | `dsr.sla_at_risk` warning at day 5, `dsr.sla_breach` critical at day 7 |
| Deletion | 30 days (incl grace period) | `dsr.sla_breach` critical at day 30 |
| Correction | 14 days | `dsr.sla_breach` at day 14 |
| Consent withdrawal | Immediate (< 24h) | `dsr.sla_breach` at day 1 |
| Objection | 14 days | `dsr.sla_breach` at day 14 |

(Số ngày cụ thể chờ luật sư VN xác nhận theo NĐ 356/2025 — xem [Doc 21](../10-legal/21-pdpl-data-policy.md))

### 11.5.4 DSR audit chain query

Mỗi DSR có 1 `request_id` xuyên suốt lifecycle. Query API cần support:

```http
GET /audit/dsr/{request_id}/timeline
```

Returns toàn bộ events theo thứ tự chronological, dùng `previous_step_audit_id` để build chain.

### 11.5.5 Legal hold mechanism

Khi có legal hold (dispute mở, fraud investigation, court order...), DSR deletion sẽ **bị block** với audit:

```json
{
 "action": "dsr.legal_hold_applied",
 "severity": "critical",
 "legal_hold_reason": "dispute_DSP-2026-001-pending_resolution",
 "subject_id": "cust_12345",
 "actor_id": "ops_admin_01",
 "estimated_release_date": "2026-08-15" // when hold can be lifted
}
```

Khi legal hold lifted, ghi `dsr.legal_hold_released` + auto-trigger queued deletion (nếu user đã request deletion trước đó).

## 12. Alerting on suspicious patterns

Real-time alerts based on audit log:

| Pattern | Alert |
|---|---|
| 5 failed logins in 5 min (same user) | Slack security channel |
| Login from new country (per user) | Email user + security log |
| Bulk export (> 1000 rows in 1 min) | Security review queue |
| Audit log query by non-security user | Security log |
| Permission grant outside business hours | Security review |
| Multiple sensitive PII views by 1 user | Security review |
| **DSR deletion attempted while legal hold active** | **Page Compliance + Security immediately** |
| **DSR SLA breach (per type)** | **Compliance dashboard + email DPO** |
| **PII unmask > 50 records in 10 min by 1 user** | **Security review queue + Slack** |
| **Cross-country PII access (PIPL/GDPR risk)** | **Block by default + Compliance review** |

Implementation: stream audit_log to Kafka → flink/streaming processor → alert.

## 13. Compliance mapping

### PDPL Việt Nam (Luật 91/2025/QH15 + NĐ 356/2025 — hiệu lực 01/01/2026)

| PDPL Article | Audit coverage |
|---|---|
| Article 3 (consent record) | `dsr.consent_granted` + `dsr.consent_withdrawn` với timestamp + IP + purposes |
| Article 4 (data minimization) | Annual audit log review · count `pii.search` events per role · flag over-access |
| Article 8 (right to access) | `dsr.request_received` (type=access) → SLA 7 days tracked qua `dsr.access_export_*` events |
| Article 9 (right to correction) | `dsr.correction_applied` |
| Article 10 (right to erasure) | `dsr.deletion_marked` → 30-day grace → `dsr.deletion_executed` |
| Article 11 (right to objection) | `dsr.objection_raised` + actions taken |
| Article 16 (security measures) | Audit log itself + tamper resistance (section 10) is part of "appropriate measures" |
| Article 22 (cross-border transfer) | `pii.access.cross_country` audit + CAC assessment record (China-specific) |

Chế tài: **tới 5% doanh thu năm trước** cho vi phạm cross-border transfer (PDPL). Audit log MUST capture mọi cross-border PII access.

### Sensitive PII categories (PDPL + NĐ 356)

Per NĐ 356, các categories sau cần extra audit attention:

| Category | Examples in SMP | Extra audit requirement |
|---|---|---|
| **Sinh trắc học** | Face ID, fingerprint (nếu dùng) | Mỗi lần access cần consent re-confirmation event |
| **Sức khỏe** | (chưa applicable cho SMP) | N/A |
| **Vị trí** | GPS live tracking SOS/share-trip | Audit per session, không log per ping |
| **Tài chính** | Bank account, transaction history | High severity audit cho mọi unmask |
| **Liên kết tổ chức** | (nếu thợ thuộc partner) | Standard audit |

### Tax & Accounting (VN)

- **NĐ 123/2020/NĐ-CP** + **NĐ 70/2025/NĐ-CP** (hiệu lực 01/06/2025): financial transactions retain **10 years**
- **Thông tư 32/2025/TT-BTC**: e-invoice retention 10 years
- **Chế tài NĐ 310/2025**: tới 200M VND cho vi phạm hóa đơn
- All `financial.*` audit entries retained **10 years** (override 7-year default)

### Multi-jurisdiction posture 

Khi expand sang country khác, audit log structure giữ nguyên nhưng compliance mapping cần extend:

| Country | Primary law | DSR SLA | Cross-border |
|---|---|---|---|
| 🇻🇳 VN | PDPL 91/2025 | 7d access, 30d deletion | Default — same region |
| 🇨🇳 CN | PIPL | Vary by request type | **Mandatory CAC assessment** |
| 🇪🇺 EU | GDPR | 30d default, 90d extension | SCC required cho non-EU transfer |
| 🇺🇸 US | CPRA (CA), state-by-state | 45 days | Notice-based |
| 🇸🇬 SG | PDPA SG | 30 days | Approved jurisdictions only |

Xem [Doc 13 · Multi-jurisdiction compliance](./13-data-classification-encryption.md) cho deployment implications.

## 14. Self-monitoring

Audit log health metrics (Prometheus):
- `audit_log_write_total` counter
- `audit_log_write_errors_total` counter
- `audit_log_write_duration_seconds` histogram
- `audit_log_queue_depth` gauge
- `audit_log_archive_lag_days` gauge

Alert if:
- Write error rate > 0.1% over 5 min
- Queue depth > 10,000 for 5 min
- No new entries in 5 min (system might be down)
