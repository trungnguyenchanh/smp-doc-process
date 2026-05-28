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

```
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
func (m Money) MulRate(rate float64) Money {
    return Money{Amount: int64(float64(m.Amount) * rate), Currency: m.Currency}
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

### 1.16 v4.0 pattern · Rules Engine (`pkg/rules`)

> Pattern này dùng cho services có business logic thay đổi theo policy (dispatch-engine, finance-svc, catalog-svc). Thay vì hardcode rule trong Go, load từ YAML config.

#### Package structure

```
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
```
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

```
<type>(<scope>): <short summary>

<body explaining what and why>

<footer with ticket ref>
```

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`, `style`

Example:
```
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
