# Deploy Guide · Push docs lên GitHub

**Repo đích**: https://github.com/trungnguyenchanh/smp-doc-process

Hướng dẫn này tương tự lần deploy `smp-mobile` và `smp-admin` trước. Làm 1 lần initial, các lần sau chỉ cần `git add → commit → push`.

---

## Phương án 1 · Tạo repo mới + push lần đầu (RECOMMENDED)

### Bước 1: Tạo repo trên GitHub
1. Vào https://github.com/new
2. Repository name: `smp-doc-process`
3. Owner: `trungnguyenchanh`
4. Visibility: **Private** (vì là internal docs)
5. ⚠️ **KHÔNG** tick "Add a README", "Add .gitignore", "Add license" — file đã có sẵn trong zip
6. Click **Create repository**

### Bước 2: Giải nén zip
```bash
# Giải nén ở thư mục bạn muốn (vd: ~/projects/)
cd ~/projects
unzip smp-doc-process-repo.zip
cd smp-doc-process-repo
```

### Bước 3: Init git + push
```bash
# Init repo local
git init
git branch -M main

# Add remote (dùng PAT cũ bạn đã tạo cho smp-mobile/admin)
git remote add origin https://github.com/trungnguyenchanh/smp-doc-process.git

# Stage + commit
git add .
git commit -m "docs: initial commit · 15 docs Batch A+B v3.4"

# Push lần đầu
git push -u origin main
```

Khi git prompt:
- Username: `trungnguyenchanh`
- Password: **Personal Access Token** (PAT cũ, scope `repo`)

✅ Done. Vào `https://github.com/trungnguyenchanh/smp-doc-process` để verify.

---

## Phương án 2 · Repo đã tồn tại (nếu bạn đã tạo trước)

```bash
# Clone repo về
cd ~/projects
git clone https://github.com/trungnguyenchanh/smp-doc-process.git
cd smp-doc-process

# Giải nén zip vào đây (overwrite README cũ nếu có)
unzip -o ~/Downloads/smp-doc-process-repo.zip -d .
# Hoặc copy nội dung từ thư mục giải nén:
# cp -r ~/Downloads/smp-doc-process-repo/* .
# cp ~/Downloads/smp-doc-process-repo/.gitignore .

# Commit + push
git add .
git commit -m "docs: add 15 process docs Batch A+B v3.4"
git push origin main
```

---

## Verify sau khi push

1. Vào https://github.com/trungnguyenchanh/smp-doc-process
2. Kiểm tra hiển thị 9 folders:
   - `01-architecture/`
   - `02-database/`
   - `03-backend/`
   - `04-frontend/` (empty placeholder, dành cho tương lai)
   - `05-ba/`
   - `06-qa/`
   - `07-devops/`
   - `08-security/`
   - `99-templates/` (empty placeholder)
3. `README.md` render tự động ở trang chính
4. Click 1 doc bất kỳ để verify markdown render đúng (bảng, code block)

---

## Deploy docs site lên GitHub Pages (MkDocs Material)

Repo đã có sẵn `mkdocs.yml` + GitHub Actions workflow tự deploy mỗi khi push lên main.

### Setup lần đầu (1 lần · ~3 phút)

1. **Vào repo settings**: https://github.com/trungnguyenchanh/smp-doc-process/settings/pages
2. **Source**: chọn **GitHub Actions** (không phải "Deploy from a branch")
3. Click **Save**

### Push code · auto-deploy

```bash
git add .
git commit -m "feat: add mkdocs config + github pages deploy"
git push origin main
```

GitHub Actions sẽ chạy workflow `.github/workflows/deploy-docs.yml`:
1. Install mkdocs-material + plugins
2. Build static site
3. Deploy lên GitHub Pages

Theo dõi build tại: `https://github.com/trungnguyenchanh/smp-doc-process/actions`

### Site URL

Sau khi deploy lần đầu (~2 phút), site sẽ available tại:
```
https://trungnguyenchanh.github.io/smp-doc-process/
```

(Optional) Custom domain: thêm `docs.smp.vn` trong Settings → Pages → Custom domain.

### Local preview

Để xem trước trước khi push:

```bash
pip install -r requirements-docs.txt
mkdocs serve
# Open http://127.0.0.1:8000
```

### Tính năng MkDocs Material

- ✅ Sidebar navigation tự động theo `mkdocs.yml`
- ✅ Search built-in (Vietnamese + English)
- ✅ Dark mode toggle
- ✅ Mobile responsive
- ✅ Code copy buttons
- ✅ Mermaid diagrams render
- ✅ Edit on GitHub button (mỗi page)
- ✅ Git revision date (last updated)

### Update docs

Mỗi PR merge → workflow chạy → site update tự động sau ~2 phút.

KHÔNG cần làm gì thêm sau setup. Toàn bộ team chỉ cần push markdown → site có ngay.

---

## (Optional) Tự build local · không qua MkDocs

## Troubleshooting

### Lỗi `Permission denied (publickey)`
Bạn đang dùng SSH URL. Switch sang HTTPS:
```bash
git remote set-url origin https://github.com/trungnguyenchanh/smp-doc-process.git
```

### Lỗi `Authentication failed`
PAT hết hạn hoặc sai scope. Tạo PAT mới tại:
https://github.com/settings/tokens → Generate new token (classic)
- Scope: ✅ `repo` (full control)
- Expiration: 90 ngày hoặc no expiration

### Lỗi `refusing to merge unrelated histories`
Nếu lúc tạo repo có tick "Add README":
```bash
git pull origin main --allow-unrelated-histories
# Resolve conflict (giữ README mới của zip)
git push origin main
```

---

## Maintenance sau khi setup

Mỗi lần sửa docs:
```bash
cd ~/projects/smp-doc-process
git pull origin main          # Lấy thay đổi mới nhất
# ... sửa file ...
git add .
git commit -m "docs(ba): update BR-DISP-003 timeout"
git push origin main
```

Xem [CONTRIBUTING.md](./CONTRIBUTING.md) cho quy trình PR đầy đủ.
