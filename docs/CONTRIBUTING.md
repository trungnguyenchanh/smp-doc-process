# Contributing to SMP Process Documentation

## Quy tắc chung

Tài liệu này là **living document**. Khuyến khích mọi thành viên đóng góp khi:
- Phát hiện sai sót / thông tin lỗi thời
- Cập nhật business rule mới
- Thêm runbook cho incident mới gặp
- Refine acceptance criteria sau khi feature implement xong

## Quy trình PR

### 1. Tạo branch
```bash
git checkout -b docs/fix-typo-glossary
# Naming: docs/<short-desc>
```

### 2. Sửa nội dung
- Giữ format Markdown chuẩn
- Tiếng Việt đầy đủ dấu (UTF-8, không xài Telex/VNI escape)
- Code block phải có language tag (```go, ```sql, ```yaml)
- Internal link dùng relative path: `[doc](./05-ba/05-glossary.md)`

### 3. Update version + changelog
Nếu thay đổi đáng kể (không phải typo):
- Mở `README.md` → cập nhật bảng changelog ở cuối
- Format: `| 3.4.1 | 2026-05-30 | Sửa BR-DISP-003 timeout | @username |`

### 4. Commit message
Theo [Conventional Commits](https://www.conventionalcommits.org/):
```text
docs(ba): cập nhật BR-DISP-003 timeout từ 60s → 45s

- Sau khi load test, 45s đủ cho 95% case
- Reference: incident INC-2026-05-15
```

Types: `docs`, `fix`, `feat`, `chore`

### 5. Tạo PR
- Title: cùng format commit message
- Body: mô tả why + screenshot nếu có diagram
- Reviewers: **1 dev + 1 BA** (cross-discipline)
- Label: `area:<role>` (vd `area:ba`, `area:devops`)

### 6. Merge
- Squash & merge (giữ history sạch)
- Sau merge: notify `#engineering-docs` trên Slack với link PR

## Khi nào không cần PR

- Sửa typo trong file của riêng mình → push thẳng main (nếu owner)
- Tuy nhiên, **đụng đến business rules / API contract / DB schema → bắt buộc PR**

## Style guide nhanh

### Tiêu đề
- H1 chỉ dùng cho doc title (1 lần duy nhất)
- H2 cho section chính
- H3-H4 cho sub-section
- Không H5/H6 (quá sâu, nên tách doc)

### Bảng
- Header bắt buộc có dấu `---` separator
- Align column rõ ràng

### Reference giữa docs
Khi nói "xem doc khác", luôn dùng:
- Format: `[Doc 02 · Database Schema](./02-database/02-database-schema.md)`
- Không viết "xem file database schema" (vague)

### Rule ID
Mọi business rule đều có ID `BR-<CATEGORY>-<NUM>`:
- `BR-DISP-001` (dispatch)
- `BR-PRICE-005` (pricing)
- `BR-KYC-003` (KYC)

Khi reference rule này từ code/test:
```go
// BR-DISP-001: round timeout 60s
const DispatchRoundTimeout = 60 * time.Second
```

## Câu hỏi?

Slack: `#engineering-docs` · Email: docs@smp.vn
