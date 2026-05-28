# SMP Glossary · Thuật ngữ thống nhất

**Audience**: All roles · BA, Dev, QC, DevOps, Security · **Updated**: v3.3

Bảng tra cứu thuật ngữ. Khi viết doc, comment code, hoặc giao tiếp team, dùng thống nhất các thuật ngữ này.

---

## A · Domain entities

| Thuật ngữ | Định nghĩa | Tương đương trong code | Notes |
|---|---|---|---|
| **Service Template** | Dịch vụ chuẩn (vd "Sửa điều hoà") = 1 chuỗi các Step | `services` table, `Service` struct | Khác Service trong "Microservice" |
| **Step** | Đơn vị công việc nguyên tử (vd "Thay tụ điện 35μF") | `steps`, `Step` | Có price theo level của thợ |
| **Skill** | Kỹ năng nghề (vd "ac-mechanic") | `skills`, `Skill` | Có 5 level L1-L5 |
| **Skill Level** | Trình độ trong 1 skill: L1 (mới vào) → L5 (chuyên gia) | `agent_skills.level` | Quyết định labor price |
| **Order** | 1 đơn hàng = 1 instance của Service Template + customer + agent | `orders`, `Order` | ID dạng `ord_01HX7K2M` |
| **Order Step** | Step nằm trong order (snapshot của master step) | `order_steps` | Lưu price tại thời điểm tạo |
| **Stage** | Giai đoạn của đơn hàng (01_created → 10_rated) | `orders.current_stage` | 10 stages trong v3.1 |
| **Agent** | Thợ thực hiện công việc | `agents`, `Agent` | KYC required, có skill matrix |
| **Customer** | Khách hàng cuối cá nhân | Ref `customer_id` to inside | SMP không own data |
| **Partner** | Đối tác B2B hoặc cá nhân lớn (v3.3) | `partners`, `Partner` | 3 types: A, B, AB |
| **End Customer** | Khách cuối của partner (vd cư dân Vinhomes) | `orders.end_customer_id` | Khác Customer (account holder) |
| **Material Type** | Loại vật tư (vd "Tụ điện 35μF") | `material_types` | Cha — generic |
| **Material Variant** | SKU cụ thể (vd "Sanyo CBB60-35") | `material_variants` | Con — có brand, model, giá |
| **BOM** (Bill of Materials) | Danh sách vật tư cần cho 1 step | `step_boms` | Định nghĩa: 1 step → N material_types |
| **Asset** (Customer Asset) | Thiết bị của khách có lịch sử (vd máy AC LG ở phòng khách) | `customer_assets` | Gắn QR code |
| **Maintenance Contract** | Hợp đồng bảo trì định kỳ | `maintenance_contracts` | Auto-create orders theo schedule |

## B · Agent roles (trong 1 order)

| Thuật ngữ | Định nghĩa |
|---|---|
| **Survey Agent (SUR)** | Thợ đi khảo sát trước (v3.1) — thu thông tin, ảnh, báo giá. Color: violet `#5b21b6` |
| **Execution Agent (EXC)** | Thợ đi thực hiện sửa chữa sau khi khách duyệt báo giá. Color: teal `#1D9E75` |
| **Dual Agent** | 1 thợ đảm nhận cả Survey + Execution (cùng người) |
| **Pool Agent** | Thợ trong pool dispatch chung, chưa assign |

## C · Dispatch terminology

| Thuật ngữ | Định nghĩa |
|---|---|
| **Dispatch** | Quá trình tìm và giao đơn cho thợ phù hợp |
| **Round** | 1 vòng dispatch · gửi đơn đến N thợ, đợi 60s, hết → round tiếp theo |
| **Invitation** | Lời mời thợ nhận đơn (push notification) |
| **Auto dispatch** | Hệ thống tự assign agent đầu tiên accept |
| **Pool dispatch** | Đơn show trong "Pool" tab cho thợ tự chọn |
| **Private dispatch** | Đơn chỉ dispatch cho thợ của 1 partner cụ thể (v3.3) |
| **Open dispatch** | Đơn dispatch cho mọi thợ qualified (default) |
| **Escalate** | Sau N rounds fail → đơn chuyển sang ops handle thủ công |

## D · Partner concepts (v3.3)

| Thuật ngữ | Định nghĩa |
|---|---|
| **Partner Type** | `business` (B2B) hoặc `individual_large` (cá nhân lớn vd chủ 12 căn cho thuê) |
| **Partner Role** | `customer` (Type A) / `supplier` (Type B) / cả hai (Type AB) |
| **Customer-side** | Vai trò partner đặt đơn cho khách của họ |
| **Supplier-side** | Vai trò partner cung cấp đội thợ |
| **Pre-paid wallet** | Hình thức partner nạp tiền trước, mỗi đơn trừ |
| **Monthly invoice** | Hình thức SMP xuất hoá đơn cuối tháng, NET 30 |
| **SaaS model** | Partner trả phí hàng tháng cố định, thợ nhận 100% |
| **Commission model** | SMP/Partner rút % mỗi đơn |
| **Hybrid model** | Kết hợp SaaS + commission, giá rẻ hơn |
| **Passthrough payout** | Tiền công thợ → SMP trả thẳng vào TK thợ |
| **Via-partner payout** | Tiền công thợ → SMP trả vào TK partner, partner trả thợ off-platform |
| **Journey mode** | `full_10steps` (khách thấy app SMP) hoặc `minimal` (B2B, khách không thấy app) |

## E · Integration terms (v3.2)

| Thuật ngữ | Định nghĩa |
|---|---|
| **inside** | Hệ thống external sở hữu Customer + Payment + Voucher |
| **wms** | Hệ thống external sở hữu Stock + Warehouse |
| **Event bus** | Kafka message broker để pub/sub events giữa các service |
| **Webhook** | HTTP callback từ inside/wms về SMP khi có event |
| **Reservation** | Reserve vật tư trong wms khi quote duyệt (TTL 24h) |
| **Commit out** | Trừ stock thật trong wms khi step xong |
| **Free-form material** | Vật tư thợ nhập tay không có trong catalog → cần ops verify |
| **BOM variance** | Vật tư actual khác với BOM expected → flag review |

## F · Financial terms

| Thuật ngữ | Định nghĩa |
|---|---|
| **GMV** (Gross Merchandise Value) | Tổng giá trị giao dịch (chưa trừ commission/refund) |
| **Labor price** | Tiền công thợ theo step + level |
| **Material price** | Giá bán vật tư = `material_variants.sell_price` (SMP set) |
| **Cost price** | Giá nhập vật tư (từ wms, chỉ tham khảo) |
| **Margin** | (sell_price − cost_price) / cost_price |
| **Surge** | Phụ phí giờ cao điểm hoặc khẩn cấp |
| **Commission** | % SMP rút từ doanh thu thợ |
| **Payout** | Khoản tiền SMP trả thợ/partner định kỳ |
| **Outstanding invoice** | Hoá đơn đã xuất chưa thu |

## G · Quality & operations

| Thuật ngữ | Định nghĩa |
|---|---|
| **Photo proof** | Ảnh chứng minh thợ đã làm step (before/after) |
| **Survey result** | Khảo sát chất lượng dịch vụ của khách cuối |
| **Rating** | Điểm khách đánh giá thợ 1-5 sao |
| **Dispute** | Tranh chấp giữa khách-thợ-SMP |
| **Material verification** | Ops review vật tư thợ nhập có khớp BOM không |
| **KYC** (Know Your Customer) | Xác minh danh tính (agent + partner) |
| **KYC Level Basic** | 3 docs cơ bản: CCCD + GPKD + STK |
| **KYC Level Full** | Basic + hợp đồng + bảo hiểm trách nhiệm |
| **Onboarding** | Quy trình đưa agent/partner mới vào hệ thống |

## H · Technical terms

| Thuật ngữ | Định nghĩa |
|---|---|
| **API Gateway** | Single entry point cho mọi request từ client |
| **Service** (microservice) | 1 deploy unit, sở hữu 1 DB (vd order-svc) |
| **Event** | Message trên Kafka, tên dạng `<service>.<resource>.<action>` (vd `order.created`) |
| **Idempotency** | Cùng request gửi 2 lần → kết quả như 1 lần (qua `Idempotency-Key` header) |
| **Eventual consistency** | Data ở 2 service không đồng bộ ngay, eventually sync qua events |
| **Saga** | Pattern handle distributed transaction qua chuỗi local transactions + compensations |
| **Circuit breaker** | Pattern tự stop call service đang down để tránh cascade failure |
| **Rate limit** | Giới hạn request/giây để chống abuse |
| **DLQ** (Dead Letter Queue) | Queue chứa message không xử lý được, cần manual review |
| **RBAC** (Role-Based Access Control) | Phân quyền theo role |
| **Scope** | 1 quyền cụ thể (vd `orders.create`) |
| **JWT** | Token định danh user, signed by issuer |
| **ULID** | Format ID 26 chars, sortable by time (thay UUID) |

## I · Geographic terms

| Thuật ngữ | Định nghĩa |
|---|---|
| **Province** | Tỉnh/Thành phố (vd HCMC = VN-79) |
| **District** | Quận/Huyện (vd Q.1, Thủ Đức) |
| **Ward** | Phường/Xã |
| **Coverage zone** | Khu vực phục vụ, có agents on standby, có surge multiplier riêng |
| **Service area** | Tập hợp các zone mà 1 service có hoạt động |
| **Travel buffer** | Thời gian thợ di chuyển giữa các đơn (default 30 phút) |

## J · Version & release

| Thuật ngữ | Định nghĩa |
|---|---|
| **v3.1** | 2-Agent Flow (Survey + Execution) — 10 stages |
| **v3.2** | Integration với inside + wms + Material BOM |
| **v3.3** | Partner platform 3-sided marketplace |
| **Pilot** | Giai đoạn chạy thử với 3 partners + 1 service area |
| **GA** (General Availability) | Release chính thức cho mọi khách |
| **Hotfix** | Sửa khẩn cấp trên production |

## K · Vietnamese-specific terms

| Tiếng Việt | English equivalent | Notes |
|---|---|---|
| Thợ | Agent / Technician | Trong code dùng `agent`, UI VN dùng "Thợ" |
| Khách | Customer | UI customer-facing |
| Đơn / Đơn hàng | Order | |
| Giai đoạn | Stage | Của order lifecycle |
| Bước | Step | Trong service |
| Khảo sát | Survey | Bước khảo sát của Survey Agent |
| Báo giá | Quote | Bước báo giá khách duyệt |
| Hợp đồng (bảo trì) | Maintenance Contract | |
| Tài sản (của khách) | Customer Asset | Máy có lịch sử |
| Vật tư | Material | |
| Vật tư cha / con | Material Type / Variant | v3.2 model 2 tầng |
| Hoá đơn | Invoice | Partner monthly invoice |
| Ví | Wallet | Pre-paid wallet |
| Chiết khấu | Discount | |
| Phụ phí | Surge fee | |
| Tỉ lệ ăn chia | Commission rate | |
| Đơn riêng | Private order | Partner private dispatch |
| Đơn mở | Open order | Partner open dispatch |
| Phí SaaS | SaaS fee | |
| Đối tác | Partner | |

## L · Status enums

### Order status
| Code | Vietnamese | Meaning |
|---|---|---|
| `01_created` | Đã tạo | Đơn vừa tạo, chờ dispatch |
| `02_dispatched_survey` | Đã dispatch khảo sát | Đang tìm Survey Agent |
| `03_survey_accepted` | Thợ khảo sát accept | SUR sắp đến |
| `04_arrived_survey` | Đã đến khảo sát | SUR check-in |
| `05_surveyed` | Đã khảo sát xong | Đã có báo giá |
| `06_quote_approved` | Khách duyệt báo giá | Sẵn sàng dispatch execution |
| `07_dispatched_execution` | Đã dispatch sửa chữa | Đang tìm EXC |
| `08_in_progress` | Đang sửa | EXC làm việc |
| `09_completed` | Hoàn thành | Khách xác nhận |
| `10_rated` | Đã đánh giá | Đơn close cycle |
| `cancelled` | Đã huỷ | Bất kỳ stage nào |

### Agent status
- `pending_kyc` — chưa hoàn thành KYC
- `active` — hoạt động bình thường
- `suspended` — tạm dừng (vi phạm, ốm, vv.)
- `terminated` — chấm dứt

### Partner status
- `pending_kyc` — chưa hoàn thành KYC
- `active` — hoạt động
- `suspended` — tạm dừng
- `terminated` — chấm dứt hợp tác

### Payment status (mirror từ inside)
- `pending` — chờ thanh toán
- `paid` — đã trả
- `refunded` — đã hoàn
- `failed` — fail

## M · Acronyms tóm tắt

| Acronym | Full form |
|---|---|
| SMP | Service Management Platform |
| SUR | Survey Agent |
| EXC | Execution Agent |
| BOM | Bill of Materials |
| SKU | Stock Keeping Unit (từ wms) |
| KYC | Know Your Customer |
| RBAC | Role-Based Access Control |
| GMV | Gross Merchandise Value |
| GPKD | Giấy Phép Kinh Doanh |
| CCCD | Căn Cước Công Dân |
| STK | Số Tài Khoản |
| NET 30 | Net 30 days payment terms |
| TTL | Time To Live |
| DLQ | Dead Letter Queue |
| SLA | Service Level Agreement |
| SLO | Service Level Objective |
| RPO | Recovery Point Objective |
| RTO | Recovery Time Objective |
| PII | Personally Identifiable Information |
| PDPA | Personal Data Protection Act (VN) |
| CPRA | California Privacy Rights Act (US) |
| PIPL | Personal Information Protection Law (China) |
| GDPR | General Data Protection Regulation (EU) |
| ISO 4217 | Currency codes standard |
| ISO 3166 | Country codes standard |
| ISO 639 | Language codes standard |
| BCP 47 | Locale tag standard (vd `vi-VN`, `en-US`) |
| IANA tz | Timezone database (vd `Asia/Ho_Chi_Minh`) |
| UTC | Coordinated Universal Time |
| CQRS | Command Query Responsibility Segregation |
| CDC | Change Data Capture |
| YAML | Yet Another Markup Language |

---

## N · Internationalization (i18n) terms · v4.0

> Các thuật ngữ này phục vụ chuẩn bị scale ra nước ngoài. v3.x hiện chỉ support Vietnam, nhưng schema + code design phải ready cho v4.0.

| Thuật ngữ | Định nghĩa |
|---|---|
| **ISO 4217 Currency Code** | Chuẩn 3-letter code cho tiền tệ. Ví dụ: `VND`, `USD`, `CNY`, `THB`, `IDR`, `PHP`, `SGD`, `MYR`, `EUR`, `JPY`. **Bắt buộc** mọi cột tiền lưu kèm currency_code. |
| **ISO 3166-1 Alpha-2 Country Code** | Chuẩn 2-letter code cho quốc gia. Ví dụ: `VN`, `US`, `CN`, `TH`, `ID`, `PH`, `SG`, `MY`. Dùng cho sharding, pricing rules per country, tax rules. |
| **ISO 639-1 Language Code** | Chuẩn 2-letter code cho ngôn ngữ. Ví dụ: `vi`, `en`, `zh`, `th`, `id`. Dùng cho UI translation, notification template, glossary translation. |
| **BCP 47 Locale Tag** | Combo `<language>-<region>`. Ví dụ: `vi-VN` (Vietnamese, Vietnam), `en-US` (English, US), `zh-CN` (Chinese, China). Dùng để chọn ngôn ngữ + format số/ngày/giờ phù hợp. |
| **UTC (Coordinated Universal Time)** | Múi giờ chuẩn quốc tế. **Mọi timestamp trong DB lưu UTC**, không lưu local time. Convert sang local chỉ ở tầng API Gateway / Frontend. |
| **Local Time** | Giờ địa phương theo timezone user. Tính bằng UTC + offset (vd `Asia/Ho_Chi_Minh` = UTC+7). |
| **IANA Timezone Database (tz)** | Database chuẩn timezones do IANA maintain. Identifier dạng `<Region>/<City>`. Ví dụ: `Asia/Ho_Chi_Minh`, `America/Los_Angeles`, `Asia/Shanghai`, `Asia/Bangkok`. |
| **Tax Configuration** | Cấu hình thuế suất linh hoạt theo country + service category. VN: VAT 10% (mặc định), 8% (giảm tạm thời), 5% (một số ngành). US: Sales Tax theo bang. SG: GST 9%. |
| **Currency Conversion Rate** | Tỷ giá quy đổi giữa các currency. Lưu lịch sử theo ngày để báo cáo nhất quán. Source: ngân hàng trung ương hoặc API như exchangerate-api.com. |
| **Money Type** | Struct `{ amount: BIGINT, currency_code: CHAR(3) }`. Không bao giờ lưu money như FLOAT (sai số). Amount lưu dạng minor units (cents/đồng, không decimal). |
| **Multi-tenant** | Một codebase phục vụ nhiều "tenants" (quốc gia/khách lớn) độc lập. Có thể chia logical (cùng DB, partition by tenant_id) hoặc physical (mỗi tenant 1 DB cluster). |
| **i18n Dictionary (Open Dictionary)** | Thay vì hardcode chuỗi tiếng Việt trong code, lưu vào `i18n_translations` table với key. Frontend lookup theo `(key, locale)`. Vd: key `order.status.in_progress`, locale `vi-VN` → "Đang thực hiện", `en-US` → "In progress". |
| **Locale Fallback** | Nếu thiếu translation, fallback theo chain: requested locale → language only → default (en-US). Vd thiếu `zh-HK` → thử `zh` → fallback `en-US`. |

---

## O · Rules Engine terms · v4.0

> Hệ thống không hardcode business rules vào Go code. Thay vào đó, dùng file YAML + library `antonmedv/expr` để eval expressions runtime. Cho phép update policy không cần deploy lại.

| Thuật ngữ | Định nghĩa |
|---|---|
| **Rules Engine** | Component đọc rules từ file YAML, eval expressions với context input, return decision. Implementation: thư viện Go `antonmedv/expr` (lightweight, ~20kb). |
| **Rule** | 1 quy tắc nghiệp vụ. Gồm `id`, `when` (predicate), `then` (action/value), `priority`, `enabled`. Vd rule "dispatch round timeout = 60s nếu country=VN, 90s nếu country=US". |
| **Expression** | Biểu thức logic/toán học. Cú pháp `antonmedv/expr` tương tự JavaScript. Vd: `order.country == 'VN' && agent.rating >= 4.5`. |
| **Predicate** | Expression trả về boolean. Dùng trong field `when`. Nếu predicate eval = true → rule active. |
| **Context** | Input data cho engine khi eval rule. Là 1 map/struct chứa các biến rule có thể reference. Vd `{ order, agent, customer, now }`. |
| **Action / Value** | Output của rule. Có thể là 1 số (multiplier, fee), 1 string (status), hoặc 1 object (config). |
| **Rule Priority** | Nếu nhiều rule match cùng lúc, rule có `priority` cao hơn thắng. Default = 100. Rules đặc biệt set priority 1-10 hoặc 200+. |
| **Hot Reload** | Khả năng update rules trong production mà không restart service. Cơ chế: file watcher monitor ConfigMap mount path, reload vào memory khi file changes. |
| **ConfigMap** | Kubernetes resource lưu config (key-value hoặc whole files). Mount vào pod như volume. Update ConfigMap → file ở pod tự update sau ~30s. |
| **Config-as-Code** | Pattern: config + rules được version control trong Git, review qua PR, deploy qua CD. Không edit live trong UI admin (tránh sai sót, dễ rollback). |
| **antonmedv/expr** | Thư viện Go open source eval expression. Type-safe, sandbox (không exec arbitrary Go code), benchmark ~1µs/eval. Repo: https://github.com/expr-lang/expr |
| **Decision Table** | Cách biểu diễn rules dưới dạng bảng (input columns → output column). Tools như GoRules Zen có hỗ trợ. Hiện SMP chưa dùng — chỉ dùng YAML rules list. |

---

## P · Event-Driven & CQRS terms · v4.0

> Kiến trúc hiện tại (v3.x) chủ yếu request-response sync. v4.0 chuyển sang event-driven với Kafka cho dispatch + pricing, CDC cho reporting.

| Thuật ngữ | Định nghĩa |
|---|---|
| **Event** | 1 fact đã xảy ra trong hệ thống. Immutable. Vd `OrderCreated`, `AgentAccepted`, `PaymentSucceeded`. Format JSON với schema versioning. |
| **Event Schema** | Cấu trúc cố định của 1 event type. Có schema_version để backward-compatible khi evolve. Lưu trong Schema Registry (Confluent). |
| **Producer** | Service phát ra events. Vd `order-svc` produces `OrderCreated`. |
| **Consumer** | Service subscribe events. Vd `dispatch-engine`, `notification-svc` cùng consume `OrderCreated`. |
| **Topic** | Logical stream của Kafka. Group các events cùng loại. Vd topic `orders.events`, `payments.events`, `agents.events`. |
| **Partition** | Sub-stream trong 1 topic. Cho phép parallel processing. Số partition = parallelism tối đa. Khuyến nghị: 6-12 partitions/topic. |
| **Partition Key** | Field dùng để route event vào partition cụ thể. Cùng key → cùng partition → preserve order. SMP dùng `order_id` làm key cho orders.events. |
| **Consumer Group** | Group các consumer instance share workload. 1 partition chỉ assign cho 1 consumer trong group. Vd dispatch-engine có 3 replicas → cùng consumer group `dispatch-engine-cg`. |
| **Offset** | Position của consumer trong partition. Commit offset = "đã xử lý xong tới đây". Restart vẫn tiếp tục từ offset. |
| **At-least-once delivery** | Mặc định Kafka. Event có thể delivered nhiều lần nếu consumer crash trước commit. Consumer phải **idempotent**. |
| **Exactly-once semantics (EOS)** | Kafka transaction guarantee. Setup phức tạp + slower. SMP chọn at-least-once + idempotency cho đơn giản. |
| **Dead Letter Queue (DLQ)** | Topic riêng lưu events fail xử lý nhiều lần. Cho phép review thủ công sau. Naming: `<topic>.dlq`. |
| **CQRS (Command Query Responsibility Segregation)** | Pattern tách Write (Commands → MySQL) và Read (Queries → Elasticsearch/ClickHouse). Cho phép optimize 2 luồng riêng biệt. |
| **Write Model** | Schema optimized cho transaction (normalized, MySQL). Chỉ phục vụ ghi và validate. |
| **Read Model** | Schema optimized cho query (denormalized, Elasticsearch). Phục vụ dashboard, search, report. Sync từ Write Model qua CDC. |
| **CDC (Change Data Capture)** | Capture mọi INSERT/UPDATE/DELETE từ DB primary, publish sang Kafka. Tool chuẩn: **Debezium** (free, open source). |
| **Debezium** | Open source CDC connector cho MySQL/Postgres/Mongo. Đọc binlog → publish Kafka events. Production-grade, used by Uber/Netflix. |
| **Eventual Consistency** | Read Model "catch up" với Write Model sau 1-5 giây. Khi user đặt đơn xong → ngay UI confirm (Write OK) nhưng dashboard analytics sẽ update sau vài giây. |
| **Saga Pattern** | Quản lý long-running transaction qua nhiều services bằng sequence of events + compensation. Vd: nếu PaymentFailed sau OrderCreated → emit OrderCancelled (compensating action). |
| **Outbox Pattern** | Để đảm bảo "ghi DB + publish event" atomic, dùng `outbox` table. App ghi event vào outbox trong cùng transaction với business data. Debezium CDC pickup outbox → publish Kafka. |
| **Idempotency Key** | Header `Idempotency-Key` ở API request. Server cache result 24h theo key. Cùng key gọi lại → return cached response. Required cho payment, order create. |

---

## Q · Security & Compliance terms · v4.0

> Khi scale global, các luật bảo vệ dữ liệu khác nhau giữa quốc gia. SMP phải design system tuân thủ được PDPA (VN), CPRA (US), PIPL (China), GDPR (EU).

| Thuật ngữ | Định nghĩa |
|---|---|
| **PII (Personally Identifiable Information)** | Dữ liệu định danh cá nhân: tên, số điện thoại, email, địa chỉ, CCCD, STK ngân hàng. Bắt buộc bảo vệ ở mọi layer. |
| **Dynamic Data Masking** | Che dữ liệu PII khi return từ API tùy theo scope của user gọi. Vd CSKH thấy `0912****890`, Finance Admin thấy full `0912345890`. Implement tại API Gateway middleware. |
| **Masking Rule** | Cấu hình che dữ liệu cho 1 field. Format: `{field: 'phone', pattern: 'first_3_last_3', required_scope: 'pii.unmask.phone'}`. |
| **Tokenization** | Thay PII bằng token random, lưu mapping trong vault riêng. Mạnh hơn masking. Dùng cho card number (PCI-DSS). |
| **Data Sovereignty** | Yêu cầu pháp lý: dữ liệu công dân nước X phải lưu trên server đặt tại nước X. Ví dụ: PIPL (TQ) yêu cầu data citizen TQ lưu tại TQ; GDPR (EU) cho phép xuyên biên giới với điều kiện. |
| **Data Residency** | Concept tương tự sovereignty nhưng nhẹ hơn, do contract chứ không phải luật. |
| **Sharding by Country** | Strategy chia DB physical theo quốc gia. Vd: cluster `smp-vn` ở Singapore (gần VN), cluster `smp-cn` ở Beijing, cluster `smp-us` ở Virginia. Routing theo `country_code` của user. |
| **Logical Sharding** | Chia trong cùng 1 DB cluster bằng partition theo column. Nhẹ hơn physical sharding. Vd partition `orders` table by `(country_code, created_at)`. |
| **Right to Erasure** | Quyền "bị quên" của user. Tất cả PII của họ phải xóa khỏi system trong X ngày sau khi yêu cầu. PDPA VN: 30 ngày. GDPR: 30 ngày. CPRA: 45 ngày. |
| **Right to Data Portability** | User có quyền yêu cầu export toàn bộ data của họ dạng máy đọc được (JSON/CSV). Trả trong 7 ngày (PDPA), 30 ngày (GDPR). |
| **Consent Management** | Hệ thống lưu trữ và verify user đã đồng ý cụ thể với từng mục đích xử lý data (vd: marketing, analytics, sharing với partner). Audit-able. |
| **PDPA (Vietnam)** | Personal Data Protection Act. Nghị định 13/2023/NĐ-CP. Hiệu lực 1/7/2023. Yêu cầu: consent, breach notification 72h, DPO appointment. |
| **CPRA (California, US)** | California Privacy Rights Act. Mở rộng từ CCPA. Yêu cầu opt-out of sale, sensitive PII categories, breach response. |
| **PIPL (China)** | Personal Information Protection Law. Hiệu lực 1/11/2021. Strict nhất: data localization mandatory, cross-border transfer cần security assessment của CAC. |
| **GDPR (EU)** | General Data Protection Regulation. Hiệu lực 2018. Phạt max 4% revenue toàn cầu. |
| **DPO (Data Protection Officer)** | Vai trò bắt buộc theo PDPA/GDPR. Responsible cho compliance, breach response, user requests. |
| **Reconciliation (Đối soát)** | Quy trình kiểm tra khớp 2 bộ data từ 2 sources khác nhau. Vd đối soát partner wallet (SMP DB) vs payment gateway transactions. Daily job. |
| **Automated Reconciliation** | Job tự động chạy hàng ngày, phát hiện mismatch (vd wallet SMP báo -10M nhưng gateway báo -10.1M), alert tới Finance. |
| **Fraud Detection Rules** | Rules engine pattern matching trên transaction stream để flag suspicious: top up từ nhiều IP, multiple wallets cùng device, abnormal velocity. |
| **Breach Notification** | Khi data breach xảy ra, phải báo cáo authority + users bị ảnh hưởng trong 72h (PDPA/GDPR), 60 days (CPRA). |
