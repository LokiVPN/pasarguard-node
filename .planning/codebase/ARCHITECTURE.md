# Architecture

**Analysis Date:** 2026-03-09

## Pattern Overview

**Overall:** Layered service architecture with pluggable backend and dual-protocol controller (gRPC + REST)

**Key Characteristics:**
- Interface-driven backend abstraction allowing multiple VPN core implementations (currently only Xray)
- Controller layer embeds shared logic; REST and gRPC services compose via Go struct embedding
- Protobuf-first wire format: both gRPC and REST endpoints serialize/deserialize with protobuf (not JSON)
- Process management pattern: the node spawns and supervises an external Xray binary as a child process
- Single-client model: only one control panel client connects at a time; new connections take over from previous

## Layers

**Config Layer:**
- Purpose: Load and validate environment-based configuration
- Location: `config/config.go`
- Contains: `Config` struct, env var loading helpers (`GetEnv`, `GetEnvAsInt`, `GetEnvAsBool`, `GetEnvAsUUID`)
- Depends on: `godotenv`, `google/uuid`
- Used by: `main.go`, `controller`, `backend/xray`

**Controller Layer:**
- Purpose: Shared business logic for API handling, client connection lifecycle, backend orchestration
- Location: `controller/controller.go`, `controller/users_chunk.go`
- Contains: `Controller` struct with `Connect`/`Disconnect`/`StartBackend`/`SystemStats`/`BaseInfoResponse` methods, `Service` interface, chunk assembly logic
- Depends on: `backend`, `config`, `tools`, `common`
- Used by: `controller/rest`, `controller/rpc` (via struct embedding)

**REST Controller:**
- Purpose: HTTP/TLS API server using chi router
- Location: `controller/rest/`
- Contains: Route definitions (`service.go`), handlers (`base.go`, `user.go`, `stats.go`, `log.go`), middleware (`middleware.go`)
- Depends on: `controller.Controller` (embedded), `go-chi/chi`, `backend`, `common`
- Used by: `main.go` when `SERVICE_PROTOCOL=rest`

**gRPC Controller:**
- Purpose: gRPC/TLS API server implementing `NodeService` protobuf service
- Location: `controller/rpc/`
- Contains: Service implementation (`service.go`), handlers (`base.go`, `user.go`, `stats.go`, `log.go`), interceptors (`middleware.go`)
- Depends on: `controller.Controller` (embedded), `google.golang.org/grpc`, `common` (generated protobuf)
- Used by: `main.go` when `SERVICE_PROTOCOL=grpc` (default)

**Backend Interface:**
- Purpose: Define contract for VPN core backends
- Location: `backend/backend.go`
- Contains: `Backend` interface with 10 methods (lifecycle, user sync, stats), context keys (`ConfigKey`, `UsersKey`)
- Depends on: `common`
- Used by: `controller`, `backend/xray`

**Xray Backend:**
- Purpose: Implement `Backend` interface by managing an Xray VPN core process
- Location: `backend/xray/`
- Contains: `Xray` struct (facade), `Core` (process management), `Config` (Xray JSON config manipulation), user management, stats delegation, health checks
- Depends on: `backend`, `backend/xray/api`, `common`, `config`, `logger`, `xtls/xray-core`
- Used by: `controller` via `Backend` interface

**Xray API Client:**
- Purpose: gRPC client to the local Xray process's internal API (handler service + stats service)
- Location: `backend/xray/api/`
- Contains: `XrayHandler` (gRPC client wrapper), `Account` interface and implementations (Vmess, Vless, Trojan, Shadowsocks), stats query methods, proxyman operations
- Depends on: `xtls/xray-core` gRPC services, `common`
- Used by: `backend/xray`

**Common (Protobuf):**
- Purpose: Shared protobuf-generated types and utility functions
- Location: `common/`
- Contains: Generated protobuf Go code (`service.pb.go`, `service_grpc.pb.go`), helpers (`helper.go`) for proto body reading, response sending, gRPC-to-HTTP code mapping
- Depends on: `google.golang.org/grpc`, `google.golang.org/protobuf`
- Used by: Every other package

**Logger:**
- Purpose: File-based logging with access/error log separation
- Location: `logger/logger.go`
- Contains: `Logger` struct with log levels, file management, thread-safe logging
- Depends on: Standard library only
- Used by: `backend/xray` (for Xray process log capture)

**Tools:**
- Purpose: Cross-cutting utilities
- Location: `tools/`
- Contains: TLS credential loading (`tls.go`), system stats collection (`sys.go`), port finding (`port.go`), file reading (`file.go`)
- Depends on: `gopsutil`, `common`
- Used by: `controller`, `main.go`

## Data Flow

**Client Connection & Backend Start:**

1. Panel client sends `Start` request (REST POST `/start` or gRPC `Start`) with Xray config JSON, user list, and keep-alive timeout
2. Controller parses the backend config via `detectBackend()`, creates `xray.Config` from JSON, stores config and users in context values
3. Any existing backend is disconnected (`Disconnect()` kills previous Xray process)
4. `Controller.StartBackend()` instantiates `xray.NewXray()` which: syncs initial users into config, generates config bytes, spawns `xray -c stdin:` subprocess, connects to Xray's internal gRPC API
5. `Connect()` records client IP, starts system stats recording goroutine, starts keep-alive tracker goroutine
6. Health check loop (`checkXrayHealth`) runs in background, auto-restarts Xray after 3 consecutive API failures

**User Sync (Single User):**

1. Panel sends user data (protobuf-encoded `User`)
2. `SyncUser` handler calls `Xray.SyncUser()` which removes the user from all inbounds via Xray gRPC API, then re-adds to active inbounds
3. In-memory `Inbound.clients` map is updated simultaneously

**User Sync (Bulk / Chunked):**

1. Panel streams `UsersChunk` messages (indexed, with `last` flag)
2. `BuildUsersFromChunks()` assembles ordered user list
3. If >100 users: `UpdateUsersAndRestart()` updates in-memory config and restarts Xray process
4. If <=100 users: `UpdateUsers()` updates via Xray gRPC API without restart

**Stats Query:**

1. Panel requests stats (user traffic, online status, system metrics, backend runtime stats)
2. Controller delegates to `Xray.GetStats()` / `Xray.GetSysStats()` which calls Xray's internal `StatsService` gRPC API
3. Xray stats use `>>>` delimiter format: `user>>>email>>>traffic>>>uplink`

**State Management:**
- `Controller` holds: current backend reference, client IP, last request time, system stats, cancel function
- `Xray` holds: parsed config with inbound client maps, Core process handle, API handler
- `Core` holds: OS process, log channel, startup diagnostics ring buffer
- All state is protected by `sync.RWMutex` at each level
- Single-client model: `Controller.clientIP` tracks the connected panel, `Disconnect()` clears all state

## Key Abstractions

**Backend Interface:**
- Purpose: Decouple controller from specific VPN core implementation
- Examples: `backend/backend.go` (interface), `backend/xray/xray.go` (implementation)
- Pattern: Strategy pattern -- `Controller.StartBackend()` switches on `BackendType` enum to instantiate the correct implementation

**Account Interface:**
- Purpose: Represent proxy user accounts across different protocols (Vmess, Vless, Trojan, Shadowsocks)
- Examples: `backend/xray/api/account.go`
- Pattern: Each protocol has its own struct (`VmessAccount`, `VlessAccount`, `TrojanAccount`, `ShadowsocksAccount`, `ShadowsocksTcpAccount`) implementing `Account` with `GetEmail()`, `GetLevel()`, `Message()` methods

**Controller Embedding:**
- Purpose: Share connection lifecycle, backend management, and stats logic between REST and gRPC services
- Examples: `controller/rest/service.go` (`Service` embeds `controller.Controller`), `controller/rpc/service.go` (`Service` embeds `controller.Controller`)
- Pattern: Go struct embedding -- both protocol services inherit all `Controller` methods

**Config as Context Value:**
- Purpose: Pass backend-specific config through the request context from handler to backend constructor
- Examples: `backend/backend.go` (`ConfigKey{}`, `UsersKey{}`), `controller/rest/base.go`, `controller/rpc/base.go`
- Pattern: Context value keys using empty struct types

## Entry Points

**main.go:**
- Location: `main.go`
- Triggers: Binary execution
- Responsibilities: Load config, load TLS creds, select protocol (REST or gRPC), start listener, handle OS signals for graceful shutdown

**REST StartHttpListener:**
- Location: `controller/rest/service.go`
- Triggers: `main.go` when `SERVICE_PROTOCOL=rest`
- Responsibilities: Create chi router with middleware chain, start HTTPS server, return shutdown function

**gRPC StartGRPCListener:**
- Location: `controller/rpc/service.go`
- Triggers: `main.go` when `SERVICE_PROTOCOL=grpc` (default)
- Responsibilities: Create gRPC server with interceptor chain, register `NodeService`, start TLS listener, return shutdown function

## Error Handling

**Strategy:** Errors propagate up through return values (standard Go pattern). gRPC status codes used internally even for REST.

**Patterns:**
- REST handlers map gRPC status codes to HTTP status codes via `common.GrpcCodeToHTTP()` in `common/helper.go`
- `common.InterceptNotFound()` wraps errors ending with "not found." into gRPC `NotFound` status
- Backend startup errors include tail of recent Xray logs via `startupErrorWithTail()` in `backend/xray/jobs.go`
- User sync errors are aggregated into a single error string (not returned individually)
- Startup diagnostics captured in a ring buffer (`backend/xray/startup_diagnostics.go`) for debugging failed starts

## Cross-Cutting Concerns

**Logging:** Standard library `log` package throughout. `logger/logger.go` provides file-based logging only for Xray process output (access/error log files). No structured logging framework.

**Validation:** Minimal -- API key validated via middleware, user email checked for emptiness, IP address format validated in config loading. No request body validation framework.

**Authentication:** UUID-based API key in `x-api-key` header. Validated in middleware (`controller/rest/middleware.go`, `controller/rpc/middleware.go`). Single shared key per node instance. No session tokens or token rotation.

**TLS:** Required for all external communication. Server cert/key loaded from file paths. Both REST and gRPC servers use mutual TLS configuration from `tools/tls.go`. Internal Xray API uses insecure local gRPC (127.0.0.1 only).

**Concurrency:** `sync.RWMutex` used at Controller, Xray, Core, Inbound, and Logger levels. Keep-alive tracker, system stats recorder, health checker, and log capturer run as goroutines with context cancellation.

**Process Management:** Platform-specific process handling via build tags (`core_unix.go`, `core_windows.go`, `process_unix.go`, `process_windows.go`). Orphaned process cleanup, process tree killing, zombie detection.

---

*Architecture analysis: 2026-03-09*
