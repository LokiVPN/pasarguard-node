# Technology Stack

**Analysis Date:** 2026-03-09

## Languages

**Primary:**
- Go 1.25.7 - All application code

**Secondary:**
- Protobuf (proto3) - gRPC service contract definition (`common/service.proto`)
- Shell/Make - Build automation and Xray installation scripts

## Runtime

**Environment:**
- Go 1.25.7 (compiled binary, no runtime dependency beyond OS)
- Alpine Linux (production Docker image)

**Package Manager:**
- Go Modules (`go.mod`)
- Lockfile: `go.sum` present (256 lines)

## Frameworks

**Core:**
- go-chi/chi v5.2.5 - REST HTTP router (`controller/rest/`)
- google.golang.org/grpc v1.78.0 - gRPC server and client (`controller/rpc/`, `backend/xray/api/`)
- google.golang.org/protobuf v1.36.11 - Protobuf serialization (`common/`)

**Testing:**
- Go standard `testing` package - All tests
- No external test framework

**Build/Dev:**
- GNU Make - Build orchestration (`Makefile`)
- protoc with go/go-grpc plugins - Code generation from `.proto` files
- Docker multi-stage build (`Dockerfile`)

## Key Dependencies

**Critical:**
- `github.com/xtls/xray-core` v1.260204.0 - Xray proxy core library; the entire application manages Xray instances. Used for proxyman/stats gRPC client types in `backend/xray/api/`
- `github.com/go-chi/chi/v5` v5.2.5 - REST API routing with middleware chaining (`controller/rest/service.go`)
- `google.golang.org/grpc` v1.78.0 - Both the node's own gRPC API server and the internal client to Xray's gRPC API

**Infrastructure:**
- `github.com/shirou/gopsutil/v4` v4.26.1 - System stats collection: CPU, memory, network bandwidth (`tools/sys.go`)
- `github.com/joho/godotenv` v1.5.1 - Environment variable loading from `.env` files (`config/config.go`)
- `github.com/google/uuid` v1.6.0 - API key generation and validation (`config/config.go`, middleware)
- `github.com/grpc-ecosystem/go-grpc-middleware` v1.4.0 - gRPC interceptor chaining (`controller/rpc/middleware.go`)

**Indirect (notable):**
- `github.com/apernet/quic-go` - QUIC protocol support (via xray-core)
- `github.com/refraction-networking/utls` - TLS fingerprint support (via xray-core)
- `github.com/sagernet/sing-shadowsocks` - Shadowsocks protocol (via xray-core)
- `github.com/xtls/reality` - REALITY protocol (via xray-core)
- `gvisor.dev/gvisor` - Network stack (via xray-core)
- `golang.zx2c4.com/wireguard` - WireGuard support (via xray-core)

## Configuration

**Environment:**
- All configuration via environment variables or `.env` file
- Loaded by `config/config.go` using `godotenv`
- Key variables:
  - `SERVICE_PORT` (default: 62050) - API listener port
  - `SERVICE_PROTOCOL` (default: "grpc") - Protocol selection: "grpc" or "rest"
  - `API_KEY` - UUID for request authentication (required)
  - `NODE_HOST` (default: "0.0.0.0") - Bind address
  - `XRAY_EXECUTABLE_PATH` (default: "/usr/local/bin/xray") - Xray binary location
  - `XRAY_ASSETS_PATH` (default: "/usr/local/share/xray") - Xray assets (geoip/geosite)
  - `SSL_CERT_FILE` (default: "/var/lib/pg-node/certs/ssl_cert.pem") - TLS certificate
  - `SSL_KEY_FILE` (default: "/var/lib/pg-node/certs/ssl_key.pem") - TLS private key
  - `GENERATED_CONFIG_PATH` (default: "/var/lib/pg-node/generated/") - Debug config output
  - `DEBUG` (default: false) - Enable debug logging and config file generation
  - `LOG_BUFFER_SIZE` (default: 10000) - Log channel buffer size
  - `STARTUP_LOG_TAIL_SIZE` (default: 200) - Startup log lines to retain

**Build:**
- `Makefile` - Build targets, cross-compilation, Xray installation
- `Dockerfile` - Multi-stage build: golang:1.25.7-alpine builder, alpine:latest runtime
- `.github/workflows/` - CI/CD pipelines (test, release, Docker build)

## Platform Requirements

**Development:**
- Go 1.25.7+
- Make
- protoc compiler (for regenerating gRPC stubs)
- OpenSSL (for generating dev TLS certificates)
- Xray binary (for integration tests; install via `make install_xray`)

**Production:**
- Linux (primary target: Alpine in Docker)
- Supported architectures: amd64, arm64, arm (v5/v6/v7), 386, s390x
- Also builds for: FreeBSD (amd64/arm64/armv7), OpenBSD (amd64/arm64)
- Docker image: `pasarguard/node` (Docker Hub) or `ghcr.io/pasarguard/node` (GHCR)
- Network mode: host (required for proxy functionality)
- Volume mount: `/var/lib/pg-node` for certs and generated configs
- TLS certificates required for API access

**Version:**
- Node version: `0.2.9` (defined as `NodeVersion` constant in `controller/controller.go`)

---

*Stack analysis: 2026-03-09*
