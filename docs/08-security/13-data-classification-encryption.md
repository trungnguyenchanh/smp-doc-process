# SMP Data Classification + Encryption Policy

**Audience**: Security, Backend, DevOps, Legal/Compliance · **Updated**: v3.3

---

## 1. Data classification levels

| Level | Definition | Examples | Handling |
|---|---|---|---|
| **L0 · Public** | Có thể công khai | Service catalog, partner names (if consented) | No restriction |
| **L1 · Internal** | Chỉ dùng trong công ty | Internal docs, dashboards, ops notes | Authenticated access |
| **L2 · Confidential** | Phải bảo vệ, leak gây hại business | Order data, partner contracts, financial reports | Encryption + RBAC |
| **L3 · Restricted (PII)** | Personal data, regulated | CCCD, address, phone, email | Encryption + audit + access controls |
| **L4 · Highly Restricted** | Most sensitive | Bank account, payment intent, password hash | Encryption + minimal access + extra audit |

## 2. Data inventory · Per domain

### 2.1 Customer data (mostly in `inside`, SMP holds references)

| Field | Level | Storage | Notes |
|---|---|---|---|
| Full name | L3 | inside | SMP cache 30s |
| Phone | L3 | inside | Masked in UI: `+84••••2841` |
| Email | L3 | inside | |
| Address | L3 | inside + SMP order | SMP store per-order snapshot |
| Date of birth | L3 | inside | SMP not store |
| CCCD | L4 | inside | SMP NEVER store |
| Payment method | L4 | inside | SMP only get intent_id |
| Order history (in SMP) | L2 | smp_order MySQL | RBAC scoped |
| Rating + review | L2 | smp_quality | RBAC scoped |

### 2.2 Agent data

| Field | Level | Storage | Notes |
|---|---|---|---|
| Full name | L3 | smp_agent MySQL | |
| Phone | L3 | smp_agent | Masked in non-privileged views |
| Email | L3 | smp_agent | |
| Home address | L3 | smp_agent | Used for dispatch radius |
| CCCD number | L4 | smp_agent (encrypted) | Application-level AES-256 |
| CCCD photos | L4 | S3/R2 + smp_agent.kyc_docs | URLs signed, expire 15 min |
| Bank account | L4 | smp_agent (encrypted) | Application-level AES-256 |
| Skills + levels | L1 | smp_agent | |
| Location (realtime) | L3 | Redis (TTL 60s) | |
| Earnings | L2 | smp_finance | Self-view OK, others RBAC |

### 2.3 Partner data (v3.3)

| Field | Level | Storage | Notes |
|---|---|---|---|
| Business name | L0/L1 | smp_partner | Public if consent |
| Tax code | L2 | smp_partner | |
| Rep name + CCCD | L4 | smp_partner (encrypted) | Application-level AES-256 |
| Rep phone, email | L3 | smp_partner | |
| Wallet balance | L2 | smp_partner | RBAC scoped |
| Contract docs | L2 | S3/R2 + smp_partner | |
| Pricing config | L2 | smp_partner | Confidential per partner |
| Commission rate | L2 | smp_partner | |

### 2.4 Order data

| Field | Level | Storage | Notes |
|---|---|---|---|
| Order ID, timestamps | L1 | smp_order | |
| Customer address (snapshot) | L3 | smp_order | Per-order copy |
| Service + steps | L1 | smp_order | |
| Pricing breakdown | L2 | smp_order | |
| Photo proof | L3 | S3/R2 + meta | URLs signed |
| Customer notes | L3 | smp_order | May contain PII |
| Internal notes (ops) | L2 | smp_order | |

### 2.5 System data

| Field | Level | Storage | Notes |
|---|---|---|---|
| Audit logs | L2 | smp_audit MongoDB | Encrypted at rest |
| Event logs | L1 | smp_events MongoDB | TTL 90 days |
| Access logs | L1 | Loki | Retention 30 days |
| Session tokens | L4 | Redis | TTL 8h |
| API keys | L4 | DB (hashed) + Vault | Bcrypt hash |
| JWT signing private key | L4 | Vault only | RS256 private |
| DB passwords | L4 | Vault | Quarterly rotation |
| Webhook signing secrets | L4 | Vault | Per partner |

## 3. Encryption requirements

### 3.1 At rest

| Storage | Method | Key management |
|---|---|---|
| MySQL data files | InnoDB encryption (AES-256) | Managed by cloud provider (AWS KMS / GCP KMS) |
| MongoDB data files | WiredTiger encryption (AES-256) | Cloud KMS |
| Redis | Disk encryption (provider managed) | N/A (in-memory primarily) |
| S3/R2 objects | SSE-KMS (AES-256) | KMS per environment |
| K8s secrets | Sealed Secrets + KMS | Vault for app secrets |
| Backups (DB snapshots) | Encrypted (cloud default) | KMS |

### 3.2 In transit

| Connection | Protocol | Cert |
|---|---|---|
| Client ↔ API Gateway | TLS 1.3 (1.2 min) | Cloudflare-managed (ACM-like) |
| Service ↔ Service (internal) | mTLS via Istio | Auto-rotated by Istio CA |
| App ↔ MySQL | TLS required | Cloud-managed |
| App ↔ Redis | TLS required (prod) | Cloud-managed |
| App ↔ MongoDB | TLS required | Atlas-managed |
| App ↔ External (inside, wms) | TLS 1.2+ | Public CA |

### 3.3 Application-level field encryption

L4 fields: PII identifiers, bank accounts → encrypt in application BEFORE save to DB.

```go
package crypto

// AES-256-GCM with separate key per env
type FieldCipher struct {
    aead cipher.AEAD
}

func (c *FieldCipher) Encrypt(plaintext string) (string, error) {
    nonce := make([]byte, c.aead.NonceSize())
    if _, err := rand.Read(nonce); err != nil { return "", err }
    ct := c.aead.Seal(nil, nonce, []byte(plaintext), nil)
    return base64.StdEncoding.EncodeToString(append(nonce, ct...)), nil
}

func (c *FieldCipher) Decrypt(encoded string) (string, error) {
    b, err := base64.StdEncoding.DecodeString(encoded)
    if err != nil { return "", err }
    if len(b) < c.aead.NonceSize() { return "", errors.New("invalid") }
    nonce, ct := b[:c.aead.NonceSize()], b[c.aead.NonceSize():]
    pt, err := c.aead.Open(nil, nonce, ct, nil)
    return string(pt), err
}
```

Store encrypted: `agents.cccd_number_encrypted` VARCHAR(255).
Storage of plain field name still kept in app code but DB column is `_encrypted`.

Searchable fields (vd search by CCCD):
- Use deterministic encryption OR
- Use hash column for equality search: `cccd_hash CHAR(64)` = SHA-256(CCCD + global_salt)
- Don't enable substring search on encrypted PII

### 3.4 Hashing (one-way)

| Data | Algorithm | Cost |
|---|---|---|
| User passwords | bcrypt | cost = 12 |
| API keys | bcrypt | cost = 10 (faster lookup) |
| Session ID validation | HMAC SHA-256 with rotating secret | N/A |

## 4. Key management

### 4.1 Vault layout

```text
secret/smp/<env>/
├── service-keys/
│   ├── jwt-signing-private.pem
│   ├── jwt-signing-public.pem
│   └── webhook-signing-secret
├── field-encryption-keys/
│   ├── agent-pii-aes256-v1
│   ├── partner-pii-aes256-v1
│   └── field-encryption-salt
├── db-credentials/
│   ├── mysql-app-password
│   ├── mysql-readonly-password
│   ├── mongo-app-password
│   └── redis-password
└── external-integrations/
    ├── inside-api-token
    ├── wms-api-token
    ├── sendgrid-api-key
    └── twilio-auth-token
```

### 4.2 Key rotation schedule

| Key type | Rotation | Process |
|---|---|---|
| JWT signing | Annual | Issue new kid, keep old for 30d, then expire |
| Field encryption | Annual | Add v2 key, re-encrypt async, retire v1 |
| Webhook signing | Annual | Coordinate with partners, transition period 30d |
| DB passwords | Quarterly | Use Vault dynamic secrets or scheduled rotation |
| API keys | On request OR annual | User-initiated regeneration |
| TLS certs | Auto (Let's Encrypt-like) | Cloudflare-managed |
| MFA backup codes | On user request | Regenerate set of 10 |

### 4.3 Key compromise response

1. Detect (alert from SIEM, suspicious logs)
2. Revoke compromised key in Vault
3. Force rotate to new key
4. Re-encrypt data using old key (background job)
5. Audit log inventory of all operations during exposure window
6. Notify affected users if PII exposed
7. Post-mortem + regulatory notification (PDPA: within 72h if "significant risk")

## 5. PII handling rules

### 5.1 Display masking

UI hiển thị PII với masking by default:

| Field | Full | Masked |
|---|---|---|
| Phone | `+84907842931` | `+84•••842931` (last 4) |
| Email | `hung@example.com` | `h***@example.com` |
| CCCD | `079203012345` | `079•••12345` (first 3 + last 5) |
| Bank account | `0123456789012345` | `••••••••5678` |
| Full name | `Nguyễn Văn A` | `N*** A` (first letter only) |

Privileged roles (ops_admin với scope `data.pii.view`) có thể click "Reveal" → audit log entry.

### 5.2 Logs

NEVER log full PII. Mask before logging:
```go
logger.Info("create agent",
    zap.String("agent_id", agent.ID),
    zap.String("name_masked", mask.Name(agent.FullName)),
    zap.String("phone_masked", mask.Phone(agent.Phone)),
)
```

Zap logger has middleware `MaskPII` for known field names.

### 5.3 Exports

Data exports (CSV reports, debug dumps):
- Default: PII masked
- Full PII export: requires `data.export.pii` scope + manager approval + audit
- Export file: encrypted ZIP, password sent separately
- Download link: expires 24h, IP-bound

### 5.4 Customer data sharing with partners

Partners see end-customer data ONLY for their orders:
- Name + phone (for service delivery)
- Service address (operational need)
- NOT: payment method, other orders, customer history outside partner's orders

Enforced at API + DB row-level filtering.

## 5.5 Dynamic Data Masking (v4.0)

> Implementation pattern cho PII masking ở API Gateway middleware. Default-deny: response chứa masked data, scope unlock specific fields.

### 5.5.1 Masking patterns catalog

Mỗi field type có pattern riêng. Pattern phải:
- Đủ để identify match (cho CSKH "có phải khách đầu 0912... cuối ...890?")
- Không đủ để leak (không thấy full)

| Field | Plain | Masked pattern | Display |
|---|---|---|---|
| Phone (VN) | `0912345890` | First 3 + last 3 | `0912****890` |
| Phone (intl) | `+14155551234` | First 4 + last 3 | `+1415*****234` |
| Email | `nguyen.van.a@gmail.com` | First 3 of local + domain | `ngu***@gmail.com` |
| CCCD | `001202012345` | First 4 + last 4 | `0012****2345` |
| Old CMND | `024567890` | First 3 + last 2 | `024****90` |
| Bank account | `1234567890123` | First 4 + last 4 | `1234*****0123` |
| Tax code (VN MST) | `0123456789-001` | Full only with scope | `01234****89-001` |
| Credit card | `4532123456789010` | First 6 + last 4 (PCI-DSS) | `453212******9010` |
| Full name | `Nguyễn Văn A` | First name + initial | `Nguyễn V. A.` (light masking) hoặc `Nguyễn ***` (strong) |
| Address line | `123 Lê Lợi, P. Bến Nghé, Q.1` | District + city only | `*** Q.1, TP.HCM` |
| Date of birth | `1990-05-28` | Year only | `1990-**-**` |
| ID photo URL | `https://s3.../front.jpg` | Whole URL hidden | `[CCCD photo - request unmask]` |

### 5.5.2 API Gateway middleware (Go)

```go
// pkg/masking/middleware.go
package masking

import (
    "encoding/json"
    "net/http"
    "regexp"
    "strings"
)

// Tag-based masking config
// Usage: tag struct field với mask info
type Customer struct {
    ID       string `json:"id"`
    Name     string `json:"name"      mask:"name,scope=pii.unmask.name"`
    Phone    string `json:"phone"     mask:"phone,scope=pii.unmask.phone"`
    Email    string `json:"email"     mask:"email,scope=pii.unmask.email"`
    CCCD     string `json:"cccd"      mask:"cccd,scope=pii.unmask.cccd"`
    BankAcc  string `json:"bank_acc"  mask:"bank,scope=pii.unmask.bank_account"`
    Address  string `json:"address"   mask:"address,scope=pii.unmask.address_full"`
}

// Middleware applied AFTER service returns response, BEFORE serialize
func MaskingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Wrap ResponseWriter to capture
        rec := newResponseRecorder(w)
        next.ServeHTTP(rec, r)

        // Parse response if JSON
        if !strings.Contains(rec.Header().Get("Content-Type"), "application/json") {
            rec.Flush()
            return
        }

        scopes := scopesFromContext(r.Context())
        var body interface{}
        if err := json.Unmarshal(rec.body, &body); err != nil {
            rec.Flush()
            return
        }

        // Walk tree, apply masking based on struct tags
        masked := applyMasking(body, scopes)

        // Audit each unmasked field access
        if rec.metadata.UnmaskedFields != nil {
            audit.LogPIIAccess(r.Context(), audit.PIIAccess{
                ActorID:    userFromContext(r.Context()),
                Fields:     rec.metadata.UnmaskedFields,
                Subject:    rec.metadata.Subject,
                RequestID:  requestIDFromContext(r.Context()),
            })
        }

        out, _ := json.Marshal(masked)
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(rec.statusCode)
        w.Write(out)
    })
}
```

### 5.5.3 Field-level masker functions

```go
// pkg/masking/maskers.go

var maskers = map[string]MaskerFunc{
    "phone":   maskPhone,
    "email":   maskEmail,
    "cccd":    maskCCCD,
    "bank":    maskBank,
    "name":    maskName,
    "address": maskAddress,
}

type MaskerFunc func(plain string) string

func maskPhone(plain string) string {
    // Normalize first
    digits := digitsOnly(plain)
    if len(digits) < 7 { return "***" }
    
    // Vietnam phone format
    if strings.HasPrefix(plain, "+84") || strings.HasPrefix(plain, "0") {
        // Show first 3 (0912 or +84) + last 3
        return digits[:3] + strings.Repeat("*", len(digits)-6) + digits[len(digits)-3:]
    }
    // International: first 4 + last 3
    return plain[:4] + strings.Repeat("*", len(plain)-7) + plain[len(plain)-3:]
}

func maskEmail(plain string) string {
    parts := strings.SplitN(plain, "@", 2)
    if len(parts) != 2 { return "***@***" }
    local := parts[0]
    if len(local) <= 3 {
        return strings.Repeat("*", len(local)) + "@" + parts[1]
    }
    return local[:3] + strings.Repeat("*", len(local)-3) + "@" + parts[1]
}

func maskCCCD(plain string) string {
    digits := digitsOnly(plain)
    if len(digits) != 12 { return "************" }
    return digits[:4] + "****" + digits[8:]
}

func maskBank(plain string) string {
    if len(plain) < 8 { return "********" }
    return plain[:4] + strings.Repeat("*", len(plain)-8) + plain[len(plain)-4:]
}
```

### 5.5.4 Service-side helper (selective unmask)

Đôi khi service muốn explicitly return unmasked data (vd internal service call). Dùng wrapper:

```go
// Trong handler
func (h *Handler) GetCustomer(c *gin.Context) {
    customer := h.repo.Get(c.Param("id"))
    
    // Default behavior: middleware masks based on scope
    c.JSON(200, customer)
}

// Special case: financial reconciliation cron — bypass masking
func (h *Handler) RunReconciliation(ctx context.Context) {
    customers := h.repo.ListAll()
    
    // Mark context to skip masking
    ctx = masking.SkipMaskingContext(ctx, "reason:reconciliation_job")
    
    // Audit log records skip with reason
    audit.LogMaskSkip(ctx, customers)
    
    // Process with full data
    for _, c := range customers {
        h.reconcile(c.BankAccount, ...)
    }
}
```

### 5.5.5 Tokenization vs Masking · Decision tree

```text
Is the field used for:
│
├─ Display only to certain roles?
│   └─► MASKING (reversible, server-side based on scope)
│
├─ PCI-DSS compliance (credit card)?
│   └─► TOKENIZATION (vault lookup, no plaintext in app DB)
│
├─ Search by exact value (vd lookup by phone)?
│   └─► HASHED INDEX + MASKED VALUE (hash for lookup, masked for display)
│
├─ Aggregation (count, sum)?
│   └─► No masking needed (no PII exposed)
│
└─ Used in expression (vd CCCD as login)?
    └─► HASHED with pepper (one-way, store hash only)
```

### 5.5.6 Performance considerations

- Mask functions = pure functions, no I/O → ~100ns per field
- Reflection overhead: cache field tags per type (lazy init)
- Total overhead: ~10-50µs per response (negligible vs 50-500ms total latency)
- Benchmark: `go test -bench=BenchmarkMaskCustomer`

### 5.5.7 Multi-country masking patterns

Mỗi country có rules riêng:

```go
var countryMaskingProfile = map[string]Profile{
    "VN": {PhoneRule: "vn_format", IDField: "cccd", IDLength: 12},
    "US": {PhoneRule: "intl_format", IDField: "ssn", IDLength: 9},
    "CN": {PhoneRule: "intl_format", IDField: "id_card_cn", IDLength: 18},
    "SG": {PhoneRule: "intl_format", IDField: "nric", IDLength: 9},
}
```

Khi user country VN truy cập customer country US, áp dụng US masking profile (vd SSN format `123-**-6789`).

## 6. Data lifecycle

### 6.1 Creation
- Validate input (sanitize, length limits)
- Classify level → apply storage controls
- Audit log: resource_created

### 6.2 Access
- Authenticate + authorize (scope check)
- Apply masking based on role
- Audit log: pii.viewed (for L3/L4 access)

### 6.3 Modification
- Audit log: before + after snapshot
- Encryption re-applied if PII field changed

### 6.4 Deletion / Anonymization

User-requested deletion (PDPA):
1. Verify user identity (re-auth, fresh token)
2. Mark account `deletion_pending` (30-day grace period for rollback)
3. After 30 days, automated process:
   - PII fields → anonymized (vd `Nguyễn Văn A` → `deleted_user_<uuid>`)
   - Order history retained (anonymized customer_id)
   - Audit logs retained (PII anonymized in actor_name)
   - Photos: customer-uploaded deleted, agent-uploaded retained anonymized
   - Bank accounts deleted
4. Audit log: account.deletion.completed
5. Notify user via email if reachable

System-driven retention expiry:
- Order data > 5 years → archive to cold storage, then delete after legal review
- Event log > 90 days → MongoDB TTL auto-delete
- Quality photos > 2 years → archive

## 7. Vulnerability management

### 7.1 Dependency scanning
- `govulncheck` in CI per service
- `npm audit` for frontend (no critical allowed)
- Trivy for container images
- Alert: any HIGH/CRITICAL → block deploy

### 7.2 Secret scanning
- Gitleaks pre-commit hook (developer machine)
- Gitleaks in CI (PR check)
- TruffleHog for repos history scan (monthly)

### 7.3 Security testing
- Static analysis: SonarCloud (weekly)
- DAST: OWASP ZAP scan (weekly staging)
- Pen test: annually + before major release
- Bug bounty: planned for GA (Phase 2)

## 8. Compliance checklist

### 8.1 PDPA Việt Nam

- [x] Privacy policy published, customer accept on signup
- [x] Consent recorded (audit_log entry on signup)
- [x] Right to information: customer can see their data via "Hồ sơ"
- [x] Right to erasure: account deletion flow (30-day grace)
- [x] Right to correction: edit profile in app
- [x] Data minimization: only collect what's needed
- [x] Purpose limitation: each field has documented use
- [x] Security: encryption + access controls (this doc)
- [x] Breach notification: incident response within 72h
- [x] Cross-border transfer: only to providers in approved list (Cloudflare, AWS, MongoDB Atlas)

### 8.2 Tax + Accounting (Việt Nam)

- [x] Financial transactions retained 10 years
- [x] Invoice records retained 10 years
- [x] Audit log for financial operations
- [x] VAT calculation correct
- [x] Reports export-able for tax filing

### 8.3 PCI DSS (Payment Card Data)

SMP **does NOT** store, process, or transmit card data:
- Payment handled entirely by `inside` (payment partner)
- SMP only stores `payment_intent_id` reference
- No PAN, CVV, magstripe ever touches SMP systems
- → SMP is **out of PCI scope**

If we ever need to take cards directly (future), follow PCI DSS Level 4 minimum.

---

## 8.5 Multi-jurisdiction compliance (v4.0)

> Khi expand global, mỗi country/region có luật riêng. Bảng so sánh + checklist mapping per region.

### 8.5.1 Regulation comparison matrix

| Aspect | **PDPA VN** | **GDPR EU** | **CPRA US-CA** | **PIPL CN** |
|---|---|---|---|---|
| Effective | 1/7/2023 | 25/5/2018 | 1/1/2023 | 1/11/2021 |
| Data localization | Optional (recommended) | Within EU/EEA | None | **Mandatory** for CN citizen data |
| Cross-border transfer | Approval for some categories | SCC / BCR / adequacy decision | Notice to consumer | **CAC security assessment** required |
| Breach notification (authority) | 72h | 72h | "without unreasonable delay" | **Immediate** + 24h for high-risk |
| Breach notification (users) | Without delay | 72h if high risk | "without unreasonable delay" | When required by authority |
| Right to erasure | 30 days | 30 days (1 month) | 45 days | 30 days |
| Right to data portability | Yes (7 days) | Yes (30 days) | Yes (45 days) | Yes |
| Consent | Specific + informed | Specific + informed + revocable | Opt-out of sale | Specific + revocable |
| DPO mandate | Required if processing 10k+ records | Required if large-scale or sensitive | Not mandatory | Required for large processors |
| Penalty (max) | 5% revenue | **4% global revenue** or €20M | $7.5k per intentional violation | **5% revenue** or ¥50M |
| Children data (special protection) | < 16 | < 16 (state varies 13-16) | < 13 + opt-in 13-16 | < 14 |
| Sensitive PII categories | health, race, religion, biometric | + political opinion, sexual orientation, genetic | + immigration, geolocation | + medical, financial, biometric, religious |

### 8.5.2 Per-country checklist (mappable to deployment)

#### PDPA Việt Nam (default · v3.x baseline)
- ✅ See section 8.1 above
- Specific: register processing activities với Cục An toàn thông tin
- Specific: appoint DPO if processing sensitive data > 10k records
- Specific: data subject requests via email/in-app form

#### GDPR (EU - future, when launch)
- Lawful basis documentation (consent vs legitimate interest vs contract)
- DPIA (Data Protection Impact Assessment) cho high-risk processing
- Records of Processing Activities (ROPA) - Article 30
- Designate EU representative if no establishment in EU
- Cookie consent banner (no pre-checked boxes)
- "Privacy by design and default" engineering principles
- 72h breach notification with specific format to relevant DPA
- Cross-border: SCC (Standard Contractual Clauses) cho non-adequate countries

#### CPRA (California, US)
- "Do Not Sell or Share My Personal Information" link prominently displayed
- Opt-out preference signal (Global Privacy Control - GPC) honored
- Sensitive PII opt-in (not opt-out default)
- Annual privacy notice update
- Consumer request portal: access, deletion, correction, opt-out
- Service provider contracts must include CPRA terms
- 45-day response window cho all requests

#### PIPL (China)
- **Critical**: data of CN citizens MUST stored in CN
- Cross-border export requires **CAC (Cyberspace Administration of China) security assessment** if:
  - Transferring sensitive PII
  - Processing > 1M CN individuals' data
  - Critical Information Infrastructure Operator (CIIO)
- Consent must be "separate consent" for sensitive data (not bundled)
- Local DPO + local representative required
- Audit logs accessible to authorities upon request
- **No data export without explicit consent + assessment**

### 8.5.3 Deployment implications

| Action | VN | EU | US (CA) | CN |
|---|---|---|---|---|
| Where data stored | SG or VN | Frankfurt (eu-central-1) | Virginia (us-east-1) | **CN-only cluster (Beijing)** |
| Where backups | Same region | Same region (EU) | Same region (US) | **CN-only** |
| Cross-region transfer | Yes with conditions | SCC required | Notice | **CAC assessment + consent** |
| User consent flow | Standard checkbox | Granular (per purpose) | Opt-out for sale | Separate per sensitive category |
| Marketing/analytics tracking | Opt-in | Opt-in | Opt-out | Opt-in + separate consent |

Implementation pattern (xem Doc 02 section 11.5 sharding strategy):
- `smp-china` cluster fully isolated, no cross-region replication
- `smp-eu` cluster với encryption keys managed in EU
- `smp-us` cluster với CPRA-specific UI

### 8.5.4 DPO + Contact info per region

Mỗi region phải có DPO + DPA contact:

```yaml
# config/dpo-contacts.yaml
dpo_contacts:
  VN:
    dpo_name: "Nguyễn Văn DPO"
    email: "dpo.vn@smp.vn"
    address: "HCMC, Vietnam"
    dpa: "Cục An toàn thông tin - Bộ TT&TT"
  EU:
    dpo_name: "TBD when launch"
    representative: "TBD"
    email: "dpo.eu@smp.vn"
    dpa_per_country:
      DE: "BfDI (Bundesbeauftragte für den Datenschutz)"
      FR: "CNIL"
  US-CA:
    privacy_officer: "TBD"
    email: "privacy.us@smp.vn"
    agency: "California Privacy Protection Agency (CPPA)"
  CN:
    dpo_name: "TBD when launch"
    local_rep: "TBD"
    email: "dpo.cn@smp.cn"
    authority: "CAC (Cyberspace Administration of China)"
```

### 8.5.5 Compliance review cadence

| Review | Frequency | Owner |
|---|---|---|
| Privacy policy audit | Annual + on major feature change | Legal + DPO |
| Compliance gap analysis per region | Quarterly | DPO + Security |
| Vendor DPA review | Annual | Legal |
| Penetration test (per region) | Annual | Security + external |
| Cross-border transfer documentation | Quarterly | DPO |
| Employee training (compliance) | Annual | HR + DPO |
| Incident response drill (breach simulation) | Semi-annual | Security + Ops |

## 9. Vendor risk

External services with access to data:

| Vendor | Data shared | Level | Agreement |
|---|---|---|---|
| Cloudflare | Web traffic (CDN, WAF) | L1-L2 (metadata) | DPA signed |
| AWS (compute, S3) | All data (encrypted) | L1-L4 | DPA + BAA |
| MongoDB Atlas | smp_events, smp_audit | L1-L2 | DPA |
| Twilio | Phone numbers (SMS) | L3 | DPA |
| SendGrid | Email addresses | L3 | DPA |
| Sentry | Error traces (no PII) | L1 | DPA, scrub config |
| inside | All customer + payment | L3-L4 | Internal · same group |
| wms | Material refs | L1 | Internal |

Annual vendor review.

## 10. Incident response (data breach)

If suspected PII leak:

1. **0-1h**: contain (revoke keys, isolate systems)
2. **1-4h**: assess scope (how much data, how many users)
3. **4-24h**: notify internal stakeholders (legal, exec)
4. **24-72h** (PDPA): notify Ministry of Information & Communications if "significant risk"
5. **Within 72h**: notify affected users with:
   - What data was exposed
   - When and how
   - What we're doing
   - What they should do
6. **Post**: post-mortem, regulatory cooperation, customer support

Maintain `incident-response-plan.md` separate doc with detailed playbook.

## 11. Training

All engineers complete (annual):
- Secure coding course (2h)
- This data classification doc
- PDPA training (1h)
- Incident response drill (table-top, 1x/year)

Track completion in HR system.

## 12. Self-audit

Quarterly review:
- [ ] All L3/L4 fields encrypted at rest
- [ ] Key rotation up to date
- [ ] Access logs reviewed for anomalies
- [ ] Vulnerability scan results addressed
- [ ] Vendor DPAs current
- [ ] Sample 10 PII access events → verify legitimate
- [ ] Document any gaps + assign owners
