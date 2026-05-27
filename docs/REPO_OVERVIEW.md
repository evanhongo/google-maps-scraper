# Google Maps Scraper

Open-source Google Maps scraper. Single Go binary that runs in six modes (file/CSV, local web UI, PostgreSQL queue, AWS Lambda runner, AWS Lambda invoker, Playwright installer) and a separate `gmapssaas` binary that adds a multi-user SaaS edition (River queue, admin UI, REST API, cloud provisioning).

For a deep architectural tour of the codebase, see [`docs/REPO_WALKTHROUGH.md`](docs/REPO_WALKTHROUGH.md).

## Tech Stack

- **Language:** Go 1.26.2 (set in `go.mod`; CI matrix pins this version exactly).
- **Scraping framework:** [`github.com/gosom/scrapemate`](https://github.com/gosom/scrapemate) v1.1.0 (author's own framework; provides browser pool, job provider interface, result writers).
- **Headless browser:** Playwright via `github.com/playwright-community/playwright-go` v0.5700.1, driving Chromium (Firefox in fast/stealth mode).
- **HTML parsing:** `github.com/PuerkitoBio/goquery` v1.12.0.
- **HTTP routing:**
  - Local web runner uses the standard `net/http` mux.
  - SaaS edition uses `github.com/go-chi/chi/v5` v5.2.4.
- **Persistence:**
  - PostgreSQL via `github.com/jackc/pgx/v5` v5.9.0 (used by `postgres/`, `runner/databaserunner`, `cmd/gmapssaas`).
  - SQLite via `modernc.org/sqlite` v1.37.0 (pure-Go driver, used by local web runner only).
- **Migrations:** `github.com/rubenv/sql-migrate` v1.8.1 with embedded `.sql` files in `migrations/`.
- **Job queue (SaaS):** [River](https://riverqueue.com/) `github.com/riverqueue/river` v0.30.1, plus `riverui` v0.14.0 for the embedded queue dashboard.
- **CLI (SaaS):** `github.com/urfave/cli/v3` v3.6.2 (the main binary uses the stdlib `flag` package).
- **AWS:** `github.com/aws/aws-sdk-go-v2` v1.36.3, `aws-lambda-go` v1.48.0, `s3` v1.79.3, `lambda` v1.71.2.
- **Cloud providers (SaaS provisioning):** DigitalOcean (`godo` v1.173.0), Hetzner (`hcloud-go/v2` v2.36.0).
- **Other notable libs:** `google/uuid`, `mcnijman/go-emailaddress` (email extraction), `pquerna/otp` + `skip2/go-qrcode` (TOTP 2FA), `speps/go-hashids/v2` (opaque job IDs), `gosom/go-leadsdb` (LeadsDB writer), `posthog-go` (telemetry), `swaggo/swag` + `http-swagger` (API docs).
- **Testing:** `github.com/stretchr/testify` v1.11.1 (require/assert only — no mocking framework).
- **Lint:** `golangci-lint` v1.64.8 invoked as a Go tool (`go tool golangci-lint`).

## Commands

All commands run from the repository root. The `Makefile` is the source of truth — run `make help` to list targets with their descriptions.

### Build

```bash
# Build the main binary (./bin/google_maps_scraper)
make build

# Build the SaaS binary (./bin/gmapssaas)
make build-saas

# Cross-compile linux/darwin/windows amd64 binaries
make cross-compile

# Docker images
make docker        # main binary, includes Playwright + Chromium
make docker-saas   # SaaS binary
```

### Lint, format, vet, vulnerability check

```bash
# Format (gofmt -s -w on the whole tree)
make format

# Run golangci-lint (enabled linters listed in .golangci.yaml)
make lint

# Run go vet
make vet

# Vulnerability scan (govulncheck)
make vuln
```

CI runs `gofmt -s -w . && git diff --exit-code`, `make lint`, `go vet ./...`, `go mod tidy && git diff --exit-code`, `go mod verify`, `make vuln`, and `go build -o /dev/null ./...` on every push/PR to `main` (see `.github/workflows/build.yml`).

### Test

```bash
# Run all unit tests with race detection (5 min timeout)
make test
# Equivalent: go test -v -race -timeout 5m ./...

# Coverage statistics to stdout
make test-cover

# HTML coverage report (opens browser)
make test-cover-report

# Single package
go test -v -race ./gmaps/...

# Single test by name (use the full package path)
go test -v -race -run TestWaitForFlushResultTimesOut ./rqueue/...
```

### Run the main binary

```bash
# File mode (CLI → CSV/JSON)
./bin/google_maps_scraper -input queries.txt -results out.csv
./bin/google_maps_scraper -input queries.txt -results out.json -json
./bin/google_maps_scraper -input queries.txt -email -results out.csv

# Web UI mode (default when no -input/-dsn given). Visit http://localhost:8080.
./bin/google_maps_scraper -web -data-folder ./webdata -addr :8080

# Grid scraping (tiles bounding box to bypass ~120-results-per-search cap)
./bin/google_maps_scraper -input queries.txt -grid-bbox 40.30,-3.80,40.50,-3.60 -grid-cell 1.0 -zoom 15

# PostgreSQL queue: producer (one-shot, exits after inserting seed jobs)
./bin/google_maps_scraper -dsn 'postgres://user:pass@host/db?sslmode=disable' -input queries.txt -produce

# PostgreSQL queue: worker (long-running, processes jobs until -exit-on-inactivity)
./bin/google_maps_scraper -dsn 'postgres://user:pass@host/db?sslmode=disable' -exit-on-inactivity 5m

# Fast mode (HTTP-only, no browser; requires -geo)
./bin/google_maps_scraper -input queries.txt -fast-mode -geo 52.52,13.40 -zoom 14

# Install Playwright browsers (use PLAYWRIGHT_INSTALL_ONLY=1 env var)
PLAYWRIGHT_INSTALL_ONLY=1 ./bin/google_maps_scraper

# Disable anonymous telemetry
DISABLE_TELEMETRY=1 ./bin/google_maps_scraper -input queries.txt -results out.csv
```

### Run the SaaS edition

```bash
# Local dev: start Postgres in Docker, apply migrations, create admin user, run server with hot reload (requires `air`)
make saas-dev
make saas-dev-stop          # stop containers (data preserved)
make saas-dev-reset         # stop + drop volumes

# Run API server / worker locally without hot reload
make saas-run-server        # serves on :8080
make saas-run-worker        # pulls from River default queue

# Manually invoke subcommands
go run ./cmd/gmapssaas serve  --encryption-key "$(openssl rand -hex 32)"
go run ./cmd/gmapssaas worker --database-url "postgres://..."
go run ./cmd/gmapssaas provision         # interactive cloud-provisioning wizard
go run ./cmd/gmapssaas admin create-user -u admin -p '...'

# Migrations
make saas-migrate-up
make saas-migrate-down
make saas-migrate-status
make saas-migrate-new name=add_xxx

# Regenerate Swagger docs (after changing api/*.go godoc annotations)
make saas-gen

# Connect to dev DB
make saas-psql
```

## Project Structure

```
.
├── main.go                          # Main-binary entry point + runnerFactory switch
├── cmd/
│   ├── gmapssaas/                   # SaaS-binary entry point with urfave/cli subcommands
│   │   ├── main.go
│   │   ├── cmdserve/                # `serve` — API + admin UI + maintenance queue
│   │   ├── cmdworker/               # `worker` — scrape-job processor (one River job at a time)
│   │   ├── cmdprovision/            # `provision` — interactive cloud-provisioning wizard
│   │   ├── cmdadmin/                # `admin` — CLI maintenance (e.g. create-user)
│   │   └── cmdupdate/               # `update` — in-place binary update
│   └── tools/                       # Build-tooling helpers
│
├── runner/                          # Runner interface, ParseConfig, six concrete modes
│   ├── runner.go                    # Config struct, RunMode constants, Telemetry singleton
│   ├── jobs.go                      # CreateSeedJobs, CreateGridSeedJobs, LoadCustomWriter
│   ├── filerunner/                  # RunModeFile — CLI → CSV/JSON
│   ├── webrunner/                   # RunModeWeb — local web UI
│   ├── databaserunner/              # RunModeDatabase[Produce] — PostgreSQL queue
│   ├── lambdaaws/                   # RunModeAwsLambda + RunModeAwsLambdaInvoker
│   └── installplaywright/           # RunModeInstallPlaywright — bootstraps Chromium
│
├── gmaps/                           # Core scraping engine
│   ├── job.go                       # GmapJob (search results page) — scrolls + extracts links
│   ├── place.go                     # PlaceJob (place detail) — parses APP_INITIALIZATION_STATE
│   ├── emailjob.go                  # EmailExtractJob — visits website for emails
│   ├── searchjob.go                 # SearchJob — fast-mode HTTP-only search
│   ├── entry.go                     # Entry result struct + EntryFromJSON parser
│   ├── reviews.go                   # Extra-reviews RPC + DOM fallback
│   └── multiple.go                  # ParseSearchResults for fast mode
│
├── grid/                            # Bounding-box → grid cells (tiling to bypass 120-result cap)
├── deduper/                         # Thread-safe FNV-hashed seen-URL set
├── exiter/                          # Job-completion bookkeeping; cancels root context when done
│
├── postgres/                        # JobProvider + ResultWriter for PostgreSQL queue
├── leadsdb/                         # Optional ResultWriter for LeadsDB
├── s3uploader/                      # Thin AWS S3 wrapper used by Lambda runner
├── internal/jsonbsanitize/          # Strips NUL bytes from entries before JSONB insert
│
├── web/                             # Local web runner internals
│   ├── web.go                       # HTTP server + handlers (HTML + REST API)
│   ├── service.go, job.go           # Service layer + Job/JobData domain types
│   ├── sqlite/                      # SQLite JobRepository implementation
│   └── static/                      # Embedded templates + CSS + ReDoc spec
│
├── admin/                           # SaaS admin UI (sessions, 2FA, API keys, workers, terminal)
│   ├── routes.go, *.go              # Chi handlers
│   ├── postgres/                    # Admin store implementation
│   ├── templates/, static/          # Server-rendered HTML
├── api/                             # SaaS REST API (/api/v1/*)
│   ├── api.go                       # Chi routes + handlers
│   ├── middleware.go                # KeyAuth, request logging
│   ├── postgres/                    # API store (API-key validation)
│   └── docs/                        # swagger-generated OpenAPI spec
├── rqueue/                          # River queue integration
│   ├── rqueue.go                    # NewClient (server) + NewWorkerClient + ScrapeWorker
│   ├── jobs.go, worker_jobs.go      # Maintenance + scrape job args + handlers
│   └── ui.go                        # Embedded riverui dashboard wiring
├── scraper/                         # SaaS scraper lifecycle
│   ├── scraper.go                   # ScraperManager (restart loop, atomic provider swap)
│   ├── centralwriter.go             # Buffers entries per River job, flushes on completion
│   └── provider.go                  # In-memory channel-based JobProvider
│
├── infra/                           # Cloud-provisioning abstraction
│   ├── infra.go                     # Provisioner interface + config structs
│   ├── digitalocean/, hetzner/      # Provider implementations
│   ├── planetscale/                 # Managed-Postgres alternative
│   ├── vps/, cloudinit/             # Generic VPS provisioning + cloud-init scripts
│   └── ssh_tofu.go, worker.go       # SSH client + worker bootstrap commands
├── cli/                             # Shared CLI prompts and pretty-printing helpers
│
├── cryptoext/                       # AES-GCM helper for encrypted app_config values
├── env/                             # Tiny env-var loader (LogUnsetEnvs)
├── httpext/                         # net/http server wrapper + chi middleware
├── log/                             # Structured logging (log/slog wrapper)
├── ratelimit/                       # Postgres-backed rate-limiter store
├── migrations/                      # Embedded .sql migrations (sql-migrate format)
├── saas/                            # SaaS constants (env-var names)
│
├── tlmt/                            # Anonymous PostHog telemetry
│   ├── tlmt.go                      # Event + Telemetry interface
│   ├── gonoop/                      # No-op implementation (used when DISABLE_TELEMETRY=1)
│   └── goposthog/                   # PostHog backend
│
├── docs/                            # Markdown docs
│   ├── REPO_WALKTHROUGH.md          # End-to-end code tour (generated via showboat)
│   ├── saas.md, development-saas.md
│   ├── recipes.md, proxies.md
├── skills/google-maps-scraper/      # Anthropic skill for AI-agent invocation
├── examples/                        # Example query files
├── testdata/                        # Captured Google Maps payloads used by entry_test.go
├── scripts/                         # Maintenance scripts
├── Dockerfile                       # Main binary image (multi-stage; bakes Chromium)
├── Dockerfile.saas                  # SaaS binary image
├── docker-compose.dev.yaml          # Local dev (main binary)
├── docker-compose.saas.yaml         # Local dev (Postgres for SaaS)
├── Makefile                         # Source of truth for all dev tasks
└── go.mod, go.sum
```

## External Dependencies

| Dependency | Where it shows up | Required? |
|---|---|---|
| **Chromium (via Playwright)** | All non-fast-mode scraping (`gmaps.GmapJob.BrowserActions`, `gmaps.PlaceJob.BrowserActions`). Bootstrapped by `runner/installplaywright` or `PLAYWRIGHT_INSTALL_ONLY=1`. Baked into the Docker image. | Yes for browser modes |
| **PostgreSQL 14+** | `runner/databaserunner` (jobs queue + results), entire SaaS edition (`admin`, `api`, `rqueue` River queue, `scraper.CentralWriter`, rate limits, sessions). Connection via `pgx/v5`. | Yes for DB / SaaS modes |
| **SQLite** | Local web runner's job queue (`web/sqlite/`). Uses `modernc.org/sqlite` (CGO-free, pure-Go), so no system SQLite is needed. | Bundled |
| **AWS S3** | Lambda runner uploads result CSVs (`s3uploader/`). Bucket name + AWS creds passed via flags or `MY_AWS_ACCESS_KEY` / `MY_AWS_SECRET_KEY` / `MY_AWS_REGION` env vars. | Only for Lambda mode |
| **AWS Lambda** | `runner/lambdaaws` runs *as* a Lambda; `runner/lambdaaws/invoker.go` *invokes* many Lambdas in parallel from a local host. | Only for Lambda mode |
| **PostHog** (`eu.i.posthog.com`) | Anonymous run-mode telemetry via `tlmt/goposthog`. Set `DISABLE_TELEMETRY=1` to opt out. | Optional |
| **LeadsDB** | Optional result writer (`leadsdb/`). Enabled via `-leadsdb-api-key` flag or `LEADSDB_API_KEY` env var. | Optional |
| **DigitalOcean** | SaaS provisioning of App Platform apps + managed Postgres (`infra/digitalocean/`). | Optional, SaaS only |
| **Hetzner Cloud** | SaaS provisioning of cloud servers (`infra/hetzner/`). | Optional, SaaS only |
| **PlanetScale** | SaaS managed-Postgres alternative (`infra/planetscale/`). | Optional, SaaS only |
| **Proxies** | Any HTTP / SOCKS proxy. Passed via `-proxies user:pass@host:port,...` or per-job from the web UI. | Optional, recommended at scale |
| **External IP echo services** | `ipify.org`, `ifconfig.me`, `icanhazip.com`, `ident.me`, `ifconfig.co` — queried *once* on startup to derive the anonymous telemetry machine ID. Disabled by `DISABLE_TELEMETRY=1`. | Optional |
| **`air`** (file watcher) | Used only by `make saas-dev` for hot-reload during SaaS development. | Dev only |
| **`sql-migrate`** | CLI used by `make saas-migrate-*` for managing SaaS migrations from the shell. The binary itself runs migrations programmatically via `migrations.Run`, so this is dev-only. | Dev only |
| **`swag`** | CLI used by `make saas-gen` to regenerate Swagger docs from `api/*.go` godoc annotations. | Dev only |
