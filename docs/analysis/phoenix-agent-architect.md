# Phoenix Agent (Sunbird) — Architectural Overview

## What Is This?

Sunbird is the endpoint component of the Phoenix EDR/XDR platform, developed by Cybereason. It is a single Rust binary deployed on customer machines running Linux, macOS, or Windows. Its job is to be a silent, low-footprint observer: it watches OS-level activity (process launches, file operations, network connections, shell commands), streams that telemetry to the Phoenix cloud backend, and enforces security policies in real-time by blocking or terminating malicious processes inline — before they have time to cause damage.

Unlike a traditional AV scanner, Sunbird operates as a sensor in a larger detection architecture. Most complex detection logic lives server-side in Apache Flink CEP rules and the ClickHouse analytics engine. Sunbird handles what must happen locally: high-frequency event capture with minimal latency, reputation-based allow/block decisions on process creation, local Sigma rule evaluation, and the execution of response commands (isolate host, kill process, quarantine file, run script) received from the Phoenix command plane.

The agent connects outbound to two Phoenix server endpoints: the Sensor Gateway (HTTPS/REST with mTLS) for event delivery and agent registration, and a Sensor MQTT broker (MQTTs with mTLS) for receiving commands. The two-channel design separates high-throughput telemetry delivery from low-latency command reception without either blocking the other.

---

## High-Level Architecture Diagram

```mermaid
flowchart LR
    subgraph Endpoint ["Endpoint (Linux / macOS / Windows)"]
        subgraph Collectors ["OS Collectors"]
            ETW["ETW / ferrisetw\n(Windows)"]
            ES["Endpoint Security fw\n(macOS)"]
            OWL["owLSM (eBPF)\n(Linux)"]
            BD["BitDefender\non-access scanner\n(Windows, opt)"]
        end

        subgraph Pipeline ["Detection Pipeline"]
            direction TB
            INLINE["Inline handler\n(latency-sensitive)"]
            ASYNC["Async handler\n(bulk rules)"]
            SIGMA["Sigma engine"]
            BOSS["Boss / YARA\n(VFP + VPP)"]
            STATEFUL["Stateful engine\n(injection detection)"]
            TAGGING["Tagging engine"]
            REP["Reputation cache\n(SQLite)"]
        end

        subgraph Cloud ["Cloud Communication"]
            GW_CLIENT["SensorGatewayClient\n(HTTPS mTLS)"]
            MQTT_CLIENT["MQTT subscriber\n(MQTTs mTLS)"]
        end

        STORAGE["SQLite local DB\n(encrypted)"]
        CNC["CommandAndControl\n(CnC handler)"]
        ACTIONS["ActionExecutor\n(kill / quarantine / isolate)"]
    end

    subgraph Server ["Phoenix Server"]
        GATEWAY["sensor-gateway\n(REST + auth)"]
        MQTT_BROKER["sensor-mqtt-mtls\n(MQTT broker)"]
        KAFKA["Redpanda\n(raw-events / detections)"]
    end

    ETW -->|SensorEvent| INLINE
    ES  -->|SensorEvent| INLINE
    OWL -->|FlatBuffer events| ASYNC
    BD  -->|inline callback| INLINE

    INLINE -->|decision + event| ASYNC
    ASYNC --> SIGMA
    ASYNC --> BOSS
    ASYNC --> STATEFUL
    ASYNC --> TAGGING
    ASYNC --> REP

    ASYNC -->|EventBatch\nProtobuf + zstd| GW_CLIENT
    ASYNC -->|DetectionBatch\nProtobuf + zstd| GW_CLIENT

    GW_CLIENT -->|PUT /api/v1/orgs/{org}/agents/{id}| GATEWAY
    GW_CLIENT -->|POST /api/v1/orgs/{org}/sensors/{id}/events| GATEWAY
    GW_CLIENT -->|POST /api/v1/orgs/{org}/sensors/{id}/detections| GATEWAY

    GATEWAY -->|publishes| KAFKA

    MQTT_BROKER -->|commands/organizations/{org}/sensors/{id}| MQTT_CLIENT
    MQTT_CLIENT --> CNC
    CNC --> ACTIONS
    CNC --> GW_CLIENT

    INLINE <-->|inline allow/deny| BD

    STORAGE --- REP
    STORAGE --- CNC
```

---

## Crate Catalog

| Crate | Platform | Role | Key Dependencies |
|---|---|---|---|
| `phoenix-sunbird` | all | Main binary — wires all crates together, owns `Application` struct and startup | all agent crates |
| `phoenix-sunbird-macros` | all | Proc-macro helpers used by the main crate | `syn`, `proc-macro2`, `quote` |
| `sunbird-config` | all | Runtime + installation config abstraction (registry/plist/env) | `serde`, `clap` |
| `sunbird-identity` | all | Agent ID (UUID-like), `OrganizationId`, `OrganizationKey`, Protobuf conversion for `CoreAgentInfo` | `phoenix-protobuf`, `bon`, `mac_address` |
| `sunbird-auth` | all | Enrollment engine, certificate challenge/solve, short-/long-term cert lifecycle | `sunbird-keystore`, `sunbird-identity`, `sensor-gateway-client`, `ed25519-dalek`, `p256`, `prost` |
| `sunbird-keystore` | all | Key/cert storage abstraction; Windows = Windows Certificate Store (CNG), others = file-based | `rustls`, `sunbird-config` |
| `sunbird-secret` | all | Symmetric secret derivation (HKDF) for SQLite DEK; Windows = DPAPI, others = file | `sunbird-identity` |
| `sunbird-detection-pipeline` | all | Core detection pipeline — inline + async handlers, engine registry, tagging, reputation | `sigma-rust`, all engine crates, `crossbeam` |
| `sigma-rust` | all | Sigma rule parser and evaluator (custom implementation) | — |
| `sunbird-cloud` | all | Cloud provider metadata detection (AWS IMDSv2, Azure `dsregcmd`) | `reqwest` |
| `sensor-gateway-client` | all | HTTPS client for Sensor Gateway REST API — sends events, detections, heartbeats, fetches policy | `reqwest`, `prost`, `phoenix-protobuf`, `zstd` |
| `phoenix-protobuf` | all | Pre-generated Protobuf types (`EventBatch`, `DetectionBatch`, `FullAgentInfo`, auth messages, etc.) | `prost` |
| `sunbird-agent-linux` | linux | Thin Linux upgrade utilities; event collection done by `owlsm-rs` | — |
| `sunbird-agent-macos` | macos | macOS upgrade utilities | — |
| `sunbird-agent-windows` | windows | Windows upgrade utilities | — |
| `sunbird-endpoint-sec-macos` | macos | Apple Endpoint Security framework client — subscribes to ES notify events | `endpoint_sec`, `sunbird-detection-pipeline` |
| `owlsm-rs` | linux | Spawns the external OwlSM eBPF binary, reads FlatBuffer events from its stdout, routes into detection pipeline | `flatbuffers`, `crossbeam-channel`, `sunbird-detection-pipeline` |
| `wfp-rs` | windows | Safe Rust wrapper around Windows Filtering Platform (WFP) API for host isolation / network blocking | `windows-sys` |
| `sunbird-netmon-macos` | macos | macOS network monitoring (wraps system network APIs) | — |
| `sunbird-shell` | all | Shell command execution utilities (used for response actions) | — |
| `sunbird-paths` | all | OS-specific canonical paths (`install_dir`, `data_dir`, `logs_dir`, `owlsm_dir`, etc.) | — |
| `sunbird-metrics` | all | Lightweight metrics collection; writes periodic JSON files to log directory | `metrics` |
| `sunbird-bundle` | all | Bundle loading and packing utilities | — |
| `sunbird-devtools` | non-prod | Developer tooling (not included in production builds) | — |
| `into` / `into_derive` | all | Derive macro for field-level `Into` conversions | `syn` |
| `bitdefender/*` | windows | BitDefender AV integration (on-access scanning, AMSI, ATC, ransomprotect); optional feature | `bitdefender-cst` |
| `windows-notification` | windows | Windows push notification (WNS) support for receiving commands without polling MQTT | — |
| `windows-security-center-rs` / `-sys` | windows | Windows Security Center registration | — |

---

## Agent Lifecycle

1. **Process start**: The OS launches the `phoenix-sunbird` binary (as a systemd service on Linux, a LaunchDaemon on macOS, or a Windows service). `clap` parses CLI flags/env vars (`--org-key`, `--enrollment-auth-type`, `--sensor-gateway-url`, `--sensor-gateway-mtls-url`, `--mqtt-endpoint`, `--discovery-base-url`).

2. **Observability init**: `tracing-subscriber` is configured. A panic hook routes panics into the structured log. `sunbird-metrics` is initialized and a metrics file-writer thread is spawned.

3. **Agent ID resolution**: The agent reads `AgentId` from the runtime config store (registry on Windows, plist on macOS, or a config file). If absent, a new UUID-like `Id` (type `StaticAgentId`) is generated and persisted. This ID is stable for the lifetime of the installation.

4. **Local state init**: `AgentStorage` opens the SQLite database at `{data_dir}/cybereason-agent.db`. The DB is encrypted using a Data Encryption Key (DEK) derived via HKDF from the agent ID and a platform secret (DPAPI on Windows, file-based HKDF elsewhere). On first boot the DEK is generated and stored. If the DB cannot be decrypted, the agent panics (unrecoverable). Other DB errors trigger a delete-and-recreate recovery path.

5. **Detection pipeline spawn**: Before network is available, the detection pipeline is launched. Sigma rules, Boss/YARA VFP and VPP bundles, and the tagging engine are loaded from embedded binaries compiled into the agent via `bundle-manifest.toml`. The pipeline allocates two unbounded crossbeam channels: `inline_receiver` (latency-sensitive path, up to `cpu_count/4` parallel threads, minimum 4) and `async_receiver` (single thread).

6. **Policy and reputation load from cache**: `SensorPolicyController` loads the last-known policy from SQLite. `ReputationController` loads the reputation table. Both are available offline so detection keeps working without network connectivity.

7. **Platform-specific collector startup**:
   - **Windows**: `EtwSessionHandle` is created via `ferrisetw` — an ETW real-time consumer session that receives kernel-level events (process creation, network, file, registry, image load). Optionally, BitDefender on-access scanning is initialized, which provides inline callbacks.
   - **macOS**: `spawn_endpoint_security()` registers an Endpoint Security client with the `ES_EVENT_TYPE_NOTIFY_*` family to receive process, file, and network events. If the process lacks Full Disk Access, the client retries every 60 seconds.
   - **Linux**: `owlsm-rs::spawn_owlsm_pipeline()` launches the external `owlsm` eBPF binary (located at `{owlsm_dir}/bin/owlsm`). owLSM delivers events as size-prefixed FlatBuffers on its stdout pipe. The agent reads and converts these to `SensorEvent` values and routes them into the async detection channel. If the binary is missing, the agent continues without Linux kernel-level collection.

8. **Network initialization (background thread)**: A dedicated `network_init` thread runs the endpoint discovery and enrollment loop with exponential backoff (2 s, doubling up to 300 s max).

   a. **Endpoint discovery**: If `--sensor-gateway-url` / `--mqtt-endpoint` are provided, they are used directly. Otherwise, a discovery HTTP request is made to `--discovery-base-url` to retrieve the three server URLs (`sensor_gateway`, `sensor_gateway_mtls`, `sensor_mqtt_mtls`).

   b. **Enrollment or re-enrollment**: If `OrgId` is already persisted in the runtime config, the agent constructs an `AuthContext<Enrolled>` and calls `ensure_certificates()`. Otherwise, it calls `initial_enrollment()`. Enrollment uses a three-step protocol over the plain (non-mTLS) gateway URL: initiate challenge (POST `/auth/challenge/initiate`) → solve proof-of-work challenge (Argon2-style nonce iteration) + sign with ephemeral Ed25519 key → complete enrollment (POST `/auth/challenge/solve`). The server returns a long-term X.509 certificate signed by the Phoenix CA.

   c. **Short-term cert issuance**: Immediately after enrollment, a short-term certificate is fetched by submitting a renewal request authenticated with the just-issued long-term cert. The short-term cert is used for mTLS connections to the gateway.

   d. **Authentication worker**: A background thread (`authentication_worker`) checks certificate expiry every minute. Long-term certs renew at ~70% of their lifetime (± 5% jitter) to avoid thundering-herd. Short-term certs renew if missing or within 5 minutes of expiry.

9. **Gateway client creation**: `SensorGatewayClient` is built with the mTLS `ClientConfig` derived from the short-term cert. It connects to the `sensor_gateway_mtls_url`.

10. **First agent-info push**: A `FullAgentInfo` (including `CoreAgentInfo`, OS info, IP/MAC, agent version, active policy) is PUT to `/api/v1/orgs/{org}/agents/{id}`.

11. **Policy fetch**: The backend policy is fetched from `/api/v1/orgs/{org}/sensors/{id}/policy/{id}?version={n}` and applied. Policy changes (rules engine enabled/disabled, path exclusions, action policy) propagate via `add_on_change` callbacks. On Linux, owLSM is restarted with new configuration when policy changes.

12. **MQTT command subscription**: `CommandAndControl::subscribe()` connects to the MQTT broker with mTLS (TLS 1.3 via rustls). On Windows, Windows Notification Service (WNS) is attempted first as a lower-latency push channel; MQTT is used as fallback. The MQTT topic subscribed is `commands/organizations/{org_id}/sensors/{sensor_id}` at QoS 1. The session is `clean_session=true`; on `ConnAck` the client re-subscribes.

13. **Steady-state event loop**: Collectors produce events → detection pipeline → events batched and sent to Sensor Gateway. Commands arrive via MQTT → CnC handler routes to `ActionExecutor` or internal handlers.

14. **Shutdown**: On SIGTERM/SIGINT, `ShutdownManager` signals all components. ETW, BitDefender, owLSM, detection pipeline, MQTT, auth worker, and storage are stopped in sequence. A 20-second drain timeout is applied before a forced exit.

---

## Platform Collectors

### Windows — ETW (Event Tracing for Windows)

**API surface**: `ferrisetw` crate wrapping the Windows ETW consumer API. The agent subscribes to kernel providers (Microsoft-Windows-Kernel-Process, Microsoft-Windows-Kernel-Network, Microsoft-Windows-Kernel-File, and others).

**Event types collected**:
- Process creation / termination (with full command line, SHA-256 hash, parent PID)
- Network connections (TCP/UDP, IPv4/IPv6, remote IP/port, protocol)
- File access events (open, read, write, create, delete, rename) — filtered by policy
- Image/DLL loads (module load events for injection detection)
- Registry reads and writes
- DNS query/response (via DNS client ETW provider)

**BitDefender integration (optional feature)**: When compiled with the `bitdefender` feature, the BitDefender on-access scanning engine is loaded as a DLL. It registers callbacks that block or allow file accesses inline before they complete. These callbacks produce `InlineRequest` events that flow into the `inline_sender` channel. The inline handler evaluates Sigma + Boss/YARA rules synchronously and returns `InlineDecision::Allow` or `::Deny` within the callback deadline (3-second budget).

**Privilege**: Requires administrator (LocalSystem) to access kernel ETW providers.

### macOS — Apple Endpoint Security Framework

**API surface**: `endpoint_sec` crate (Rust bindings to the Apple ESF C API, `libEndpointSecurity`). The agent creates a single notify-mode ES client using `ES_NEW_CLIENT_RESULT_SUCCESS` and subscribes to ES notify event types.

**Event types collected**: Process execution (`ES_EVENT_TYPE_NOTIFY_EXEC`), process fork, process exit, file open/create/write/unlink/rename, network connection events (where ESF exposes them). The exact set of subscribed event types is defined in `sunbird-endpoint-sec-macos/src/clients/`.

**Retry behavior**: If the process lacks Full Disk Access entitlement, the client is not permitted (`ES_NEW_CLIENT_RESULT_ERR_NOT_PERMITTED`). The agent logs a guidance message and retries every 60 seconds. If the process is not entitled (entitlement not present in the provisioning profile), it stops retrying.

**Privilege**: Must be in the macOS System Extensions list with the `com.apple.developer.endpoint-security.client` entitlement. Installed as a LaunchDaemon under `/Library/LaunchDaemons/` running as root.

### Linux — OwlSM (eBPF-based security module)

**API surface**: The `owlsm-rs` crate does not call eBPF APIs directly. It spawns an external binary (`{install_dir}/owlsm/bin/owlsm`) that runs as a privileged process and writes FlatBuffer-encoded events on its stdout.

**Event types collected**:
- Process fork, exec, exit
- File operations: create, unlink, read, write, mkdir, rmdir, chmod, chown, rename
- Network connections (TCP/UDP, with source/dest IP and port)
- Shell command execution (if `shell_commands_monitoring` is enabled in policy)

**IPC protocol**: Size-prefixed FlatBuffers (4-byte little-endian length + FlatBuffer payload). stderr carries size-prefixed FlatBuffer `Error` frames; the agent logs each error with hook name and code.

**Policy propagation**: owLSM configuration is serialized as JSON and delivered to the subprocess via stdin at launch time. Policy changes trigger a full restart of owLSM (stop old process, start new one) because config cannot be changed at runtime.

**Privilege**: owLSM requires `CAP_BPF` / `CAP_SYS_ADMIN` or root to load eBPF programs.

---

## Cloud Communication

### Protocol and Target

The `sensor-gateway-client` crate wraps a `reqwest` blocking HTTP client. All post-enrollment communication uses HTTPS (TLS 1.3 via rustls) with mTLS — both the server and the agent authenticate with X.509 certificates.

The target is `sensor_gateway_mtls_url` (resolved at startup), which corresponds to the `sensor-gateway` server. Based on the server architecture documented in CLAUDE.md, sensor-gateway listens on port 50051 for gRPC internally, but the agent speaks HTTP REST to it (the gateway exposes REST endpoints that it translates internally).

### Event Sending

**Event batching**: `SunbirdLowPrioritySender` accumulates `SensorEvent` values (ordinary telemetry) into an `EventBatch` Protobuf message. `SunbirdHighPrioritySender` sends detections as `DetectionBatch` messages immediately. Both are compressed with `zstd` at level 3 before HTTP transmission (this can be disabled via compile-time env `SUNBIRD_DISABLE_COMPRESSION`).

**Endpoints**:
- Events: `POST /api/v1/orgs/{org_id}/sensors/{sensor_id}/events`
- Detections: `POST /api/v1/orgs/{org_id}/sensors/{sensor_id}/detections`
- Agent heartbeat / info: `PUT /api/v1/orgs/{org_id}/agents/{sensor_id}`

**Request timeout**: 30 seconds.

### Command Reception

Commands are received via MQTT over TLS (`mqtts://`, port 443 by default). The `rumqttc` client library is used. The subscription topic is `commands/organizations/{org_id}/sensors/{sensor_id}`.

The MQTT session uses `clean_session=true`. On every `ConnAck` (initial connect or reconnect), the client explicitly re-subscribes. This is intentional: with `clean_session=true` the broker does not persist subscriptions, so re-subscribing is required. The library auto-reconnects after disconnection.

On Windows only, Windows Push Notification Service (WNS) is attempted as a primary channel. If WNS is not available, MQTT is used as a fallback.

**Payload decoding**: MQTT payloads are Protobuf-encoded `CommandControlResponse` messages processed by `CommandHandler`.

### Authentication

The agent presents its short-term X.509 certificate (ECDSA P-256) for mTLS. The server validates it against the Phoenix CA. For enrollment (before certs exist), connections to the plain (non-mTLS) gateway URL use the HTTPS certificate from the server CA only.

### Reconnection / Retry

Discovery and enrollment use exponential backoff: starting at 2 seconds, doubling on each failure, capped at 300 seconds. The `rumqttc` library handles MQTT reconnection internally with its own backoff. When the MQTT internal channel disconnects due to a library bug (see the comment referencing bytebeamio/rumqtt#820), the agent recreates the full MQTT connection with its own backoff.

---

## Local Detection

### Detection Engines

The `sunbird-detection-pipeline` crate orchestrates five detection engines:

1. **Sigma engine** (`sigma-rust`): Evaluates YARA-like Sigma rules against `SensorEvent` fields. Rules are embedded as a binary bundle (`pcp_rules_bep.bundle.bin`) compiled into the binary at build time from `bundle-manifest.toml`. In non-production builds, an external directory can be passed via `--sigma-dir`. Both inline (latency-sensitive, for prevention) and async (full rule set) paths run Sigma.

2. **Boss engine + YARA VFP** (`sunbird-detection-pipeline::BossEngine` + `YaraEngine`): Variant File Prevention — evaluates file-based variant signatures against executable content at file-open time. Loaded from `variant_rules_vfp.bundle.bin`.

3. **Boss engine + YARA VPP** (`ContentScannerEngine`): Variant Payload Prevention — evaluates memory-loaded content (mapped executable sections) against variant payload signatures. Loaded from `variant_rules_vpp.bundle.bin`. Up to 1,000 concurrent stateful machine instances.

4. **Stateful engine** (`StatefulEngine`): Cross-event correlation for injection detection. Maintains a pool of up to 1,000 concurrent finite state machine instances. Expired machines are swept every second. This engine is async-only (multi-event, too expensive for inline).

5. **Tagging engine**: Applies process-level tags (e.g., "browser", "office") that persist in the enrichment cache and influence rule matching.

### Inline vs. Async

**Inline path** (latency-sensitive): Receives events from the OS driver/ESF/ETW inline callbacks. Must return a decision (Allow/Deny) before the OS unblocks the operation. The stateful engine is skipped. Runs Sigma + Boss/YARA VFP + VPP if the action policy permits prevention. Uses `cpu_count / 4` parallel threads (minimum 4). Events older than 3 seconds are aged out and sent directly to async.

**Async path**: Receives all events (including post-inline events with their inline context). Runs the full engine registry (including stateful). If the inline handler already denied the event and async evaluates a higher-priority action (e.g., Delete > Prevent), the async action is executed additionally.

### Reputation Check

Before running any detection engine, each process-creation event is checked against an in-memory reputation table (synchronized from the server via `GET /api/v1/orgs/{org}/sensors/{id}/reputations`). Whitelist entries short-circuit inline processing with `ForceAllow`. Blacklist entries with `DetectPrevent` immediately deny inline; the detection is reported via the async path.

### What Happens on Local Detection

On detection:
1. `ActionExecutor` executes the policy decision: kill process (`KillProcess`), deny inline execution (`PreventInline`), quarantine file (via BitDefender on Windows), or delete file.
2. A `DetectionBatch` is sent to the Sensor Gateway via `POST /detections`.
3. If the endpoint UI is configured to show notifications, `SunbirdNotifier` displays a system notification.
4. The detection result is stored in SQLite for deduplication and pending-outcome replay.

Server-side CEP rules (in Apache Flink) receive the raw events and detections from the `raw-events` and `detections` Kafka topics and perform multi-machine correlation that the agent cannot do locally.

---

## Security and Identity Model

### Agent Identity

Each Sunbird installation has a stable `Id` — a 16-byte UUID-like identifier of type `StaticAgentId`. It is generated once on first boot and persisted in the runtime config store (registry on Windows, plist on macOS). The ID is stable across upgrades and reboots; it changes only if the agent is uninstalled and reinstalled.

The `OrganizationKey` (a human-readable string provided at install time via `--org-key`) identifies which Phoenix tenant the agent belongs to. The `OrganizationId` (a numeric `u64`) is assigned by the server during enrollment and persisted to the runtime config.

### Certificate Lifecycle

The enrollment and certificate protocol is a two-phase cryptographic flow:

**Phase 1 — Initial enrollment** (over plain HTTPS, non-mTLS gateway):
1. Agent generates an ephemeral Ed25519 keypair and a long-term P-256 ECDSA keypair (or retrieves the existing one from the keystore).
2. Agent sends `ChallengeInitiationRequest` wrapped in an `EphemeralSignedEnvelope` (signed with the ephemeral key) to `POST /auth/challenge/initiate`.
3. Server returns a proof-of-work challenge.
4. Agent solves the challenge (iterative nonce search to meet a difficulty mask) and signs the `ChallengeSolutionRequest` with the ephemeral key. Auth mode is included: `NoAuth`, `InstallationKey`, or `OneTimeToken`.
5. Server returns a signed long-term X.509 certificate (`not_after` typically years away) and assigns `org_id`.

**Phase 2 — Short-term cert issuance** (over mTLS using long-term cert):
1. Agent immediately requests a short-term certificate by repeating the challenge flow using the long-term cert for mTLS.
2. The short-term cert (`not_after` typically hours to days) is used for all subsequent mTLS connections.

**Renewal**:
- Long-term cert: renewed at `(not_before + (not_after - not_before) * (0.3 ± 0.05))` of its lifetime (jittered to avoid thundering herd). A background thread (`authentication_worker`) checks every minute.
- Short-term cert: renewed if missing or within 5 minutes of expiry.
- If the server rejects the long-term cert during short-term renewal (401/400), the agent fully re-enrolls.

After initial enrollment, the installation key and one-time token are deleted from the config store.

### Key Storage

| Platform | Long-term keypair storage | Certificate storage |
|---|---|---|
| Windows | Windows Certificate Store (CNG via `CertOpenStore`) | Windows Certificate Store |
| macOS | File-based (DER in `{data_dir}/keystore/`) | File-based |
| Linux | File-based (DER in `{data_dir}/keystore/`) | File-based |

### Database Encryption

The SQLite database is encrypted. The Data Encryption Key (DEK) is derived via HKDF using the agent ID as salt and a platform secret as input keying material:
- **Windows**: DPAPI (`CryptProtectData`) wraps the static key stored in the agent binary.
- **macOS / Linux**: A static key baked into the binary combined with the agent ID via HKDF.

The DEK is stored encrypted in the database itself. If the platform secret or DEK cannot be loaded, the agent panics.

### mTLS: Both Sides Verified

Yes. The server presents its certificate (signed by the Phoenix CA); the agent validates it. The agent presents its short-term certificate; the server validates it. Both validations are enforced by rustls with a configurable CA bundle.

---

## OS Integration Details

### Windows

| Crate | OS API | Privilege |
|---|---|---|
| `phoenix-sunbird` (ETW) | `ferrisetw` — `ITraceEventCallback`, kernel ETW providers | SYSTEM |
| `wfp-rs` | `FwpmFilterAdd0`, `FwpmLayerConnect*` — WFP sublayer for host isolation | SYSTEM |
| `bitdefender/*` | BitDefender SDK DLL callbacks via `BDC_*` C API | SYSTEM |
| `sunbird-keystore::windows` | `CertOpenStore`, `CryptProtectData`, CNG | SYSTEM |
| `sunbird-secret::dpapi` | `CryptProtectData` / `CryptUnprotectData` | SYSTEM |
| `windows-security-center-rs` | Windows Security Center registration | SYSTEM |
| `windows-notification` | WNS HTTP push channel | SYSTEM |

The agent is installed under `SOFTWARE\Cybereason\Sunbird` in the registry for configuration, and BitDefender self-protection (`SelfProtectionLocation`) prevents other processes from modifying the registry key or the install/data directories.

### macOS

| Crate | OS API | Privilege |
|---|---|---|
| `sunbird-endpoint-sec-macos` | `es_new_client`, `ES_EVENT_TYPE_NOTIFY_*` (Endpoint Security framework) | root + FDA + entitlement |
| `sunbird-keystore::macos` | File-based (future: macOS Keychain) | root |
| `sunbird-netmon-macos` | System network monitoring APIs | root |

Installed as an app bundle under `/Library/Cybereason/` with a LaunchDaemon plist at `/Library/LaunchDaemons/com.cybereason.sunbird.agent.plist`. The user-facing tray app is installed under `/Applications/Cybereason Sunbird.app` with a LaunchAgent.

The `Agent.entitlements` file includes `com.apple.developer.endpoint-security.client`. Requires a provisioning profile (`MACOS_SUNBIRD_AGENT_PROVISIONING_PROFILE` at build time) for distribution.

### Linux

| Crate | OS API | Privilege |
|---|---|---|
| `owlsm-rs` | External eBPF binary using `bpf_ktime_get_ns()`, LSM hooks, tracepoints | root / CAP_BPF |
| `sunbird-keystore::file` | File-based DER storage | root |

Installed via `.deb` or `.rpm` package under `/opt/cybereason/sunbird/` with a systemd unit at `/etc/systemd/system/sunbird-agent.service`. Data at `/var/opt/cybereason/sunbird/`, config at `/etc/opt/cybereason/sunbird/`. Directory permissions: 750.

---

## Packaging and Distribution

### Linux

**Package formats**: `.deb` and `.rpm` (both built from the same `build_package.sh` using `fpm`).

**Install layout**:
```
/opt/cybereason/sunbird/bin/agent       # main binary
/opt/cybereason/sunbird/owlsm/          # owLSM eBPF directory (bin/owlsm, rules, etc.)
/var/opt/cybereason/sunbird/            # data directory (SQLite DB, keystore)
/etc/opt/cybereason/sunbird/            # config directory
/etc/systemd/system/sunbird-agent.service
```

**Privilege**: Runs as root via the systemd service unit.

### macOS

**Package format**: macOS installer `.pkg` built with a custom `cli.py` toolchain and Apple `pkgbuild`/`productbuild`.

**Install layout**:
```
/Library/Cybereason/                    # agent install + data dir
/Library/LaunchDaemons/com.cybereason.sunbird.agent.plist
/Applications/Cybereason Sunbird.app/  # tray UI app
/Library/LaunchAgents/com.cybereason.sunbird.tray.plist
```

A custom macOS installer plugin (`SunbirdInstallerSection`) is compiled into the installer package to handle the `--org-key` configuration step during the installer UI flow.

**Privilege**: LaunchDaemon runs as root; LaunchAgent runs as the logged-in user (tray).

### Windows

**Package format**: MSI installer. The MSI writes `SOFTWARE\Cybereason\Sunbird` registry keys for configuration including `OrgKey`, `ProxyList`, `ProxyType`, enrollment token/installation key.

**Privilege**: SYSTEM service.

### Update Mechanism

Self-update is triggered by a command received via MQTT (the `upgrade` module). The CnC handler receives an upgrade command containing the target version, OS, and architecture. It calls `GET /api/v1/orgs/{org}/sensors/{id}/installer/download-url?version=X&os=Y&arch=Z&agent_type=sunbird&format=deb/rpm` to obtain a pre-signed download URL. The new installer is downloaded, verified by SHA-256 hash, and executed. On Linux, this invokes the `.deb`/`.rpm` installer; on macOS, the `.pkg` installer; on Windows, the MSI installer. The installer replaces the running binary and restarts the service.

---

## Connection to Phoenix Server

### Endpoints Used

| Purpose | Method | URL pattern |
|---|---|---|
| Enrollment challenge initiation | POST | `{gateway_url}/auth/challenge/initiate` |
| Enrollment challenge solution | POST | `{gateway_url}/auth/challenge/solve` |
| Agent info / heartbeat | PUT | `{mtls_url}/api/v1/orgs/{org}/agents/{sensor}` |
| Send event batch | POST | `{mtls_url}/api/v1/orgs/{org}/sensors/{sensor}/events` |
| Send detection batch | POST | `{mtls_url}/api/v1/orgs/{org}/sensors/{sensor}/detections` |
| Send action response | POST | `{mtls_url}/api/v1/orgs/{org}/sensors/{sensor}/actions/response` |
| Fetch sensor policy | GET | `{mtls_url}/api/v1/orgs/{org}/sensors/{sensor}/policy/{id}?version={n}` |
| Fetch reputation sync | GET | `{mtls_url}/api/v1/orgs/{org}/sensors/{sensor}/reputations[?since=ts]` |
| Get installer download URL | GET | `{mtls_url}/api/v1/orgs/{org}/sensors/{sensor}/installer/download-url` |
| Get fetch-log upload URL | GET | `{mtls_url}/api/v1/orgs/{org}/sensors/{sensor}/actions/{id}/fetch-log-upload-url` |
| Get script for execution | GET | `{mtls_url}/api/v1/orgs/{org}/sensors/{sensor}/scripts/{id}?version={n}` |
| Get script upload URL | GET | `{mtls_url}/api/v1/orgs/{org}/sensors/{sensor}/actions/{id}/script-upload-url` |
| Update WNS channel | POST | `{mtls_url}/api/v1/orgs/{org}/sensors/{sensor}/wns-channel` |
| Health check | GET | `{gateway_url}/health` |

### Protobuf Messages Sent (Agent → Server)

| Message type | Protobuf file | When sent |
|---|---|---|
| `EphemeralSignedEnvelope` wrapping `ChallengeInitiationRequest` | `auth.v2` | Enrollment / cert renewal step 1 |
| `EphemeralSignedEnvelope` wrapping `ChallengeSolutionRequest` | `auth.v2` | Enrollment / cert renewal step 2 |
| `FullAgentInfo` (contains `CoreAgentInfo` + host info + policy) | `agent.v1`, `ecs.agent.v1`, `ecs.host.v1` | Startup, policy change |
| `EventBatch` (array of `EventSpecific`) | `events.raw.v1` | Continuous event streaming |
| `DetectionBatch` | `detection.v1` | On local detection |
| `CommandControlResponse` | `commandcontrol.v1` | After executing a CnC command |

### Protobuf Messages Received (Server → Agent, via MQTT)

| Message type | Protobuf file | Purpose |
|---|---|---|
| `CommandControlResponse` (decoded from MQTT payload) | `commandcontrol.v1` | Commands: kill process, isolate host, collect file, run script, upgrade, fetch logs, policy update trigger, etc. |
| `PolicyResponse` | `sensor_gateway.v1` | Sensor policy delivered on policy fetch |
| `ReputationSyncResponse` | `sensor_gateway.v1` | Allow/block reputation table sync |

### Kafka Topics (via Server-side Processing)

| Topic | Schema | Source |
|---|---|---|
| `raw-events` | `SingleEvent` | Agent events forwarded by sensor-gateway |
| `detections` | `Detection` | Agent detections forwarded by sensor-gateway |
| `agents_v2` | `CoreAgentInfo` | Agent registration/heartbeat forwarded by sensor-gateway |
| `actions` | `Action` | Outbound commands from server to sensors |
| `actions-response` | `ActionResponse` | Agent action outcomes forwarded by sensor-gateway |

---

## Performance Characteristics

### Resource Footprint

The agent is designed for a low steady-state footprint. The `release` profile uses `lto = "thin"`, `codegen-units = 1`, and `panic = "abort"`. The `mimalloc` allocator is used (with override on Windows when the `mimalloc-override` feature is active) for lower per-allocation overhead.

SQLite uses `sqlcipher` (encrypted) which adds CPU cost on every DB read/write.

### Inline Detection Budget

The inline handler has a hard deadline of 3 seconds per event (`MAX_AGE_INLINE_DETECTION`). Events that exceed this age are passed directly to the async path. The inline thread pool uses `cpu_count / 4` threads (minimum 4) to allow parallel callback handling without starving the OS.

### Event Throughput

The perf directory (`perf/`) contains PowerShell-based infrastructure for running wet-run performance tests with InfluxDB metrics, Telegraf collection, and Grafana dashboards. Tests are run on VM fleets and measured in events/second throughput, CPU utilization, and memory growth. Specific throughput targets are not hardcoded in the agent source but are validated externally by the perf test suite.

### Stateful Engine Capacity

The `StatefulEngine` maintains at most 1,000 concurrent state machine instances (`STATEFUL_POOL_CAPACITY`). Expired machines are swept every second (`STATEFUL_EXPIRE_TICK`).

### Benchmarks

`criterion`-based micro-benchmarks exist in `crates/phoenix-sunbird/benches/`.

---

## Glossary

| Term | Definition |
|---|---|
| **Sunbird** | Codename for the Phoenix endpoint agent binary. Named after the Cybereason Sunbird product. |
| **Sensor** | Synonym for the Sunbird agent in server-side terminology (e.g., `sensor_id`, `sensor-gateway`). |
| **OwlSM** | "Owl Security Module" — an eBPF-based Linux security module that runs as a separate privileged binary alongside the agent. It uses Linux LSM hooks and BPF tracepoints to observe and optionally block process, file, and network events, delivering them to the agent via FlatBuffers on stdout. |
| **ETW** | Event Tracing for Windows — a Windows kernel-level tracing infrastructure used by Sunbird to capture process, file, network, and registry events with near-zero overhead. |
| **WFP** | Windows Filtering Platform — the Windows kernel network filtering API. Sunbird uses it (via `wfp-rs`) for host network isolation (blocking all network traffic) during containment actions. |
| **Endpoint Security framework** | Apple's kernel extension replacement API (`libEndpointSecurity`) that gives privileged user-space processes access to kernel security events (process, file, network). Used by Sunbird on macOS for event collection. |
| **Sigma rules** | An open, vendor-agnostic rule format for describing attack patterns against security event logs. Sunbird evaluates Sigma rules locally in real time against each event. |
| **Boss engine** | Cybereason's proprietary binary signature matching engine used for VFP and VPP. Paired with YARA for flexible pattern matching. |
| **VFP** | Variant File Prevention — prevents execution of known-bad file variants at the file-open (execute-access) level, checked inline. |
| **VPP** | Variant Payload Prevention — detects and prevents malicious in-memory payloads (shellcode, reflective DLL injection) by scanning mapped executable memory content. |
| **mTLS** | Mutual TLS — both the server and the agent present and validate X.509 certificates during the TLS handshake. Sunbird uses rustls for all mTLS. |
| **PCP rules** | Pre-compiled pattern rules (the `bep_rules.bin` bundle), part of Cybereason's proprietary detection content shipped inside the agent binary. |
| **CoreAgentInfo** | The Protobuf message describing the agent: ID, hostname, OS, version, group. Sent on startup and changes. Ends up in the `agents_v2` Kafka topic. |
| **EventBatch** | The Protobuf container for raw security telemetry events posted to sensor-gateway, which forwards them to the `raw-events` Kafka topic. |
| **DetectionBatch** | The Protobuf container for locally-detected alerts posted to sensor-gateway, which forwards them to the `detections` Kafka topic. |
| **CnC** | Command and Control — the subsystem in Sunbird that receives commands from the Phoenix server and dispatches them to the appropriate handler (action executor, policy updater, script runner, upgrade handler, etc.). |
| **DPAPI** | Windows Data Protection API — used on Windows to encrypt the SQLite database key using the SYSTEM account's machine-scope credentials, ensuring only the same machine account can decrypt it. |
| **AMSI** | Antimalware Scan Interface — a Windows API that allows scripts (PowerShell, VBScript, etc.) to be scanned by registered AV engines before execution. Sunbird wires AMSI into BitDefender for fileless protection. |
| **ATC** | Advanced Threat Control — BitDefender's behavior-based detection module for ransomware and document-exploit detection. |
| **WNS** | Windows Notification Service — Microsoft's push notification infrastructure used by Sunbird on Windows to receive commands without polling MQTT. |
