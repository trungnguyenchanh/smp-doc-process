# SMP Backend · Coding Standards + Local Dev Setup

**Stack**: Go 1.22 · MySQL 8 · Redis 7 · MongoDB 6 · **Audience**: Backend Developers

---

## 1. Go coding standards

### 1.1 Style
- Follow [Effective Go](https://go.dev/doc/effective_go) + [Google Go Style Guide](https://google.github.io/styleguide/go/)
- Format: `gofmt` (auto, no debate)
- Lint: `golangci-lint` (config in repo root `.golangci.yml`)
- Imports: `goimports` auto-organize. Order: stdlib, third-party, internal — separated by blank line.

### 1.2 Naming
| Type | Convention | Example |
|---|---|---|
| Package | lowercase, single word | `order`, `dispatch` |
| Exported | PascalCase | `OrderService`, `CreateOrder` |
| Unexported | camelCase | `orderRepo`, `validateInput` |
| Const | PascalCase hoặc UPPER_SNAKE | `MaxRetries = 3`, `STAGE_CREATED` |
| Interface | tên + `-er` hoặc behavioral | `OrderRepository`, `Notifier` |
| Receiver | 1-2 chars | `func (s *Service)`, `func (o *Order)` |
| Error | bắt đầu `Err` | `ErrOrderNotFound`, `ErrInsufficientBalance` |

### 1.3 Project structure

Mỗi service repo follow:

```text
smp-order-svc/
├── cmd/
│   └── server/
│       └── main.go              # entry point
├── internal/
│   ├── api/                     # HTTP handlers
│   │   ├── handler/
│   │   ├── middleware/
│   │   └── router.go
│   ├── domain/                  # business logic
│   │   ├── order/
│   │   │   ├── service.go       # OrderService
│   │   │   ├── repository.go    # interface
│   │   │   └── model.go         # entity structs
│   │   └── dispatch/
│   ├── infra/                   # adapters
│   │   ├── mysql/
│   │   ├── redis/
│   │   └── kafka/
│   ├── config/                  # viper config loader
│   └── pkg/                     # internal utilities
├── api/
│   └── openapi.yaml             # API spec
├── migrations/                  # golang-migrate sql files
├── deployments/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── k8s/                     # k8s manifests
├── docs/
│   └── adr/                     # architecture decision records
├── scripts/
│   ├── setup.sh
│   └── seed.sh
├── test/
│   ├── integration/
│   └── fixtures/
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

### 1.4 Error handling

**Wrap errors with context:**
```go
if err := repo.Save(ctx, order); err != nil {
    return fmt.Errorf("save order %s: %w", order.ID, err)
}
```

**Sentinel errors at domain layer:**
```go
package order

var (
    ErrNotFound          = errors.New("order not found")
    ErrInvalidStage      = errors.New("invalid stage transition")
    ErrInsufficientFunds = errors.New("insufficient wallet balance")
)
```

**Check with `errors.Is`/`errors.As`:**
```go
if errors.Is(err, order.ErrNotFound) {
    return c.JSON(404, problemDetails{...})
}
```

**HTTP layer translate domain error to ProblemDetails (RFC 7807)** trong middleware.

### 1.5 Context usage

- Mọi function I/O nhận `ctx context.Context` là param đầu
- KHÔNG store ctx trong struct
- Timeout per request: 30s default, dispatch endpoint 60s
- Cancellation propagate qua HTTP client, DB query, Redis call

```go
func (s *Service) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    ctx, span := s.tracer.Start(ctx, "order.create")
    defer span.End()
    // ...
}
```

### 1.6 Logging với Zap

```go
s.log.Info("order created",
    zap.String("order_id", order.ID),
    zap.String("customer_id", order.CustomerID),
    zap.String("source", order.Source),
    zap.Int("total_amount", order.TotalAmount),
)
```

**NEVER** log:
- Passwords, OTP codes, tokens
- Full CCCD, full credit card numbers
- Full customer address (chỉ log district + city)

Mask sensitive fields trong logger config.

### 1.7 Testing

- Unit test files: `*_test.go` cùng package
- Test func: `Test<Function>_<scenario>` vd `TestCreateOrder_PartnerInsufficientBalance`
- Mock interfaces với `mockery` hoặc `gomock`
- Integration test: `test/integration/` với testcontainers (MySQL + Redis up trong test)
- Coverage target: 70% domain layer, 50% overall

```go
func TestCreateOrder_PartnerPrivate(t *testing.T) {
    // Arrange
    ctx := context.Background()
    repo := &MockOrderRepo{}
    svc := order.NewService(repo, ...)
    
    // Act
    got, err := svc.Create(ctx, CreateOrderRequest{
        Source:    "partner_customer",
        PartnerID: "ptn_hung_acservice",
    })
    
    // Assert
    require.NoError(t, err)
    assert.Equal(t, "private", got.DispatchVisibility)
}
```

### 1.8 Concurrency safety

- Goroutine có bound: dùng worker pool (vd `errgroup` với SetLimit)
- Channel close: producer close, không close trong consumer
- Sync primitives: prefer `sync.RWMutex` nếu read heavy
- Race detector: `go test -race` chạy trong CI

### 1.9 Dependency injection

Manual DI (no framework). Wire trong `main.go`:

```go
func main() {
    cfg := config.Load()
    db := mysql.New(cfg.MySQL)
    cache := redis.New(cfg.Redis)
    
    repo := mysql.NewOrderRepo(db)
    publisher := kafka.NewPublisher(cfg.Kafka)
    svc := order.NewService(repo, publisher, cache)
    
    h := handler.New(svc)
    router := api.NewRouter(h)
    
    log.Fatal(http.ListenAndServe(":8080", router))
}
```

### 1.10 Forbidden patterns

❌ `panic()` in business logic (chỉ dùng cho impossible-state)
❌ `time.Sleep()` trong production code (dùng ticker hoặc backoff library)
❌ Global mutable state (singleton, package-level vars except const/error)
❌ SQL string concat (always use prepared statements với `?` placeholders)
❌ `fmt.Println` (dùng zap)
❌ `interface{}` hoặc `any` khi có type cụ thể
❌ Goroutine không có recovery (wrap với `defer recover()` ở root)
❌ `time.Now()` direct trong business logic (dùng `clock.Now()` injectable cho testability + timezone safety)
❌ `float32/float64` cho money (dùng `Money` type với int64 minor units)
❌ Hardcoded `"VND"` strings (dùng `pkg/money.VND` const)

### 1.15 v4.0 patterns · Money + Time + i18n

> Các pattern này bắt buộc từ v3.5 onwards. Existing code v3.x sẽ refactor dần qua kế hoạch migration.

#### Money type · Multi-currency safe

```go
// pkg/money/money.go
package money

type Money struct {
    Amount   int64  // minor units (đồng VND, cent USD)
    Currency string // ISO 4217: "VND", "USD", "CNY", ...
}

// Construction helpers
func VND(amount int64) Money { return Money{Amount: amount, Currency: "VND"} }
func USD(amount int64) Money { return Money{Amount: amount, Currency: "USD"} }
func FromAmount(amount int64, ccy string) Money {
    return Money{Amount: amount, Currency: ccy}
}

// Arithmetic chỉ cho cùng currency
func (m Money) Add(other Money) (Money, error) {
    if m.Currency != other.Currency {
        return Money{}, ErrCurrencyMismatch
    }
    return Money{Amount: m.Amount + other.Amount, Currency: m.Currency}, nil
}

func (m Money) Sub(other Money) (Money, error) { /* tương tự */ }

// Multiply with rate (cho surge, VAT, discount %)
// ⚠️ WARNING: float64 mất precision khi rate có decimal nhiều.
// CHỈ DÙNG cho display approximation. Khi cần exact (tính VAT, commission, refund),
// dùng MulBps() với basis points (1 bps = 0.01%) hoặc SplitBreakdown() — xem section 1.15.1.
func (m Money) MulRate(rate float64) Money {
    return Money{Amount: int64(float64(m.Amount) * rate), Currency: m.Currency}
}

// MulBps: nhân với basis points (integer-safe). 1000 bps = 10%, 800 bps = 8%
// Recommended cho mọi tính toán money exact.
func (m Money) MulBps(bps int) Money {
    return Money{Amount: m.Amount * int64(bps) / 10000, Currency: m.Currency}
}

// Format cho display theo locale
func (m Money) Format(locale string) string {
    // 100 minor units VND = "100 ₫"
    // 10050 minor units USD = "$100.50"
    decimals := DecimalPlaces(m.Currency)
    // ... format theo locale rules
}

// JSON marshal — luôn output { amount, currency }
func (m Money) MarshalJSON() ([]byte, error) {
    return json.Marshal(struct {
        Amount   int64  `json:"amount"`
        Currency string `json:"currency"`
    }{m.Amount, m.Currency})
}
```

**Usage trong domain**:
```go
type Order struct {
    ID              string
    SubtotalLabor   money.Money
    SubtotalMaterial money.Money
    VAT             money.Money
    Total           money.Money
}

// Tính tổng — tất cả phải cùng currency
func (o *Order) RecalcTotal() error {
    if o.SubtotalLabor.Currency != o.SubtotalMaterial.Currency {
        return money.ErrCurrencyMismatch
    }
    subtotal, _ := o.SubtotalLabor.Add(o.SubtotalMaterial)
    o.Total, _ = subtotal.Add(o.VAT)
    return nil
}
```

**Banned**:
```go
// ❌ AVOID: float gây sai số
type Order struct { Total float64 }

// ❌ AVOID: hardcode VND
type Order struct { TotalVND int64 }

// ❌ AVOID: int riêng, currency riêng nhưng không enforce
type Order struct {
    TotalAmount   int64
    TotalCurrency string  // dễ quên sync
}

// ✅ PREFER: 1 struct, type-safe
type Order struct { Total money.Money }
```

#### Time handling · UTC always

```go
// pkg/clock/clock.go
package clock

type Clock interface {
    NowUTC() time.Time
}

type realClock struct{}
func (realClock) NowUTC() time.Time { return time.Now().UTC() }

func New() Clock { return realClock{} }

// Testable
type MockClock struct{ now time.Time }
func (m *MockClock) NowUTC() time.Time { return m.now }
func (m *MockClock) SetNow(t time.Time) { m.now = t }
```

**Usage**:
```go
// ✅ Injection pattern
type OrderService struct {
    clock clock.Clock
    repo  OrderRepository
}

func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) error {
    order := Order{
        ID:           generateID(),
        CreatedAtUTC: s.clock.NowUTC(),  // ✅ UTC always
        CreatedAtTZ:  req.UserTimezone,  // audit: user ở múi giờ nào
    }
    return s.repo.Save(ctx, order)
}
```

**Convert sang local cho display**:
```go
// pkg/timezone/convert.go
func ToLocal(utc time.Time, ianaTZ string) (time.Time, error) {
    loc, err := time.LoadLocation(ianaTZ)
    if err != nil {
        return time.Time{}, err
    }
    return utc.In(loc), nil
}

// Frontend nhận về local time, format theo locale
// API response always trả về UTC ISO 8601: "2026-05-28T10:30:00Z"
```

**Banned**:
```go
// ❌ time.Now() trả về local time của server — không reliable cross-region
created := time.Now()

// ❌ Lưu DB với CURRENT_TIMESTAMP() — phụ thuộc session timezone
// SQL: INSERT INTO orders (..., created_at) VALUES (..., CURRENT_TIMESTAMP())

// ✅ Always: clock.NowUTC() trong app + UTC_TIMESTAMP(6) trong DB
```

#### Locale-aware string formatting

```go
// pkg/i18n/translator.go
type Translator interface {
    T(key string, locale string, params ...any) string
}

// Lookup `i18n_translations` table với fallback chain
func (t *translator) T(key, locale string, params ...any) string {
    // Try exact: vi-VN
    if v := t.cache.Get(key, locale); v != "" {
        return interpolate(v, params...)
    }
    // Try language only: vi
    lang := strings.Split(locale, "-")[0]
    if v := t.cache.GetByLang(key, lang); v != "" {
        return interpolate(v, params...)
    }
    // Fallback: en-US
    if v := t.cache.Get(key, "en-US"); v != "" {
        return interpolate(v, params...)
    }
    return "[" + key + "]" // dev hint
}

// Usage
status := translator.T("order.status.in_progress", "vi-VN")
// → "Đang thực hiện"

notification := translator.T("notification.order_assigned", "en-US", "Mr. Smith", "08:30")
// → "Hello Mr. Smith, your order will be served at 08:30"
```

#### Linter rules · Bắt buộc

Add vào `.golangci.yml`:

```yaml
linters-settings:
  forbidigo:
    forbid:
      - p: '^time\.Now$'
        msg: "Use clock.NowUTC() instead. Inject clock for testability + UTC safety."
      - p: '^time\.Now\(\)\.UTC\(\)'
        msg: "Use clock.NowUTC() directly."
      - p: '"VND"|"USD"|"CNY"|"THB"|"IDR"|"SGD"|"MYR"|"PHP"|"EUR"|"JPY"'
        msg: "Use money.VND, money.USD, etc. constants."
      - p: 'float64\s*\)\s*\(?\s*price|amount|total'
        msg: "Never use float for money. Use money.Money."

linters:
  enable:
    - forbidigo
    - depguard
```

Add custom linter `pkg/lint/no_currency_string.go` để catch các pattern phức tạp hơn.

### 1.15.1 Rounding rule for breakdown · `SplitBreakdown` (v3.5+)

> Critical pattern khi chia 1 số tiền P thành nhiều phần (commission C, VAT V, agent net N). Naive `round(C)` + `round(V)` + `round(N)` riêng có thể lệch 1 đồng → entry **không cân** trong double-entry bookkeeping. Xem [Doc 16 · Finance Ledger section 4](../09-finance/16-finance-ledger-spec.md) cho full context.

#### Problem statement

Cho `P = 99,999` VND. Cần tính:
- VAT (8%) — đọc rate từ `tax_config`
- Commission SMP (15% trên base = P − V)
- Agent net N = P − C − V

Naive approach (SAI):
```go
// ❌ AVOID
v := P.MulBps(800)           // VAT 8% → 7,999.92 → round → 7,999 hoặc 8,000?
base := P.Sub(v)              // base = 92,000 hoặc 92,001
c := base.MulBps(1500)        // commission 15% → 13,800.15 → round
n := P.Sub(c).Sub(v)          // residual
// → Σ có thể = 99,998 hoặc 100,000 ≠ P → entry KHÔNG cân
```

#### Solution · Rule N = P − C − V (residual)

**Quy tắc**:
1. Tính `V` (VAT) trước, làm tròn về integer minor units.
2. Tính `C` (commission) trên base (P − V), làm tròn về integer minor units.
3. **`N` = P − V − C** (residual) — N hấp thụ toàn bộ phần dư → `V + C + N = P` luôn tuyệt đối đúng.

```go
// pkg/money/breakdown.go
package money

// SplitBreakdown chia P thành 3 phần (agent_net, commission, vat) sao cho
// agent_net + commission + vat = P (no rounding drift).
// vatBps: VAT rate basis points (800 = 8%, 1000 = 10%)
// commBps: commission rate basis points trên base (P - VAT)
//
// Reference: Doc 16 section 4 (Rounding rule)
func SplitBreakdown(p Money, vatBps, commBps int) (agentNet, commission, vat Money) {
    // Step 1: VAT first (rounded down via integer division)
    vat = Money{
        Amount:   p.Amount * int64(vatBps) / 10000,
        Currency: p.Currency,
    }
    
    // Step 2: Commission on base = P - VAT
    base := p.Amount - vat.Amount
    commission = Money{
        Amount:   base * int64(commBps) / 10000,
        Currency: p.Currency,
    }
    
    // Step 3: Agent net = residual (N = P - C - V)
    // Hấp thụ mọi phần dư → tổng luôn = P
    agentNet = Money{
        Amount:   p.Amount - commission.Amount - vat.Amount,
        Currency: p.Currency,
    }
    
    return agentNet, commission, vat
}
```

#### Example walkthrough

```go
P := money.VND(99999)
n, c, v := money.SplitBreakdown(P, 800, 1500)  // VAT 8%, commission 15%

// V = 99999 × 800 / 10000 = 7,999
// base = 99999 - 7999 = 92,000
// C = 92000 × 1500 / 10000 = 13,800
// N = 99999 - 13800 - 7999 = 78,200  ← residual (no drift)

// Verify entry cân:
// 78,200 (agent_payable) + 13,800 (revenue_commission) + 7,999 (vat_payable) = 99,999 ✓
```

#### Cross-entry rounding (rare)

Nếu phân bổ nhiều orders trong 1 batch (vd settlement nhiều đơn cùng partner), chênh dồn cuối batch → ghi vào account `rounding_adjustment` (xem [Doc 16 Chart of Accounts](../09-finance/16-finance-ledger-spec.md)). Alert nếu chênh > ngưỡng (suggested: > 100 VND/batch).

#### Test cases bắt buộc

```go
// pkg/money/breakdown_test.go
func TestSplitBreakdown_NoDrift(t *testing.T) {
    cases := []struct {
        name string
        p    int64
        vatBps, commBps int
    }{
        {"perfect division", 100000, 1000, 1500},
        {"with drift", 99999, 800, 1500},      // ← edge case
        {"prime number", 99991, 800, 1500},     // ← worst case
        {"small amount", 1000, 800, 1500},
        {"large amount", 99999999999, 800, 1500},
    }
    
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            p := money.VND(tc.p)
            n, c, v := money.SplitBreakdown(p, tc.vatBps, tc.commBps)
            
            // Invariant: N + C + V = P luôn đúng
            sum := n.Amount + c.Amount + v.Amount
            require.Equal(t, tc.p, sum, "rounding drift detected")
        })
    }
}
```

### 1.15.2 VAT rate · đọc từ `tax_config`, không hardcode

> v3.4 hiện tại có `tax_configs` table (Doc 02 section 7.5.4). v3.5+ MUST đọc VAT rate runtime, không hardcode 10%.

#### Why

VAT VN hiện đang **8% (giảm tạm)** cho hàng/dịch vụ đủ điều kiện theo **VAT Law 48/2024/QH15**, có hiệu lực đến **31/12/2026**. Sau ngày này có thể trở lại 10%, hoặc gia hạn → policy thay đổi mà không cần redeploy code.

Đồng thời multi-country: VN=VAT, SG=GST 9%, US=Sales Tax theo state. Hardcode `0.10` chỉ work cho VN.

#### Pattern · Cache + fallback

```go
// pkg/finance/tax.go
package finance

import (
    "context"
    "time"
)

type TaxConfig struct {
    Country       string  // ISO 3166-1 alpha-2: "VN", "SG"
    Category      string  // "default", "service", "material"
    TaxType       string  // "VAT", "GST", "Sales Tax"
    RateBps       int     // 800 = 8%, 1000 = 10%, 900 = 9%
    EffectiveFrom time.Time
    EffectiveTo   *time.Time  // nullable
}

type TaxResolver struct {
    repo  TaxConfigRepository
    cache *ttlCache  // refresh every 5 min, fallback to last-known
}

// Resolve lấy VAT rate cho 1 transaction.
// Lookup theo (country, category, transaction_date).
// Fallback nếu DB down: dùng last-cached value + log warning.
func (r *TaxResolver) Resolve(ctx context.Context, country, category string, txnDate time.Time) (int, error) {
    key := taxCacheKey(country, category, txnDate)
    if cached, ok := r.cache.Get(key); ok {
        return cached.RateBps, nil
    }
    
    cfg, err := r.repo.FindActive(ctx, country, category, txnDate)
    if err != nil {
        // Fallback: last known cached value (degraded mode)
        if last, ok := r.cache.GetStale(key); ok {
            r.log.Warn("tax_config DB lookup failed, using stale cache",
                zap.String("country", country),
                zap.Int("stale_rate_bps", last.RateBps),
                zap.Error(err))
            return last.RateBps, nil
        }
        return 0, fmt.Errorf("tax_config not found: %w", err)
    }
    
    r.cache.Set(key, cfg, 5*time.Minute)
    return cfg.RateBps, nil
}
```

#### Usage trong finance-svc

```go
// finance-svc/internal/settlement/calculate.go
func (s *SettlementService) CalculateBreakdown(ctx context.Context, order Order) (Breakdown, error) {
    // Lookup VAT rate theo country của order + ngày
    vatBps, err := s.taxResolver.Resolve(ctx, 
        order.CountryCode,         // "VN"
        "service",                  // category
        order.CompletedAt,         // ngày completed → áp đúng rate hiệu lực
    )
    if err != nil {
        return Breakdown{}, fmt.Errorf("vat lookup: %w", err)
    }
    
    // Commission rate đọc từ rules_engine.yaml (xem section 1.16)
    commBps := s.rulesEngine.Get("pricing", ctx).GetInt("commission_bps", 1500)
    
    // Apply rounding rule
    n, c, v := money.SplitBreakdown(order.Total, vatBps, commBps)
    
    return Breakdown{
        AgentNet:   n,
        Commission: c,
        VAT:        v,
        VATRateBps: vatBps,  // snapshot vào order để audit
    }, nil
}
```

#### Audit snapshot

Order/settlement record phải lưu **snapshot** `vat_rate_bps` tại thời điểm tạo entry, KHÔNG re-compute từ tax_config (vì rate có thể đổi). Reason: 1 đơn completed 30/11/2026 với VAT 8% phải giữ 8% mãi mãi, dù sau đó policy đổi sang 10%.

```sql
-- Trong settlement table
vat_rate_bps INT NOT NULL  -- snapshot, immutable after create
```

#### Forbidden pattern

```go
// ❌ AVOID
const VAT_VN = 0.10
vat := order.Total.MulRate(VAT_VN)

// ❌ AVOID
const VAT_BPS = 1000
vat := order.Total.MulBps(VAT_BPS)

// ❌ AVOID — magic number trong tính toán
vat := money.VND(order.Total.Amount * 8 / 100)

// ✅ PREFER
vatBps, _ := taxResolver.Resolve(ctx, "VN", "service", order.CompletedAt)
n, c, v := money.SplitBreakdown(order.Total, vatBps, commBps)
```

#### Linter rules update

Add vào `.golangci.yml`:

```yaml
linters-settings:
  forbidigo:
    forbid:
      - p: '0\.[0-9]+\s*\*.*(?:vat|tax|commission)'
        msg: "Don't hardcode tax/VAT/commission rates. Use tax_config / rules_engine."
      - p: 'const\s+VAT'
        msg: "VAT rate must be runtime config, not const."
```

---

### 1.15.3 Multi-agent step split · `MultiAgentSplit` (v3.5+)

> Pattern này dùng cho dispatch-engine + finance-svc khi 1 order_step có nhiều agents tham gia với split ratio riêng. Logic ghép giữa **step weight** (% step trong order) và **role split** (% trong step).

#### Pattern problem

Order = 1,000,000 VND, service "Sửa máy lạnh" (4 steps, weights 30/20/30/20 bps).
Step 3 "Lắp đặt": lead A (50%), specialist C electrician (50%).

```go
// Sai: chia thẳng commission cho N agents → mất precision + không phân biệt step
agentAmount := totalCommission / numAgents  // BUG · không dùng split_bps
```

Đúng: dùng pattern hierarchical breakdown.

#### Step 1 · Compute step_revenue per step

```go
// pkg/finance/multiagent.go
package finance

import (
    "fmt"
    "smp/pkg/money"
)

// ComputeStepRevenue derives revenue allocated to each order_step.
//
//   step_revenue[i] = order_revenue × step_weight_bps[i] / 10000
//
// Residual rounding rule: last step absorbs rounding to ensure
//   SUM(step_revenue) == order_revenue exactly.
func ComputeStepRevenue(orderRevenue money.Money, stepWeightsBps []int) ([]money.Money, error) {
    if len(stepWeightsBps) == 0 {
        return nil, fmt.Errorf("at least one step required")
    }
    
    // Validate sum = 10000
    sum := 0
    for _, w := range stepWeightsBps {
        if w < 0 || w > 10000 {
            return nil, fmt.Errorf("step weight bps out of range: %d", w)
        }
        sum += w
    }
    if sum != 10000 {
        return nil, fmt.Errorf("step weights sum must equal 10000 bps (100%%), got %d", sum)
    }
    
    n := len(stepWeightsBps)
    result := make([]money.Money, n)
    
    // Compute first N-1 steps with MulBps
    var allocated money.Money
    for i := 0; i < n-1; i++ {
        result[i] = orderRevenue.MulBps(stepWeightsBps[i])
        allocated = allocated.Add(result[i])
    }
    
    // Last step = residual (absorbs rounding)
    result[n-1] = orderRevenue.Sub(allocated)
    
    return result, nil
}
```

#### Step 2 · Compute agent earnings per step

```go
// AgentSplit describes one agent's share of a step.
type AgentSplit struct {
    AgentID  int64
    Role     string  // "lead" | "helper" | "specialist"
    SplitBps int     // sum across all agents per step = 10000
}

// SplitStepRevenue computes how much each agent earns from one order_step.
//
//   amount_earned[i] = step_revenue × split_bps[i] / 10000
//
// Residual rounding rule: the LEAD agent absorbs rounding to ensure
//   SUM(amount_earned) == step_revenue exactly.
func SplitStepRevenue(stepRevenue money.Money, splits []AgentSplit) (map[int64]money.Money, error) {
    if len(splits) == 0 {
        return nil, fmt.Errorf("at least one agent required")
    }
    
    // Validate: exactly 1 lead + sum bps = 10000
    leadCount, sumBps := 0, 0
    leadIdx := -1
    for i, s := range splits {
        if s.Role == "lead" {
            leadCount++
            leadIdx = i
        }
        if s.SplitBps < 0 || s.SplitBps > 10000 {
            return nil, fmt.Errorf("split bps out of range for agent %d: %d", s.AgentID, s.SplitBps)
        }
        sumBps += s.SplitBps
    }
    if leadCount != 1 {
        return nil, fmt.Errorf("exactly 1 lead required, got %d", leadCount)
    }
    if sumBps != 10000 {
        return nil, fmt.Errorf("split bps sum must equal 10000, got %d", sumBps)
    }
    
    result := make(map[int64]money.Money, len(splits))
    var allocated money.Money
    
    // Compute non-lead agents
    for i, s := range splits {
        if i == leadIdx {
            continue
        }
        amt := stepRevenue.MulBps(s.SplitBps)
        result[s.AgentID] = amt
        allocated = allocated.Add(amt)
    }
    
    // Lead absorbs residual (rounding adjustment)
    result[splits[leadIdx].AgentID] = stepRevenue.Sub(allocated)
    
    return result, nil
}
```

#### Step 3 · Combined helper · earnings for full order

```go
// OrderStepConfig describes one order_step with its agents.
type OrderStepConfig struct {
    StepNo         int
    StepWeightBps  int
    Agents         []AgentSplit
}

// ComputeOrderEarnings computes amount_earned per agent for entire order.
// Returns map[agent_id]map[step_no]amount.
func ComputeOrderEarnings(
    orderRevenue money.Money,
    steps []OrderStepConfig,
) (map[int64]map[int]money.Money, error) {
    // Extract step weights
    weights := make([]int, len(steps))
    for i, s := range steps {
        weights[i] = s.StepWeightBps
    }
    
    // Compute step revenues
    stepRevenues, err := ComputeStepRevenue(orderRevenue, weights)
    if err != nil {
        return nil, fmt.Errorf("compute step revenue: %w", err)
    }
    
    // Compute agent earnings per step
    result := make(map[int64]map[int]money.Money)
    for i, step := range steps {
        agentEarnings, err := SplitStepRevenue(stepRevenues[i], step.Agents)
        if err != nil {
            return nil, fmt.Errorf("split step %d: %w", step.StepNo, err)
        }
        
        for agentID, amount := range agentEarnings {
            if result[agentID] == nil {
                result[agentID] = make(map[int]money.Money)
            }
            result[agentID][step.StepNo] = amount
        }
    }
    
    return result, nil
}
```

#### Usage example

```go
// Order 1,000,000 VND with 4 steps
orderRevenue := money.NewVND(1_000_000)

steps := []finance.OrderStepConfig{
    {StepNo: 1, StepWeightBps: 3000, Agents: []finance.AgentSplit{
        {AgentID: 1001, Role: "lead", SplitBps: 7000},
        {AgentID: 1002, Role: "helper", SplitBps: 3000},
    }},
    {StepNo: 2, StepWeightBps: 2000, Agents: []finance.AgentSplit{
        {AgentID: 1001, Role: "lead", SplitBps: 6000},
        {AgentID: 1002, Role: "helper", SplitBps: 4000},
    }},
    {StepNo: 3, StepWeightBps: 3000, Agents: []finance.AgentSplit{
        {AgentID: 1001, Role: "lead", SplitBps: 4000},
        {AgentID: 1003, Role: "specialist", SplitBps: 4000},
        {AgentID: 1004, Role: "helper", SplitBps: 2000},
    }},
    {StepNo: 4, StepWeightBps: 2000, Agents: []finance.AgentSplit{
        {AgentID: 1001, Role: "lead", SplitBps: 10000},
    }},
}

earnings, err := finance.ComputeOrderEarnings(orderRevenue, steps)
// earnings[1001] = {1: 210k, 2: 120k, 3: 120k+residual, 4: 200k} → total ~650k
// earnings[1002] = {1: 90k, 2: 80k}                              → 170k
// earnings[1003] = {3: 120k}                                     → 120k
// earnings[1004] = {3: 60k}                                      → 60k
// SUM all = 1,000,000 ✓
```

#### Validation helpers

```go
// ValidateStepWeights checks SUM(weights) == 10000 + range.
// Use BEFORE inserting/updating service_steps rows.
func ValidateStepWeights(weights []int) error { /* ... */ }

// ValidateAgentSplits checks exactly 1 lead + SUM == 10000 + ranges.
// Use BEFORE inserting/updating order_step_agents rows.
func ValidateAgentSplits(splits []AgentSplit) error { /* ... */ }
```

#### Test cases (REQUIRED · ≥ 95% coverage)

| Case | Input | Expected |
|---|---|---|
| Happy path 4 steps × 2-3 agents | 1M VND order | SUM all earnings = 1M exactly |
| Single agent (lead 100%) | 1 step, 1 lead | All revenue → lead |
| Rounding edge case | 99,999 VND prime, 3 steps × 3 agents | No drift (lead absorbs) |
| Invalid weights sum | weights = [3000, 3000, 3000] | Error: sum != 10000 |
| Multiple leads | 2 agents both role=lead | Error: exactly 1 lead required |
| Negative split | split_bps = -100 | Error: out of range |

```go
func TestComputeOrderEarnings_NoDrift(t *testing.T) {
    for _, total := range []int64{1, 99, 99_999, 1_000_000, 9_999_999_991} {
        revenue := money.NewVND(total)
        // 4 steps with various weights + 2-3 agents each
        config := buildTestSteps()
        
        earnings, err := finance.ComputeOrderEarnings(revenue, config)
        require.NoError(t, err)
        
        // SUM all amounts MUST equal original revenue
        var sum money.Money
        for _, byStep := range earnings {
            for _, amt := range byStep {
                sum = sum.Add(amt)
            }
        }
        require.True(t, sum.Equal(revenue), "drift detected: got %v expected %v", sum, revenue)
    }
}
```

#### Linter rule

```yaml
# .forbidigo.yaml addition
forbidigo:
  forbid:
    - p: 'commission\s*/\s*len\(agents\)'
      msg: "Don't divide commission by agent count. Use SplitStepRevenue with explicit split_bps."
    - p: 'amount\s*\*\s*0\.[0-9]'
      msg: "Don't multiply Money by float. Use MulBps() or SplitStepRevenue()."
```

---

### 1.15.4 Warranty package logic · `pkg/warranty` (v3.5+)

> Pattern cho maintenance subscription packages. Quota check + claim validation + deferred revenue recognition. Schema [Doc 02 §7.8](../02-database/02-database-schema.md), accounting [Doc 16 §3.6](../09-finance/16-finance-ledger-spec.md).

#### Package structure

```text
pkg/warranty/
├── resolver.go         # WarrantyResolver · validate claims
├── quota.go            # Quota tracking + atomic decrement
├── recognition.go      # Deferred revenue recognition cron
├── refund.go           # Refund calculator (cooling-off + proportional)
└── warranty_test.go
```

#### WarrantyResolver · validate claim

```go
package warranty

import (
    "context"
    "fmt"
    "time"
)

// ClaimRequest describes a customer's request to use a warranty.
type ClaimRequest struct {
    CustomerWarrantyID int64
    ClaimType          string  // cleaning | repair_basic | repair_full
    ServiceCode        string  // for cleaning
    IssueCategory      string  // for repair · must match covered_issues
}

// ValidationError reports why a claim is rejected.
type ValidationError struct {
    Code    string  // EXPIRED | NO_QUOTA | NOT_COVERED | TOO_SOON | etc.
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Code, e.Message)
}

// ValidateClaim runs all BR-WPKG-004 checks.
// Returns nil if claim can proceed, ValidationError otherwise.
func (r *WarrantyResolver) ValidateClaim(ctx context.Context, req ClaimRequest, now time.Time) error {
    warranty, err := r.repo.GetWarranty(ctx, req.CustomerWarrantyID)
    if err != nil {
        return fmt.Errorf("get warranty: %w", err)
    }
    
    // Check 1: warranty active
    if warranty.Status != "active" {
        return &ValidationError{Code: "NOT_ACTIVE", Message: "Gói không còn hoạt động"}
    }
    
    // Check 2: not expired
    if now.After(warranty.EndDateUTC) {
        return &ValidationError{Code: "EXPIRED", Message: "Gói đã hết hạn"}
    }
    
    // Check 3: quota available
    quota, err := r.repo.GetQuotaBalance(ctx, req.CustomerWarrantyID, req.ClaimType, req.ServiceCode)
    if err != nil {
        return fmt.Errorf("get quota: %w", err)
    }
    if quota.CountRemaining <= 0 {
        return &ValidationError{Code: "NO_QUOTA", Message: "Đã hết lượt cho loại này"}
    }
    
    // Check 4: rate limit (per_period)
    if quota.NextEligibleAtUTC != nil && now.Before(*quota.NextEligibleAtUTC) {
        return &ValidationError{
            Code:    "TOO_SOON",
            Message: fmt.Sprintf("Phải đợi đến %s", quota.NextEligibleAtUTC.Format("2006-01-02")),
        }
    }
    
    // Check 5: covered issue (only for repairs)
    if req.ClaimType == "repair_basic" || req.ClaimType == "repair_full" {
        covered, err := r.repo.IsCoveredIssue(ctx, warranty.PackageID, req.IssueCategory)
        if err != nil {
            return fmt.Errorf("check covered: %w", err)
        }
        if !covered {
            return &ValidationError{
                Code:    "NOT_COVERED",
                Message: fmt.Sprintf("Lỗi '%s' không nằm trong scope gói. Đặt order trả phí.", req.IssueCategory),
            }
        }
    }
    
    return nil
}
```

#### Quota tracking · atomic decrement

```go
// DecrementQuota atomically decrements count_remaining and updates last_used.
// Uses row-level lock to prevent race conditions.
func (r *WarrantyResolver) DecrementQuota(ctx context.Context, req ClaimRequest, now time.Time) error {
    return r.db.WithTransaction(ctx, func(tx Tx) error {
        // Lock the quota row
        quota, err := r.repo.LockQuotaBalance(tx, req.CustomerWarrantyID, req.ClaimType, req.ServiceCode)
        if err != nil {
            return err
        }
        
        // Re-validate (defense in depth · concurrent claim might have consumed it)
        if quota.CountRemaining <= 0 {
            return &ValidationError{Code: "NO_QUOTA_RACE", Message: "Quota consumed by another claim"}
        }
        
        // Compute next_eligible based on per_period rule
        var nextEligible *time.Time
        if quota.PeriodDays > 0 {
            t := now.AddDate(0, 0, quota.PeriodDays)
            nextEligible = &t
        }
        
        // Update
        return r.repo.UpdateQuota(tx, quota.ID, QuotaUpdate{
            CountRemaining:    quota.CountRemaining - 1,
            CountUsed:         quota.CountUsed + 1,
            LastUsedAtUTC:     now,
            NextEligibleAtUTC: nextEligible,
        })
    })
}
```

#### Deferred revenue recognition (cron)

```go
// RecognizeRevenueForPeriod processes all due recognition entries.
// Called nightly by cron · idempotent.
func (r *WarrantyResolver) RecognizeRevenueForPeriod(ctx context.Context, now time.Time) error {
    dueEntries, err := r.repo.GetDueRecognitionEntries(ctx, now)
    if err != nil {
        return err
    }
    
    for _, entry := range dueEntries {
        // Idempotency: skip if already posted
        if entry.JournalEntryID != 0 {
            continue
        }
        
        // Post journal: Dr deferred_revenue / Cr revenue_warranty_subscription
        journalID, err := r.finance.PostJournal(ctx, finance.JournalRequest{
            EntryType:     "warranty_revenue_recognition",
            ReferenceType: "warranty_revenue_recognition",
            ReferenceID:   fmt.Sprintf("%d", entry.ID),
            Lines: []finance.Line{
                {Account: "deferred_revenue", Side: "Dr", Amount: entry.Amount, OwnerType: "customer_warranty", OwnerID: entry.CustomerWarrantyID},
                {Account: "revenue_warranty_subscription", Side: "Cr", Amount: entry.Amount},
            },
        })
        if err != nil {
            return fmt.Errorf("post journal for entry %d: %w", entry.ID, err)
        }
        
        // Mark posted
        if err := r.repo.MarkRecognitionPosted(ctx, entry.ID, journalID, now); err != nil {
            return err
        }
        
        // Update warranty.total_amount_recognized
        if err := r.repo.IncrementRecognized(ctx, entry.CustomerWarrantyID, entry.Amount); err != nil {
            return err
        }
    }
    
    return nil
}
```

#### Refund calculator

```go
// CalculateRefund computes refund amount for warranty cancellation.
// Per BR-WPKG-003: 100% within 7 days, proportional after.
func CalculateRefund(warranty *Warranty, now time.Time) (money.Money, string) {
    daysSinceStart := int(now.Sub(warranty.StartDateUTC).Hours() / 24)
    
    if daysSinceStart <= 7 {
        // Cooling-off · full refund (including VAT portion)
        return warranty.PricePaid, "cooling_off_full_refund"
    }
    
    // Proportional refund based on unused duration
    totalDays := int(warranty.EndDateUTC.Sub(warranty.StartDateUTC).Hours() / 24)
    usedDays := daysSinceStart
    remainingDays := totalDays - usedDays
    if remainingDays <= 0 {
        return money.Zero(warranty.Currency), "no_refund_expired"
    }
    
    // refund = price × remaining/total
    ratio := float64(remainingDays) / float64(totalDays)
    refundAmount := warranty.PricePaid.MulBps(int(ratio * 10000))
    
    return refundAmount, "proportional_refund"
}
```

#### Test cases (REQUIRED)

| Case | Input | Expected |
|---|---|---|
| Valid cleaning claim | active warranty, quota=3 | Pass, quota → 2 |
| Expired warranty | end_date < now | Error EXPIRED |
| No quota | quota=0 | Error NO_QUOTA |
| Issue not covered | issue=compressor, whitelist=[capacitor] | Error NOT_COVERED |
| Too soon | last_used + period_days > now | Error TOO_SOON |
| Race condition | 2 concurrent claims, quota=1 | Only 1 succeeds |
| Cooling-off refund | day 3 | Full refund |
| Proportional refund | day 90 of 365 | 75% refund |
| Revenue recognition | 12-month warranty, month 1 | Post 1/12 of net |
| Revenue recognition residual | month 12 | Absorb remainder |

```go
func TestRecognizeRevenue_NoDrift(t *testing.T) {
    for _, price := range []int64{1_200_000, 999_999, 1, 100} {
        warranty := buildWarranty(price, 12)
        
        var total money.Money
        for month := 1; month <= 12; month++ {
            now := warranty.StartDateUTC.AddDate(0, month, 0)
            recognized := recognizeMonth(warranty, now)
            total = total.Add(recognized)
        }
        
        // After 12 months, total recognized MUST equal net amount
        net := warranty.PricePaid.Sub(warranty.VATAmount)
        require.True(t, total.Equal(net), "drift: got %v expected %v", total, net)
    }
}
```

---

### 1.16 v4.0 pattern · Rules Engine (`pkg/rules`)

> Pattern này dùng cho services có business logic thay đổi theo policy (dispatch-engine, finance-svc, catalog-svc). Thay vì hardcode rule trong Go, load từ YAML config.

#### Package structure

```text
pkg/rules/
├── engine.go          # Engine, load YAML, compile expressions
├── types.go           # Rule, Context, Decision types
├── loader.go          # File watcher với fsnotify
├── cache.go           # Compiled program cache
├── engine_test.go
└── testdata/
    └── sample_rules.yaml
```

#### Core types

```go
// pkg/rules/types.go
package rules

type Rule struct {
    ID          string                 `yaml:"id"`
    Name        string                 `yaml:"name"`
    Category    string                 `yaml:"category"`
    Description string                 `yaml:"description"`
    Enabled     bool                   `yaml:"enabled"`
    Priority    int                    `yaml:"priority"`
    When        string                 `yaml:"when"`         // expression
    Then        map[string]interface{} `yaml:"then"`
    Overrides   []Override             `yaml:"overrides"`
    LegacyID    string                 `yaml:"legacy_id"`    // BR-* mapping
}

type Override struct {
    When string                 `yaml:"when"`
    Then map[string]interface{} `yaml:"then"`
}

type Context map[string]interface{}

type Decision struct {
    Matched []string                          // rule IDs matched
    Values  map[string]interface{}            // merged then values
}

func (d Decision) GetInt(key string, def int) int {
    if v, ok := d.Values[key].(int); ok { return v }
    if v, ok := d.Values[key].(int64); ok { return int(v) }
    return def
}

func (d Decision) GetFloat(key string, def float64) float64 { /* ... */ }
func (d Decision) GetString(key, def string) string { /* ... */ }
func (d Decision) GetBool(key string, def bool) bool { /* ... */ }
```

#### Engine

```go
// pkg/rules/engine.go
package rules

import (
    "sync"
    "github.com/expr-lang/expr"
    "github.com/expr-lang/expr/vm"
    "gopkg.in/yaml.v3"
    "go.uber.org/zap"
)

type Engine struct {
    mu       sync.RWMutex
    rules    []Rule
    programs map[string]*vm.Program  // compiled expressions
    log      *zap.Logger
}

func New(log *zap.Logger) *Engine {
    return &Engine{
        programs: make(map[string]*vm.Program),
        log:      log,
    }
}

func (e *Engine) LoadFromFile(path string) error {
    data, err := os.ReadFile(path)
    if err != nil {
        return fmt.Errorf("read rules file: %w", err)
    }

    var doc struct {
        Rules []Rule `yaml:"rules"`
    }
    if err := yaml.Unmarshal(data, &doc); err != nil {
        return fmt.Errorf("parse yaml: %w", err)
    }

    // Pre-compile all expressions
    newPrograms := make(map[string]*vm.Program, len(doc.Rules)*2)
    for _, r := range doc.Rules {
        if !r.Enabled {
            continue
        }
        p, err := expr.Compile(r.When, expr.AsBool())
        if err != nil {
            return fmt.Errorf("compile rule %s: %w", r.ID, err)
        }
        newPrograms[r.ID] = p

        for i, ov := range r.Overrides {
            key := fmt.Sprintf("%s#override-%d", r.ID, i)
            p, err := expr.Compile(ov.When, expr.AsBool())
            if err != nil {
                return fmt.Errorf("compile override %s: %w", key, err)
            }
            newPrograms[key] = p
        }
    }

    // Atomic swap
    e.mu.Lock()
    defer e.mu.Unlock()
    e.rules = doc.Rules
    e.programs = newPrograms

    e.log.Info("rules loaded",
        zap.String("path", path),
        zap.Int("count", len(doc.Rules)))
    return nil
}

func (e *Engine) Evaluate(category string, ctx Context) Decision {
    e.mu.RLock()
    defer e.mu.RUnlock()

    // Collect matched rules sorted by priority DESC
    type match struct {
        rule Rule
        then map[string]interface{}
    }
    var matched []match

    for _, r := range e.rules {
        if r.Category != category || !r.Enabled {
            continue
        }
        prog, ok := e.programs[r.ID]
        if !ok {
            continue
        }
        result, err := expr.Run(prog, ctx)
        if err != nil {
            e.log.Warn("rule eval error",
                zap.String("rule_id", r.ID),
                zap.Error(err))
            continue
        }
        if pass, ok := result.(bool); !ok || !pass {
            continue
        }

        // Base then
        thenMap := copyMap(r.Then)

        // Apply overrides in order
        for i, ov := range r.Overrides {
            key := fmt.Sprintf("%s#override-%d", r.ID, i)
            ovProg := e.programs[key]
            ovResult, err := expr.Run(ovProg, ctx)
            if err == nil {
                if pass, ok := ovResult.(bool); ok && pass {
                    mergeMap(thenMap, ov.Then)
                }
            }
        }

        matched = append(matched, match{r, thenMap})
    }

    // Sort by priority desc
    sort.Slice(matched, func(i, j int) bool {
        return matched[i].rule.Priority > matched[j].rule.Priority
    })

    // Merge thens: lower priority loses on conflict (higher wins)
    merged := make(map[string]interface{})
    matchedIDs := make([]string, 0, len(matched))
    for i := len(matched) - 1; i >= 0; i-- {
        mergeMap(merged, matched[i].then)
        matchedIDs = append(matchedIDs, matched[i].rule.ID)
    }

    return Decision{Matched: matchedIDs, Values: merged}
}
```

#### Loader · hot reload với fsnotify

```go
// pkg/rules/loader.go
package rules

import (
    "github.com/fsnotify/fsnotify"
    "time"
)

func (e *Engine) WatchFile(ctx context.Context, path string) error {
    watcher, err := fsnotify.NewWatcher()
    if err != nil { return err }
    defer watcher.Close()

    if err := watcher.Add(filepath.Dir(path)); err != nil {
        return err
    }

    // Debounce: re-load tối đa 1 lần / 500ms
    var debounce *time.Timer
    reload := func() {
        if err := e.LoadFromFile(path); err != nil {
            e.log.Error("rules hot reload FAILED, keeping old rules",
                zap.Error(err))
            // alert Slack qua webhook
            return
        }
        e.log.Info("rules hot reloaded successfully")
    }

    for {
        select {
        case <-ctx.Done():
            return nil
        case event, ok := <-watcher.Events:
            if !ok { return nil }
            // ConfigMap mount uses symlinks. Watch for create/write.
            if event.Op&(fsnotify.Create|fsnotify.Write) != 0 &&
               event.Name == path {
                if debounce != nil { debounce.Stop() }
                debounce = time.AfterFunc(500*time.Millisecond, reload)
            }
        case err := <-watcher.Errors:
            e.log.Error("watcher error", zap.Error(err))
        }
    }
}
```

#### Usage trong service (DI)

```go
// cmd/dispatch-engine/main.go
func main() {
    log, _ := zap.NewProduction()

    // Load rules engine
    engine := rules.New(log)
    if err := engine.LoadFromFile("/etc/smp/rules/rules_engine.yaml"); err != nil {
        log.Fatal("initial rules load failed", zap.Error(err))
    }

    // Hot reload
    ctx := context.Background()
    go engine.WatchFile(ctx, "/etc/smp/rules/rules_engine.yaml")

    // Inject into services
    roundCtrl := dispatch.NewRoundController(engine, log, ...)
    server := http.NewServer(roundCtrl)
    server.Start()
}
```

#### Testing

```go
// pkg/rules/engine_test.go
func TestEngine_DispatchTimeout(t *testing.T) {
    log := zap.NewNop()
    e := rules.New(log)
    require.NoError(t, e.LoadFromFile("testdata/sample_rules.yaml"))

    tests := []struct {
        name string
        ctx  rules.Context
        want int
    }{
        {
            name: "VN prod = 60s",
            ctx:  rules.Context{"country_code": "VN", "env": "prod"},
            want: 60,
        },
        {
            name: "VN dev = 30s (override)",
            ctx:  rules.Context{"country_code": "VN", "env": "dev"},
            want: 30,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            d := e.Evaluate("dispatch", tt.ctx)
            assert.Equal(t, tt.want, d.GetInt("round_timeout_seconds", 0))
        })
    }
}
```

#### Required dependencies

```bash
go get github.com/expr-lang/expr
go get github.com/fsnotify/fsnotify
go get gopkg.in/yaml.v3
```

---

## 2. Local dev setup

### 2.1 Required tools

```bash
# Install
brew install go@1.22 mysql@8.0 redis mongodb-community docker docker-compose

# Verify
go version          # → go1.22.x
mysql --version     # → 8.0.x
redis-server --version
mongod --version
docker --version
```

**Windows**: dùng WSL2 + Ubuntu 22.04 LTS, sau đó install như macOS bằng `apt` hoặc `brew (linuxbrew)`.

### 2.2 IDE setup

**VS Code** (recommended):
- Extensions: `golang.go`, `ms-azuretools.vscode-docker`, `ms-vscode-remote.remote-containers`
- `.vscode/settings.json` (share trong repo):
```json
{
  "go.useLanguageServer": true,
  "go.lintTool": "golangci-lint",
  "go.lintOnSave": "workspace",
  "editor.formatOnSave": true,
  "[go]": { "editor.defaultFormatter": "golang.go" }
}
```

**GoLand**: enable `gofmt on save`, `golangci-lint` integration.

### 2.3 Repo clone & setup

```bash
# Clone all services (mono-folder setup)
mkdir -p ~/work/smp && cd ~/work/smp

for svc in api-gateway order-svc dispatch-engine catalog-svc agent-svc partner-svc finance-svc quality-svc integration-svc; do
  git clone git@github.com:trungnguyenchanh/smp-$svc.git
done

# Each service:
cd smp-order-svc
make setup           # download deps + install tools
make seed            # seed dev DB with master-data-v3.3.json
make test            # run unit tests
make run             # start local server :8081
```

### 2.4 Docker compose for infrastructure

Mỗi developer chạy infra locally bằng `docker-compose`:

```yaml
# ~/work/smp/infra/docker-compose.yml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: dev_root_pass
      MYSQL_DATABASE: smp_dev
    ports: ["3306:3306"]
    volumes: ["./mysql-data:/var/lib/mysql", "./init-sql:/docker-entrypoint-initdb.d"]
  
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  
  mongo:
    image: mongo:6
    ports: ["27017:27017"]
  
  kafka:
    image: bitnami/kafka:3.6
    ports: ["9092:9092"]
    environment:
      KAFKA_CFG_NODE_ID: 0
      KAFKA_CFG_PROCESS_ROLES: controller,broker
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@kafka:9093
  
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports: ["16686:16686", "4317:4317"]
```

Start: `cd infra && docker-compose up -d`

### 2.5 Environment variables

Mỗi service có `.env.example` (commit) và `.env` (gitignored):

```bash
# .env.example for smp-order-svc
SERVER_PORT=8081
SERVER_READ_TIMEOUT=30s

MYSQL_DSN=root:dev_root_pass@tcp(localhost:3306)/smp_order?parseTime=true&loc=Asia%2FHo_Chi_Minh
REDIS_ADDR=localhost:6379
MONGO_URI=mongodb://localhost:27017/smp_events

KAFKA_BROKERS=localhost:9092
KAFKA_CONSUMER_GROUP=order-svc-local

JWT_PUBLIC_KEY_PATH=./keys/jwt_public.pem

INSIDE_API_BASE=http://localhost:9001
WMS_API_BASE=http://localhost:9002

LOG_LEVEL=debug
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

Copy: `cp .env.example .env` và fill in.

### 2.6 Makefile common targets

```makefile
.PHONY: setup test build run lint migrate-up migrate-down seed clean

setup:
	@go mod download
	@go install github.com/golang/mock/mockgen@latest
	@go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest
	@go install github.com/swaggo/swag/cmd/swag@latest

test:
	@go test -race -coverprofile=coverage.out ./...

test-integration:
	@go test -tags=integration ./test/integration/...

build:
	@go build -o bin/server ./cmd/server

run:
	@go run ./cmd/server

lint:
	@golangci-lint run ./...

migrate-up:
	@migrate -database "$$MYSQL_DSN" -path migrations up

migrate-down:
	@migrate -database "$$MYSQL_DSN" -path migrations down 1

seed:
	@go run ./scripts/seed/main.go

clean:
	@rm -rf bin/ coverage.out
```

### 2.7 Initial seed data

Mỗi service seed từ master data JSON:
```bash
make seed   # gọi script load master-data-v3.3.json vào DB
```

Output sample:
```text
Seeded 8 skills
Seeded 22 steps
Seeded 20 material_types
Seeded 21 material_variants
Seeded 16 step_boms
Seeded 8 partners
Seeded 7 partner_admin_users
```

### 2.8 Run all services

```bash
# Terminal 1: infra
cd ~/work/smp/infra && docker-compose up

# Terminal 2-N: services (mỗi service 1 terminal)
cd ~/work/smp/smp-order-svc && make run     # :8081
cd ~/work/smp/smp-dispatch && make run      # :8082
cd ~/work/smp/smp-catalog && make run       # :8083
cd ~/work/smp/smp-agent-svc && make run     # :8084
cd ~/work/smp/smp-partner-svc && make run   # :8085
# ...

# Or use overmind/foreman:
overmind start -f Procfile.dev
```

### 2.9 Verify setup

```bash
curl http://localhost:8081/health
# → {"status":"ok","version":"3.3.0","commit":"abc123"}

curl http://localhost:8081/api/v1/catalog/services
# → [{"service_code":"svc_ac_repair","name":"Sửa điều hoà",...}]
```

### 2.10 Troubleshooting

| Issue | Fix |
|---|---|
| `mysql: connection refused` | Check `docker ps` MySQL running, `docker logs <id>` |
| `port already in use` | `lsof -ti:3306 \| xargs kill` |
| Migration fail | Check DB exists: `mysql -uroot -p < init-sql/create-dbs.sql` |
| Tests fail with race | Run `go test -race -v ./domain/order/` để pinpoint |
| Slow `go mod download` | Set `GOPROXY=https://goproxy.io,direct` (VN-friendly mirror) |

---

## 3. Git workflow

### 3.1 Branching strategy

Trunk-based với short-lived feature branches:
- `main` — protected, always deployable, auto-deploy to staging
- `feat/<ticket-id>-<short-desc>` — new feature, từ JIRA/Linear ticket
- `fix/<ticket-id>-<short-desc>` — bug fix
- `hotfix/<ticket-id>` — production emergency

Max branch life: 3 days. Sau đó force rebase + ship hoặc abandon.

### 3.2 Commit message (Conventional Commits)

```text
<type>(<scope>): <short summary>

<body explaining what and why>

<footer with ticket ref>
```

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`, `style`

Example:
```text
feat(order): support partner_customer source with private dispatch

Add Source, PartnerID, DispatchVisibility fields to order domain.
Validate partner exists and has sufficient wallet balance before insert.
Publish order.created event with partner metadata.

Refs: SMP-3142
```

### 3.3 PR rules

- Tên PR = commit summary
- Mô tả: link ticket, describe changes, screenshots if UI, breaking changes flagged
- Required reviewer: 1 from same squad, 1 from another (cross-pollination)
- CI must pass: lint + test + coverage không giảm
- Squash merge to main (1 PR = 1 commit on main)
- Delete branch after merge

### 3.4 Protected branches

`main` branch settings (GitHub):
- Require PR
- Require status checks: lint, test, security-scan
- Require linear history (no merge commits)
- Require signed commits (Phase 2)
- Restrict who can push: only via PR

### 3.5 Code review checklist

Reviewer check:
- [ ] Tests added/updated, all pass
- [ ] No SQL string concat
- [ ] Error wrapping has context
- [ ] No PII/secrets in logs
- [ ] No hardcoded URLs/keys
- [ ] OpenAPI updated nếu API thay đổi
- [ ] Migration script up + down
- [ ] Breaking changes documented in CHANGELOG.md
