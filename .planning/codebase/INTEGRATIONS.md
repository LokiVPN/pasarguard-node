# External Integrations

**Analysis Date:** 2026-03-09

## APIs & External Services

**Xray-core (Local Process):**
- The primary integration. PasarGuard Node manages Xray-core as an external subprocess
- Binary location: configurable via `XRAY_EXECUTABLE_PATH` (default: `/usr/local/bin/xray`)
- Communication: gRPC client over localhost on a random free port
  - SDK/Client: `backend/xray/api/base.go` - `XrayHandler` struct
  - Handler service: `github.com/xtls/xray-core/app/proxyman/command` - add/remove inbound/outbound handlers
  - Stats service: `github.com/xtls/xray-core/app/stats/command` - query traffic/user stats
- Config passed via stdin (`-c stdin:`) as JSON
- Process lifecycle managed in `backend/xray/core.go` (start, stop, restart, orphan cleanup)
- Platform-specific process management: `backend/xray/process_unix.go`, `backend/xray/process_windows.go`

**PasarGuard Panel (Upstream Client):**
- The node does NOT call the panel; instead, the panel connects to this node's API
- Dual-protocol API exposed:
  - gRPC: `controller/rpc/service.go` - full bidirectional streaming support
  - REST: `controller/rest/service.go` - Chi-based HTTP API
- Authentication: UUID-based API key via `x-api-key` header on every request
- Keep-alive mechanism: panel sends `keep_alive` duration on `Start`; node auto-disconnects after timeout

**Xray Install Script (Build-time only):**
- Downloaded from `https://github.com/PasarGuard/scripts/raw/main/install_core.sh`
- Used during Docker build and CI to install Xray binary
- Not used at runtime

## Data Storage

**Databases:**
- None. This is a stateless node agent. All state is held in memory.

**File Storage:**
- Generated Xray config files: `GENERATED_CONFIG_PATH` (default: `/var/lib/pg-node/generated/`) - only written in debug mode
- Xray access/error logs: file paths defined within the Xray config JSON
- TLS certificates: `SSL_CERT_FILE` and `SSL_KEY_FILE` paths

**Caching:**
- None. System stats are continuously polled and held in memory (`controller/controller.go` `recordSystemStats`)

## Authentication & Identity

**Auth Provider:**
- Custom UUID-based API key authentication
- Implementation:
  - REST: `controller/rest/middleware.go` `validateApiKey` - checks `x-api-key` HTTP header
  - gRPC: `controller/rpc/middleware.go` `validateApiKey` - checks `x-api-key` gRPC metadata
- Key format: UUID (any version), set via `API_KEY` environment variable
- All requests require valid API key; mismatch returns 403/PermissionDenied

**TLS:**
- All API communication over TLS (both REST and gRPC)
- Self-signed certificates supported (generated via `make generate_server_cert`)
- TLS config loaded in `tools/tls.go` `LoadTLSCredentials`
- Client certificate auth available but not enforced (`tls.NoClientCert`)

## Monitoring & Observability

**Error Tracking:**
- None (no external error tracking service)

**Logs:**
- Standard library `log` package throughout
- Custom log writer for Xray process output: `logger/logger.go`
- Log streaming API: clients can stream Xray logs in real-time via `GetLogs` (gRPC stream or REST endpoint)
- Startup diagnostics: `backend/xray/startup_diagnostics.go` captures and analyzes startup logs for error detection
- Log buffer size configurable via `LOG_BUFFER_SIZE` env var (default: 10000)

**System Stats:**
- CPU usage, memory, network bandwidth collected via gopsutil (`tools/sys.go`)
- Exposed via `GetSystemStats` API endpoint
- Backend (Xray) runtime stats: goroutines, GC, memory allocation via `GetBackendStats`
- User online stats and per-IP tracking via `GetUserOnlineStats` and `GetUserOnlineIpListStats`

## CI/CD & Deployment

**Hosting:**
- Docker containers (primary deployment method)
- Docker Hub: `pasarguard/node`
- GitHub Container Registry: `ghcr.io/pasarguard/node`
- Bare binary deployment also supported (release artifacts)

**CI Pipeline:**
- GitHub Actions (4 workflows in `.github/workflows/`)
  - `run-tests.yml` - Runs on PRs to all branches and pushes to `dev`; Go 1.25.7, installs Xray, generates certs, runs `make test`
  - `docker-build.yml` - On release: builds multi-platform Docker image (linux/amd64, linux/arm64), pushes to Docker Hub and GHCR
  - `docker-build-dev.yml` - Development Docker builds
  - `release.yml` - On release: cross-compiles binaries for multiple OS/arch combos, uploads as release assets (linux, freebsd, openbsd; amd64, 386, arm64, armv5-7, s390x)

**Container Configuration:**
- `docker-compose.yml` - Reference deployment config
- Network mode: host (required for proxy port access)
- Volume: `/var/lib/pg-node` for persistent cert/config storage

## Environment Configuration

**Required env vars:**
- `API_KEY` - UUID for authentication (functionally required; node logs error if missing but still starts)

**Optional env vars with defaults:**
- `SERVICE_PORT` (62050)
- `SERVICE_PROTOCOL` ("grpc")
- `NODE_HOST` ("0.0.0.0")
- `SSL_CERT_FILE`, `SSL_KEY_FILE` (paths to TLS certs)
- `XRAY_EXECUTABLE_PATH`, `XRAY_ASSETS_PATH` (Xray binary/assets paths)
- `GENERATED_CONFIG_PATH` (debug config output)
- `DEBUG` (false)
- `LOG_BUFFER_SIZE` (10000)
- `STARTUP_LOG_TAIL_SIZE` (200)

**Secrets location:**
- `.env` file (local development, gitignored)
- `.env.example` exists as a reference template
- Docker: environment variables via `docker-compose.yml` `environment` section or `env_file`
- CI: GitHub Secrets (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `GITHUB_TOKEN`)

## Webhooks & Callbacks

**Incoming:**
- None (the gRPC/REST API serves as the primary interface; no webhook endpoints)

**Outgoing:**
- None (the node does not call external services; it only communicates with the local Xray process)

## Internal gRPC API (Node Service)

**Defined in:** `common/service.proto`

**Unary RPCs:**
- `Start(Backend) -> BaseInfoResponse` - Start Xray backend with config and users
- `Stop(Empty) -> Empty` - Stop backend
- `GetBaseInfo(Empty) -> BaseInfoResponse` - Node/core version and status
- `GetSystemStats(Empty) -> SystemStatsResponse` - CPU, memory, bandwidth
- `GetBackendStats(Empty) -> BackendStatsResponse` - Xray runtime stats
- `GetStats(StatRequest) -> StatResponse` - Traffic stats (inbound/outbound/user)
- `GetUserOnlineStats(StatRequest) -> OnlineStatResponse` - User online count
- `GetUserOnlineIpListStats(StatRequest) -> StatsOnlineIpListResponse` - User IP list

**Streaming RPCs:**
- `GetLogs(Empty) -> stream Log` - Server-streaming log output
- `SyncUser(stream User) -> Empty` - Client-streaming single user sync
- `SyncUsers(Users) -> Empty` - Bulk user sync (unary)
- `SyncUsersChunked(stream UsersChunk) -> Empty` - Client-streaming chunked user sync for large lists

**REST Equivalents:**
- `POST /start`, `PUT /stop`, `GET /info`
- `GET /logs`, `GET /stats/*`, `PUT /user/sync`, `PUT /users/sync`, `PUT /users/sync/chunked`

---

*Integration audit: 2026-03-09*
