# Phoenix Proto — Project Summary

## One-Line Description
Canonical Protobuf schema repository defining every wire contract (Kafka, gRPC, sensor binary frames) exchanged across the Phoenix EDR/XDR platform.

## Purpose
`phoenix-proto` is the single source of truth for all message contracts shared between the Phoenix Server (cloud microservices), the Phoenix Agent / Sunbird (endpoint sensor), and downstream analytics or integration pipelines. Both the Rust services and Go services on the server side, and the Rust agent on endpoints, consume code generated from this repo. There are no `.proto` files anywhere else in the platform — schema changes flow outward from this single repository.

## Tech Stack
- Protobuf 3 (proto3 syntax, `optional` used explicitly)
- buf CLI v2 (module `com.cybereason/cybereason/phoenix`, BSR-published)
- Generated Go (`buf.build/protocolbuffers/go` v1.31.0) + Go gRPC (`buf.build/grpc/go` v1.3.0) into `gen/go/`
- Rust generation handled per-crate via `prost-build` in each consumer's `build.rs` (not configured in this repo)
- Custom `cybereason.common.v1.Timestamp` instead of `google.protobuf.Timestamp` (prost compatibility)

## Scale
- Files: 236 proto files
- Messages: 1,607
- Enums: 284
- gRPC services: 61

## Package Structure
- `grpc/health/v1/` — Standard Kubernetes-compatible gRPC health checking
- `cybereason/common/v1/` — Primitives: Timestamp, Duration, Struct, Value, DynamicFilter, Pagination
- `cybereason/common/health/v1/` — Cybereason HealthService Ping RPC
- `cybereason/ecs/agent/v1/` — `CoreAgentInfo`, `AgentPolicy`, `AgentType`
- `cybereason/ecs/event/v1/` — `SingleEvent`, `EventCommon`, `EventSpecific`, `EventInfo`, `EventStatus`
- `cybereason/ecs/detect/v1/` — `Detection`, `Threat`, `Vulnerability`, rule/engine schemas
- `cybereason/ecs/host/v1/` — `Host`, `Process`, `File`, `User`, `Authentication`, `LogonSession`, `Registry`
- `cybereason/ecs/net/v1/` — `Network`, `Peer`, DNS/HTTP/TLS/Email
- `cybereason/ecs/os/windows/v1/` — Windows-specific telemetry (ETW, event logs, services)
- `cybereason/ecs/rpc/`, `services/`, `software/`, `infra/`, `observer/` — supporting ECS domains
- `cybereason/phoenix/agent/v1` + `agent/sensor/v1` — `FullAgentInfo`, `AgentMetrics`, sensor config/status
- `cybereason/phoenix/events/raw/v1/` — `EventBatch`, `OpaqueEventBatch`
- `cybereason/phoenix/commandcontrol/v1/` — All 31 command types (`CommandMessage`, request/response)
- `cybereason/phoenix/cep/v1/` — `ChainTrigger`, `StageEventBatch`
- `cybereason/phoenix/detection/`, `detection_store/` — Detection gateway, attack path, store API
- `cybereason/phoenix/platform_store/v1/` — `Sensor`, `Policy`, groups, tags, scripts
- `cybereason/phoenix/organization_store/v1/` — Multi-tenant org management
- `cybereason/phoenix/auth/v1` (deprecated) + `v2/` — Sensor enrollment
- `cybereason/phoenix/etl_service/v2/` — `VendorRawEnvelope` for XDR ingest
- `cybereason/phoenix/{vault,rulecontrol,sensor_gateway,sensor_action_store,asset_store,case_*,integration_*,xdr_*,mitre_service,...}` — Service-specific schemas

## Key Message Types
| Message | Purpose | Producer | Consumer | Transport |
|---------|---------|----------|----------|-----------|
| `ecs.event.v1.SingleEvent` | Top-level security event | Agent, XDR via ETL | Flink, ClickHouse, correlator | Kafka `raw-events` |
| `ecs.detect.v1.Detection` | Detection/alert with embedded events | Flink, EDR sensor, XDR, CEP | detection-store, correlator | Kafka `detections` |
| `ecs.agent.v1.CoreAgentInfo` | Sensor identity (per-batch + heartbeat) | Agent | Sensor Gateway, platform-store | Kafka `agents_v2` (in `EventCommon`) |
| `phoenix.events.raw.v1.EventBatch` | Sensor-to-gateway batch payload | Agent | Sensor Gateway | ppRPC HTTP/2 |
| `phoenix.commandcontrol.v1.CommandMessage` | Typed command (oneof of 31 variants) | command-service | Agent | Kafka `actions` (signed envelope) |
| `phoenix.commandcontrol.v1.CommandResponseMessage` | Sensor command response | Agent | sensor-action-store | Kafka `actions-response` |
| `phoenix.cep.v1.ChainTrigger` | Correlation chain completion | cep-chainmaker | detection-correlator | Kafka `triggers` |
| `phoenix.organization_store.v1.OrganizationChangeNotification` | Org mutation broadcast | organization-store | sensor-gateway, sensor-auth | Kafka `organization-changes` |
| `phoenix.rulecontrol.v1.PhoenixRuleControlEvent` | CEP rule hot-reload | rule-control-service | Flink, cep-service | Kafka `rule-control-events` |
| `phoenix.etl_service.v2.VendorRawEnvelope` | XDR vendor payload (billions/day) | xdr-worker-v2 | etl-service-v2 | Kafka `xdr-vendor-raw` |

## Kafka Topic to Message Mapping
| Topic | Message | Notes |
|-------|---------|-------|
| `raw-events` | `ecs.event.v1.SingleEvent` | One event per message after gateway unpacks `OpaqueEventBatch` |
| `detections` | `ecs.detect.v1.Detection` | Includes full triggering events inside `Detection.events` |
| `agents_v2` | `ecs.agent.v1.CoreAgentInfo` (in `FullAgentInfo`) | Agent registration + heartbeat |
| `actions` | `phoenix.commandcontrol.v1.CommandControlRequest` | Signed envelope wrapping `CommandMessage` |
| `actions-response` | `phoenix.commandcontrol.v1.CommandControlResponse` | Wraps `CommandResponseMessage` |
| `triggers` | `phoenix.cep.v1.ChainTrigger` | Multi-stage correlation completion |
| `organization-changes` | `phoenix.organization_store.v1.OrganizationChangeNotification` | CREATE/UPDATE/PATCH/DELETE |
| `rule-control-events` | `phoenix.rulecontrol.v1.PhoenixRuleControlEvent` | Rule YAML hot-reload |
| `xdr-vendor-raw` (internal) | `phoenix.etl_service.v2.VendorRawEnvelope` | Inline payload or S3 URL for oversized |
| `cep-stage-events` (internal) | `phoenix.cep.v1.StageEventBatch` | Keyed by `rule_id` for deterministic routing |

## Who Uses This Repo
| Component | How | Location |
|-----------|-----|----------|
| Phoenix Server | Git submodule, generated Go in `modules/proto/gen/go/`, generated Rust in `rust/pbgen/` | `projects/Phoenix/modules/proto/` |
| Phoenix Agent | `prost-build` at compile time against canonical schemas | `projects/phoenix-agent/crates/phoenix-protobuf/` |
| Flink CEP jobs | Generated Java/Scala stubs | consumes `SingleEvent`, publishes `ChainTrigger`/`StageEventBatch` |
| Sensor Gateway | Generated Rust, decodes `OpaqueEventBatch`, repacks `OpaqueSingleEvent` | Phoenix Server Rust services |

## Code Generation
```bash
buf generate   # regenerate all language bindings
buf lint       # check style compliance (STANDARD ruleset)
buf breaking --against '.git#branch=main'  # check backward compat
```

## Schema Quality Snapshot
| Metric | Value |
|--------|-------|
| Overall score | 7/10 |
| Top strength | All 284 enums have `_UNSPECIFIED` zero value; consistent custom `Timestamp` usage |
| Top concern | Submodule drift — Phoenix server pinned 1 commit behind (`cd67b45e` vs HEAD `446c777f`) |
| Breaking change detection | `FILE` mode configured but missing `against` baseline reference |
| Field documentation coverage | 37.4% of 4,445 fields have comments; 234 of 236 files lack package-level docs |
| Multi-tenancy enforcement | `org_id`/`organization_id` mandatory on every tenant-scoped message |

## Top 3 Issues
1. **Submodule drift** — Phoenix server submodule pinned 1 commit behind HEAD; missing `EventSpecific.rpc` (field 37), four new `EVENT_ACTION_RPC_*` enum values, and the `px_rpc.proto` / `px_msrpc.proto` files. Drift is additive only (no wire breakage), but RPC telemetry will be silently dropped on the server until the pin is bumped.
2. **No `buf breaking` baseline** — `buf.yaml` has no `against` reference under `breaking:`; the check is a silent no-op unless CI passes `--against` explicitly. Critical gap for a wire protocol shared with deployed endpoint sensors.
3. **Raw epoch integers** — 4+ locations use raw `int64` / `uint64 *_ns` timestamps instead of `cybereason.common.v1.Timestamp` (across `platform_store`, `sensor_gateway`, `commandcontrol`, `xdr_integration_store`, `integration_manager`, `integration_proxy`, `vault/v2`). `int64` variants do not document their epoch unit.

## How to Work with This Repo
```bash
cd projects/phoenix-proto
mise install    # install buf and tools
buf lint        # lint all protos
buf generate    # generate bindings (Go + Go gRPC into gen/go/)
buf breaking --against '.git#branch=main'  # backward-compat check
```

## Key Files to Know
- `proto/cybereason/ecs/event/v1/px_event.proto` — `SingleEvent`, `EventCommon`, `EventSpecific`, `EventInfo`, `EventAction` enum (150+ values). The most heavily commented file in the repo.
- `proto/cybereason/ecs/detect/v1/px_detection.proto` — `Detection` schema published to Kafka `detections`.
- `proto/cybereason/phoenix/commandcontrol/v1/px_command_control.proto` — `CommandMessage` oneof of 31 command types and signed envelope/response wrapper.
- `proto/cybereason/common/v1/px_notation.proto` — Custom `Timestamp`, `Duration`, `Struct`, `Value` primitives used across the entire codebase.
- `buf.yaml` / `buf.gen.yaml` — Module config, lint exceptions, breaking-change strategy, code-gen targets.
