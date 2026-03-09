# Codebase Concerns

**Analysis Date:** 2026-03-09

## Tech Debt

**Duplicated REST/gRPC logic across protocol layers:**
- Issue: The REST and gRPC service layers (`controller/rest/` and `controller/rpc/`) duplicate nearly identical business logic for Start, Stop, SyncUser, SyncUsers, SyncUsersChunked, stats, and logs. Each endpoint is implemented twice with protocol-specific marshalling but the same core flow.
- Files: `controller/rest/base.go`, `controller/rpc/base.go`, `controller/rest/user.go`, `controller/rpc/user.go`, `controller/rest/stats.go`, `controller/rpc/stats.go`, `controller/rest/log.go`, `controller/rpc/log.go`
- Impact: Every new feature or bug fix must be applied in two places. Divergence between REST and gRPC behavior is likely over time (e.g., the chunked sync threshold of 100 users is hardcoded in both `controller/rest/user.go:133` and `controller/rpc/user.go:76`).
- Fix approach: Extract shared business logic into `controller/controller.go` methods. REST/gRPC layers should only handle protocol-specific marshalling and call shared controller methods.

**Duplicated stats parsing in `backend/xray/api/stats.go`:**
- Issue: Seven stat-fetching functions (`GetUsersStats`, `GetInboundsStats`, `GetOutboundsStats`, `GetUserStats`, `GetInboundStats`, `GetOutboundStats`) contain near-identical `strings.Split(data, ">>>")` parsing logic with inconsistent field ordering. Some use `parts[1]=name, parts[2]=link, parts[3]=statType` while `GetUserStats`/`GetInboundStats`/`GetOutboundStats` use `parts[1]=name, parts[2]=statType, parts[3]=link`.
- Files: `backend/xray/api/stats.go`
- Impact: The inconsistent field ordering between "list all" vs "get specific" functions suggests either a bug or an undocumented Xray stat name format difference. A malformed stat name will cause an index-out-of-bounds panic.
- Fix approach: Extract a `parseStat(name string) *common.Stat` helper. Add bounds checking on the `parts` slice before indexing.

**Hardcoded magic number for chunked sync threshold:**
- Issue: The threshold of 100 users to decide between API-based update vs restart is hardcoded in both protocol layers.
- Files: `controller/rest/user.go:133`, `controller/rpc/user.go:76`
- Impact: Cannot tune without code change. Inconsistency risk if changed in one place but not the other.
- Fix approach: Move to `config.Config` as a configurable env var with a sensible default.

**Version string manually maintained:**
- Issue: `NodeVersion` is a hardcoded constant in `controller/controller.go:19`. Bumping the version requires editing source code.
- Files: `controller/controller.go:19`
- Impact: Easy to forget during release. No way to inject version at build time.
- Fix approach: Use `-ldflags` to inject version at build time from git tags.

## Known Bugs

**`recordSystemStats` hot-loops without sleep:**
- Symptoms: The `recordSystemStats` goroutine in `controller/controller.go:142-158` runs in a tight loop with no delay between iterations. The `default` case in the select fires continuously.
- Files: `controller/controller.go:142-158`
- Trigger: Every time `Connect()` is called, this goroutine starts and immediately consumes 100% of one CPU core.
- Workaround: The `GetSystemStats()` function inside calls `time.Sleep(1*time.Second)` twice (for CPU sampling and bandwidth), which provides some implicit throttling, but this is wasteful -- it blocks the goroutine for ~2 seconds per iteration while still semantically hot-looping.
- Fix: Replace `default` with a `time.Ticker` case (e.g., 5-10 second interval) to poll system stats at a reasonable frequency.

**`Config.ToBytes()` mutates shared state under read lock:**
- Symptoms: `Config.ToBytes()` in `backend/xray/config.go:293-385` acquires RLocks on all inbounds, then mutates `i.Settings["clients"]` (line 310-361) and mutates `c.LogConfig.AccessLog`/`ErrorLog` (lines 365-368, restored on 373-374) while holding only read locks.
- Files: `backend/xray/config.go:293-385`
- Trigger: Concurrent calls to `ToBytes()` (e.g., during restart + health check) cause data races on the Settings map and LogConfig fields.
- Workaround: None currently.
- Fix: Either use write locks, or build a separate serialization struct instead of mutating the live config.

**`FindFreePort` can return a port beyond 65535 or fail silently:**
- Symptoms: `FindFreePort()` in `tools/port.go:18-26` starts at a random port and increments until finding a free one. If the random start is high (e.g., 59999), the loop increments past 65535, returning an invalid port. The loop also has no fallback if no port is free.
- Files: `tools/port.go:18-26`
- Trigger: On a system with many ports in use, or when the random start is near 65535.
- Workaround: None.
- Fix: Use `net.Listen("tcp", ":0")` to let the OS assign a free port, then extract the port number from the listener address.

**Potential nil pointer in `Xray.SyncUser` and `Xray.UpdateUsers`:**
- Symptoms: `SyncUser()` and `UpdateUsers()` in `backend/xray/user.go` access `x.handler` and `x.config` without holding the mutex lock.
- Files: `backend/xray/user.go:128-164`, `backend/xray/user.go:177-209`
- Trigger: If `Shutdown()` is called concurrently (e.g., via keep-alive timeout), `x.handler` becomes nil causing a panic.
- Workaround: None.
- Fix: Add mutex protection around handler/config access, or copy references under lock before use.

## Security Considerations

**API key comparison is not constant-time:**
- Risk: The UUID comparison `key != apiKey` in both `controller/rest/middleware.go:27` and `controller/rpc/middleware.go:37` uses standard equality which is susceptible to timing attacks.
- Files: `controller/rest/middleware.go:27`, `controller/rpc/middleware.go:37`
- Current mitigation: TLS transport encryption makes timing attacks harder but not impossible.
- Recommendations: Use `subtle.ConstantTimeCompare` on the raw UUID bytes for constant-time comparison.

**No rate limiting on authentication endpoints:**
- Risk: No rate limiting on API key validation allows brute-force attempts against the UUID API key.
- Files: `controller/rest/middleware.go`, `controller/rpc/middleware.go`
- Current mitigation: TLS requirement, UUID keyspace (128-bit) makes brute force impractical.
- Recommendations: Add rate limiting per source IP for failed authentication attempts.

**Xray internal API uses insecure gRPC (no TLS):**
- Risk: The internal gRPC connection from the node to Xray's API on `127.0.0.1` uses `insecure.NewCredentials()`.
- Files: `backend/xray/api/base.go:30-32`
- Current mitigation: Bound to `127.0.0.1` only. The API inbound in Xray config also restricts source IPs to local addresses via `apiRuleSources()` in `backend/xray/config.go:499-546`.
- Recommendations: Acceptable for localhost communication. No action needed unless multi-container deployment changes networking.

**Config file written with broad permissions:**
- Risk: Generated Xray config (which may contain user credentials/keys) is written to disk via `os.Create` which uses 0666 permissions (modified by umask).
- Files: `backend/xray/core.go:75`
- Current mitigation: Only written in debug mode.
- Recommendations: Use `os.OpenFile` with 0600 permissions when writing config files.

**Log files opened with 0666 permissions:**
- Risk: Log files may contain user email addresses and IP addresses.
- Files: `logger/logger.go:48`
- Current mitigation: None.
- Recommendations: Use 0640 or 0600 permissions for log files.

**API key load failure is non-fatal:**
- Risk: In `config/config.go:47-50`, if `API_KEY` env var is missing or invalid, the error is logged but the server starts with a nil/zero UUID key. This means any request with the zero UUID could authenticate.
- Files: `config/config.go:47-50`
- Current mitigation: None.
- Recommendations: Make API key loading fatal. If no valid API key is configured, the service should refuse to start.

## Performance Bottlenecks

**System stats polling blocks for ~2 seconds per call:**
- Problem: `GetSystemStats()` in `tools/sys.go` calls `cpu.Percent(time.Second, false)` (1s blocking) and `getBandwidthSpeed()` (another 1s `time.Sleep`). Each stats recording cycle takes ~2 seconds minimum.
- Files: `tools/sys.go:13-45`, `tools/sys.go:50-93`
- Cause: Both CPU percentage and bandwidth speed require two samples with a 1-second delay between them.
- Improvement path: Run CPU and bandwidth sampling concurrently using goroutines within `GetSystemStats()`. This would reduce the blocking time from ~2s to ~1s.

**`SyncUsers` always triggers a full Xray restart:**
- Problem: `Xray.SyncUsers()` in `backend/xray/user.go:166-175` replaces all users in config and does a full process restart, even for small changes.
- Files: `backend/xray/user.go:166-175`
- Cause: The sync operation is designed for full replacement, not incremental updates.
- Improvement path: For small deltas, compute a diff and use the Xray API to add/remove users without restart (as `UpdateUsers` already does).

**Log channel drops messages when full:**
- Problem: The log channel in `backend/xray/log.go:49-53` and `backend/xray/log.go:59-65` uses non-blocking sends, silently dropping logs when the buffer (default 10000) is full.
- Files: `backend/xray/log.go:49-53`, `backend/xray/log.go:59-65`
- Cause: Intentional design to prevent blocking the Xray process, but log loss is invisible.
- Improvement path: Add a dropped-log counter and periodically log how many messages were dropped.

## Fragile Areas

**`Config.ToBytes()` lock ordering and mutation:**
- Files: `backend/xray/config.go:293-385`
- Why fragile: Acquires all inbound RLocks in a loop, mutates shared state under read lock, temporarily modifies LogConfig and restores it. Any panic between mutation and restoration leaves the config in an inconsistent state. Lock release happens in a separate loop which won't run if marshal panics.
- Safe modification: If changing serialization, build a deep copy of the config before marshalling. Use `defer` for all lock releases.
- Test coverage: No direct tests for `ToBytes()`.

**Process lifecycle management in `backend/xray/core.go`:**
- Files: `backend/xray/core.go:119-189`, `backend/xray/core.go:191-242`, `backend/xray/core.go:244-265`
- Why fragile: Complex state machine with process start, stop, restart. The `Restart()` method releases the lock between `Stop()` and `Start()` calls, creating a window where another goroutine could interfere. The `Start()` method calls `c.Started()` while holding the lock, but `Started()` does not use the lock -- it checks `c.process` directly.
- Safe modification: Always test with concurrent health checks running. Verify no zombie processes after stop.
- Test coverage: `backend/xray/xray_test.go` exists but tests are integration tests requiring an Xray binary.

**Keep-alive tracker calling `Disconnect()` from goroutine:**
- Files: `controller/controller.go:122-140`
- Why fragile: The `keepAliveTracker` goroutine calls `c.Disconnect()` which acquires locks and shuts down the backend. This runs concurrently with API request handlers that also access the backend. The `Disconnect()` method is not idempotent -- calling it multiple times could panic if `cancelFunc` is nil after first call.
- Safe modification: Ensure `Disconnect()` is idempotent. Guard `cancelFunc` against nil.
- Test coverage: No tests for keep-alive timeout behavior.

**`/proc` stat parsing in `process_unix.go`:**
- Files: `backend/xray/process_unix.go:100-148`, `backend/xray/process_unix.go:220-240`
- Why fragile: Parses `/proc/[pid]/stat` by splitting on whitespace. The comm field (field 2) can contain spaces and parentheses (e.g., `(xray core)`), which would shift all subsequent field indices. The standard approach is to find the last `)` and parse from there.
- Safe modification: Use a proper `/proc/stat` parser or the `gopsutil` library which already handles this edge case.
- Test coverage: `backend/xray/process_unix_test.go` has tests but may not cover edge cases with spaces in process names.

## Scaling Limits

**Single backend instance per node:**
- Current capacity: One Xray process per node service instance.
- Limit: Cannot run multiple backends simultaneously. A new `Start()` call disconnects the previous client.
- Scaling path: Run multiple node instances behind a load balancer, or refactor to support multiple backend instances keyed by client/session.

**64MB gRPC message size limit:**
- Current capacity: 64MB max message size for gRPC (`controller/rpc/service.go:35`).
- Limit: Very large user lists or configs exceeding 64MB will fail.
- Scaling path: The chunked sync (`SyncUsersChunked`) already addresses this for user data. Config payloads are unlikely to hit this limit.

**Log channel buffer of 10000:**
- Current capacity: 10000 buffered log messages (`config/config.go:43`).
- Limit: High-traffic Xray instances generating logs faster than consumers read them will silently drop logs.
- Scaling path: Increase `LOG_BUFFER_SIZE` env var, or implement log persistence/rotation.

## Dependencies at Risk

**`github.com/grpc-ecosystem/go-grpc-middleware` v1.4.0:**
- Risk: This is the v1 API which is deprecated in favor of `github.com/grpc-ecosystem/go-grpc-middleware/v2`. The v1 API may not receive security updates.
- Files: `controller/rpc/middleware.go:10`
- Impact: Used only for `grpcmiddleware.ChainUnaryServer` and `ChainStreamServer` in middleware chaining.
- Migration plan: Migrate to v2 or use the built-in `grpc.ChainUnaryInterceptor`/`grpc.ChainStreamInterceptor` which were added in recent gRPC versions.

**`github.com/xtls/xray-core`:**
- Risk: Tightly coupled dependency. The node imports Xray's protobuf types (`conf.LogConfig`, `conf.RouterConfig`, `conf.PolicyConfig`, `conf.APIConfig`) for config generation and its gRPC service types for the API client.
- Files: `backend/xray/config.go`, `backend/xray/api/base.go`, `backend/xray/api/stats.go`, `backend/xray/api/proxyman.go`, `backend/xray/api/account.go`
- Impact: Xray version upgrades may break config struct compatibility. Pinned to `v1.260204.0`.
- Migration plan: None needed -- this is the core managed process. Just be cautious with version bumps.

## Missing Critical Features

**No request body size limits on REST API:**
- Problem: The REST API reads entire request bodies into memory (`io.ReadAll(r.Body)` in `controller/rest/user.go:19`, `controller/rest/user.go:55`) without any size limit.
- Blocks: Production hardening. A malicious client could send an arbitrarily large body to exhaust memory.

**No structured logging:**
- Problem: All logging uses `log.Println`/`log.Printf` from the standard library. No log levels, no structured fields, no correlation IDs.
- Blocks: Operational visibility in production. Cannot filter logs by severity or trace requests across log lines.

**No health check endpoint:**
- Problem: No dedicated HTTP/gRPC health check endpoint for load balancers or container orchestrators.
- Blocks: Proper Kubernetes/Docker health checks. The Dockerfile has no `HEALTHCHECK` instruction.

**No graceful handling of concurrent `Start()` calls:**
- Problem: If two clients call `Start()` simultaneously, the second call disconnects the first without error. There is no session management or client identification beyond IP tracking.
- Blocks: Multi-tenant or multi-panel deployments sharing a node.

## Test Coverage Gaps

**No unit tests for controller business logic:**
- What's not tested: `controller/controller.go` -- Connect, Disconnect, keepAliveTracker, recordSystemStats, StartBackend, SystemStats.
- Files: `controller/controller.go`
- Risk: Race conditions in concurrent access patterns, keep-alive timeout behavior, and disconnect cleanup are untested.
- Priority: High

**No unit tests for config generation:**
- What's not tested: `backend/xray/config.go` -- `ToBytes()`, `ApplyAPI()`, `syncUsers()`, `updateUsers()`, `checkPolicy()`, JSON serialization correctness.
- Files: `backend/xray/config.go`
- Risk: Config generation bugs cause Xray startup failures in production with no test safety net. The `ToBytes()` mutation-under-read-lock bug is undetected.
- Priority: High

**No tests for REST/gRPC middleware:**
- What's not tested: API key validation, backend status checks, request tracking middleware.
- Files: `controller/rest/middleware.go`, `controller/rpc/middleware.go`
- Risk: Authentication bypass or middleware ordering issues go undetected.
- Priority: Medium

**Integration tests require external Xray binary:**
- What's not tested: All tests in `controller/rest/rest_test.go` and `controller/rpc/rpc_test.go` require a running Xray binary, making them integration tests that cannot run in CI without special setup.
- Files: `controller/rest/rest_test.go`, `controller/rpc/rpc_test.go`
- Risk: Tests are likely skipped in CI environments, reducing effective coverage.
- Priority: Medium

**No tests for stats parsing:**
- What's not tested: `backend/xray/api/stats.go` -- all stat name parsing with `strings.Split`. No test verifies that the `>>>` delimiter parsing handles malformed stat names.
- Files: `backend/xray/api/stats.go`
- Risk: Index-out-of-bounds panic on malformed stat names from Xray.
- Priority: Medium

---

*Concerns audit: 2026-03-09*
