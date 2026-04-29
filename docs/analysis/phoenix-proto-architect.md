# Phoenix Proto — Architectural Overview

## What Is This?

`phoenix-proto` is the canonical Protobuf schema repository for the Phoenix EDR/XDR platform. It is the single source of truth for every message contract exchanged across the three major platform components: the Phoenix Server (cloud microservices), the Phoenix Agent / Sunbird (endpoint sensor), and any downstream analytics or integration pipelines. All wire formats — Kafka payloads, gRPC service interfaces, sensor-to-gateway binary frames — are defined here.

The module name is `com.cybereason/cybereason/phoenix` (buf.yaml v2 format). Proto sources live under `proto/` and are organized into two top-level namespaces: `cybereason.ecs.*` (ECS-derived data model) and `cybereason.phoenix.*` (platform-specific services and messages).

---

## Role in the Ecosystem

| Component | Relationship to phoenix-proto |
|-----------|-------------------------------|
| **Phoenix Server** (`projects/Phoenix/`) | Consumes generated Go code via `buf generate`. Go stubs land in `gen/go/`. Rust services use `prost`-generated code (via `modules/proto/` which references this repo's content). |
| **Phoenix Agent / Sunbird** (`projects/phoenix-agent/`) | Serializes `EventBatch` / `FullAgentInfo` / `CommandMessage` / `CommandResponseMessage` on the wire. Uses prost-generated Rust structs. |
| **Flink CEP jobs** | Consume `SingleEvent` from Redpanda `raw-events` topic; publish `ChainTrigger` and `StageEventBatch`. |
| **ETL service** | Consumes `VendorRawEnvelope` from XDR workers; outputs `SingleEvent`. |
| **Sensor Gateway** | Decodes `OpaqueEventBatch`, re-packs as `OpaqueSingleEvent` per-event onto `raw-events`; serves `PolicyResponse` to sensors; forwards `CommandControlRequest` / `CommandControlResponse`. |

**Code generation targets (buf.gen.yaml):**
- Go (protocolbuffers/go v1.31.0) — output to `gen/go/`, paths source-relative.
- Go gRPC stubs (grpc/go v1.3.0) — same output directory.
- No Rust or TypeScript generation is configured in this repo; Rust code generation is handled separately inside each Rust crate (build.rs with prost-build).

**Cross-component sync status:** Neither `projects/Phoenix/` nor `projects/phoenix-agent/` contains any `.proto` files. Both components pull the generated artifacts directly from this repository (or a CI-published package). There is no diverged local copy. The server's `modules/proto/` directory contains only generated Go/Rust output, not proto sources. This is the single canonical source.

---

## Proto Package Structure

```
proto/
├── grpc/health/v1/                    # Standard gRPC Health Checking Protocol (Kubernetes-compatible)
│   └── health.proto
└── cybereason/
    ├── common/
    │   ├── health/v1/                 # Cybereason HealthService (Ping RPC)
    │   │   ├── px_health_data.proto
    │   │   └── px_health_svc.proto
    │   └── v1/                        # Primitive types used across all packages
    │       ├── px_notation.proto      # Timestamp, Duration, Struct, Value
    │       ├── px_filter.proto        # DynamicFilter for query APIs
    │       ├── px_content_preview.proto
    │       └── px_pagination.proto    # PaginationRequest/Response
    └── ecs/                           # ECS (Elastic Common Schema) data model
        ├── agent/v1/px_agent.proto    # CoreAgentInfo, AgentPolicy, AgentType enum
        ├── detect/
        │   ├── v1/px_detection.proto  # Detection (Kafka: detections topic)
        │   ├── v1/px_detection_action.proto
        │   ├── v1/px_detection_classification.proto
        │   ├── v1/px_rule.proto
        │   ├── v1/px_threat.proto
        │   ├── v1/px_threat_attack.proto
        │   ├── v1/px_vulnerability.proto
        │   └── engine/v1/             # Engine-specific detection schemas
        │       ├── px_behavioral_execution_protection_engine.proto
        │       ├── px_bitdefender_engine.proto
        │       ├── px_document_engine.proto
        │       ├── px_exploit_engine.proto
        │       ├── px_script_engine.proto
        │       ├── px_variant_file_protection_engine.proto
        │       └── px_variant_payload_protection_engine.proto
        ├── event/v1/
        │   ├── px_event.proto         # SingleEvent, EventBatch root schemas
        │   └── px_event_status.proto  # EventStatus, errno catalog
        ├── host/v1/                   # Host entity schemas
        │   ├── px_host.proto          # Host, CoreHostInfo, OsInfo, HostMetrics
        │   ├── px_process.proto       # Process, Thread, IntegrityLevel
        │   ├── px_file.proto          # File, FileType, FileAttribute, FileProvenance
        │   ├── px_file_meta.proto
        │   ├── px_executable.proto
        │   ├── px_auth.proto          # Authentication
        │   ├── px_user.proto          # User
        │   ├── px_logon_session.proto # LogonSession
        │   ├── px_network_interface.proto
        │   └── px_registry.proto      # Registry (Windows)
        ├── infra/v1/
        │   ├── px_cloud.proto         # Cloud
        │   └── px_cloud_types.proto
        ├── net/v1/                    # Network entity schemas
        │   ├── px_network.proto       # Network, Peer
        │   ├── px_dns.proto
        │   ├── px_http.proto
        │   ├── px_email.proto
        │   ├── px_tls.proto
        │   ├── px_interface.proto
        │   ├── px_network_protocol.proto
        │   ├── px_as.proto
        │   └── px_vlan.proto
        ├── observer/v1/px_observer.proto
        ├── os/windows/v1/             # Windows-specific telemetry
        │   ├── px_cross_process_operation.proto
        │   ├── px_windows_event_log.proto
        │   ├── px_windows_info.proto
        │   └── px_windows_service.proto
        ├── platform/v1/px_platform_integration.proto
        ├── rpc/
        │   ├── v1/px_rpc.proto
        │   └── msrpc/v1/px_msrpc.proto
        ├── services/
        │   ├── v1/px_scheduled_task.proto
        │   ├── v1/px_service_instance.proto
        │   ├── v1/px_service_type.proto
        │   └── ldap/v1/
        │       ├── px_ldap.proto
        │       └── px_ldap_query.proto
        ├── software/v1/
        │   ├── px_software_product.proto
        │   └── px_swid.proto
        └── v1/                        # Shared ECS primitives
            ├── px_code_signature.proto
            ├── px_geo.proto
            ├── px_hash.proto          # (deprecated, see v2)
            ├── px_mime_type.proto
            ├── px_url.proto
            ├── px_x500.proto
            └── px_x509.proto
        └── v2/
            └── px_hash_v2.proto       # Hash (bytes-based, replaces v1)

    └── phoenix/                       # Platform-specific services
        ├── agent/
        │   ├── sensor/v1/             # Sensor configuration & status
        │   │   ├── px_sensor_antimalware.proto
        │   │   ├── px_sensor_collection.proto
        │   │   ├── px_sensor_config.proto
        │   │   ├── px_sensor_host_config.proto
        │   │   └── px_sensor_status.proto
        │   └── v1/
        │       ├── px_agent_metrics.proto   # AgentMetrics
        │       └── px_full_agent_info.proto  # FullAgentInfo (ppRPC)
        ├── asset_store/v1/             # Asset management
        ├── audit_log_store/v1/         # Audit logging
        ├── auth/
        │   ├── v1/                     # Legacy sensor onboarding (deprecated)
        │   └── v2/                     # Current challenge-based enrollment
        ├── case_service/v1/            # Case/investigation service
        ├── case_store/v1/              # Case/investigation storage
        ├── cep/v1/                     # CEP correlation (ChainTrigger, StageEventBatch)
        ├── cep_xsf_lookup/v1/
        ├── command_service/v1/         # Command orchestration service
        ├── commandcontrol/v1/          # All command types (31 distinct commands)
        ├── detection/v1/               # Detection gateway, attack path
        ├── detection_store/v1/         # Detection storage and query
        ├── etl_service/
        │   ├── v1/                     # Legacy ETL
        │   └── v2/                     # Current VendorRawEnvelope format
        ├── event_store/v1/             # ClickHouse event query service
        ├── events/raw/v1/              # EventBatch / OpaqueEventBatch
        ├── integration_manager/v1, v2/
        ├── integration_proxy/v1, v2/
        ├── ioc_filter/v1/
        ├── legacysensorpolicyconfig/v1/ # Legacy ActiveConsole policy config
        ├── malop_sensor_notification/v1/
        ├── mitre_service/v1/
        ├── notification_dispatcher/v1/
        ├── notification_store/v1/
        ├── organization_store/v1/       # Multi-tenant org management
        ├── platform_store/v1/           # Sensor, Policy, Groups, Tags, Scripts
        ├── pusher_auth/v1/
        ├── rule_converter/v1/
        ├── rulecontrol/v1/              # Detection rule lifecycle management
        ├── sensor_action_store/v1/      # Batch command tracking
        ├── sensor_gateway/v1/           # Sensor policy response, WNS
        ├── threat_intel/v1/
        ├── threat_intel_store/v1/
        ├── vault/v1, v2/                # Encryption/decryption service
        ├── xdr_action_store/v1/
        ├── xdr_command_service/v1/
        ├── xdr_integration_store/v1/
        └── xdr_worker/v2/
```

---

## Message Type Catalog

### Core Primitive Types (`cybereason.common.v1`)

#### `common.v1.Timestamp`
**File:** `proto/cybereason/common/v1/px_notation.proto`
**Purpose:** Custom timestamp replacing `google.protobuf.Timestamp` (prost compatibility).

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | seconds | int64 | UTC seconds since Unix epoch |
| 2 | nanos | int32 | Sub-second nanoseconds (0..999,999,999) |

#### `common.v1.Duration`
**File:** `proto/cybereason/common/v1/px_notation.proto`
**Purpose:** Custom duration replacing `google.protobuf.Duration`.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | seconds | int64 | Signed seconds (-315,576,000,000 to +315,576,000,000) |
| 2 | nanos | int32 | Signed nanoseconds |

---

### ECS Event Layer (`cybereason.ecs.event.v1`)

#### `ecs.event.v1.SingleEvent`
**File:** `proto/cybereason/ecs/event/v1/px_event.proto`
**Purpose:** Top-level event message. One event per Redpanda `raw-events` message after sensor gateway unpacks batches.
**Produced by:** Phoenix Agent (Sunbird / ActiveConsole), XDR integrations (via ETL)
**Consumed by:** Flink (rule evaluation), ClickHouse ingestion, detection-correlator
**Transport:** Kafka topic `raw-events`

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | common | EventCommon (optional) | Mandatory. Shared batch-level fields (agent, host, org) |
| 2 | event | EventSpecific (optional) | Mandatory. Event-specific fields |

#### `ecs.event.v1.OpaqueSingleEvent`
**File:** `proto/cybereason/ecs/event/v1/px_event.proto`
**Purpose:** Sensor gateway optimization — unpack batch without parsing individual events.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | common | bytes (optional) | Serialized EventCommon |
| 2 | event | bytes (optional) | Serialized EventSpecific |

#### `ecs.event.v1.EventCommon`
**File:** `proto/cybereason/ecs/event/v1/px_event.proto`
**Purpose:** Fields shared by all events in a batch — sent once per batch, merged onto every event.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | ecs_version | EcsVersion | ECS version (always ECS_VERSION_9_0_0) |
| 2 | event_source | EventSource | EDR or XDR |
| 3 | agent | CoreAgentInfo (optional) | Mandatory for EDR. Agent identity and policy |
| 4 | org_id | uint64 | Tenant ID. Multi-tenancy partition key |
| 5 | retention_days | uint32 (optional) | ClickHouse retention (default 30) |
| 7 | host | CoreHostInfo (optional) | Mandatory for EDR. Core host info |
| 8 | integration | Integration (optional) | EDR/XDR integration context |

#### `ecs.event.v1.EventSpecific`
**File:** `proto/cybereason/ecs/event/v1/px_event.proto`
**Purpose:** All fields specific to a single event. Approximately 37 optional sub-messages covering every ECS domain.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | event | EventInfo (optional) | Mandatory. Event ID, kind, action, outcome, timestamps, severity |
| 2 | tags | repeated string | Enrichment tags |
| 3 | message | string (optional) | Human-readable event description |
| 4 | labels | map<string,string> | Free-form key-value pairs |
| 6 | observer | Observer (optional) | Observer metadata |
| 7 | host | Host (optional) | Deprecated. Use EventCommon.host |
| 8 | user | User (optional) | User who triggered the event |
| 9 | process | Process (optional) | Primary process |
| 10 | file | File (optional) | Primary file |
| 11 | registry | Registry (optional) | Windows registry (Windows only) |
| 12 | cloud | Cloud (optional) | Cloud platform metadata |
| 13 | authentication | Authentication (optional) | Auth event data |
| 14 | network | Network (optional) | Network connection metadata |
| 15 | source | Peer (optional) | Network source endpoint |
| 16 | destination | Peer (optional) | Network destination endpoint |
| 17 | dns | Dns (optional) | DNS query/response data |
| 18 | tls | Tls (optional) | TLS handshake data |
| 19 | http | Http (optional) | HTTP request/response data |
| 20 | email | Email (optional) | Email metadata |
| 21 | url | Url (optional) | URL data |
| 22 | user_agent | UserAgent (optional) | User-agent string |
| 23 | session | LogonSession (optional) | Logon session |
| 24 | ingestion_source_id | string (optional) | XDR ingestion source identifier |
| 25 | windows | WindowsInfo (optional) | Windows-specific telemetry |
| 26 | target_process | Process (optional) | Target process in cross-process events |
| 28 | target_user | User (optional) | Target user (e.g., for user modification events) |
| 29 | target_session | LogonSession (optional) | Target logon session |
| 30 | related | Related (optional) | Related entities not captured by standard fields |
| 31 | ldap | Ldap (optional) | LDAP query data |
| 32 | target_file | File (optional) | Target file in rename/copy events |
| 33 | process_injection | ProcessInjection (optional) | Injection classification |
| 34 | cross_process_operation | CrossProcessOperation (optional) | Windows ETW cross-process telemetry |
| 35 | service | ServiceInstance (optional) | Service event data |
| 36 | scheduled_task | ScheduledTask (optional) | Scheduled task data |
| 37 | rpc | Rpc (optional) | RPC call data |

#### `ecs.event.v1.EventInfo`
**File:** `proto/cybereason/ecs/event/v1/px_event.proto`
**Purpose:** Core event classification and metadata. Maps to ECS `event.*` fields.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | id | bytes (optional) | ULID (16 bytes, Crockford Base32). Mandatory |
| 2 | kind | EventKind (optional) | Always EVENT_KIND_EVENT currently |
| 3 | created_at | Timestamp (optional) | When event occurred |
| 4 | category | EventCategory (optional) | Deprecated. ECS category |
| 5 | type | repeated EventType | Deprecated |
| 6 | action | EventAction (optional) | Primary event discriminator (150+ values) |
| 7 | outcome | EventOutcome (optional) | SUCCESS / FAILURE / BLOCKED / QUARANTINED |
| 8 | reason | string (optional) | Human-readable reason |
| 9 | start | Timestamp (optional) | Event start time |
| 10 | end | Timestamp (optional) | Event end time |
| 11 | duration | Duration (optional) | Event duration |
| 12 | ingested | Timestamp (optional) | Set by ingestion pipeline only |
| 13 | risk_score | float (optional) | Source-specific risk score |
| 14 | risk_score_norm | float (optional) | Normalized risk score (0-100) |
| 15 | severity | EventSeverityLevel (optional) | LOW/MEDIUM/HIGH/CRITICAL |
| 16 | confidence | DetectionConfidence (optional) | LOW/MEDIUM/HIGH |
| 17 | dataset | string (optional) | Event dataset identifier |
| 18 | module | string (optional) | Module that generated the event |
| 20 | sequence | uint64 (optional) | Sequence number |
| 21 | original | string (optional) | Raw original log record |
| 23 | code | int64 (optional) | Numeric event code (e.g., Windows Event ID) |
| 24 | status | EventStatus (optional) | Status code with cause chaining |

#### `ecs.event.v1.EventStatus`
**File:** `proto/cybereason/ecs/event/v1/px_event_status.proto`
**Purpose:** Structured status code supporting chaining and Win32/NTSTATUS/HRESULT/errno kinds.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | kind | StatusKind (optional) | SUCCESS / FAILURE / NEUTRAL |
| 2 | code_kind | StatusCodeKind (optional) | WIN32_ERROR / NTSTATUS / HRESULT / ERRNO |
| 3 | code | uint64 (oneof) | Unsigned status code |
| 4 | code_signed | sint64 (oneof) | Signed status code (NTSTATUS, HRESULT) |
| 5 | sub_code | uint64 (oneof) | Sub-code |
| 6 | sub_code_signed | sint64 (oneof) | Signed sub-code |
| 7 | reason | string (optional) | Human-readable description |
| 8 | cause | EventStatus (optional) | Direct cause (inner exception) |
| 9 | sub_failures | repeated EventStatus | Sub-failures (validation errors) |
| 10 | errno_value | Errno (optional) | ABI-independent errno name |

#### `ecs.event.v1.Related`
**File:** `proto/cybereason/ecs/event/v1/px_event.proto`
**Purpose:** Related entities not captured by standard event fields.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | hash | repeated Hash | Related hashes |
| 2 | hosts | repeated CoreHostInfo | Related hosts |
| 3 | ip | repeated bytes | Related IPs (4 or 16 bytes) |
| 4 | user | repeated User | Related users |
| 5 | files | repeated File | Related files |
| 6 | processes | repeated Process | Related processes |

#### `ecs.event.v1.ProcessInjection`
**File:** `proto/cybereason/ecs/event/v1/px_event.proto`
**Purpose:** Injection type classification for process injection events.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | type | ProcessInjectionType | Injection method (LOAD_LIBRARY, SHELLCODE_COBALT, etc.) |

---

### Agent Information (`cybereason.ecs.agent.v1`)

#### `ecs.agent.v1.CoreAgentInfo`
**File:** `proto/cybereason/ecs/agent/v1/px_agent.proto`
**Purpose:** Core sensor identity sent with every event batch and on the `agents_v2` Kafka topic.
**Produced by:** Phoenix Agent
**Consumed by:** Sensor Gateway, platform-store, event ingestion
**Transport:** Kafka `raw-events` (embedded in EventCommon), Kafka `agents_v2`

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | id | bytes | 128-bit sensor ID (little endian ULID) |
| 2 | type | AgentType (optional) | ACTIVE_CONSOLE / SUNBIRD / ENDPOINT / etc. |
| 3 | version | string (optional) | Agent software version |
| 4 | policy | AgentPolicy (optional) | Currently assigned policy |
| 5 | group_id | int64 (optional) | Sensor group ID |
| 6 | hostname | string (optional) | Agent-reported hostname |
| 7 | ephemeral_id | bytes (optional) | ULID reset on restart (session tracking) |

#### `ecs.agent.v1.AgentPolicy`
**File:** `proto/cybereason/ecs/agent/v1/px_agent.proto`
**Purpose:** Policy assignment for an agent.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | id | uint64 | Policy ID (mandatory for storage) |
| 2 | name | string (optional) | Policy name (read-only) |
| 3 | version | string (optional) | Policy version string (read-only) |

---

### Host Information (`cybereason.ecs.host.v1`)

#### `ecs.host.v1.Host`
**File:** `proto/cybereason/ecs/host/v1/px_host.proto`
**Purpose:** Full host information — sent on agent start or when host data changes.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | id | string (optional) | Unique host identifier |
| 2 | name | string (optional) | Deprecated. Use fqdn |
| 3 | hostname | string (optional) | Local unqualified hostname |
| 4 | ip | repeated string | All IP addresses (deprecated) |
| 5 | mac | repeated string | MAC addresses (deprecated) |
| 6 | os | OsInfo (optional) | OS information |
| 7 | architecture | CpuArchitecture (optional) | CPU architecture |
| 8 | domain | string (optional) | Windows domain |
| 9 | type | HostType (optional) | DESKTOP / SERVER / LAPTOP / VM / etc. |
| 10 | containerized | bool (optional) | Running in container |
| 11 | metrics | HostMetrics (optional) | CPU/memory usage |
| 12 | ip_external | string (optional) | Deprecated. External IP |
| 13 | fqdn | string (optional) | Fully qualified domain name |
| 14 | boot_id | string (optional) | Boot-unique ID |
| 15 | device_model | string (optional) | Hardware model string |
| 16 | installer_architecture | string (optional) | Computed: "x64" / "arm64" / "x86" |
| 17 | nics | repeated NetworkInterface | Network interfaces |

#### `ecs.host.v1.CoreHostInfo`
**File:** `proto/cybereason/ecs/host/v1/px_host.proto`
**Purpose:** Minimal host info sent with every event (embedded in EventCommon).

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | hostname | string (optional) | Unqualified hostname |
| 2 | fqdn | string (optional) | FQDN |
| 3 | type | HostType | Device type |
| 4 | os_type | OsType | OS family (WINDOWS / LINUX / DARWIN / etc.) |
| 5 | os_family | OsFamily | Distro family |
| 6 | os_platform | OsPlatform | Specific distro |
| 7 | cpu_architecture | CpuArchitecture | CPU arch |
| 8 | containerized | bool (optional) | Container flag |

#### `ecs.host.v1.Process`
**File:** `proto/cybereason/ecs/host/v1/px_process.proto`
**Purpose:** OS process telemetry. Recursive (parent, session_leader, entry_leader chains).

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | entity_id | bytes (optional) | Unique process entity ID |
| 2 | name | string (optional) | Mandatory. Process name |
| 3 | pid | int64 (optional) | Mandatory. Process ID |
| 4 | vpid | int64 (optional) | Virtual PID (containers) |
| 5 | exit_code | int64 (optional) | Exit code |
| 6 | title | string (optional) | Process title |
| 7 | user | User (optional) | Owning user |
| 8 | start | Timestamp (optional) | Start time |
| 9 | end | Timestamp (optional) | End time |
| 11 | command_line | string (optional) | Full command line |
| 12 | executable | string (optional) | Deprecated. Use image_file.path |
| 13 | args | repeated string | Process arguments |
| 14 | image_file | File (optional) | Process image file details |
| 15 | working_directory | string (optional) | Working directory |
| 16 | env_vars | map<string,string> | Environment variables (selective) |
| 17 | interactive | bool (optional) | Is interactive |
| 18 | thread | Thread (optional) | Thread details |
| 23 | parent | Process (optional) | Parent process (recursive) |
| 24 | session_leader | Process (optional) | Unix session leader |
| 25 | entry_leader | Process (optional) | Unix entry point process |
| 31 | integrity_level | IntegrityLevel (optional) | Windows integrity level |
| 32 | session | LogonSession (optional) | Logon session |
| 33 | changed | Process (optional) | Changed attributes (for modification events) |

#### `ecs.host.v1.File`
**File:** `proto/cybereason/ecs/host/v1/px_file.proto`
**Purpose:** File system object telemetry covering all platforms.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | type | FileType | FILE / DIR / SYMLINK / SOCKET / NAMED_PIPE / etc. |
| 2 | name | string (optional) | File name (without directory) |
| 3 | path | string (optional) | Full path including drive letter |
| 4 | directory | string (optional) | Deprecated. Use path |
| 5 | extension | string (optional) | Deprecated. Use path |
| 6 | mime_type | MimeType (optional) | IANA MIME type |
| 7 | size | int64 (optional) | File size in bytes |
| 8 | inode | int64 (optional) | Unix inode |
| 9 | device | string (optional) | Device source |
| 10 | drive | string (optional) | Deprecated. Windows drive letter |
| 11 | fork_name | string (optional) | macOS fork name |
| 12 | target_path | string (optional) | Symlink/hardlink target |
| 13 | atime | Timestamp (optional) | Last access time |
| 14 | created | Timestamp (optional) | Creation time |
| 15 | ctime | Timestamp (optional) | Attribute change time |
| 16 | mtime | Timestamp (optional) | Content modification time |
| 17 | attributes | repeated FileAttribute | Extended attributes |
| 18 | mode | int32 (optional) | Unix mode bits (octal, no type bits) |
| 19 | gid | int64 (optional) | Unix GID |
| 20 | uid | int64 (optional) | Unix UID |
| 21 | owner | string (optional) | Owner username |
| 22 | group | string (optional) | Group name |
| 23 | hash | Hash (optional) | Deprecated. Use hash_bytes |
| 24 | code_signature | CodeSignature (optional) | Code signature details |
| 25 | pe | Pe (oneof) | Windows PE header info |
| 26 | meta | FileMetadata (optional) | File metadata |
| 27 | provenance | FileProvenance (optional) | Download origin metadata |
| 28 | reference_id | uint32 (optional) | Intra-event file reference ID |
| 29 | hash_bytes | Hash (optional) | Current hash representation |
| 30 | software_product | SoftwareProduct (optional) | Detected software product |

#### `ecs.host.v1.HostMetrics`
**File:** `proto/cybereason/ecs/host/v1/px_host.proto`
**Purpose:** CPU/memory metrics sent in every event batch.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | uptime | Duration (optional) | System uptime |
| 2 | cpu_usage_norm_ppm | uint64 (optional) | Normalized CPU usage in ppm (0=0%, 1,000,000=100%) |
| 3 | memory_actual_used_bytes | uint64 (optional) | Memory used in bytes |
| 4 | memory_actual_used_ppm | uint64 (optional) | Memory usage in ppm |

---

### Event Batch (`cybereason.phoenix.events.raw.v1`)

#### `phoenix.events.raw.v1.EventBatch`
**File:** `proto/cybereason/phoenix/events/raw/v1/px_event_batch.proto`
**Purpose:** Sensor-to-gateway payload. Agent sends batches; gateway unpacks to individual `SingleEvent` messages.
**Produced by:** Phoenix Agent (Sunbird / ActiveConsole)
**Consumed by:** Sensor Gateway
**Transport:** ppRPC (agent to gateway HTTP/2)

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | sent_at | Timestamp (optional) | Batch send timestamp |
| 2 | unique_id | bytes (optional) | Batch ULID |
| 3 | sequence_id | uint64 (optional) | Monotonic batch counter (resets on restart) |
| 11 | common | EventCommon (optional) | Shared event fields for the batch |
| 12 | events | repeated EventSpecific | Individual event payloads |

#### `phoenix.events.raw.v1.OpaqueEventBatch`
**File:** `proto/cybereason/phoenix/events/raw/v1/px_event_batch.proto`
**Purpose:** Gateway-optimized format — parses batch header without decoding individual events.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | sent_at | Timestamp (optional) | Batch send timestamp |
| 2 | unique_id | bytes (optional) | Batch ULID |
| 3 | sequence_id | uint64 (optional) | Monotonic counter |
| 11 | common | bytes (optional) | Serialized EventCommon |
| 12 | events | repeated bytes | Serialized EventSpecific blobs |

---

### Detection Layer (`cybereason.ecs.detect.v1`)

#### `ecs.detect.v1.Detection`
**File:** `proto/cybereason/ecs/detect/v1/px_detection.proto`
**Purpose:** Top-level detection/alert produced by rule engines. Kafka payload for `detections` topic.
**Produced by:** Flink (stateless/stateful), EDR sensor, XDR vendor, CEP service
**Consumed by:** detection-store (ingestion), correlation-service
**Transport:** Kafka topic `detections`

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | id | bytes (optional) | Detection ULID (16 bytes) |
| 2 | created_at | Timestamp (optional) | Detection creation timestamp |
| 3 | source | DetectionSource (optional) | FLINK_STATELESS / EDR_SENSOR / XDR_VENDOR / CEP_SERVICE |
| 4 | org_id | uint64 (optional) | Tenant ID |
| 5 | rule | CoreRuleInfo (optional) | Triggering rule metadata |
| 6 | threat | Threat (optional) | MITRE ATT&CK classification and threat actor |
| 7 | vulnerabilities | repeated Vulnerability | Associated CVEs |
| 8 | engine_classification | DetectionEngineClassification (optional) | Legacy engine-specific classification |
| 9 | action | DetectionAction (optional) | Mandatory. Action taken (REPORTED_ONLY, BLOCKED, QUARANTINED, etc.) |
| 10 | events | repeated SingleEvent | Triggering events (full event data) |
| 11 | tags | repeated string | Textual tags |
| 12 | source_guid | string (optional) | Legacy Malop GUID (migration only) |

---

### CEP Correlation (`cybereason.phoenix.cep.v1`)

#### `phoenix.cep.v1.ChainTrigger`
**File:** `proto/cybereason/phoenix/cep/v1/px_chain_trigger.proto`
**Purpose:** Published when all stages of a correlation rule fire. Consumed by detection-correlator.
**Produced by:** cep-chainmaker
**Consumed by:** detection-correlator
**Transport:** Kafka topic `triggers`

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | org_id | uint64 (optional) | Tenant ID |
| 2 | rule_id | bytes (optional) | Correlation rule UUID (16 bytes) |
| 3 | rule_name | string (optional) | Rule display name |
| 4 | rule_version | uint32 (optional) | Rule version number |
| 5 | trigger_stage_number | uint32 (optional) | 1-based stage that completed the chain |
| 6 | total_stages | uint32 (optional) | Total stages in the rule |
| 7 | chain_values | map<string,string> | Chain field values that linked events |
| 8 | trigger_event_id | bytes (optional) | Event ULID that completed the chain |
| 9 | event_timestamp | Timestamp (optional) | Completing event timestamp |
| 10 | window_start | Timestamp (optional) | Earliest event in matched chain |
| 11 | window_end | Timestamp (optional) | Latest event in matched chain |
| 12 | created_at | Timestamp (optional) | Trigger creation timestamp |

#### `phoenix.cep.v1.StageEventBatch`
**File:** `proto/cybereason/phoenix/cep/v1/px_stage_events.proto`
**Purpose:** Batch of stage-match events published by cep-service; keyed by rule_id for deterministic routing.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | org_id | uint64 (optional) | Tenant ID |
| 2 | rule_id | bytes (optional) | Rule UUID (Kafka partition key) |
| 3 | events | repeated StageEvent | Batched stage events |
| 4 | batch_timestamp | Timestamp (optional) | Batch flush timestamp |

#### `phoenix.cep.v1.StageEvent`
**File:** `proto/cybereason/phoenix/cep/v1/px_stage_events.proto`
**Purpose:** Single stage match extracted from a raw event.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | org_id | uint64 (optional) | Tenant ID |
| 2 | event_id | bytes (optional) | Source event ULID |
| 3 | rule_id | bytes (optional) | Parent correlation rule UUID |
| 4 | stage_number | uint32 (optional) | Stage ordinal (1-based) |
| 5 | total_stages | uint32 (optional) | Total stages in rule |
| 6 | rule_version | uint32 (optional) | Rule version |
| 7 | chain_values | map<string,string> | Extracted chain field values |
| 8 | event_timestamp | Timestamp (optional) | Original event timestamp |
| 9 | created_at | Timestamp (optional) | Stage event creation time |

---

### Organization Events (`cybereason.phoenix.organization_store.v1`)

#### `phoenix.organization_store.v1.OrganizationChangeNotification`
**File:** `proto/cybereason/phoenix/organization_store/v1/px_organization_event.proto`
**Purpose:** Published to `organization-changes` Kafka topic on any org mutation.
**Produced by:** organization-store service
**Consumed by:** Services maintaining org caches (sensor-gateway, sensor-auth, etc.)
**Transport:** Kafka topic `organization-changes`

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | org_id | uint64 (optional) | Changed organization ID |
| 2 | change_type | EntityChangeType (optional) | CREATE / UPDATE / PATCH / DELETE |
| 3 | fields | repeated OrganizationFieldId | Specific fields changed (PATCH only) |
| 4 | updated_at | Timestamp (optional) | Change timestamp |

---

### Rule Control Events (`cybereason.phoenix.rulecontrol.v1`)

#### `phoenix.rulecontrol.v1.PhoenixRuleControlEvent`
**File:** `proto/cybereason/phoenix/rulecontrol/v1/px_rule_control_event.proto`
**Purpose:** Hot-reload notification for CEP rule changes.
**Produced by:** rule-control-service
**Consumed by:** Flink CEP jobs, cep-service
**Transport:** Kafka topic `rule-control-events`

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | event_type | PhoenixRuleControlEventType | RULE_UPDATED / RULE_DELETED |
| 2 | event_id | bytes (optional) | Event ULID |
| 3 | rule_id | bytes (optional) | Rule UUID (16 bytes) |
| 4 | rule_yaml | string (optional) | Full rule YAML (for RULE_UPDATED) |

---

### Command and Control (`cybereason.phoenix.commandcontrol.v1`)

#### `phoenix.commandcontrol.v1.CommandMessage`
**File:** `proto/cybereason/phoenix/commandcontrol/v1/px_command_control.proto`
**Purpose:** Typed command container serialized into `CommandControlRequest.opaque_command`. Signed for replay protection.
**Produced by:** command-service / dispatcher
**Consumed by:** Phoenix Agent (sensor)
**Transport:** Kafka `actions` topic (via dispatcher → MQTT/WNS → sensor)

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | timestamp | Timestamp (optional) | Command creation time (replay protection) |
| 2 | ttl_seconds | uint32 (optional) | TTL in seconds (typical: 300) |
| 3..31 | command | oneof | One of 31 command types (see below) |

The 31 command variants cover:
- **Core:** Ping, Restart, SetPolicy, SetControlBackend
- **Remediation:** Quarantine, Unquarantine, PreventExecution, KillProcess
- **Lifecycle:** Archive, Unarchive, Decommission, RevertDecommissioned, Delete, Purge, Uninstall
- **Config:** SetRansomwareMode, ResetRansomwareMode, SetAntimalwareStatus, ResetAntimalwareStatus, SetPowershellStatus, ResetPowershellStatus
- **Security:** Isolate, Unisolate
- **DFIR:** ExecuteScript, FetchLog, SystemScan
- **Reputation:** SetReputation
- **Remote Shell:** RemoteShellConnection
- **Upgrade:** Upgrade

#### `phoenix.commandcontrol.v1.CommandControlRequest`
**File:** `proto/cybereason/phoenix/commandcontrol/v1/px_command_control.proto`
**Purpose:** Signed outer envelope for command dispatch.
**Transport:** Kafka `actions` topic

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | command_id | uint64 (optional) | Same as batch_action_id |
| 2 | opaque_command | bytes (optional) | Serialized CommandMessage |
| 3 | command_signature | bytes (optional) | Signature over opaque_command |
| 4 | signature_key_id | string (optional) | Key ID for signature verification |

#### `phoenix.commandcontrol.v1.CommandControlResponse`
**File:** `proto/cybereason/phoenix/commandcontrol/v1/px_command_control.proto`
**Purpose:** Sensor response to a command.
**Produced by:** Phoenix Agent
**Consumed by:** Sensor Gateway, dispatched to `actions-response` topic
**Transport:** Kafka `actions-response` topic

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | command_id | uint64 (optional) | Command this responds to |
| 2 | opaque_response | bytes (optional) | Serialized CommandResponseMessage |

#### `phoenix.commandcontrol.v1.CommandResponseMessage`
**File:** `proto/cybereason/phoenix/commandcontrol/v1/px_command_control.proto`
**Purpose:** Typed response deserialized from CommandControlResponse.opaque_response.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | common | CommandResponseCommon (optional) | org_id, sensor_id, status, timestamp, error |
| 2..30 | response | oneof | One of 31 response types matching CommandMessage |

#### `phoenix.commandcontrol.v1.CommandResponseCommon`
**File:** `proto/cybereason/phoenix/commandcontrol/v1/px_command_common.proto`
**Purpose:** Common fields for all command responses.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | organization_id | uint64 (optional) | Tenant ID |
| 2 | sensor_id | bytes (optional) | Sensor ULID |
| 3 | status | CommandStatus (optional) | Execution status (46 values) |
| 4 | event_time | Timestamp (optional) | Execution timestamp |
| 5 | error_message | string (optional) | Error details |
| 6 | error_code | SystemErrorCode (optional) | Structured error code |

---

### XDR ETL (`cybereason.phoenix.etl_service.v2`)

#### `phoenix.etl_service.v2.VendorRawEnvelope`
**File:** `proto/cybereason/phoenix/etl_service/v2/px_etl_envelope.proto`
**Purpose:** High-throughput XDR vendor data envelope. Billions of messages/day.
**Produced by:** xdr-worker-v2
**Consumed by:** etl-service-v2
**Transport:** Kafka topic `xdr-vendor-raw` (internal)

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | org_id | uint64 (optional) | Tenant ID |
| 2 | org_integration_id | uint64 (optional) | Integration instance ID |
| 5 | payload | bytes (oneof) | Inline vendor payload |
| 6 | s3_url | string (oneof) | S3 URL for oversized payloads |
| 7 | fetched_at | Timestamp (optional) | When data was fetched |
| 8 | schedule_fire_ts | Timestamp (optional) | Scheduled fetch trigger time |
| 9 | response_format | ResponseFormat (optional) | JSON/NDJSON/XML/CSV/PARQUET/etc. |
| 10 | source_id | uint32 (optional) | Source within integration (stable ID) |
| 11 | integration_version | uint32 (optional) | Bundle version (monotonic) |
| 12 | integration_name | string (optional) | Integration slug (e.g., "aws", "okta") |
| 13 | stage_index | uint32 (optional) | Request stage index for VRL routing |

---

### Sensor Registration (`cybereason.phoenix.agent.v1`)

#### `phoenix.agent.v1.FullAgentInfo`
**File:** `proto/cybereason/phoenix/agent/v1/px_full_agent_info.proto`
**Purpose:** Full agent registration payload sent via ppRPC `/agent/{sensor_id}` endpoint.
**Produced by:** Phoenix Agent
**Consumed by:** Sensor Gateway → platform-store
**Transport:** ppRPC (HTTP/2, protobuf)

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | core | CoreAgentInfo | Core agent identity |
| 2 | extended | ExtendedAgentInfo | Pylum ID (legacy), proxy URL |
| 3 | host | Host | Full host information |
| 4 | sensor_status | SensorStatus | Current sensor operational status |
| 5 | sensor_configuration | SensorConfiguration | Current configuration |
| 6 | cloud | repeated Cloud | Cloud provider metadata |
| 7 | software | repeated SoftwareProduct | Installed software |

#### `phoenix.agent.v1.AgentMetrics`
**File:** `proto/cybereason/phoenix/agent/v1/px_agent_metrics.proto`
**Purpose:** Agent performance metrics.
**Transport:** ppRPC `/agent/metrics` endpoint

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | start_time | Timestamp (optional) | Agent instance start time |
| 2 | cpu_user_time | Duration (optional) | User-mode CPU time |
| 3 | cpu_kernel_time | Duration (optional) | Kernel-mode CPU time |
| 4 | memory_usage | uint64 (optional) | Current memory usage in bytes |
| 5 | peak_memory_usage | uint64 (optional) | Peak memory usage in bytes |
| 6 | disk_usage | uint64 (optional) | Installation disk usage in bytes |
| 10 | minion_host | MinionHostMetrics (optional) | Legacy ActiveConsole MinionHost metrics |

---

### Sensor Policy (`cybereason.phoenix.sensor_gateway.v1`)

#### `phoenix.sensor_gateway.v1.PolicyResponse`
**File:** `proto/cybereason/phoenix/sensor_gateway/v1/px_sensor_policy.proto`
**Purpose:** Sensor policy response served by sensor-gateway to agent requests.
**Transport:** REST GET `/api/v1/policies/{policyId}?version={v}` with mTLS (returns protobuf)

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | policy_id | uint64 | Policy identifier |
| 2 | policy_version | uint64 | Version for staleness detection |
| 3 | configuration | CustomerConfiguration (optional) | Legacy format for ActiveConsole sensors |
| 4 | organization_id | uint64 | Organization ID |
| 5 | updated_at | int64 (optional) | Unix epoch milliseconds |
| 6 | raw_policy | Policy (optional) | Rust sensor (Sunbird) format |

---

### Organization Management (`cybereason.phoenix.organization_store.v1`)

#### `phoenix.organization_store.v1.Organization`
**File:** `proto/cybereason/phoenix/organization_store/v1/px_organization_data.proto`
**Purpose:** Organization entity in the multi-tenant hierarchy.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | id | uint64 (optional) | Organization ID |
| 2 | name | string (optional) | Display name |
| 3 | idp_id | string (optional) | Identity provider ID |
| 4 | parent_id | uint64 (optional) | Parent org (null = root) |
| 5 | status | OrganizationStatus (optional) | ACTIVE / SUSPENDED / DELETED |
| 6 | region | string (optional) | Geographic region |
| 7 | description | string (optional) | Description |
| 8 | is_tenant | bool (optional) | True for leaf/tenant orgs |
| 9 | plan_id | uint64 (optional) | Licensing plan |
| 10 | email_domains | repeated string | Allowed email domains |
| 11 | is_mfa_required | bool (optional) | MFA requirement flag |
| 12 | created_at | Timestamp (optional) | Creation timestamp |
| 13 | updated_at | Timestamp (optional) | Last update timestamp |
| 14 | detailed | OrganizationDetailed (optional) | Plan name, license pools, allocations |
| 15 | enrollment_auth_level | EnrollmentSecurity (optional) | UNSECURED or INSTALLATION_KEY |
| 16 | org_key | string (optional) | Public org identifier |
| 17 | enable_control_backend_action | bool (optional) | Expose Set Controlling Backend action |

---

### Detection Storage (`cybereason.phoenix.detection_store.v1`)

#### `phoenix.detection_store.v1.Detection`
**File:** `proto/cybereason/phoenix/detection_store/v1/px_detection_data.proto`
**Purpose:** Detection as stored in PostgreSQL and served by the detection API.

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | id | bytes | 16-byte UUID |
| 2 | organization_id | uint64 | Tenant ID |
| 3 | created_at | Timestamp | Creation time |
| 4 | updated_at | Timestamp | Last update |
| 5 | detection_source | DetectionSource | Source engine |
| 6..9 | detection_rule_* | string (optional) | Rule ID, name, path, version |
| 10 | detection_rule_source_language | RuleSourceLanguage (optional) | SIGMA |
| 11 | detection_rule_severity_level | DetectionSeverityLevel (optional) | INFORMATIONAL..CRITICAL |
| 12 | detection_threat_confidence | double (optional) | 0-100 confidence score |
| 13 | detection_threat_tlp_marking | TlpMarking (optional) | WHITE/GREEN/AMBER/RED |
| 14..16 | detection_threat_tactic/technique/subtechnique_ids | repeated string | MITRE ATT&CK IDs |
| 17..18 | detection_threat_group_id/name | string (optional) | Threat actor group |
| 19..21 | detection_threat_software_* | string/ThreatSoftwareType | Software name and type |
| 22 | important_events | repeated Event | Triggering events |
| 23 | tags | repeated string | Categorization tags |
| 24 | investigation_status | InvestigationStatus (optional) | NEW..ESCALATED |
| 25 | summary | string (optional) | LLM-generated summary |
| 26 | affected_machines | repeated AffectedMachine | Affected hosts |
| 27 | affected_users | repeated AffectedUser | Affected users |
| 28 | child_detections | repeated Detection | Child detections (MalOp tree) |
| 29 | has_xdr_events | bool (optional) | Contains XDR events |
| 30 | detection_engine | DetectionEngine (optional) | Engine classification |
| 31 | legacy_detection_type | LegacyDetectionType (optional) | Legacy classification |
| 32 | detection_engine_specific_name | string (optional) | Engine-specific rule name |
| 33 | detection_rule_title | string (optional) | Human-readable rule title |
| 34 | detection_rule_description | string (optional) | Rule description |
| 35 | decision_statuses | repeated string | Action status summary |
| 36 | associated_assets | repeated DetectionAssociatedAsset | Canonical asset associations |

---

### Sensor Management (`cybereason.phoenix.platform_store.v1`)

#### `phoenix.platform_store.v1.Sensor`
**File:** `proto/cybereason/phoenix/platform_store/v1/px_sensor_data.proto`
**Purpose:** Full sensor record stored in PostgreSQL platform_store.

Key fields (abbreviated — full record has 91 fields):

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | sensor_primary_key | uint64 (optional) | DB primary key (unstable, avoid for correlation) |
| 2 | sensor_id | bytes (optional) | ULID sensor identity (stable) |
| 3 | organization_id | uint64 (optional) | Tenant ID |
| 4 | type | AgentType (optional) | SUNBIRD / ACTIVE_CONSOLE |
| 5 | version | string (optional) | Agent version |
| 6 | policy | AgentPolicy (optional) | Direct policy assignment |
| 7 | group_id | uint64 (optional) | Sensor group |
| 21 | created_at | Timestamp | Creation timestamp |
| 22 | updated_at | Timestamp | Last update |
| 23 | last_seen_time | Timestamp (optional) | Last heartbeat |
| 76..79 | short/long_term_certificate_* | bytes/Timestamp | mTLS certificate tracking |
| 82 | host | Host (optional) | Full host data |
| 83 | lifecycle | SensorLifecycle (optional) | ONLINE/OFFLINE/STALE/ARCHIVED/DECOMMISSIONED |
| 84 | status | SensorStatus (optional) | Operational status flags |
| 85 | configuration | SensorConfiguration (optional) | Current configuration |
| 90 | control_backend_type | ControlBackendType (optional) | LEGACY or PHOENIX backend |
| 91 | extended_details | string (optional) | JSON blob for cloud_info and similar metadata |

---

### Batch Action Tracking (`cybereason.phoenix.sensor_action_store.v1`)

#### `phoenix.sensor_action_store.v1.CommandDispatchEnvelope`
**File:** `proto/cybereason/phoenix/sensor_action_store/v1/px_batch_action_data.proto`
**Purpose:** Kafka envelope from command-service to dispatcher.
**Transport:** Internal Kafka topic (command dispatch)

| # | Name | Type | Description |
|---|------|------|-------------|
| 1 | sensor_ids | repeated bytes | Target sensor ULIDs |
| 2 | organization_id | uint64 (optional) | Tenant ID |
| 3 | sensor_group_id | uint64 (optional) | Target group ID |
| 4 | target_type | TargetType (optional) | SINGLE/MULTIPLE/SENSOR_GROUP/ORGANIZATION |
| 5 | os_type | string (optional) | OS type for routing |
| 6 | command_id | uint64 (optional) | Same as batch_action_id |
| 7 | command_bytes | bytes (optional) | Serialized CommandControlRequest |
| 8 | wns_channel_uri | string (optional) | WNS channel (deprecated) |
| 9 | is_notification_hub_available | bool (optional) | Azure Notification Hub flag |
| 10 | action_type | ActionType (optional) | Action for dispatcher routing |
| 11 | initiated_by | string (optional) | Audit trail identity |
| 12 | remaining_attempts | uint32 (optional) | Retry attempts remaining |

---

### Vault Service (`cybereason.phoenix.vault.v1`)

#### `phoenix.vault.v1.VaultService` (gRPC)
**File:** `proto/cybereason/phoenix/vault/v1/px_vault_svc.proto`
**Purpose:** Org-specific encryption/decryption using Shamir secret sharing for key management.

---

### Policy Configuration (`cybereason.phoenix.platform_store.v1.Policy`)
**File:** `proto/cybereason/phoenix/platform_store/v1/px_policy_data.proto`
**Purpose:** Full sensor security policy with all protection module settings.

The `Policy` message has 115 primary settings fields organized into grouped sub-messages (EndpointUiSettings, CollectionSettings, AppControlSettings, AntiRansomwareSettings, AntiMalwareSettings, PowerShellSettings, EndpointProtectionSettings, AdvancedFlagsSettings, CmsSettings, InfrastructureSettings, RulesEngineSettings, ArwSettings, VmSettings, SensorManagementSettings, ResponseSettings) plus 21 exclusion/allowlist lists.

---

### Raw Event Store (`cybereason.phoenix.event_store.v1`)

#### `phoenix.event_store.v1.RawEvent`
**File:** `proto/cybereason/phoenix/event_store/v1/px_raw_event.proto`
**Purpose:** Flat projection of ClickHouse `raw_events` table — all 333 possible event fields. Used by `RawEventService` gRPC.

The message has 293 explicit fields covering: event classification, agent info, observer, host, user, process (with parent), process code signature/hashes/PE, file (with hashes), network (source/dest with geo), DNS, session, authentication, registry, tags/labels. Different query modes populate different subsets (list: ~107 fields; detail: all 333).

---

## Enum Catalog

### EventAction
150+ values covering all EDR/XDR event categories:
- **Process:** PROCESS_CREATED (10), PROCESS_ENDED (11), PROCESS_FORK (16), PROCESS_REPLACEMENT (17)
- **Module:** MODULE_LOADED (21), MODULE_UNLOADED (22)
- **Network:** NETWORK_CONNECTION_ATTEMPTED..DENIED (30-34), DNS_QUERY/RESPONSE (35-36), RPC_* (38-41)
- **File:** FILE_CREATED..FILE_STREAM_CREATED (60-71)
- **Registry:** REGISTRY_KEY_CREATED..REGISTRY_VALUE_READ (80-86)
- **Auth/User/Session:** USER_LOGIN_SUCCESS..SESSION_UNLOCKED (100-117, 400-439)
- **System/Service:** SERVICE_STARTED..KERNEL_MODULE_UNLOADED (130-148)
- **Security:** VULNERABILITY_DETECTED..SCAN_RESUMED (210-489)
- **Cloud:** COMPUTE_INSTANCE_CREATED..FUNCTION_INVOKED (250-263)
- **Email:** EMAIL_SENT..EMAIL_LINK_BLOCKED (280-284, 540-543)
- **Database:** DATABASE_STATEMENT_EXECUTED..DATABASE_DML_EXECUTED (300-311)
- **Cross-process (Windows ETW):** CROSS_PROCESS_VIRTUAL_PROTECT..CROSS_PROCESS_ALLOCATE_VIRTUAL_MEMORY (572-583)
- **Cloud Security:** ENCRYPTION_KEY_DISABLED..LDAP_QUERY (600-606)
- **Scheduled Task:** SCHEDULED_TASK_CREATED..SCHEDULED_TASK_MODIFIED (607-610)

### CommandStatus
46 values: PENDING, IN_PROGRESS, FAILED_SENDING, PRIMED, SUCCEEDED, TIMEOUT, INVALID_STATE, ABORTED, ABORTING, and many upgrade-specific statuses.

### AgentType
AGENT_TYPE_ACTIVE_CONSOLE (1), AGENT_TYPE_SUNBIRD (2), AGENT_TYPE_ENDPOINT (3), AGENT_TYPE_CLOUDBEAT (4), AGENT_TYPE_FILEBEAT (5), AGENT_TYPE_OSQUERY (6), AGENT_TYPE_SYSMON (7)

### DetectionSource
FLINK_STATELESS (1), FLINK_STATEFUL (2), EDR_SENSOR (3), XDR_VENDOR (4), CEP_SERVICE (5)

### InvestigationStatus
NEW (1), REOPENED (2), INVESTIGATING (3), ON_HOLD (4), CLOSED (5), FALSE_POSITIVE (6), RESOLVED (7), ESCALATED (8)

### ActionType
29 values covering all command types (RESTART, PING, SET_POLICY, QUARANTINE, ISOLATE, EXECUTE_SCRIPT, REMOTE_SHELL_CONNECTION, etc.)

### OsType
WINDOWS (1), LINUX (2), DARWIN (3), SOLARIS (4), OTHER (6)

### OsFamily
17 values: WINDOWS, MACOS, IOS, ANDROID, CHROME_OS, REDHAT, DEBIAN, SUSE, ALPINE, ARCH, AIX, SOLARIS, FREEBSD, OPENBSD, NETBSD, DRAGONFLY_BSD

### CpuArchitecture
17 values: X86_64, AMD64, X86, I386, I686, AARCH64, ARM64, ARM, ARM_V7L, ARM_V8L, PPC64, PPC64LE, S390X, MIPS, MIPS64, RISC_V64, OTHER

### SensorLifecycleStatus
ONLINE (1), OFFLINE (2), STALE (3), ARCHIVED (4), DECOMMISSIONED (5), PURGED (6), UNINSTALLED (7)

### PhoenixRuleControlEventType
RULE_UPDATED (1), RULE_DELETED (2)

### EntityChangeType (org events)
CREATE (1), UPDATE (2), PATCH (3), DELETE (4)

### ProcessInjectionType
14 values: LOAD_LIBRARY, PROTECTED_PROCESS, UNMAPPED_MEMORY_SECTION, NEW_ANON_RWX, PROCESS_MEMORY_DUMP, SHELLCODE_STAGER, SHELLCODE_MIGRATE, SHELLCODE_EXPLOIT, SHELLCODE_COBALT, SET_THREAD_CONTEXT, PROCESS_HOLLOWING, QUEUE_USER_APC, SHELLCODE_HASHDUMP, ATOM_BOMBING

### Errno
Full Linux errno catalog (133 values): EPERM through EHWPOISON, with ABI-independent naming for cross-platform normalization.

---

## gRPC Service Definitions

| Service | File | Port | Methods |
|---------|------|------|---------|
| `HealthService` | `common/health/v1/px_health_svc.proto` | All services | Ping |
| `SensorOnboardingService` | `phoenix/auth/v1/px_sensor_onboarding_svc.proto` | 50051 | Challenge, CertificateIssue (deprecated) |
| `SensorAuthChallengeService` | `phoenix/auth/v2/px_sensor_challenge_auth_svc.proto` | 50051 | ProxyInitiateChallenge, ProxySolveChallenge |
| `SensorService` | `phoenix/platform_store/v1/px_sensor_svc.proto` | 50052 | ListSensors, GetSensor, RegisterSensor, UpdateOrRegisterSensor, UpdateSensor, BatchUpdateSensorHeartbeats, RunSensorLifecycleTransitions, DeleteSensor, AssignSensorPolicy, RemoveSensorPolicy, GetEffectivePolicyForSensor, GetFilterMetadata |
| `RawEventService` | `phoenix/event_store/v1/px_event_svc.proto` | 50053 | QueryEvents, QueryEvent, HealthCheck, GenerateEventSummaryReport |
| `BatchActionService` | `phoenix/sensor_action_store/v1/px_batch_action_svc.proto` | 50054 | CreateBatchAction, GetBatchAction, ListBatchActions, CreateSensorActions, UpsertSensorActionStatuses, GetBatchActionStatus, GetBatchActionStatuses, GetPaginatedSensorStatuses, UpdateBatchActionCreationStatus, GetBatchActionsForRetry, GetSensorActionsForRetry, UpdateSensorActionRetryState |
| `CommandService` | `phoenix/command_service/v1/px_command_svc.proto` | 50055 | CreateAndDispatchBatchAction, GetFetchLogUploadUrl, DownloadLog, GetScriptUploadUrl, DownloadScriptLog, GetInstallerDownloadUrl |
| `AssetService` | `phoenix/asset_store/v1/px_asset_svc.proto` | 50056 | (asset CRUD) |
| `MitreService` | `phoenix/mitre_service/v1/px_mitre_svc.proto` | 50057 | (MITRE ATT&CK enrichment) |
| `RuleControlService` | `phoenix/rulecontrol/v1/px_rule_control_svc.proto` | 50058 | AppendVersion, RollbackRule, DeactivateRule, ActivateRule, ListRules, GetRule, ListRuleAuditLogs, DistributeRule, ListRuleDistributions, ListRuleGroups, CreateRuleGroup, AssignRulesToRuleGroup |
| `OrganizationService` | `phoenix/organization_store/v1/px_organization_svc.proto` | 50059 | ListOrganizations, GetOrganization, GetOrganizationHierarchy, GetDescendantOrganizationIds, ValidateOrganizationScope, CheckOrganizationExists, ValidateEmailDomain, CreateOrganization, UpdateOrganization, CreateInstallationKey, GetInstallationKey, RevokeInstallationKey, ListInstallationKeys, GetAllOrganizationEnrollmentSettings, GetAllOrganizationsDiscoverySettings |
| `DetectionService` | `phoenix/detection_store/v1/px_detection_query_svc.proto` | (detection-store) | ListDetections, GetDetection, GetDetectionCountsByAssetIds, SaveDetectionSummary, UpdateDetectionInvestigationStatus, GetDetectionOverview, GetDetectionTimeline, GetDetectionAttackPath, GetMalopDetectionIds, GetAttackPathNodeDetail, GetAttackPathGraphOwner, ListDetectionComments, CreateDetectionComment, DeleteDetectionComment, ToggleDetectionCommentPin |
| `DetectionIngestService` | `phoenix/detection_store/v1/px_detection_ingest_svc.proto` | (detection-store) | IngestDetection |
| `VaultService` | `phoenix/vault/v1/px_vault_svc.proto` | (vault) | Decrypt, Encrypt, GetStatus, Seal, Unseal, GenerateOrgKey, GetOrgPublicKey |
| `EtlTransformService` | `phoenix/etl_service/v2/px_etl_transform_svc.proto` | (etl) | (transform) |

---

## Kafka Topic → Message Mapping

| Topic | Message Type | Producer | Consumer(s) | Notes |
|-------|-------------|----------|-------------|-------|
| `raw-events` | `ecs.event.v1.SingleEvent` | Sensor Gateway (unpacked from EventBatch) | Flink (rules), ClickHouse ingestion, detection-correlator | One message per event. Gateway unpacks OpaqueEventBatch, merges common into each event |
| `detections` | `ecs.detect.v1.Detection` | Flink stateless/stateful, EDR sensor, XDR via ETL, CEP | detection-store (DetectionIngestService), correlation-service | Includes full triggering SingleEvent objects inside Detection.events |
| `agents_v2` | `ecs.agent.v1.CoreAgentInfo` (embedded in FullAgentInfo) | Phoenix Agent (via sensor-gateway) | platform-store, asset-store | Agent registration / heartbeat on connect |
| `actions` | `phoenix.commandcontrol.v1.CommandControlRequest` | dispatcher (receives CommandDispatchEnvelope) | Sensor Gateway → MQTT/WNS/Azure NotificationHub → Agent | Serialized+signed CommandMessage inside opaque_command |
| `actions-response` | `phoenix.commandcontrol.v1.CommandControlResponse` | Sensor Gateway (from agent response) | sensor-action-store | Opaque CommandResponseMessage inside |
| `triggers` | `phoenix.cep.v1.ChainTrigger` | cep-chainmaker | detection-correlator | Correlation chain completion |
| `organization-changes` | `phoenix.organization_store.v1.OrganizationChangeNotification` | organization-store | sensor-gateway, sensor-auth, local caches | Published on CREATE/UPDATE/PATCH/DELETE |
| `rule-control-events` | `phoenix.rulecontrol.v1.PhoenixRuleControlEvent` | rule-control-service | Flink CEP jobs, cep-service | Hot-reload on rule changes |
| `xdr-vendor-raw` (internal) | `phoenix.etl_service.v2.VendorRawEnvelope` | xdr-worker-v2 | etl-service-v2 | Billions/day; inline or S3 URL for oversized |
| `cep-stage-events` (internal) | `phoenix.cep.v1.StageEventBatch` | cep-service instances | cep-chainmaker | Batched per rule_id for deterministic routing |

---

## buf Configuration

**Module:** `com.cybereason/cybereason/phoenix`
**Proto root:** `proto/`

**Lint rules:** `STANDARD` ruleset (the complete buf standard set, which enforces package naming, enum prefixing, field naming, service/RPC naming conventions, reserved ranges, etc.)

**Lint exceptions:**
- `proto/grpc/health/v1/health.proto` — excluded from lint to preserve Kubernetes-compatible standard gRPC health check names that would otherwise violate Cybereason's naming conventions.

**Breaking change detection:** Enabled with `FILE` strategy. This performs wire-compatibility checks at file granularity, detecting field number reuse, type changes, removal of required fields, service/method removal, and enum value removal.

**Code generation targets:**
- Go protobuf (buf.build/protocolbuffers/go v1.31.0) → `gen/go/`, paths=source_relative
- Go gRPC (buf.build/grpc/go v1.3.0) → `gen/go/`, paths=source_relative
- No Rust generation in this repo — Rust crates use prost-build in their own build.rs

**Managed mode:** Enabled. buf auto-manages `go_package` options.

---

## Cross-Component Sync Status

**Phoenix Server (`projects/Phoenix/`):** Zero `.proto` source files found. The server consumes proto definitions exclusively via generated Go output in `modules/proto/gen/go/` and generated Rust output in `rust/pbgen/`. There is no diverged local copy of the proto sources.

**Phoenix Agent (`projects/phoenix-agent/`):** Zero `.proto` source files found in `crates/phoenix-protobuf/`. The agent generates Rust structs via prost-build at compile time, referencing the canonical schemas from this repository. No diverged copy exists.

**Conclusion:** There is exactly one canonical source of proto definitions — this `phoenix-proto` repository. All components consume generated artifacts, not duplicated proto sources. Schema changes flow from this repository outward. This is a healthy single-source-of-truth architecture with no divergence risk from local copies.

---

## Schema Governance

**Breaking change detection:** `buf breaking` with `FILE` strategy is configured. This should be enforced in CI to prevent backward-incompatible schema changes (field number reuse, type changes, required field removal).

**Lint enforcement:** `STANDARD` ruleset with buf. Enforces consistent naming, enum zero-value suffixes (`_UNSPECIFIED`), and proper package structure. The buf lint configuration is minimal exceptions — only the third-party gRPC health proto is excluded.

**Versioning approach:**
- API versions are encoded in the package path (`.v1`, `.v2`). Three packages have both v1 and v2: `auth/`, `vault/`, `integration_manager/`, `integration_proxy/`, `xdr_worker/`, and `etl_service/`. v2 supersedes v1 in all these cases.
- Field deprecation is handled via proto comments (`[+Deprecated]`) and `reserved` field numbers. Many deprecated fields use pre-release reservations with a note that field numbers may be reassigned.
- The `agents_v2` Kafka topic name itself encodes the version of the schema (v1 was the original ActiveConsole schema; v2 is the current Phoenix schema).

**Coordination across components:** Schema changes require coordinated deployment since both Go (server) and Rust (agent) consumers must be updated together. The agent's generated Rust structs and the server's generated Go structs must remain wire-compatible. Breaking changes require a two-phase deployment (new field deployed to consumers before it's sent by producers).

**Multi-tenancy enforcement at schema level:** `org_id` is present in every Kafka message (SingleEvent, Detection, ChainTrigger, OrganizationChangeNotification, VendorRawEnvelope, StageEventBatch, CommandDispatchEnvelope) and in every gRPC request/response that touches tenant data. This is a schema-level guarantee that no cross-tenant data leak is possible through missing partition keys.
