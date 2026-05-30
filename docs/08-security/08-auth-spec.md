# SMP Authentication & Authorization
**Standards**: OAuth 2.1 + JWT (RS256) + RBAC · **Audience**: Backend, Frontend, Security

---

## 1. Identity sources

Có 4 loại user · từng nguồn:

| User type | Auth method | Identity provider | Token type |
|---|---|---|---|
| End Customer | Phone + OTP | inside (SSO) | JWT 8h + refresh 30d |
| Technician | Phone + OTP | SMP self | JWT 8h + refresh 30d |
| Ops (internal) | Email + password + 2FA | SMP self | JWT 4h + refresh 7d |
| Partner admin | Email + password + 2FA | SMP self | JWT 4h + refresh 7d |

## 2. JWT structure

### 2.1 Header
```json
{ "alg": "RS256", "typ": "JWT", "kid": "smp-2026-key1" }
```

### 2.2 Payload (claims)
```json
{
 "iss": "smp.vn",
 "sub": "usr_01HX7K2M",
 "aud": ["api.smp.vn"],
 "iat": 1716800000,
 "exp": 1716828800,
 "jti": "tok_01HX7K2N",
 
 "role": "customer | technician | ops_admin | partner_owner | partner_manager | ...",
 "scopes": ["orders.read", "orders.create", "agents.read"],
 
 "user": {
 "id": "usr_01HX7K2M",
 "name": "Nguyễn Hùng",
 "phone_masked": "+84••••2841"
 },
 
 "ctx": {
 "partner_id": "ptn_hung_acservice", 
 "agent_id": null, 
 "customer_id": "cus_01HX5K" 
 }
}
```

### 2.3 Signing keys
- Algorithm: RS256 (asymmetric)
- Key rotation: annual or on compromise
- Key ID (`kid`) header used to support multiple active keys during rotation
- Public key endpoint: `GET https://api.smp.vn/.well-known/jwks.json`

```bash
# Generate keypair
openssl genrsa -out smp-jwt-private.pem 2048
openssl rsa -in smp-jwt-private.pem -pubout -out smp-jwt-public.pem
```

Private key stored in Vault, public key exposed via JWKS endpoint for services to fetch.

## 3. OAuth flows

### 3.1 Customer login (Phone OTP)

```text
Customer mobile app
 │ POST /auth/otp/request {"phone": "+84..."}
 ▼
auth-svc → SMS gateway → user receives OTP
 │ POST /auth/login {"phone": "+84...", "otp": "123456"}
 ▼
auth-svc verify OTP → issue JWT + refresh token
 │
 ▼
App store tokens in secure storage (Keychain iOS, Keystore Android)
```

### 3.2 Ops/Partner login (Email + Password + 2FA)

```text
Browser
 │ POST /auth/login {"email":"hung@hungac.vn", "password":"..."}
 ▼
auth-svc verify password (bcrypt cost 12)
 │ → if MFA enabled: respond {"mfa_required": true, "mfa_token": "..."}
 ▼
Browser prompt TOTP
 │ POST /auth/login/mfa {"mfa_token":"...", "totp_code":"123456"}
 ▼
auth-svc verify TOTP → issue JWT
```

### 3.3 Refresh token rotation

```text
Mobile app (token expired)
 │ POST /auth/refresh {"refresh_token": "..."}
 ▼
auth-svc:
 1. Validate refresh token + check not in revocation list
 2. Issue NEW access + refresh token
 3. Invalidate OLD refresh token (one-time use)
 4. Detect reuse → security alert (compromise indicator)
```

## 4. Roles & scopes catalog

### 4.1 Customer scopes
```text
orders.read.own
orders.create.own
orders.cancel.own
ratings.create.own
assets.read.own
contracts.read.own
voucher.read
profile.update.own
```

### 4.2 Technician scopes
```text
orders.read.assigned
orders.accept
orders.transit_stage
orders.upload_photo
orders.report_material
pool.read
profile.update.own
earnings.read.own
```

### 4.3 Ops admin scopes (default · super-admin có all)
```text
# Operations
orders.read.all
orders.cancel.any
dispatch.manage

# Catalog
catalog.read
catalog.manage

# Agents
agents.read.all
agents.kyc.approve
agents.suspend

# Partners
partners.read.all
partners.create
partners.manage
partners.kyc.approve

# Finance
finance.read.all
finance.payout.approve

# Quality
quality.read.all
quality.material_verify

# Integration
integration.read
integration.manage

# System
system.health.read
system.settings.manage
audit.read
```

### 4.4 Partner admin scopes (RBAC v3.3)

Auto-scoped to `current_user.ctx.partner_id`:

| Role | Scopes |
|---|---|
| `partner_owner` | `all_within_partner` (super-admin của partner) |
| `partner_manager` | `orders.view.own_partner`, `orders.create.own_partner`, `orders.cancel.own_partner`, `agents.view.own_partner`, `agents.toggle.own_partner`, `finance.view.own_partner` |
| `partner_finance` | `finance.view.own_partner`, `finance.export.own_partner`, `invoices.view.own_partner`, `invoices.download.own_partner`, `wallet.topup.own_partner` |
| `partner_dispatcher` | `orders.view.own_partner`, `orders.create.own_partner`, `orders.dispatch.own_partner`, `agents.view.own_partner` |

### 4.5 PII unmask scopes 

> **Default-deny** pattern: mọi PII field default trả về **masked** (e.g., `0912****890`). Để xem full, caller phải có scope tương ứng. Mỗi lần unmask = 1 audit log entry.

#### Catalog scopes

| Scope | Description | Risk | Granted to |
|---|---|---|---|
| `pii.unmask.phone` | Số điện thoại customer/agent | Medium | Ops Admin, Finance, Partner Owner (own partner only) |
| `pii.unmask.email` | Email customer/agent | Low | Ops Admin, Marketing, Partner Manager (own partner) |
| `pii.unmask.address_full` | Địa chỉ chi tiết (line + ward) | Medium | Ops Dispatch, Agent assigned to order, Customer Support |
| `pii.unmask.cccd` | Số CCCD | **High** | Finance Admin (KYC verify), Super Admin, **Compliance** |
| `pii.unmask.bank_account` | Số tài khoản ngân hàng | **High** | Finance Admin, Super Admin |
| `pii.unmask.tax_code` | MST partner/agent | Medium | Finance Admin, Partner Owner (own) |
| `pii.unmask.dob` | Ngày sinh | Low | Ops Admin, KYC team |
| `pii.unmask.id_card_photo` | Ảnh CCCD (front/back) | **Critical** | Compliance only, **MFA fresh required** |
| `pii.unmask.bulk` | Export bulk PII | **Critical** | Super Admin, Compliance with **MFA + reason text** |

#### Role-to-PII-scope mapping

| Role | Default scopes | What they see by default |
|---|---|---|
| **Customer Support L1** | NONE | `phone: 0912****890`, `email: ng***@gmail.com` (suficient cho lookup + xác nhận) |
| **Customer Support L2** | `pii.unmask.phone`, `pii.unmask.address_full` | Có thể call back, send tech đến |
| **Ops Admin** | `pii.unmask.phone`, `pii.unmask.email`, `pii.unmask.address_full`, `pii.unmask.dob` | Full operational data |
| **Finance Admin** | All above + `pii.unmask.cccd`, `pii.unmask.bank_account`, `pii.unmask.tax_code` | Cho KYC + payout + invoice |
| **Compliance Officer** | All `pii.unmask.*` scopes | Audit + data subject request fulfillment |
| **Super Admin** | All scopes | Emergency access, every action audit-logged |
| **Partner Owner** | `pii.unmask.phone.own_partner`, `pii.unmask.email.own_partner` | Chỉ thợ của partner đó, **không** thấy customer cuối |
| **Agent (technician)** | NONE by default | After assigned → see assigned customer phone temporarily |

#### Special rules

1. **Time-bounded unmask**: Một số scope chỉ valid trong 1 session ngắn (vd 1h sau MFA fresh). Sau đó re-auth.
2. **Field-level audit**: Mỗi lần API trả về unmasked PII, ghi 1 record audit (xem [Doc 12 · Audit Log](./12-audit-log-spec.md#cross-cutting-pii-access-events)).
3. **Bulk export rate limit**: scope `pii.unmask.bulk` giới hạn max 100 records/request + 5 requests/day.
4. **Geographic restriction**: scope `pii.unmask.*` cho data của user country X chỉ caller cùng country/sovereignty_region được dùng (tránh CSKH VN unmask data US — vi phạm CPRA).

#### Unmask request flow

```text
Caller request data
 ▼
API Gateway middleware checks JWT scope
 ▼
 ┌────┴────┐
 ▼ ▼
has scope no scope
 │ │
 ▼ ▼
unmask mask (default)
 │ │
 ▼ ▼
log "PII return masked
access"
 │
 ▼
return full data
```

#### One-time unmask endpoint (sensitive fields)

Cho các field critical (CCCD, bank account, ID photo), even với scope đầy đủ, caller phải call endpoint riêng + cung cấp `reason`:

```http
POST /api/v1/pii/unmask
Authorization: Bearer <jwt>
Authorization-Fresh: <mfa_proof>
Content-Type: application/json

{
 "entity_type": "customer",
 "entity_id": "cus_01HX7K2M",
 "fields": ["bank_account", "cccd"],
 "reason": "Customer requested refund, verifying bank account for transfer",
 "ticket_id": "ZD-12345"
}
```

Response includes audit_id để track:
```json
{
 "unmask_session_id": "ums_01HXABCD",
 "expires_at_utc": "2026-05-28T11:30:00Z",
 "audit_id": "aud_01HXABCE",
 "data": {
 "bank_account": "1234567890",
 "cccd": "001202012345"
 }
}
```

Session valid 30 phút, sau đó field trở lại masked. Reason text được lưu permanent trong audit log.

## 5. Scope enforcement

### 5.1 Middleware (Go)

```go
func RequireScope(scope string) gin.HandlerFunc {
 return func(c *gin.Context) {
 claims := c.MustGet("jwt_claims").(*JWTClaims)
 if !hasScope(claims.Scopes, scope) {
 c.JSON(403, ProblemDetails{
 Type: "/errors/forbidden",
 Title: "Insufficient permissions",
 Status: 403,
 Detail: fmt.Sprintf("Missing scope: %s", scope),
 })
 c.Abort
 return
 }
 c.Next
 }
}

// Usage
router.GET("/api/v1/orders", RequireScope("orders.read"), handler.ListOrders)
router.POST("/api/v1/orders", RequireScope("orders.create"), handler.CreateOrder)
```

### 5.2 Resource-level authorization

Beyond scope check, verify ownership:

```go
func (h *Handler) GetOrder(c *gin.Context) {
 claims := c.MustGet("jwt_claims").(*JWTClaims)
 orderID := c.Param("order_id")
 
 order, err := h.svc.GetOrder(c, orderID)
 if err != nil { ... }
 
 if !h.canAccess(claims, order) {
 c.JSON(403, ProblemDetails{...})
 return
 }
 c.JSON(200, order)
}

func (h *Handler) canAccess(claims *JWTClaims, order *Order) bool {
 switch claims.Role {
 case "customer":
 return order.CustomerID == claims.User.ID
 case "technician":
 return order.SurveyAgentID == claims.Ctx.AgentID || order.ExecutionAgentID == claims.Ctx.AgentID
 case "partner_owner", "partner_manager":
 return order.PartnerID == claims.Ctx.PartnerID
 case "ops_admin":
 return true
 }
 return false
}
```

### 5.3 SQL row-level filtering

For list endpoints, inject filter automatically:

```go
func (r *OrderRepo) List(ctx context.Context, claims *JWTClaims, filter OrderFilter) ([]*Order, error) {
 q := sq.Select("*").From("orders")
 
 switch claims.Role {
 case "customer":
 q = q.Where(sq.Eq{"customer_id": claims.User.ID})
 case "partner_owner", "partner_manager", "partner_dispatcher":
 q = q.Where(sq.Eq{"partner_id": claims.Ctx.PartnerID})
 case "technician":
 q = q.Where(sq.Or{
 sq.Eq{"survey_agent_id": claims.Ctx.AgentID},
 sq.Eq{"execution_agent_id": claims.Ctx.AgentID},
 })
 // ops_admin: no filter
 }
 
 // Apply additional filters...
 return r.query(ctx, q)
}
```

## 6. Password & credential policy

### 6.1 Password requirements (ops + partner)
- Min 12 characters
- Must contain: uppercase, lowercase, number, special char
- Check against breach list (Have I Been Pwned API)
- Forbid common 10000 (top dictionary)
- No reuse of last 5 passwords

### 6.2 Storage
- Hash: bcrypt cost 12
- Never log password in plain
- Never include in error messages

### 6.3 Reset flow
1. User enter email → email link with one-time token (TTL 15 min)
2. Click link → enter new password
3. Token invalidated immediately
4. Invalidate all existing sessions for that user
5. Audit log entry

### 6.4 Account lockout
- 5 failed login attempts in 15 min → lock 30 min
- Notify user via email
- Ops can manually unlock

## 7. 2FA (TOTP)

Required for: ops_admin, partner_owner, partner_finance

Setup:
1. User enable 2FA in settings
2. App show QR code (TOTP secret)
3. User scan in Google Authenticator / Authy
4. Verify with 1 code → enable
5. Show 10 backup codes (one-time use)

Verify:
- TOTP window: ±1 step (30s before/after)
- Backup code: single use, audit log

## 8. Session management

### 8.1 Active sessions

User can view + revoke active sessions in profile:
- Device name, IP, last active
- "Logout from this device"
- "Logout all devices"

Stored in Redis: `session:<token_jti>` → user_id, expires_at

### 8.2 Logout

```text
POST /auth/logout
Header: Authorization: Bearer <token>

→ Add token jti to blacklist (Redis SET with TTL = remaining token lifetime)
→ Invalidate refresh token
→ Audit log
```

### 8.3 Concurrent sessions limit

- Customer: unlimited (multi-device OK)
- Technician: max 2 (phone + tablet)
- Ops: max 3
- Partner: max 5

Over limit → oldest session auto-invalidated.

## 9. API key auth (for service-to-service)

External integrations (inside, wms) use API keys, not JWT:

```text
Authorization: Bearer api_<32-char-token>
```

API keys:
- Stored hashed in DB (bcrypt)
- Created via admin UI, shown ONCE
- Scoped to specific endpoints
- Rate limited per key
- Auditable: every call logged

## 10. Audit logging requirements

ALL these events MUST audit log:
- Login (success + fail)
- Logout
- Password change
- 2FA enable/disable
- Role/scope change
- Sensitive resource access (financial data, KYC docs)
- Data export
- Admin actions on other accounts
- Settings change

Audit log format:
```json
{
 "audit_id": "aud_01HX",
 "timestamp": "2026-05-27T08:14:32Z",
 "actor_type": "user",
 "actor_id": "usr_01HX",
 "actor_role": "ops_admin",
 "action": "agent.kyc.approve",
 "resource_type": "agent",
 "resource_id": "agt_T4K9",
 "ip_address": "210.245.x.x",
 "user_agent": "...",
 "request_id": "req_01HX",
 "result": "success",
 "before": null,
 "after": { "kyc_level": "full" }
}
```

Stored in MongoDB `smp_audit.audit_log`, retention 7 năm.

## 11. CORS policy

API allows:
- `https://app.smp.vn` (mobile web)
- `https://admin.smp.vn`
- `https://smp-mobile.pages.dev` (preview)
- `https://smp-admin.pages.dev`
- `http://localhost:*` (dev only, never prod)

Block all others. No wildcard origins in prod.

## 12. Rate limiting

| Type | Limit | Scope |
|---|---|---|
| Per IP | 600/min | Anonymous endpoints |
| Per user | 300/min | Read endpoints |
| Per user | 60/min | Write endpoints |
| Auth endpoints | 10/min | Per IP (anti-bruteforce) |
| Webhook senders | 1000/min | Per source IP |

Headers:
```text
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 58
X-RateLimit-Reset: 1716800060
Retry-After: 30
```

## 13. Sensitive endpoints (extra protection)

These require fresh authentication (re-enter password):
- Change password
- Change phone/email
- Disable 2FA
- Delete account
- Wallet topup amount > 10M VND
- Approve partner KYC

Implementation: `Authorization-Fresh: Bearer <token>` token issued after re-auth, TTL 5 min.

## 14. Token revocation

Scenarios:
- User logout: add jti to Redis blacklist
- Password change: invalidate all tokens for that user
- Account suspend: invalidate all + lock
- Token compromised (suspected): manual revoke via ops UI

Check on every request:
```go
if redis.Exists(ctx, "blacklist:"+claims.JTI) > 0 {
 return ErrTokenRevoked
}
```

## 15. Frontend security

### 15.1 Token storage
- Mobile native: Keychain (iOS) / EncryptedSharedPreferences (Android)
- Web: HttpOnly Secure SameSite=Strict cookies (preferred) OR memory only (no localStorage for tokens)
- Refresh token in HttpOnly cookie, access token in memory

### 15.2 XSS prevention
- Content Security Policy headers
- Sanitize all user input rendering
- Use framework auto-escape (React/Vue)

### 15.3 CSRF
- SameSite=Strict cookies prevent most
- Additional CSRF token for non-idempotent ops (POST/PUT/DELETE) if cookie auth used

## 16. Compliance notes

- PII access logged
- Right to erasure (PDPA VN): customer can request account deletion → anonymize within 30 days
- Data export: customer can request all their data via support → ZIP file within 7 days
- Children: app requires phone OTP, no service for &lt; 16

## 17. Onboarding new role/scope (process)

To add new scope:
1. Define in scope catalog (this doc)
2. Implement check in code (middleware + repo)
3. Add to relevant role(s) in DB seed
4. Audit log new permission grant
5. Update API contract OpenAPI
6. QA verify with test JWT
