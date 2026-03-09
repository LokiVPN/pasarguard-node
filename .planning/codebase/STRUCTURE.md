# Codebase Structure

**Analysis Date:** 2026-03-09

## Directory Layout

```
pasarguard-node/
├── backend/                # Backend interface + implementations
│   ├── backend.go          # Backend interface definition
│   └── xray/               # Xray backend implementation
│       ├── api/            # Xray internal gRPC API client
│       │   ├── account.go  # Account types (Vmess, Vless, Trojan, SS)
│       │   ├── base.go     # XrayHandler gRPC client setup
│       │   ├── message.go  # Protobuf message helpers
│       │   ├── proxyman.go # Inbound/outbound user management
│       │   └── stats.go    # Stats query methods
│       ├── config.go       # Xray JSON config parsing/manipulation
│       ├── core.go         # Process lifecycle (start/stop/restart)
│       ├── core_unix.go    # Unix process attributes (build tag)
│       ├── core_windows.go # Windows process attributes (build tag)
│       ├── jobs.go         # Health checks, startup verification
│       ├── log.go          # Log capture from Xray process stdout
│       ├── process_unix.go # Unix process tree kill, zombie detection
│       ├── process_windows.go # Windows process management
│       ├── startup_diagnostics.go # Startup log ring buffer
│       ├── stats.go        # Stats delegation to API handler
│       ├── user.go         # User sync/update logic
│       └── xray.go         # Xray struct (Backend impl facade)
├── common/                 # Shared protobuf types and utilities
│   ├── helper.go           # Proto body reading, response sending, code mapping
│   ├── service.pb.go       # Generated protobuf types (DO NOT EDIT)
│   └── service_grpc.pb.go  # Generated gRPC service stubs (DO NOT EDIT)
├── config/                 # Application configuration
│   └── config.go           # Config struct, env var loading
├── controller/             # API controller layer
│   ├── controller.go       # Shared Controller struct, connection lifecycle
│   ├── users_chunk.go      # Chunked user list assembly
│   ├── rest/               # REST/HTTP implementation
│   │   ├── base.go         # Base/Start/Stop handlers
│   │   ├── log.go          # SSE log streaming handler
│   │   ├── middleware.go    # API key, backend check, logging, request tracking
│   │   ├── service.go      # Router setup, HTTP server startup
│   │   ├── stats.go        # Stats endpoint handlers
│   │   └── user.go         # User sync endpoint handlers
│   └── rpc/                # gRPC implementation
│       ├── base.go         # Start/Stop/GetBaseInfo handlers
│       ├── log.go          # Server-streaming log handler
│       ├── middleware.go    # Interceptors (auth, backend check, logging)
│       ├── service.go      # gRPC server setup, service registration
│       ├── stats.go        # Stats RPC handlers
│       └── user.go         # User sync RPC handlers (client streaming)
├── logger/                 # File-based logger
│   └── logger.go           # Logger struct with access/error file output
├── tools/                  # Cross-cutting utilities
│   ├── file.go             # File reading helper
│   ├── port.go             # Free port finder
│   ├── sys.go              # System stats (CPU, memory, bandwidth)
│   └── tls.go              # TLS credential loading, HTTP client creation
├── .github/workflows/      # CI/CD
│   ├── docker-build.yml    # Docker image build
│   ├── docker-build-dev.yml # Dev Docker image build
│   ├── release.yml         # Release workflow
│   └── run-tests.yml       # Test runner
├── main.go                 # Application entry point
├── go.mod                  # Go module definition
├── go.sum                  # Dependency checksums
├── Makefile                # Build, test, cert generation, xray install
├── Dockerfile              # Multi-stage Docker build
├── docker-compose.yml      # Docker Compose config
├── .env.example            # Environment variable template
├── CLAUDE.md               # Claude Code instructions
├── CONTRIBUTING.md         # Contributing guidelines
├── README.md               # Project readme
└── LICENSE                 # License file (GPL-3.0)
```

## Directory Purposes

**`backend/`:**
- Purpose: Backend abstraction and implementations for VPN core management
- Contains: Interface definition and Xray-specific implementation
- Key files: `backend.go` (interface), `xray/xray.go` (main implementation)

**`backend/xray/`:**
- Purpose: Complete Xray VPN core lifecycle management
- Contains: Process management, config manipulation, user sync, stats, health monitoring, platform-specific code
- Key files: `xray.go` (facade), `core.go` (process lifecycle), `config.go` (config manipulation), `user.go` (user management)

**`backend/xray/api/`:**
- Purpose: gRPC client for Xray's internal management API (runs on localhost)
- Contains: Handler/stats service clients, protocol-specific account types
- Key files: `base.go` (client setup), `account.go` (account types), `stats.go` (stats queries), `proxyman.go` (user add/remove)

**`common/`:**
- Purpose: Shared protobuf-generated types and utility functions used across all packages
- Contains: Generated Go code from `service.proto`, helper functions
- Key files: `service.pb.go` (generated types), `service_grpc.pb.go` (generated gRPC stubs), `helper.go` (utilities)

**`config/`:**
- Purpose: Application-level configuration from environment variables
- Contains: Config struct and env loading helpers
- Key files: `config.go`

**`controller/`:**
- Purpose: API layer handling external requests from the control panel
- Contains: Shared controller logic, REST and gRPC service implementations
- Key files: `controller.go` (shared logic), `users_chunk.go` (chunk assembly)

**`controller/rest/`:**
- Purpose: HTTP/TLS REST API using chi router with protobuf serialization
- Contains: Route handlers, middleware, server setup
- Key files: `service.go` (router/server), `middleware.go` (auth/logging)

**`controller/rpc/`:**
- Purpose: gRPC/TLS API implementing `NodeService`
- Contains: RPC handlers, interceptors, server setup
- Key files: `service.go` (server setup), `middleware.go` (interceptors)

**`logger/`:**
- Purpose: File-based logging for Xray process output
- Contains: Logger with access/error log file separation
- Key files: `logger.go`

**`tools/`:**
- Purpose: Standalone utility functions with no domain logic
- Contains: TLS, system stats, port management, file helpers
- Key files: `tls.go`, `sys.go`, `port.go`, `file.go`

## Key File Locations

**Entry Points:**
- `main.go`: Application startup, protocol selection, signal handling

**Configuration:**
- `config/config.go`: All application configuration
- `.env.example`: Environment variable reference (existence noted only)
- `Makefile`: Build targets, test commands, cert generation

**Core Logic:**
- `backend/backend.go`: Backend interface contract
- `backend/xray/xray.go`: Xray backend facade
- `backend/xray/core.go`: Xray process lifecycle
- `backend/xray/config.go`: Xray config parsing, user injection, API setup
- `backend/xray/user.go`: User sync/update via config and API
- `controller/controller.go`: Connection lifecycle, backend orchestration

**Protobuf:**
- `common/service.pb.go`: All protobuf message types (generated)
- `common/service_grpc.pb.go`: gRPC service definition (generated)
- Proto source: `common/service.proto` (referenced in Makefile, regenerate with `make generate_grpc_code`)

**Testing:**
- `backend/xray/log_test.go`: Xray log parsing tests
- `backend/xray/process_unix_test.go`: Unix process management tests
- `backend/xray/startup_diagnostics_test.go`: Startup diagnostics tests
- `backend/xray/xray_test.go`: Xray integration tests
- `controller/rest/rest_test.go`: REST API tests
- `controller/rpc/rpc_test.go`: gRPC API tests
- `controller/users_chunk_test.go`: Chunk assembly tests

## Naming Conventions

**Files:**
- Snake_case for multi-word files: `users_chunk.go`, `startup_diagnostics.go`
- Platform-specific build tags: `core_unix.go`, `core_windows.go`, `process_unix.go`
- Test files: `*_test.go` co-located with source

**Packages:**
- Single lowercase words: `backend`, `common`, `config`, `controller`, `logger`, `tools`
- Sub-packages by protocol: `rest`, `rpc`, `api`
- Sub-packages by implementation: `xray`

**Go Identifiers:**
- Exported types: PascalCase (`Controller`, `Backend`, `XrayHandler`)
- Unexported fields: camelCase (`clientIP`, `apiPort`, `cancelFunc`)
- Constants: PascalCase for exported (`NodeVersion`), camelCase for package-level (`logPhaseRuntime`)
- Protocol constants: PascalCase string consts (`Vmess`, `Vless`, `Trojan`, `Shadowsocks`)

## Where to Add New Code

**New Backend Implementation (e.g., WireGuard, Sing-box):**
- Create directory: `backend/<name>/`
- Implement `backend.Backend` interface
- Add case to `Controller.StartBackend()` switch in `controller/controller.go`
- Add corresponding `BackendType` to protobuf in `common/service.proto` and regenerate

**New API Endpoint:**
- REST: Add handler method to `controller/rest/` (in relevant file or new file), add route in `controller/rest/service.go` `setRouter()`
- gRPC: Add RPC definition to `common/service.proto`, regenerate, implement in `controller/rpc/`
- If shared logic needed: Add method to `controller/controller.go`

**New Stats Type:**
- Add stat query to `backend/xray/api/stats.go`
- Add case to `backend/xray/stats.go` `GetStats()` switch
- Add `StatType` enum value to protobuf, regenerate
- Add REST handler in `controller/rest/stats.go`, gRPC handler in `controller/rpc/stats.go`

**New Utility Function:**
- General utilities: `tools/` (create new file if distinct concern)
- Protobuf helpers: `common/helper.go`
- Xray-specific: `backend/xray/api/` or `backend/xray/`

**New Protocol Account Type:**
- Add struct implementing `Account` interface in `backend/xray/api/account.go`
- Add protocol handling in `backend/xray/config.go` `syncUsers()`, `updateUsers()`, `ToBytes()`
- Add to `ProxySettings` struct in `backend/xray/api/account.go`
- Add protobuf message in `common/service.proto`, regenerate

**New Test:**
- Co-locate with source: `<package>/<name>_test.go`
- Integration tests: Use `TEST_INTEGRATION` env guard (see existing test files)

## Special Directories

**`common/`:**
- Purpose: Generated protobuf Go code
- Generated: Yes (via `make generate_grpc_code` from `common/service.proto`)
- Committed: Yes (generated files are committed)
- Note: Do not manually edit `service.pb.go` or `service_grpc.pb.go`

**`.github/workflows/`:**
- Purpose: GitHub Actions CI/CD pipeline definitions
- Generated: No
- Committed: Yes

**`certs/` (not in repo):**
- Purpose: TLS certificates for development
- Generated: Yes (via `make generate_server_cert` / `make generate_client_cert`)
- Committed: No (in `.gitignore`)

---

*Structure analysis: 2026-03-09*
