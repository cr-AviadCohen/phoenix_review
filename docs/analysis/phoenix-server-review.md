# Phoenix Server — Code Review Findings

## Summary

The Phoenix EDR/XDR platform server codebase is of high overall quality. The Rust and Go services show strong engineering discipline: structured logging via `tracing`/`slog`, proper gRPC health endpoints, consistent error propagation, and robust multi-tenancy enforcement at the database layer. The TypeScript portal follows strict TypeScript settings and correctly sources `org_id` from authenticated sessions throughout. The main concerns are: (1) an inconsistency between coding standards prescribing `thiserror` for library crates and the actual use of `anyhow` in several services; (2) a handful of `unwrap()`/`expect()` calls in production-path Rust code outside test modules; (3) the `SINGLE_EVENT_QUERY` in `event-store` uses string interpolation to inject `event_id` and `org_id` directly into ClickHouse SQL; and (4) Go services do not uniformly use the `PHOENIX_` env prefix required by standards.

---

## Standards Compliance

| Area | Status | Notes |
|------|--------|-------|
| Rust error handling | ⚠️ | `eyre`/`color-eyre` used correctly in binaries; `anyhow` leaks into library modules in `idm-service`, `notification-service`, and others where `thiserror` is required |
| Rust logging (tracing + org_id) | ⚠️ | `tracing` used consistently; `#[tracing::instrument]` present in 51 files but not universally applied; `org_id` included in spans in some services (`vault-service`, `asset-store`, `xdr-action-store`) but absent in others (`correlation-service`, `event-store`) |
| Rust config (clap + env) | ✅ | `clap::Parser` with `env = "..."` attributes used in all sampled services; `sensor-auth` uses `figment` instead — a deliberate deviation that is documented |
| Rust health endpoints | ✅ | All three sampled services expose HTTP `/healthz` and/or gRPC `grpc.health.v1.Health` |
| No unsafe code | ✅ | Production `unsafe` blocks found only in `vault-service/src/secure_memory.rs` (justified: `mlock`/`munlock` syscalls) and `cep-service` benchmark harness. `unsafe_code = "forbid"` lint is set workspace-wide |
| Go logging (slog) | ⚠️ | `clickhouse-ingester-v2` uses `log/slog` correctly; `xdr-worker` uses `github.com/sirupsen/logrus` — a standards violation |
| Go config (viper + PHOENIX_ prefix) | ⚠️ | `clickhouse-ingester-v2` uses `PHOENIX_INGESTER_` prefix (compliant); `xdr-worker` uses `XDR_WORKER_` prefix (non-compliant); `xdr-worker-v2` uses `XDR_WORKER_V2_` prefix (non-compliant) |
| Go error wrapping | ⚠️ | Most errors use `fmt.Errorf("...: %w", err)` correctly; `clickhouse-ingester-v2` has ~10 sentinel errors that do not wrap an underlying cause (acceptable for leaf errors, but inconsistent style) |
| TypeScript strict mode | ✅ | `"strict": true` and `"noUncheckedIndexedAccess": true` confirmed in `phoenix-portal/tsconfig.json` |
| TypeScript no `any` | ⚠️ | One `any[]` found in production router code (`investigation.ts:1311`); one `any` in `exclusions-list.tsx` with a comment acknowledging the issue |
| Zod validation | ✅ | All sampled tRPC procedures use `z.object(...)` input schemas |
| i18n compliance | ⚠️ | Server-side tRPC error messages contain hardcoded English strings (e.g., `investigation.ts:1262`, `investigation.ts:1430`); these are server error strings that typically do not require i18n, but it is inconsistent with front-end i18n standards |
| Multi-tenancy (org_id) | ⚠️ | Correctly enforced in most paths; `SINGLE_EVENT_QUERY` uses direct string interpolation instead of parameterized binding — see Critical Issues |

---

## Rust Services — Findings

### Services Sampled

- `rust/correlation-service` (`src/main.rs`, `src/config.rs`, `src/error.rs`, `src/infrastructure/storage/postgres.rs`)
- `rust/event-store` (`src/main.rs`, `src/config.rs`, `src/repositories/event_repository.rs`, `src/repositories/queries/single_event.rs`)
- `rust/sensor-auth` (`src/main.rs`, `src/config.rs`)

### Strengths

- **Clean startup patterns**: All three services use `color_eyre::install()` at binary entry, propagate errors with `?` and `.context()`, and call `process::exit(1)` on fatal initialization errors. No panics at startup.
- **gRPC health**: Both `event-store` and `correlation-service` register `grpc.health.v1.Health` via the shared `GrpcHealthService`; `sensor-auth` uses `common::healthz::start_health_server` for HTTP.
- **Config via clap**: `event-store` and `correlation-service` use `#[derive(Parser)]` with `env = "..."` on every field. No bare `std::env::var` calls in config modules.
- **OTel tracing**: All three services initialize OpenTelemetry through the shared `common::tracing` crate with a `GrpcServerTraceLayer` on the tonic server.
- **Parameterized Postgres queries**: Every `sqlx` query in `postgres.rs` (correlation-service) uses `$1`, `$2`, etc. with `.bind(...)`. No string interpolation into SQL.
- **Deprecated API clearly marked**: `PostgresRepositories::new_with_clickhouse` in `correlation-service/src/infrastructure/storage/postgres.rs` is annotated `#[deprecated]` with a migration note.
- **`secrecy` crate usage**: `sensor-auth/src/config.rs` uses `SecretString` and `SecretBox` from the `secrecy` crate for cryptographic keys, preventing accidental logging.
- **Secure memory in vault-service**: `vault-service/src/secure_memory.rs` uses `mlock`/`munlock` and `zeroize` for secrets in memory — security-correct use of `unsafe`.

### Concerns

- **[Critical] SQL string interpolation in `event-store`**: `SINGLE_EVENT_QUERY` at `/rust/event-store/src/repositories/queries/single_event.rs:136` injects `{event_id}` and `{org_id}` via `.replace()` in `event_repository.rs:413-414`. While `org_id` is a `u64` (safe), `event_id` is a `u128` derived from bytes — the injection path does convert to a numeric string, mitigating direct SQL injection. However, this breaks the parameterized query invariant and creates a maintenance hazard. All SQL values should go through `.bind()`.
- **[High] `anyhow` used in library crates**: `idm-service/src/extractor.rs`, `idm-service/src/cache.rs`, `idm-service/src/consumer.rs`, `idm-service/src/storage.rs`, and `notification-service/src/service.rs` import `anyhow::Result` or `anyhow::Error`. The coding standard requires `thiserror` in library crates. `anyhow` erases type information, preventing downstream callers from matching error variants.
- **[High] `#[instrument]` coverage is incomplete**: Only 51 files in the entire Rust workspace use `tracing::instrument`. The `correlation-service` gRPC handlers and `event-store` service layer have no `#[instrument]` decorations, meaning incoming requests generate no per-request spans in the trace backend.
- **[Medium] `org_id` missing from span fields in most services**: Only `vault-service`, `asset-store`, and `xdr-action-store` include `org_id` in `#[instrument(fields(org_id = ...))]`. Debugging multi-tenant issues across `correlation-service` or `event-store` requires correlating separately.
- **[Medium] `expect()` in production code paths**: `notification-dispatcher/src/sendgrid.rs:20` calls `.expect("Failed to create HTTP client for SendGrid")`; `notification-dispatcher/src/template_resolver.rs:85` calls `.expect("Failed to create HTTP client")`. These will panic the process on any `reqwest` client construction failure. `idm-service/src/cache.rs:78` calls `NonZeroUsize::new(1_600_000).unwrap()` in non-test code (the value is a literal, so it cannot be `None`, but the unwrap is still present).
- **[Medium] `unwrap()` on `fmt::Write` in `notification-store`**: `queue_repository.rs:144` and `:149` call `.unwrap()` on `write!()`. `fmt::Write` for `String` is infallible, so this will never panic in practice, but it violates the no-unwrap guideline and adds noise.
- **[Low] `correlation-service` error type is custom, not `thiserror`**: `error.rs` manually implements `fmt::Display` and `std::error::Error`. Switching to `thiserror` would eliminate the boilerplate and make the impl easier to maintain.

### Code Samples

**Good pattern — parameterized sqlx in postgres.rs (correlation-service):**
```rust
// rust/correlation-service/src/infrastructure/storage/postgres.rs:164-191
let query = r#"
    SELECT DISTINCT ...
    FROM identity_relationships
    WHERE org_id = $2
      AND (parent_identity_key = ANY($1) OR child_identity_key = ANY($1))
"#;
sqlx::query(query)
    .bind(identity_keys_vec)
    .bind(org_id)
    .fetch_all(&*pool)
    .await
```

**Concern — string-interpolated ClickHouse query (event-store):**
```rust
// rust/event-store/src/repositories/queries/single_event.rs:136
WHERE event_id = '{event_id}' AND org_id = {org_id}
// Values are injected via:
// rust/event-store/src/repositories/event_repository.rs:413
let query = SINGLE_EVENT_QUERY
    .replace("{event_id}", &event_id_u128.to_string())
    .replace("{org_id}", &org_id.to_string());
```

---

## Go Services — Findings

### Services Sampled

- `golang/clickhouse-ingester-v2` (`main.go`, `config/config.go`)
- `golang/xdr-worker` (`main.go`, `config/config.go`)

### Strengths

- **`clickhouse-ingester-v2` is well-structured**: Uses `log/slog` throughout, graceful shutdown with timeout, dual-buffer pattern for high-throughput inserts, explicit backoff/retry and pressure-controller configs, HTTP health endpoints (`/health`, `/health/ready`, `/health/live`).
- **Error wrapping**: Both services predominantly use `fmt.Errorf("context: %w", err)`.
- **`viper` for config**: Both services use `viper` with explicit env prefix and `AutomaticEnv()`.
- **OTel integration**: Both services initialize OpenTelemetry via the shared `phoenix/otelinit` package. Span lifecycle is handled correctly with `defer span.End()`.
- **Graceful shutdown**: `clickhouse-ingester-v2` implements a complete shutdown sequence with timeout context, stopping Kafka consumer, ClickHouse client, and metrics server in order.

### Concerns

- **[High] `xdr-worker` uses `logrus` instead of `slog`**: `xdr-worker/main.go:17` imports `github.com/sirupsen/logrus`. The entire `xdr-worker` service logs via `log.WithFields(log.Fields{...}).Info(...)`. This is a direct violation of the Go logging standard requiring `log/slog`. Structured field names and log levels differ between services, complicating centralized log analysis.
- **[High] Non-compliant env prefix**: `xdr-worker/config/config.go:74` sets `v.SetEnvPrefix("XDR_WORKER")`. The standard requires `PHOENIX_`. Same issue in `xdr-worker-v2` which uses `XDR_WORKER_V2`.
- **[Medium] Hardcoded default credentials in `xdr-worker` config**: `config.go:61` sets `v.SetDefault("s3_secret_key", "admin")` and `v.SetDefault("s3_access_key", "admin")`. Even as development defaults these should not appear in committed source, as they may be accidentally used in staging/production if the env vars are not set.
- **[Medium] Non-wrapped sentinel errors in `clickhouse-ingester-v2`**: Several errors like `fmt.Errorf("ClickHouse client is closed")` at `clickhouse/client.go:214` create new errors without wrapping context. This makes `errors.Is`/`errors.As` chains unreliable for callers.
- **[Low] No table-driven tests observed**: The Go test files sampled (`main_otel_test.go` in `clickhouse-ingester-v2`) test individual functions but do not use the table-driven pattern required by standards. The standard requires table-driven tests for coverage of multiple input cases.

---

## TypeScript Portal — Findings

### Files Sampled

- `phoenix-portal/src/server/api/routers/investigation.ts`
- `phoenix-portal/src/server/api/routers/asset.ts`
- `phoenix-portal/src/server/api/routers/agent.ts`
- `phoenix-portal/tsconfig.json`

### Strengths

- **Strict TypeScript enforced**: `tsconfig.json` has `"strict": true` and `"noUncheckedIndexedAccess": true`. 195 test files confirm active testing investment.
- **`org_id` from session**: All sampled routers source organization ID from `ctx.session.user.organizationId` or `ctx.session.currentOrganization.id`. The `agent.ts` router additionally validates cross-org access via `validateOrganizationAccessViaApi` when a caller supplies an explicit `organizationId` input — this is the correct IDOR-prevention pattern.
- **Zod validation on all inputs**: Every sampled procedure uses `z.object(...)` input schemas; none accept raw user input without validation.
- **`organizationScopedProcedure` abstraction**: Several routers use `organizationScopedProcedure` (visible in imports), suggesting a centralized middleware layer that injects `org_id` from the session — good architecture.
- **Observability**: `investigation.ts` uses `startSpan(...)` for OpenTelemetry and emits structured attributes (element types, page, org count) for every query.
- **Timeout guard**: `withTimeout()` wrapper at `investigation.ts:186` applies a 30-second deadline to all backend gRPC calls.

### Concerns

- **[Medium] `any[]` in `investigation.ts`**: Line 1311 declares `const allResults: any[] = []`. This accumulates results from multiple element-type queries. A union type or generic should be used here to maintain type safety across the merge logic.
- **[Medium] Hardcoded English error messages in server code**: `investigation.ts:1262` ("Invalid time range: Start time cannot be after end time.") and `:1430` ("Failed to query MalOps. Please try again.") are user-visible messages returned from tRPC that contain hardcoded English. If the platform supports i18n in the UI, these strings should either be translated or returned as error codes for client-side localization.
- **[Low] `any` in `exclusions-list.tsx`**: Line uses `name: any` with comment "Using any to avoid complex nested path types". This should be replaced with a typed `Path<T>` generic from `react-hook-form` or a discriminated union.

---

## Multi-Tenancy Audit

### Methodology

- `grep -r "SELECT|INSERT|UPDATE|DELETE"` on the Rust source tree identified 20+ files containing SQL queries.
- Five files were read in full: `postgres.rs` (correlation-service), `event_repository.rs` (event-store), `repository.rs` (case-store), `queue_repository.rs` (notification-store), `sensor_group/repository.rs` (platform-store).
- The ratio of files mentioning `org_id` to total Rust source files is 349/923 = 38%, which is high given that many files are not query files (models, config, grpc layer, etc.).
- TypeScript routers were checked for org_id sourcing.

### Findings

**Compliant patterns observed:**

- `correlation-service/postgres.rs:173`: `WHERE org_id = $2` with parameterized `.bind(org_id)`.
- `case-store/repository.rs:85-96`: `WHERE org_id = ? AND id = ?` with parameterized MySql binding. Every `InvestigationRepository` method includes `org_id` as both a parameter and a bind variable.
- `notification-store/queue_repository.rs:62`: `WHERE organization_id = $1` parameterized.
- `platform-store/sensor_group/repository.rs`: `list_groups` takes `org_ids: Vec<i64>` and builds the WHERE clause through `sqlx::QueryBuilder` — parameterized.
- All TypeScript routers source `org_id` from `ctx.session.user.organizationId`.

**Non-compliant finding:**

- `event-store/src/repositories/event_repository.rs`: The method `get_max_event_timestamp` (line 189) builds:
  ```
  SELECT toUnixTimestamp(max(event_created_at)) FROM raw_events WHERE org_id IN ({org_ids_str})
  ```
  where `org_ids_str` is assembled via `id.to_string()` from a `&[u64]` slice. Since all values are `u64` numerics this is safe from injection, but it violates the parameterized-query invariant. The `build_optimized_query` method similarly injects `org_ids_str` directly into PREWHERE SQL strings (line 576). The ClickHouse crate's `.bind()` method should be used instead.

- `event-store/src/repositories/queries/single_event.rs:136`: `WHERE event_id = '{event_id}'` — single-quoted value injected via string replace. The `event_id` is converted from bytes to a `u128`, so only numeric characters can appear. Safe in practice but structurally a SQL injection pattern.

### Risk Assessment

**Medium.** The numeric-only nature of the interpolated values (`u64` org IDs and `u128` event IDs) prevents actual injection. However, the pattern violates the project's own security invariant and creates a maintenance risk: future developers may follow the same pattern with string-typed values, where injection would be possible.

---

## Security Observations

### Findings

- **Hardcoded default passwords in Rust configs**: Both `event-store/src/config.rs:59` and `correlation-service/src/config.rs:148` declare `default_value = "edrpassword"` for `CLICKHOUSE_PASSWORD`. While these are development defaults, they appear in committed source code and could be used in misconfigured deployments.
- **Hardcoded default credentials in Go config**: `xdr-worker/config/config.go:61` sets `s3_secret_key` default to `"admin"` and `s3_access_key` to `"admin"`. Same risk as above.
- **`sensor-auth` uses `secrecy` crate correctly**: Keys are wrapped in `SecretString`/`SecretBox`, preventing them from appearing in `Debug` output or logs. This is a positive security finding.
- **`vault-service/src/secure_memory.rs`**: Uses `mlock`/`munlock` to pin secrets in RAM, and `zeroize` on drop. The `unsafe` blocks are narrow, justified, and correctly scoped. The `VAULT_REQUIRE_MLOCK` env var provides a safety valve.
- **`integration-proxy` zeroizes credentials**: `tests/credential_zeroize_test.rs` and `src/auth.rs` confirm credential zeroization is tested.
- **`asset-store/unified_asset_repository.rs:173`**: Contains a guard: `"Unified asset filter field_name rejected (not allowlisted or unsafe). Skipping."` — indicating field names from user filters are validated against an allowlist before being interpolated. This is the correct approach.
- **`platform-store/sensor/query_builders/filter_builder.rs:247`**: Similar allowlist guard: `"qualify_field_name: unsafe identifier rejected"` — confirms consistent field-name sanitization across services.

### Hardcoded Secrets Scan

No plain-text API keys, passwords, or tokens were found hardcoded as literal values in production code. All instances of `password`/`secret` in source are:
1. Configuration field names that read from environment variables.
2. `default_value` strings in clap/viper configs (development defaults only — medium risk).
3. Field names in Go config structs (`S3SecretKey string`).
4. Test utilities (`test-utils/src/bin/send_test_malop_sensor_emails.rs` — acceptable).

---

## Test Coverage Observations

- **Rust**: 259 files contain `#[test]` or `#[tokio::test]` annotations. Only 17 files live under `tests/` directories (integration tests); the remainder are unit tests inline in source modules. `correlation-service` has one integration test file (`tests/correlation_integration_test.rs`). `event-store` has no `tests/` directory — all tests are inline.
- **`rstest`/`googletest`**: Only 21 files use these frameworks despite the coding standard requiring them. Most tests use plain `#[test]` with `assert_eq!`. This divergence from standard is widespread.
- **Go**: Both `clickhouse-ingester-v2` and `xdr-worker` have test files at the package level. Tests observed in `main_otel_test.go` are unit tests of specific functions, not table-driven. No integration test suite was found in the sampled Go services.
- **TypeScript**: 195 test files (`*.test.ts`, `*.test.tsx`) present. Router-level tests exist (`asset.test.ts`, `batch-action.event-id.test.ts`, `batch-action.validation.test.ts`). This is the strongest test coverage of the three stacks.
- **Gap**: No evidence of contract tests between gRPC services. Changes to a proto schema would not be caught by existing tests until runtime.

---

## Top 5 Recommendations

1. **Replace string-interpolated ClickHouse queries with parameterized bindings** (High, Security).
   - Files: `/rust/event-store/src/repositories/event_repository.rs` (lines 189-190, 576) and `/rust/event-store/src/repositories/queries/single_event.rs` (line 136).
   - Use `clickhouse::Client::query(sql).bind(value)` for all variable values. This eliminates the structural SQL injection pattern and enforces the project's own security invariant.

2. **Migrate `xdr-worker` from `logrus` to `log/slog`** (High, Standards).
   - File: `/golang/xdr-worker/main.go` and all Go files in `golang/xdr-worker/`.
   - Also update the env prefix from `XDR_WORKER` to `PHOENIX_` in `/golang/xdr-worker/config/config.go:74` and `/golang/xdr-worker-v2/config/config.go`.
   - This aligns the service with the Go logging standard and enables consistent structured log aggregation.

3. **Replace `anyhow` with `thiserror` in library crates** (High, Standards).
   - Files: `/rust/idm-service/src/extractor.rs`, `/rust/idm-service/src/cache.rs`, `/rust/idm-service/src/consumer.rs`, `/rust/idm-service/src/storage.rs`, `/rust/notification-service/src/service.rs`.
   - Define typed error enums with `#[derive(Error)]`. This preserves error variant information across crate boundaries and is required by the coding standard.

4. **Add `org_id` to `#[instrument]` span fields in all gRPC handlers** (Medium, Observability/Security).
   - Priority files: all `grpc.rs` files in `correlation-service`, `event-store`, `case-store`, `notification-store`.
   - Pattern: `#[tracing::instrument(skip(self, request), fields(org_id = ?request.get_ref().organization_id))]` — already demonstrated in `/rust/xdr-action-store/src/grpc.rs:128`.
   - Without org_id in spans, multi-tenant debugging requires joining log lines by timestamp — fragile in high-throughput environments.

5. **Remove hardcoded default passwords from committed configuration** (Medium, Security).
   - Files: `/rust/event-store/src/config.rs:59` (`edrpassword`), `/rust/correlation-service/src/config.rs:148` (`edrpassword`), `/golang/xdr-worker/config/config.go:61` (`admin`/`admin`).
   - Replace `default_value = "edrpassword"` with no default (make the field required), or document the dev-only default in a `.env.example` file that is not committed. Operators who forget to set `CLICKHOUSE_PASSWORD` should receive a clear startup error, not silently use a known-default password.
