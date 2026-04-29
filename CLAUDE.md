# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Review repository for the **Phoenix EDR/XDR Platform** — a multi-tenant endpoint detection and response platform. The actual codebase lives under `projects/Phoenix/`.

## Architecture

Phoenix is a microservices platform:

- **Rust** (`rust/`): 16+ backend services — event processing, sensor management, correlation, detection, storage, integration, vault, etc.
- **Go** (`golang/`): High-throughput data processing — ClickHouse ingestion, XDR workers, schema generation
- **TypeScript** (`phoenix-portal/`): T3 Stack — Next.js 16, tRPC 11, Drizzle ORM, NextAuth v5
- **Apache Flink**: Real-time stream processing (CEP rules via `cep-rules/`)
- **ClickHouse**: Analytics (`edr_xdr` database, partitioned by `org_id`)
- **Redpanda** (Kafka-compatible): Message broker between services
- **Protobuf** (`modules/proto/`): Shared schemas — `SingleEvent`, `Detection`, `CoreAgentInfo`; generated into `rust/pbgen/` and `golang/pbgen/`

## Development Environment

All Docker operations go through `local/dev0`:

```bash
cd projects/Phoenix/local/dev0
./start.sh            # start dev environment
./start.sh --ci       # start with full CI build
./kill.sh             # stop all services
./kill.sh --full      # clean rebuild (removes volumes)
docker compose logs -f [service-name]
```

Tool installation: `mise install` (see `mise.toml`)

## Build & Test

```bash
# Rust
cargo test --workspace                        # all tests
cargo test -p [crate-name] [test_name]        # single test

# Go
go test ./...

# TypeScript
bun test
bun run test:e2e    # Playwright E2E (from phoenix-portal/)

# Protobuf regeneration
cd modules/proto && buf generate

# Linting
cargo clippy          # Rust
gofmt / go vet        # Go
bun run lint          # TypeScript
buf lint              # Protobuf
```

## Critical: Multi-Tenancy

**Every database query MUST filter by `org_id`.** Missing this is a security vulnerability.

- Derive `org_id` from authenticated session only — never trust user-provided IDs
- Use parameterized queries always — never string concatenation
- ClickHouse tables are partitioned by `org_id`
- Verify `org_id` at every gRPC/REST request boundary

## Rust Standards

- Applications: `color-eyre`/`eyre` for errors; Libraries: `thiserror`
- Logging: `tracing` crate with `#[instrument]`, always include `org_id` in fields
- Config: `clap::Parser` with `env = "..."` attributes
- gRPC: `tonic` + `pbgen` generated code
- DB: `sqlx` (PostgreSQL), `clickhouse` crate (ClickHouse)
- Health: `/healthz` HTTP + `grpc.health.v1.Health` gRPC
- Tests: `rstest` + `googletest`
- Lints: `unsafe_code = "forbid"`, `clippy::pedantic = "warn"`

## Go Standards

- Logging: `log/slog` with structured key-value fields
- Config: `github.com/spf13/viper` with `PHOENIX_` env prefix
- Errors: wrap with `fmt.Errorf("context: %w", err)`
- Tests: table-driven
- Patterns: backoff/retry for transient failures, circuit breaker for external services, dual-buffer for high throughput

## TypeScript Standards

- Strict TypeScript: `"strict": true`, `"noUncheckedIndexedAccess": true` — never use `any`
- Validation: `zod` schemas
- Server state: React Query via tRPC; Client state: Zustand
- Tailwind: static class names only — no dynamic class construction (use `style` prop instead)
- i18n: `next-intl`, translations in `en.json`/`ja.json`/`es.json`/`de.json`, glossary at `phoenix-portal/glossary/translation-glossary.json`
- Server components by default; add `"use client"` only when needed

## gRPC Service Ports

| Port  | Service               |
|-------|-----------------------|
| 50051 | sensor-auth           |
| 50052 | platform-store        |
| 50053 | event-store           |
| 50054 | sensor-action-store   |
| 50055 | command-service       |
| 50056 | asset-store           |
| 50057 | mitre-tagging-service |
| 50058 | rule-control-service  |
| 50059 | organization-store    |

## Kafka Topics

| Topic            | Schema            | Purpose                        |
|------------------|-------------------|--------------------------------|
| raw-events       | SingleEvent       | Security events from sensors   |
| detections       | Detection         | Generated alerts               |
| agents_v2        | CoreAgentInfo     | Agent registration             |
| actions          | Action            | Outbound commands to sensors   |
| actions-response | ActionResponse    | Command execution responses    |
| triggers         | Trigger           | Correlation trigger events     |

## Dev Service URLs

| Service         | URL                    |
|-----------------|------------------------|
| Portal (dev)    | http://localhost:3002  |
| Portal (prod)   | http://localhost:3003  |
| Flink Dashboard | http://localhost:8081  |
| Redpanda        | http://localhost:8080  |
| ClickHouse      | http://localhost:8123  |
| PostgreSQL      | localhost:5432         |

## AI Rules Management

Rule source of truth: `.rulesync/` directory. Do NOT edit `.cursor/rules/`, `.claude/rules/`, or other agent-specific files directly. Sync with `mise agents:rules:sync`.

## Commit Format

```
type(scope): description [JIRA-ticket]
```
Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`. PRs require JIRA ticket (ENG-xxxxx).
