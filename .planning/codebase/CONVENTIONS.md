# Coding Conventions

**Analysis Date:** 2026-03-09

## Naming Patterns

**Files:**
- Use lowercase `snake_case` for Go source files: `users_chunk.go`, `startup_diagnostics.go`, `core_unix.go`
- Test files use `_test.go` suffix co-located with source: `xray_test.go`, `log_test.go`
- Platform-specific files use build tag suffixes: `process_unix.go`, `process_windows.go`, `core_unix.go`, `core_windows.go`
- One file per logical concern within a package (e.g., `controller/rpc/` has `base.go`, `service.go`, `user.go`, `log.go`, `stats.go`, `middleware.go`)

**Functions:**
- Use PascalCase for exported functions: `NewXray()`, `StartGRPCListener()`, `BuildUsersFromChunks()`
- Use camelCase for unexported functions: `detectBackend()`, `validateApiKey()`, `checkBackendStatus()`
- Constructor functions use `New` prefix: `New()`, `NewXray()`, `NewXRayCore()`, `NewXRayConfig()`
- Start/listener functions use `Start` prefix: `StartGRPCListener()`, `StartHttpListener()`

**Variables:**
- Use camelCase for local variables and struct fields: `clientIP`, `apiPort`, `lastRequest`
- Package-level test vars use camelCase: `servicePort`, `nodeHost`, `sslCertFile`
- Constants use PascalCase for exported, camelCase for unexported: `NodeVersion`, `LogDebug`, `logPhaseRuntime`

**Types:**
- Struct names use PascalCase: `Controller`, `Service`, `Config`, `Core`, `Xray`
- Interface names use PascalCase without `I` prefix: `Backend`, `Service`
- Context key types use empty structs: `ConfigKey{}`, `UsersKey{}` (in `backend/backend.go`)
- Custom string types for enums: `type LogLevel string` with const block (in `logger/logger.go`)

**Acronyms:**
- Mixed casing for acronyms: `XRay` in function names (`NewXRayConfig`, `NewXRayCore`) but `Xray` for struct name
- `gRPC` in comments, `GRPC` in function names: `StartGRPCListener()`
- `HTTP`/`Http` mixed: `StartHttpListener()` (should be `StartHTTPListener` per Go conventions)

## Code Style

**Formatting:**
- Standard `gofmt` formatting (no custom formatter config detected)
- Tab indentation (Go default)
- No `.editorconfig`, `.prettierrc`, or custom formatting config files present

**Linting:**
- No linting config detected (no `.golangci.yml`, `golint` config, etc.)
- Code follows standard Go conventions with minor inconsistencies (see Acronyms above)

## Import Organization

**Order:**
1. Standard library packages (`context`, `fmt`, `log`, `sync`, `time`, etc.)
2. Third-party packages (`github.com/google/uuid`, `google.golang.org/grpc`, etc.)
3. Internal project packages (`github.com/pasarguard/node/...`)

**Separator:** Blank line between each group.

**Example from `controller/controller.go`:**
```go
import (
    "context"
    "errors"
    "log"
    "sync"
    "time"

    "github.com/google/uuid"

    "github.com/pasarguard/node/backend"
    "github.com/pasarguard/node/backend/xray"
    "github.com/pasarguard/node/common"
    "github.com/pasarguard/node/config"
    "github.com/pasarguard/node/tools"
)
```

**Package Aliases:**
- Use aliases only when necessary to avoid conflicts: `nodeLogger "github.com/pasarguard/node/logger"` (in `backend/xray/core.go`)
- gRPC middleware aliased: `grpcmiddleware "github.com/grpc-ecosystem/go-grpc-middleware"` (in `controller/rpc/middleware.go`)

## Error Handling

**Patterns:**
- Return `error` as last return value following Go conventions
- Use `errors.New("message")` for simple errors in business logic
- Use `fmt.Errorf("context: %w", err)` for wrapping with context (e.g., `fmt.Errorf("failed to create config directory: %v", err)` in `backend/xray/core.go`)
- Use `status.Errorf(codes.X, "message")` for gRPC error responses (in `controller/rpc/middleware.go`)
- Use `http.Error(w, "message", statusCode)` for REST error responses (in `controller/rest/middleware.go`)
- Fatal errors at startup use `log.Fatal(err)` in `main.go`
- Non-fatal errors logged with `log.Printf("warning: %v", err)` pattern

**Error checking:**
- Always check errors immediately after function calls
- Use `t.Fatal(err)` or `t.Fatalf()` in tests for critical failures
- Ignored errors explicitly marked with `_ =` (e.g., `_ = c.process.Process.Kill()` in `backend/xray/core.go`)

**gRPC to HTTP error mapping:**
- `common.GrpcCodeToHTTP()` in `common/helper.go` maps gRPC codes to HTTP status codes
- `common.InterceptNotFound()` wraps errors ending in "not found." as gRPC NotFound status

## Logging

**Framework:** Standard library `log` package

**Patterns:**
- Use `log.Println()` for informational messages: `log.Println("xray started, Version:", xray.Version())`
- Use `log.Printf()` for formatted messages: `log.Printf("syncing %d users on startup", len(users))`
- Use `log.Fatal()` / `log.Fatalf()` only at application startup in `main.go` and `TestMain()`
- Custom `logger.Logger` struct in `logger/logger.go` for Xray process log routing (access vs error logs)
- REST middleware prefixes log entries with `[API]`: `log.Printf("[API] New request from %s, %s, %s\n", ...)`
- Warning messages use `[Warning]` prefix in strings, not a log level framework

## Comments

**When to Comment:**
- Function-level comments on exported functions and types (inconsistent -- some exported functions lack comments)
- Inline comments for non-obvious logic steps: `// Clean up any orphaned xray processes before starting new one`
- Section comments using `//` to separate logical blocks within functions
- TODO/FIXME comments: not widely used

**JSDoc/TSDoc:**
- Not applicable (Go project)
- GoDoc-style comments used sparingly: `// ProcessInfo holds information about a process` (in `backend/xray/core.go`)
- `// BuildUsersFromChunks orders chunked user payloads by their index...` (in `controller/users_chunk.go`)

## Function Design

**Size:** Functions are generally moderate in size (20-60 lines). Larger functions include `NewXray()` (~85 lines in `backend/xray/xray.go`) and `TestMain()` functions in test files.

**Parameters:**
- Use `context.Context` as first parameter where applicable
- Use pointer receivers for struct methods: `func (c *Controller)`, `func (x *Xray)`, `func (s *Service)`
- Config structs passed as pointers: `*config.Config`, `*Config`
- Protobuf messages passed as pointers: `*common.User`, `*common.Backend`

**Return Values:**
- Follow `(result, error)` pattern: `func NewXray(...) (*Xray, error)`
- Shutdown/start functions return `(shutdownFunc, service, error)` triple
- Void operations that can fail return just `error`
- Void operations that cannot fail return nothing: `func (x *Xray) Shutdown()`

## Module Design

**Exports:**
- Each package exports a primary struct (e.g., `Controller`, `Service`, `Xray`, `Core`, `Config`)
- Constructor function `New()` or `NewX()` is the primary entry point for each package
- Interface definitions live in the package that consumes them: `Backend` interface in `backend/backend.go`, `Service` interface in `controller/controller.go`

**Barrel Files:**
- Not used. Go packages serve as natural namespaces.

## Concurrency Patterns

**Mutex usage:**
- `sync.RWMutex` for read-heavy data: `Controller.mu`, `Xray.mu`, `Logger.mu`
- `sync.Mutex` for write-heavy data: `Core.mu`, `Core.startupMu`
- Always use `defer mu.Unlock()` pattern after locking
- Lock acquisitions are minimal scope: lock, read/write, unlock

**Goroutines:**
- Background tasks launched with `go func()` pattern
- Context cancellation used for goroutine lifecycle: `context.WithCancel()` + `cancelFunc()`
- Channel-based communication for log streaming: `logsChan chan string`

## Struct Embedding

**Pattern:** Both REST and gRPC `Service` structs embed `controller.Controller` directly (value embedding, not pointer):
```go
// controller/rpc/service.go
type Service struct {
    common.UnimplementedNodeServiceServer
    controller.Controller
}

// controller/rest/service.go
type Service struct {
    controller.Controller
    Router chi.Router
}
```

This allows both protocol services to share all Controller methods without delegation.

## Configuration Pattern

**Environment-based config:** All configuration via environment variables with defaults, loaded in `config/config.go`:
```go
ServicePort: GetEnvAsInt("SERVICE_PORT", 62050),
```

**Helper functions:** `GetEnv()`, `GetEnvAsInt()`, `GetEnvAsBool()`, `GetEnvAsUUID()` in `config/config.go`

**Test config:** `config.NewTestConfig()` creates a config for testing by loading from env then overriding specific fields.

## Serialization

**Protocol Buffers** for all API communication (both REST and gRPC):
- REST uses `application/x-protobuf` content type with `proto.Marshal()`/`proto.Unmarshal()`
- Helper functions in `common/helper.go`: `ReadProtoBody()`, `SendProtoResponse()`
- Proto definitions in `common/service.proto`, generated code in `common/service.pb.go` and `common/service_grpc.pb.go`

---

*Convention analysis: 2026-03-09*
