# Testing Patterns

**Analysis Date:** 2026-03-09

## Test Framework

**Runner:**
- Go standard `testing` package
- No additional test framework (no testify, gomock, etc.)

**Assertion Library:**
- None. All assertions use manual `if` checks with `t.Fatal()` / `t.Fatalf()` / `t.Errorf()`

**Run Commands:**
```bash
make test                     # Unit tests only: TEST_INTEGRATION=false go test ./... -v -p 1
make test-integration         # Integration tests: TEST_INTEGRATION=true go test ./... -v -p 1
go test ./backend/xray/ -run TestFunctionName -v   # Run a single test
```

**Key flags:**
- `-p 1` forces serial package execution (prevents parallel package runs)
- `-v` verbose output always enabled in Makefile targets
- `TEST_INTEGRATION` environment variable gates integration tests

## Test File Organization

**Location:**
- Co-located with source files in the same package (same directory)

**Naming:**
- `*_test.go` suffix: `xray_test.go`, `log_test.go`, `rpc_test.go`, `rest_test.go`
- Test functions prefixed with `Test`: `TestDetectLogType`, `TestGRPC_GetBackendStats`, `TestREST_GetOutboundsStats`

**Structure:**
```
backend/xray/
    xray.go                        # Source
    xray_test.go                   # Integration test (requires Xray binary)
    log.go                         # Source
    log_test.go                    # Unit test
    process_unix.go                # Source
    process_unix_test.go           # Integration test (requires Xray binary)
    startup_diagnostics.go         # Source
    startup_diagnostics_test.go    # Unit test
controller/
    users_chunk.go                 # Source
    users_chunk_test.go            # Unit test
    rpc/
        *.go                       # Source
        rpc_test.go                # Integration test (requires Xray + TLS certs)
    rest/
        *.go                       # Source
        rest_test.go               # Integration test (requires Xray + TLS certs)
```

## Test Structure

**Unit Test Pattern (table-driven):**
```go
// From backend/xray/log_test.go
func TestDetectLogType(t *testing.T) {
    tests := []struct {
        name          string
        logMessage    string
        expectInError bool
    }{
        {
            name:          "Access log - TCP connection",
            logMessage:    "2025/10/06 ...",
            expectInError: false,
        },
        // ... more cases
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            tmpDir := t.TempDir()
            // ... create test fixtures

            // Act
            core.detectLogType(tt.logMessage)

            // Assert
            if !strings.Contains(string(logContent), tt.logMessage) {
                t.Errorf("Log file does not contain expected message...")
            }
        })
    }
}
```

**Unit Test Pattern (simple):**
```go
// From backend/xray/startup_diagnostics_test.go
func TestStartupLogRingTail(t *testing.T) {
    ring := newStartupLogRing(3)
    ring.add("l1")
    ring.add("l2")
    // ...
    if len(tail) != len(want) {
        t.Fatalf("unexpected tail length: got %d, want %d", len(tail), len(want))
    }
}
```

**Integration Test Pattern (TestMain setup/teardown):**
```go
// From controller/rpc/rpc_test.go
func TestMain(m *testing.M) {
    // Setup: start real server
    cfg := config.NewTestConfig(generatedConfigPath, apiKey)
    tlsConfig, err := tools.LoadTLSCredentials(sslCertFile, sslKeyFile)
    shutdownFunc, s, err := StartGRPCListener(tlsConfig, addr, cfg)
    // ... create client, start backend

    sharedTestCtx = &testContext{
        client:         client,
        ctxWithSession: ctxWithSession,
        // ...
    }

    code := m.Run()

    // Teardown: stop server
    conn.Close()
    s.Disconnect()
    shutdownFunc(ctx)

    os.Exit(code)
}
```

**Patterns:**
- `TestMain()` used in `controller/rpc/rpc_test.go` and `controller/rest/rest_test.go` for server lifecycle
- Shared test state via package-level `sharedTestCtx` variable
- `t.Helper()` used in test helper functions: `resolveXrayBinary(t)`, `startXrayProcess(t, exe)` (in `backend/xray/process_unix_test.go`)
- `t.TempDir()` for temporary file creation
- `t.Skip()` for skipping tests when prerequisites are missing
- `t.Cleanup()` for deferred resource cleanup

## Mocking

**Framework:** None

**Patterns:**
- No mocking framework is used
- Integration tests use real servers and real Xray binaries
- Unit tests operate on internal structs directly (e.g., creating `Core{}` or `startupLogRing` directly)

**What is mocked:**
- Nothing is mocked. Tests either:
  1. Test internal logic directly (unit tests on `startup_diagnostics.go`, `users_chunk.go`, `log.go`)
  2. Spin up a real server with a real Xray backend (integration tests)

**What to mock (if adding mocks):**
- The `backend.Backend` interface would be the natural mock boundary
- Currently untested: `tools/` package functions, `config/` loading

## Fixtures and Factories

**Test Data:**
```go
// Package-level vars in test files define shared paths and config
var (
    servicePort         = 8002
    nodeHost            = "127.0.0.1"
    sslCertFile         = "../../certs/ssl_cert.pem"
    sslKeyFile          = "../../certs/ssl_key.pem"
    apiKey              = uuid.New()
    generatedConfigPath = "../../generated/"
    configPath          = "../../backend/xray/config.json"
)

// Test config factory
cfg := config.NewTestConfig(generatedConfigPath, apiKey)
```

**User fixtures** are constructed inline in tests:
```go
user := &common.User{
    Email:    "test_user1@example.com",
    Inbounds: []string{"VMESS TCP NOTLS", "VLESS TCP REALITY"},
    Proxies: &common.Proxy{
        Vmess: &common.Vmess{Id: uuid.New().String()},
        Vless: &common.Vless{Id: uuid.New().String()},
    },
}
```

**Test context struct** used in integration tests:
```go
// controller/rpc/rpc_test.go
type testContext struct {
    client         common.NodeServiceClient
    ctxWithSession context.Context
    shutdownFunc   func(ctx context.Context) error
    service        controller.Service
    conn           *grpc.ClientConn
}
```

**Location:**
- No dedicated fixtures directory
- Test data constructed inline or via package-level vars
- External fixture: `backend/xray/config.json` (Xray configuration file used by integration tests)
- TLS certificates expected at `certs/ssl_cert.pem` and `certs/ssl_key.pem` (relative to project root)

## Coverage

**Requirements:** None enforced. No coverage thresholds configured.

**View Coverage:**
```bash
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out
```

## Test Types

**Unit Tests:**
- Pure logic tests that do not require external binaries or network
- Located in:
  - `backend/xray/log_test.go` - Log type detection
  - `backend/xray/startup_diagnostics_test.go` - Startup log ring buffer, failure detection
  - `controller/users_chunk_test.go` - Chunk assembly logic
- Run with: `make test` (sets `TEST_INTEGRATION=false`)

**Integration Tests:**
- Require a running Xray binary, TLS certificates, and network access
- Located in:
  - `backend/xray/xray_test.go` - Full Xray backend lifecycle
  - `backend/xray/process_unix_test.go` - Process management with real Xray binary
  - `controller/rpc/rpc_test.go` - Full gRPC service test suite
  - `controller/rest/rest_test.go` - Full REST service test suite
- Run with: `make test-integration` (sets `TEST_INTEGRATION=true`)
- Prerequisites:
  - Xray binary at `/usr/local/bin/xray` or `XRAY_EXECUTABLE_PATH` env var
  - TLS certs at `certs/ssl_cert.pem` and `certs/ssl_key.pem`
  - Xray config at `backend/xray/config.json`

**E2E Tests:**
- Not used as a separate category. The integration tests in `rpc_test.go` and `rest_test.go` function as E2E tests, starting real servers and making real API calls.

**Note:** The gating mechanism for integration vs unit tests is the `TEST_INTEGRATION` env var, but the test code itself does not check this variable. The distinction is enforced only at the Makefile level. Some integration tests use `t.Skip()` when binaries are not found (e.g., `process_unix_test.go`).

## Common Patterns

**Context with Timeout:**
```go
// Used consistently in integration tests
ctx, cancel := context.WithTimeout(sharedTestCtx.ctxWithSession, 5*time.Second)
defer cancel()
```

**Error Testing:**
```go
// Check for specific error content
_, err := BuildUsersFromChunks(map[uint64][]*common.User{}, 0, false)
if err == nil || !strings.Contains(err.Error(), "missing final chunk indicator") {
    t.Fatalf("expected missing final chunk indicator error, got: %v", err)
}

// Check gRPC error codes
st, _ := status.FromError(err)
if st.Code() != codes.NotFound {
    t.Fatalf("Expected NotFound error, got: %v", err)
}
```

**Naming convention for integration test functions:**
- gRPC tests: `TestGRPC_MethodName` (e.g., `TestGRPC_GetBackendStats`, `TestGRPC_SyncUsers`)
- REST tests: `TestREST_MethodName` (e.g., `TestREST_GetOutboundsStats`, `TestREST_SyncUser`)
- Unit tests: `TestDescriptiveName` (e.g., `TestDetectLogType`, `TestBuildUsersFromChunksOrdersByIndex`)

**Build tags for platform-specific tests:**
```go
//go:build !windows
// Used in backend/xray/process_unix_test.go
```

**Test helper functions** (in `backend/xray/process_unix_test.go`):
```go
func resolveXrayBinary(t *testing.T) string {
    t.Helper()
    // ... finds Xray binary or skips test
}

func startXrayProcess(t *testing.T, exe string) (pid int, waitCh <-chan error) {
    t.Helper()
    // ... starts process, registers t.Cleanup()
}
```

---

*Testing analysis: 2026-03-09*
