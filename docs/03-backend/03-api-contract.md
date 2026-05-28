# SMP API Contract · OpenAPI 3.0

**Version**: v1 · **Base URL**: `https://api.smp.vn/api/v1` · **Audience**: Backend, Frontend, Integration

---

## 1. Convention

- **Style**: REST + JSON
- **Auth**: JWT Bearer token in `Authorization: Bearer <token>` header
- **Versioning**: URL path · `/api/v1/`, `/api/v2/` khi breaking change
- **Pagination**: cursor-based · `?cursor=<opaque>&limit=20` (max 100)
- **Filtering**: query params · `?status=active&city=HCMC`
- **Sorting**: `?sort=created_at:desc`
- **Error format**: RFC 7807 Problem Details

```json
{
  "type": "/errors/insufficient-balance",
  "title": "Insufficient wallet balance",
  "status": 422,
  "detail": "Partner wallet balance is 100,000 VND, required 750,000 VND",
  "instance": "/api/v1/orders",
  "request_id": "req_01HX7K2M"
}
```

- **ID format**: ULID (26 chars) · vd `ord_01HX7K2M9QWRA1B2`
- **Timestamps**: ISO 8601 UTC · `2026-05-27T08:14:32.123Z`
- **Currency**: VND, no decimal · `750000`

## 2. Standard headers

**Request:**
```text
Authorization: Bearer <jwt>
X-Request-Id: <ulid>
X-Client: mobile-customer/3.3.0 | mobile-tech/3.3.0 | admin-web/3.3.0
Accept-Language: vi-VN
```

**Response:**
```text
X-Request-Id: <echoed>
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 58
X-RateLimit-Reset: 1716800000
```

## 3. Auth endpoints

### POST /auth/login
Login bằng phone + OTP (cho customer/agent), email + password (cho ops/partner).

Request:
```json
{ "method": "phone_otp", "phone": "+84907842931", "otp": "123456" }
```

Response 200:
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiI...",
  "refresh_token": "eyJ...",
  "expires_in": 28800,
  "user": {
    "user_id": "usr_01HX",
    "role": "customer",
    "name": "Nguyễn Văn A"
  }
}
```

### POST /auth/refresh
Refresh access token. Body: `{ "refresh_token": "..." }` · Response như login.

### POST /auth/logout
Invalidate token. Header Authorization required.

## 4. Orders endpoints

### POST /orders
Tạo đơn mới. Required scope: `orders.create`.

Request:
```json
{
  "service_id": "svc_ac_repair",
  "address": {
    "line": "15 Lê Lợi",
    "ward": "Bến Nghé",
    "district": "Q.1",
    "city": "HCMC",
    "lat": 10.7769,
    "lng": 106.7009
  },
  "scheduled_at": "2026-05-27T14:00:00+07:00",
  "notes": "Máy không lạnh, dàn lạnh chảy nước",
  "voucher_code": "HOTSUMMER20",
  
  // For partner-created orders
  "source": "partner_customer",
  "partner_id": "ptn_hung_acservice",
  "partner_order_ref": "VIN-2026-04821",
  "dispatch_visibility": "private",
  "end_customer_id": "cus_01HX5K"
}
```

Response 201:
```json
{
  "order": {
    "order_id": "ord_01HX7K2M",
    "current_stage": "01_created",
    "total_estimate": 750000,
    "scheduled_at": "2026-05-27T14:00:00+07:00",
    ...
  }
}
```

Errors:
- `400` validation_failed
- `402` insufficient_balance (partner wallet)
- `422` no_agent_available_in_zone

### GET /orders
List orders với filter & pagination.

Query: `?customer_id=cus_X | partner_id=ptn_X | agent_id=agt_X | status=in_progress | from=2026-05-01 | to=2026-05-31 | cursor=<opaque> | limit=20`

Response 200:
```json
{
  "items": [ { "order_id": "...", ... }, ... ],
  "next_cursor": "eyJjcmVhdGVk...",
  "has_more": true
}
```

### GET /orders/{order_id}
Get order detail. Includes order_steps, materials, stage_log.

### POST /orders/{order_id}/cancel
Body: `{ "reason": "customer_request", "notes": "..." }`

### POST /orders/{order_id}/transitions/{stage}
Transit stage. Vd: `POST /orders/ord_X/transitions/05_surveyed`

Auth check: agent gắn vào order mới được transit.

## 5. Catalog endpoints (public · cached)

### GET /catalog/services
List services. Cache 5 phút.

### GET /catalog/steps
List steps. Cache 5 phút.

### GET /catalog/steps/{step_code}/boms
List BOMs cho step. Trả về kèm material_types và variants.

### GET /catalog/material-types
List material types.

### GET /catalog/material-variants?type_id=mtyp_X
List variants của 1 type. Include stock info từ wms (real-time):

Response 200:
```json
{
  "items": [
    {
      "variant_code": "mvar_capacitor_sanyo_35uf",
      "brand": "Sanyo",
      "model": "CBB60-35",
      "sell_price": 80000,
      "warranty_months": 6,
      "stock": {
        "personal": 3,
        "shared_hcmc": 12,
        "available": true,
        "checked_at": "2026-05-27T08:14:32Z"
      }
    }
  ]
}
```

## 6. Agents endpoints

### GET /agents
Query: `?city=HCMC&district=Q.1&skill=ac-mechanic&min_level=3&is_online=true`

### GET /agents/{agent_id}

### PATCH /agents/{agent_id}
Body: partial update. Vd toggle online: `{ "is_online": true }`

### POST /agents/{agent_id}/skills
Body: `{ "skill_code": "ac-mechanic", "level": 3, "certificate_url": "..." }`

## 7. Partners endpoints (v3.3)

### GET /partners
List partners. Scope `partners.read`.

### POST /partners
Create partner. Scope `partners.create`.

```json
{
  "partner_type": "business",
  "roles": ["customer", "supplier"],
  "business_name": "Hùng AC Service",
  "rep_name": "Nguyễn Hùng",
  "rep_phone": "+84932841...",
  "customer_config": {
    "payment_mode": "prepaid_wallet",
    "journey_mode": "full_10steps",
    "default_dispatch_visibility": "private"
  },
  "supplier_config": {
    "model": "commission_only",
    "commission_percent": 6
  }
}
```

### GET /partners/{partner_id}

### PATCH /partners/{partner_id}

### POST /partners/{partner_id}/wallet/topup
Body: `{ "amount": 5000000, "method": "bank_transfer", "ref": "..." }`

### GET /partners/{partner_id}/orders
Scope partner self-service.

### GET /partners/{partner_id}/agents
List agents thuộc partner.

### POST /partners/{partner_id}/users
Tạo partner_admin_user mới. Body: `{ "name", "email", "role", "scopes" }`

## 8. Dispatch endpoints

### POST /dispatch/orders/{order_id}/start
Trigger dispatch flow. Returns 202 Accepted.

### POST /dispatch/orders/{order_id}/accept
Agent accept order. Body: `{ "agent_id": "agt_T4K9" }`

### GET /dispatch/orders/{order_id}/candidates
List candidate agents currently in queue.

### WebSocket /ws/dispatch
Realtime updates. Connect với JWT in query: `wss://api.smp.vn/ws/dispatch?token=<jwt>`

Events stream:
```json
{ "type": "order.invited", "order_id": "...", "ttl_seconds": 60 }
{ "type": "order.assigned", "order_id": "...", "agent_id": "..." }
{ "type": "order.expired", "order_id": "..." }
```

## 9. Finance endpoints

### GET /finance/wallet-transactions?partner_id=X

### GET /finance/invoices?partner_id=X

### GET /finance/payouts?partner_id=X

### POST /finance/payouts/{payout_id}/mark-paid (ops only)

## 10. Quality endpoints

### POST /orders/{order_id}/steps/{step_id}/photos
Upload photo proof. multipart/form-data.

### GET /quality/disputes

### POST /quality/disputes/{dispute_id}/resolve

### GET /quality/material-verifications
Pending material verifies (free-form + variance).

### POST /quality/material-verifications/{id}/approve
### POST /quality/material-verifications/{id}/reject

## 11. Integration endpoints (internal · webhook receivers)

### POST /integrations/inside/payment-webhook
Receive payment events from inside. HMAC signature validation.

### POST /integrations/wms/stock-alert
Receive stock low alerts from wms.

## 11.5 PII endpoints (v4.0)

### Response format · Masked vs Unmasked

Mọi response chứa PII fields đều có cấu trúc nhất quán. Caller biết field nào masked qua **header**.

#### Default response (caller không có scope unmask)

```http
GET /api/v1/customers/cus_01HX2M
Authorization: Bearer <jwt-no-unmask-scope>

HTTP/1.1 200 OK
Content-Type: application/json
X-Masked-Fields: phone,email,cccd,bank_account,address_full
X-Unmask-Available: phone,email,address_full
X-Unmask-Endpoint: /api/v1/pii/unmask

{
  "id": "cus_01HX2M",
  "name": "Nguyễn V. A.",
  "phone": "0912****890",
  "email": "ngu***@gmail.com",
  "cccd": "0012****2345",
  "bank_account": "1234*****0123",
  "address_full": "*** Q.1, TP.HCM",
  "country_code": "VN",
  "created_at_utc": "2026-05-20T10:30:00Z"
}
```

**Headers giải thích**:
- `X-Masked-Fields`: list fields bị mask trong response này
- `X-Unmask-Available`: list fields caller CÓ THỂ unmask (scope cho phép) nhưng chưa unlock — frontend hiển thị nút "Reveal" cho user
- `X-Unmask-Endpoint`: endpoint để request unmask

#### Caller có scope (auto-unmask)

```http
GET /api/v1/customers/cus_01HX2M
Authorization: Bearer <jwt-with-pii.unmask.phone-scope>

HTTP/1.1 200 OK
X-Masked-Fields: email,cccd,bank_account,address_full
X-Unmask-Available: email,address_full
X-Unmask-Audit-Id: aud_01HXABCD

{
  "id": "cus_01HX2M",
  "name": "Nguyễn Văn An",
  "phone": "0912345890",         ← unmasked
  "email": "ngu***@gmail.com",   ← still masked
  ...
}
```

**`X-Unmask-Audit-Id`** = audit log ID, returned để debug/trace nếu cần.

### POST /pii/unmask

One-time unmask cho sensitive fields (CCCD, bank account, ID photo).

#### Request

```http
POST /api/v1/pii/unmask
Authorization: Bearer <jwt>
Authorization-Fresh: <mfa_proof_token>  ← required cho sensitive fields
Content-Type: application/json

{
  "entity_type": "customer",
  "entity_id": "cus_01HX2M",
  "fields": ["bank_account", "cccd"],
  "reason": "Customer requested refund, verifying bank for transfer",
  "ticket_id": "ZD-12345"
}
```

#### Response · Success

```http
HTTP/1.1 200 OK

{
  "unmask_session_id": "ums_01HXABCD",
  "expires_at_utc": "2026-05-28T11:30:00Z",
  "audit_id": "aud_01HXABCE",
  "data": {
    "bank_account": "1234567890123",
    "cccd": "001202012345"
  }
}
```

Sau khi nhận session_id, caller có thể call lại endpoint khác trong session timeout (30 phút) với header `X-Unmask-Session: ums_01HXABCD` để get unmasked data của entity đó without re-MFA.

#### Response · Errors

| HTTP | Type | Reason |
|---|---|---|
| 401 | unauthorized | JWT invalid hoặc expired |
| 403 | insufficient_scope | Caller không có `pii.unmask.<field>` scope |
| 403 | mfa_required | `Authorization-Fresh` header missing/expired (required cho CCCD, bank, ID photos) |
| 403 | cross_country_restricted | Caller country khác data subject country, không cùng sovereignty_region |
| 422 | reason_too_short | `reason` text quá ngắn (min 20 chars cho audit purpose) |
| 429 | rate_limited | Vượt limit unmask requests (5/giờ cho sensitive fields) |

### GET /pii/audit-trail

Cho compliance officer xem ai đã access PII của 1 data subject.

```http
GET /api/v1/pii/audit-trail?entity_type=customer&entity_id=cus_01HX2M
Authorization: Bearer <jwt-with-compliance.audit-scope>

HTTP/1.1 200 OK

{
  "entity_id": "cus_01HX2M",
  "total_accesses": 23,
  "period": {"from": "2026-01-01", "to": "2026-05-28"},
  "items": [
    {
      "audit_id": "aud_01H...",
      "timestamp_utc": "2026-05-27T08:15:32Z",
      "actor_id": "usr_csv1",
      "actor_name": "Trần Thị B (CSKH)",
      "actor_country": "VN",
      "fields_accessed": ["phone"],
      "reason": null,
      "ticket_id": "ZD-12340"
    },
    {
      "audit_id": "aud_01H...",
      "timestamp_utc": "2026-05-26T14:22:10Z",
      "actor_id": "usr_finance3",
      "actor_name": "Lê Văn C (Finance)",
      "actor_country": "VN",
      "fields_accessed": ["bank_account", "cccd"],
      "reason": "Refund processing for ticket ZD-12300",
      "ticket_id": "ZD-12300"
    }
  ]
}
```

Endpoint này phục vụ Data Subject Access Request (DSAR) compliance — user có quyền xem "ai đã access data của tôi".

## 12. Rate limits

| Endpoint group | Limit | Window |
|---|---|---|
| `/auth/*` | 10 req | 1 min per IP |
| `/orders POST` | 5 req | 1 min per user |
| `/orders GET` | 60 req | 1 min per user |
| Read endpoints | 120 req | 1 min per user |
| Webhooks | 1000 req | 1 min per source |

## 13. Error codes catalog

| HTTP | Type | Title |
|---|---|---|
| 400 | validation_failed | Request validation failed |
| 401 | unauthorized | Token invalid or expired |
| 403 | forbidden | Insufficient scope |
| 404 | not_found | Resource not found |
| 409 | conflict | Resource conflict (vd order đã assigned) |
| 422 | business_rule_violation | (xem detail) |
| 422 | insufficient_balance | Partner wallet insufficient |
| 422 | no_agent_available | Không có thợ available trong zone |
| 422 | kyc_required | KYC chưa hoàn thành |
| 429 | rate_limit_exceeded | Too many requests |
| 500 | internal_error | Server error |
| 503 | service_unavailable | Downstream service down |

## 14. Webhooks SMP gửi ra ngoài

| Event | Endpoint partner | Auth |
|---|---|---|
| `order.completed` | partner.webhook_url | HMAC SHA256 |
| `order.cancelled` | partner.webhook_url | HMAC SHA256 |
| `payout.created` | partner.webhook_url | HMAC SHA256 |

Retry: exponential backoff 3 lần (1s, 10s, 60s). DLQ sau khi fail.

## 15. OpenAPI spec file

Full spec dùng Swagger UI:
- File location: `api/openapi.yaml` (mỗi service repo có file riêng)
- Tooling: oapi-codegen sinh Go server stub + client SDK
- Documentation: tự host `https://api.smp.vn/docs` (Swagger UI)
- Validation: spectral lint trong CI

Sample stub:
```yaml
openapi: 3.0.3
info:
  title: SMP Order Service
  version: 1.0.0
servers:
  - url: https://api.smp.vn/api/v1
security:
  - bearerAuth: []
paths:
  /orders:
    post:
      operationId: createOrder
      ...
```

## 16. Testing

| Type | Tool | Coverage target |
|---|---|---|
| Contract test | pact + dredd | 100% endpoints |
| Integration test | Go testing + testcontainers | Critical paths |
| Load test | k6 | p95 < 500ms for read, < 2s for write |
