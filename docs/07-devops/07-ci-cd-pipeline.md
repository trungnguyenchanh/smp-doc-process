# SMP CI/CD Pipeline

**Tooling**: GitHub Actions + ArgoCD + Helm · **Audience**: DevOps, Backend

---

## 1. Pipeline overview

```
Code push (feature branch)
       │
       ▼
┌──────────────┐
│ GH Actions   │ CI: lint → test → build → scan
│ (PR pipeline)│
└──────┬───────┘
       │ PR approved + merged
       ▼
┌──────────────┐
│ GH Actions   │ CD: build image → push to registry → update GitOps repo
│ (main branch)│
└──────┬───────┘
       │ commit to gitops-deploys repo
       ▼
┌──────────────┐
│ ArgoCD       │ Sync: pull manifest → apply to K8s cluster
│ (auto-sync)  │
└──────┬───────┘
       │
       ▼
   Dev env updated
       │ manual promote
       ▼
   Staging env updated (after manual approve)
       │ manual promote
       ▼
   Prod env updated (after release manager approve)
```

## 2. Branch & deploy mapping

| Branch | Trigger | Deploys to | Auto/Manual |
|---|---|---|---|
| `feat/*`, `fix/*` | PR open | (CI only, no deploy) | Auto CI |
| `main` | Merge | dev cluster | Auto |
| Tag `v*.*.*-rc.N` | Tag push | staging cluster | Auto |
| Tag `v*.*.*` | Tag push | prod cluster | Manual approve |

## 3. CI: PR pipeline (`.github/workflows/ci.yml`)

```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

env:
  GO_VERSION: "1.22"

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: gofmt check
        run: |
          fmt_files=$(gofmt -l .)
          if [ -n "$fmt_files" ]; then
            echo "Unformatted files: $fmt_files"
            exit 1
          fi
      - name: go vet
        run: go vet ./...
      - uses: golangci/golangci-lint-action@v3
        with:
          version: latest
          args: --timeout=5m

  test:
    name: Unit + integration tests
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: smp_test
        ports: ["3306:3306"]
        options: >-
          --health-cmd "mysqladmin ping" --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: Run migrations
        run: |
          go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest
          migrate -database "mysql://root:test@tcp(localhost:3306)/smp_test" -path migrations up
      - name: Unit tests
        run: go test -race -coverprofile=coverage.out ./...
      - name: Integration tests
        run: go test -tags=integration ./test/integration/...
        env:
          MYSQL_DSN: root:test@tcp(localhost:3306)/smp_test?parseTime=true
          REDIS_ADDR: localhost:6379
      - name: Coverage check
        run: |
          cov=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
          if (( $(echo "$cov < 60" | bc -l) )); then
            echo "Coverage $cov% below threshold 60%"
            exit 1
          fi
      - uses: codecov/codecov-action@v4
        with: { file: coverage.out }

  security:
    name: Security scans
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Gitleaks
        uses: gitleaks/gitleaks-action@v2
      - name: govulncheck
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...
      - name: Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: 'HIGH,CRITICAL'
          exit-code: '1'

  openapi:
    name: OpenAPI validation
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Spectral lint
        uses: stoplightio/spectral-action@latest
        with:
          file_glob: "api/openapi.yaml"
      - name: Breaking change check (vs main)
        run: |
          npx -y @apidevtools/swagger-cli validate api/openapi.yaml
```

## 4. CD: Build & deploy (`.github/workflows/cd.yml`)

```yaml
name: CD

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,format=short
            type=semver,pattern={{version}}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true

  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: trungnguyenchanh/smp-gitops
          token: ${{ secrets.GITOPS_PAT }}
          path: gitops
      - name: Update image tag in dev manifest
        run: |
          cd gitops/environments/dev
          yq e -i ".image.tag = \"${{ needs.build.outputs.image_tag }}\"" ${{ github.event.repository.name }}.yaml
          git config user.email "ci@smp.vn"
          git config user.name "ci-bot"
          git add . && git commit -m "deploy(dev): ${{ github.event.repository.name }} → ${{ needs.build.outputs.image_tag }}"
          git push

  deploy-staging:
    needs: build
    if: startsWith(github.ref, 'refs/tags/v') && contains(github.ref, '-rc.')
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging-api.smp.vn
    steps:
      - uses: actions/checkout@v4
        with:
          repository: trungnguyenchanh/smp-gitops
          token: ${{ secrets.GITOPS_PAT }}
          path: gitops
      - name: Update staging manifest
        run: |
          cd gitops/environments/staging
          yq e -i ".image.tag = \"${{ needs.build.outputs.image_tag }}\"" ${{ github.event.repository.name }}.yaml
          git add . && git commit -m "deploy(staging): ${{ github.event.repository.name }} → ${{ needs.build.outputs.image_tag }}"
          git push

  deploy-prod:
    needs: build
    if: startsWith(github.ref, 'refs/tags/v') && !contains(github.ref, '-rc.')
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.smp.vn
    steps:
      - uses: actions/checkout@v4
        with:
          repository: trungnguyenchanh/smp-gitops
          token: ${{ secrets.GITOPS_PAT }}
          path: gitops
      - name: Update prod manifest
        run: |
          cd gitops/environments/prod
          yq e -i ".image.tag = \"${{ needs.build.outputs.image_tag }}\"" ${{ github.event.repository.name }}.yaml
          git add . && git commit -m "deploy(prod): ${{ github.event.repository.name }} → ${{ needs.build.outputs.image_tag }}"
          git push
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {"text": "🚀 Prod deploy: ${{ github.event.repository.name }} v${{ needs.build.outputs.image_tag }}"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_PROD_WEBHOOK }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

## 5. GitOps repo structure

Repo `smp-gitops` (separate from service repos):

```
smp-gitops/
├── README.md
├── environments/
│   ├── dev/
│   │   ├── order-svc.yaml
│   │   ├── dispatch-engine.yaml
│   │   └── ...
│   ├── staging/
│   └── prod/
└── apps/                  # ArgoCD Application manifests
    ├── dev-apps.yaml
    ├── staging-apps.yaml
    └── prod-apps.yaml
```

ArgoCD configured to auto-sync `environments/dev`, manual sync staging + prod.

## 6. Dockerfile template (Go service)

Multi-stage build:

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
RUN apk add --no-cache git ca-certificates
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s -X main.version=$(git describe --tags --always) -X main.commit=$(git rev-parse HEAD)" \
    -o /app/server ./cmd/server

# Runtime stage
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/server /app/server
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/app/server"]
```

## 7. Helm chart structure

Per service repo có `deployments/helm/`:

```
deployments/helm/order-svc/
├── Chart.yaml
├── values.yaml          # default values
├── values-dev.yaml      # dev overrides
├── values-staging.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── hpa.yaml          # auto-scale
    ├── pdb.yaml          # pod disruption budget
    └── servicemonitor.yaml  # prometheus scrape
```

## 8. Deployment strategy

### Rolling update (dev, staging)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

### Blue/green (prod, critical services)

Order-svc, dispatch-engine, partner-svc use blue/green via Argo Rollouts:

```yaml
spec:
  strategy:
    blueGreen:
      activeService: order-svc-active
      previewService: order-svc-preview
      autoPromotionEnabled: false  # require manual promote
      scaleDownDelaySeconds: 300
```

Flow:
1. Deploy new version → preview service
2. SRE smoke test via preview URL
3. Manual promote → swap active/preview
4. Old version sleep 5 min then scale down
5. If issue → 1-click rollback (re-swap)

## 9. Database migrations in pipeline

Migrations run **before** service deploy:

```yaml
# Helm pre-install/pre-upgrade hook
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-order-db
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-weight: "-5"
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: migrate/migrate:v4.17.0
        args:
          - "-database=$(MYSQL_DSN)"
          - "-path=/migrations"
          - "up"
        volumeMounts:
          - name: migrations
            mountPath: /migrations
```

**Rollback safety:**
- Tất cả migrations phải reversible (có file `.down.sql`)
- Breaking schema change: 2-phase deploy (expand → migrate data → contract)
  - Step 1: add new column (nullable) → deploy code đọc cả cũ + mới
  - Step 2: backfill data
  - Step 3: deploy code chỉ dùng mới
  - Step 4: drop column cũ

## 10. Smoke test post-deploy

`.github/workflows/smoke-test.yml`:

```yaml
on:
  workflow_run:
    workflows: ["CD"]
    types: [completed]
jobs:
  smoke:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Smoke test
        run: |
          ENV_URL="${{ vars.ENV_API_URL }}"
          # Health check
          curl -f "$ENV_URL/health" || exit 1
          # Read endpoint check
          curl -f "$ENV_URL/api/v1/catalog/services" -H "Authorization: Bearer ${{ secrets.SMOKE_TEST_JWT }}" || exit 1
          # Critical create endpoint (idempotency)
          curl -f -X POST "$ENV_URL/api/v1/test/echo" -d '{"hello":"world"}' || exit 1
```

## 11. Monitoring deploys

Post-deploy alerts:
- Error rate spike > 2% in 5 min → PagerDuty
- Latency p95 > 2x baseline in 5 min → PagerDuty
- Pod crash loop → Slack

Grafana dashboard "Deploy timeline" overlays deployments with metrics.

## 12. Release versioning (SemVer)

- `MAJOR.MINOR.PATCH` (vd v3.3.0)
- Pre-release: `v3.3.0-rc.1`, `v3.3.0-rc.2`
- Tag created khi PR release merged to main

```bash
# Release flow
git checkout main && git pull
git tag -a v3.3.0 -m "Release v3.3.0 · Partner platform"
git push origin v3.3.0
# → triggers CD prod deploy with manual approval
```

CHANGELOG.md updated trong PR release.

## 13. Rollback playbook

```bash
# Option 1: ArgoCD rollback
argocd app rollback smp-order-svc-prod <previous-revision>

# Option 2: kubectl rollback (emergency)
kubectl rollout undo deployment/order-svc -n smp-prod

# Option 3: Re-tag previous version
git tag -a v3.3.0-rollback -m "Rollback to v3.2.5"
# manually update gitops repo
```

Always:
- Notify in #prod-alerts Slack
- Create incident ticket
- Post-mortem within 48h if customer impact

## 14. Branch protection rules

GitHub settings for `main`:
- Require PR with 1 approver from CODEOWNERS
- Require status checks: `lint`, `test`, `security`, `openapi`
- Require linear history
- Require signed commits (Phase 2)
- Lock force push
- Auto-delete branches after merge

## 15. Container image policy

- Base: distroless or alpine only
- No `:latest` tags allowed
- All images scanned by Trivy (block HIGH/CRITICAL)
- SBOM generated and signed (cosign)
- Image registry: GitHub Container Registry (ghcr.io)
- Retention: prod tags forever, dev tags 30 days
