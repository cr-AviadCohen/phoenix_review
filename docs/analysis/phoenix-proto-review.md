# Phoenix Proto — Schema Review Findings

## Summary

The phoenix-proto repository is a well-structured, actively-maintained Protobuf schema library covering 236 files, 1,607 messages, 284 enums, and 61 gRPC services. Governance fundamentals are solid: buf v2 with the STANDARD lint rule set is enforced, every enum correctly uses an `UNSPECIFIED`/`UNKNOWN` zero value, and a custom `Timestamp` message is consistently used rather than raw `int64` across the majority of the codebase. The most significant finding is a **submodule-pin drift**: the Phoenix server repo (`Phoenix/`) has pinned its phoenix-proto git submodule one commit behind the current HEAD of phoenix-proto, meaning the `EventSpecific.rpc` field (field 37) and four new `EVENT_ACTION_RPC_*` enum values added in commit `446c777f` are not yet available to server-side consumers. Field documentation coverage is low at 37%, and 234 of 236 files lack package-level comments. Four isolated locations use raw `int64` or `uint64` nanosecond epoch integers rather than the project's own `Timestamp` message, introducing inconsistency and potential timezone confusion for consumers.

---

## Standards Compliance

| Area | Status | Notes |
|------|--------|-------|
| Field naming (snake_case) | ✅ | All fields across all 236 files use snake_case; no violations detected by automated check |
| Message naming (PascalCase) | ✅ | All 1,607 messages use PascalCase |
| Enum zero values (UNSPECIFIED) | ✅ | All 284 enums carry an `_UNSPECIFIED` or `_UNKNOWN` zero value; NullValue uses a documented lint suppression |
| Timestamp types (not raw int64) | ⚠️ | 4 locations use raw `int64` or `uint64 *_ns` epoch integers instead of `cybereason.common.v1.Timestamp` |
| No `required` fields (proto3) | ✅ | No `required` keyword appears in any proto3 file |
| Reserved for deleted fields | ✅ | 112 `reserved` statements found across the codebase; both field-number and field-name reservations are used |
| Breaking change detection | ⚠️ | `FILE` breaking mode is configured but no `against` baseline is specified in `buf.yaml`; the check will fail unless a baseline is supplied at CI call time |
| Field documentation | ⚠️ | 37.4% of fields have inline or preceding comments; 234 of 236 files have no package-level comment |

---

## Schema Design Quality

### Strengths

- **Consistent custom Timestamp**: A project-specific `cybereason.common.v1.Timestamp` message (equivalent to `google.protobuf.Timestamp`) is used uniformly. The comment in `px_notation.proto` correctly explains the rationale: prost does not natively support `google.protobuf.Timestamp`. This is documented and intentional.
- **Universal UNSPECIFIED enum zero values**: All 284 enums follow the proto3 best practice of placing an `_UNSPECIFIED` (or equivalent) value at position 0. This prevents silent misinterpretation of default-initialised fields.
- **Field deletion hygiene**: `reserved` statements protect deleted field numbers with both numeric and name forms (e.g. `reserved 6; reserved "plugin_version";`) across platform_store, integration_manager, xdr_integration_store, case_service, and others. This is correct and important practice.
- **Opaque message pairing**: The `OpaqueEventBatch` / `OpaqueSingleEvent` design in `px_event_batch.proto` and `px_event.proto` is a clean pattern that lets the sensor gateway forward events without deserialising their content, preserving wire-format compatibility.
- **`optional` used explicitly**: proto3 `optional` fields are used extensively and correctly to distinguish missing from zero-value, which is the right practice for a sensor protocol where field absence is semantically significant.
- **Deprecation expressed at proto level**: Deprecated fields use `[deprecated = true]` option (e.g. `EVENT_ACTION_RPC_CALL`), deprecated packages carry file-level comments (e.g. `cybereason.phoenix.auth.v1`), and migration guidance is provided inline.
- **`oneof` used where appropriate**: The `ChallengeRequest.entity_id` oneof correctly expresses that only one of `tenant_id` or `organization_id` can be present, and `px_notation.proto` uses `oneof kind` for the dynamic Value type.
- **Multi-tenancy enforced at schema level**: The `org_id`/`organization_id` field is mandatory (non-optional, non-zero) on all sensor-facing event and detection messages. `AssetInstance` includes an explicit comment: "Multi-tenancy: `org_id` MUST be set and derived from authenticated scope. Never accept from untrusted input."
- **Buf BSR module registered**: The module is published as `com.cybereason/cybereason/phoenix`, enabling version-locked dependency management for consumers.

### Concerns

**Medium — Raw epoch integer timestamps (inconsistent timestamp type)**

Four locations use raw numeric epoch fields instead of `cybereason.common.v1.Timestamp`:

- `proto/cybereason/phoenix/platform_store/v1/px_policy_svc.proto:117` — `optional int64 updated_at = 5` (comment says "Unix epoch milliseconds")
- `proto/cybereason/phoenix/platform_store/v1/px_policy_data.proto:999` — `optional int64 created_at = 6` (comment says "Unix epoch milliseconds")
- `proto/cybereason/phoenix/sensor_gateway/v1/px_sensor_policy.proto:72` — `optional int64 updated_at = 5`
- `proto/cybereason/phoenix/commandcontrol/v1/px_command_kill_process.proto:17` — `optional int64 creation_time = 2`
- `proto/cybereason/phoenix/xdr_integration_store/v1/px_xdr_integration_store_svc.proto:224,356,371,381,401` — `optional uint64 *_ns` fields (nanoseconds since Unix epoch)
- `proto/cybereason/phoenix/integration_manager/v2/px_integration_manager_svc.proto:429,480` — `optional uint64 updated_at_ns`
- `proto/cybereason/phoenix/integration_proxy/v2/px_integration_proxy_svc.proto:134` — `optional uint64 created_at_ns`
- `proto/cybereason/phoenix/vault/v2/px_vault_svc.proto:152,221,236` — `optional uint64 *_ns` fields

The `int64` variants in platform_store do not document their epoch unit (seconds vs milliseconds), while the `uint64 *_ns` fields are at least named consistently. These should be migrated to `cybereason.common.v1.Timestamp` for consistency and schema clarity. This is a wire-breaking change if done, so migration must use new field numbers with reserved statements on the old ones.

**Medium — Large field number gaps without corresponding reserved ranges**

Several messages have field number gaps of 10+ that are not covered by `reserved` statements:

- `DetectionEvent` (`px_detection_event_svc.proto`): jumps from 43 to 100 — no `reserved 44 to 99`
- `AssetInstance` (`px_asset_instance.proto`): jumps from 32 to 200 — no `reserved 33 to 199`
- `Policy` (`px_policy_data.proto`): gaps at (5→33), (39→100), (115→200) — ranges not reserved
- `DocumentProtectionEngineStatus` (`px_sensor_status.proto`): jumps from 1 to 21 — no `reserved 2 to 20`

The `Sensor` message in `px_sensor_data.proto` is a positive counterexample: it has explicit `reserved 12 to 16`, `reserved 25 to 62`, and `reserved 65 to 71` with explanatory comments for each removed block.

**Low — Deprecated entire package without service-level version signal**

`cybereason.phoenix.auth.v1` (files: `px_sensor.proto`, `px_sensor_onboarding_svc.proto`) is marked deprecated at the package level with a comment directing consumers to `auth.v2`. However, the `SensorOnboardingService` gRPC service has no `deprecated` option set on the service definition itself, which means buf lint will not flag consumers importing it. The package deprecation comment is invisible to code-generation tools.

**Low — `org_id` type inconsistency (`int64` vs `uint64`)**

`CoreAgentInfo.group_id` (`px_agent.proto:57`) is typed `int64` while virtually all other tenant/org IDs in the schema use `uint64`. The field also carries a `FIXME` comment noting it is duplicated. The sign difference is benign on the wire but can cause Rust type-checking friction (signed vs unsigned integer).

**Low — `TODO` / `FIXME` comments in production proto files**

Several production files contain inline `TODO` and `FIXME` markers indicating unresolved design decisions:
- `px_event.proto` (`EventCommon.integration` field 8): `<TODO>` block noting the integration field is not correctly populated
- `px_agent.proto` field 5: `FIXME: we have group_id twice`
- `px_detection_event_svc.proto` `DetectionEvent.event_data` field 100: the jump to field 100 suggests a placeholder design rather than a deliberate reserved range

---

## Breaking Change Risk

### Assessment

**Medium**

### Findings

- **Buf breaking mode configured correctly as `FILE`**: This is the most comprehensive breaking-change detection mode (catches field renames, type changes, field number reuse, and removal of services/RPCs). However, the `buf.yaml` does not specify an `against` baseline (no `git#branch=main` or BSR reference). This means CI must supply `--against` at invocation time. If it is omitted, the breaking-change check is silently skipped.

- **`reserved` statements are used correctly**: Deleted fields and field names are reserved in platform_store, integration_manager, xdr_integration_store, case_service, and asset_store. The practice is inconsistent across the full codebase (some files reserve, others leave gaps unexplained).

- **Deprecated fields kept on the wire**: `EVENT_ACTION_RPC_CALL = 37 [deprecated = true]`, `pubkey_material = 2 [deprecated = true]`, and several `DeviceSource source` fields in asset_store retain their original field numbers, preserving wire compatibility for deployed agents that may still send these values. This is correct.

- **`RawEvent` field ordering anomaly**: In `px_raw_event.proto`, fields 219 (`tags`) and 220 (`labels`) appear physically after field 231 in the source file. Field numbers are not required to appear in order in proto3, but the out-of-sequence placement (fields 219–220 inserted after 231 in the source) increases the risk of a reviewer accidentally assigning a conflicting number during a future addition.

- **No field number reuse detected in git history**: The one commit of divergence (`446c777f`) adds field 37 in `EventSpecific` (rpc) and enum values 38–41 in `EventAction`. These are additive changes that do not break deployed agents. The old `EVENT_ACTION_RPC_CALL = 37` enum value retained with `[deprecated = true]` is correct wire-safety practice.

---

## Cross-Component Consistency

### Methodology

Three repositories were examined:
1. **phoenix-proto** (canonical source): `projects/phoenix-proto/proto/`
2. **Phoenix server** (`projects/Phoenix/`): uses phoenix-proto as a git submodule at `modules/proto/`
3. **phoenix-agent** (`projects/phoenix-agent/`): no embedded proto copies found; uses phoenix-proto via the BSR or a separate checkout

The Phoenix server git submodule configuration was read from `projects/Phoenix/.gitmodules` and the pinned commit extracted from `git submodule status`.

### Findings

**The Phoenix server submodule is pinned one commit behind the current HEAD of phoenix-proto.**

- phoenix-proto HEAD: `446c777f` ("Add RPC Protos")
- Phoenix submodule pin: `cd67b45e` (the commit immediately before "Add RPC Protos")

Files that differ between the pinned version and HEAD:

```
proto/cybereason/ecs/event/v1/px_event.proto
proto/cybereason/ecs/rpc/msrpc/v1/px_msrpc.proto
proto/cybereason/ecs/rpc/v1/px_rpc.proto
```

Content of the diff (additive only — no breaking changes):

1. `EventSpecific` gains a new `optional cybereason.ecs.rpc.v1.Rpc rpc = 37` field
2. `EventAction` gains four new enum values: `EVENT_ACTION_RPC_CLIENT_INITIATED = 38`, `EVENT_ACTION_RPC_SERVER_RECEIVED = 39`, `EVENT_ACTION_RPC_SERVER_COMPLETED = 40`, `EVENT_ACTION_RPC_CLIENT_RETURNED = 41`
3. `EVENT_ACTION_RPC_CALL = 37` is marked `[deprecated = true]` in the newer version; in the pinned version it is active
4. New files `px_msrpc.proto` and `px_rpc.proto` are not present in the Phoenix submodule at all

The phoenix-agent repository has no embedded proto copies and therefore has no divergence issue. Its integration mechanism was not inspectable from the checkout state.

### Risk

The drift is **additive only** and therefore does not introduce a wire-incompatibility between currently deployed agents and the current server. However:

- Any server-side code that attempts to use `EventAction.EVENT_ACTION_RPC_CLIENT_INITIATED` or `EventSpecific.rpc` must first update the submodule — otherwise the field and enum values will be missing from generated code, and the compiler will reject it.
- The Phoenix `ensure-proto.sh` script detects hash drift and regenerates, but only for TypeScript/portal. Go code (golang/pbgen) and Rust code (rust/pbgen) require manual submodule update and code regeneration.
- If the submodule is not bumped before a sensor release that emits `EVENT_ACTION_RPC_*` events, server-side consumers will silently ignore those events (proto3 unknown-field behaviour). This is safe but means RPC telemetry would be silently dropped until the server is updated.

**Recommended action**: Bump the Phoenix submodule pin from `cd67b45e` to `446c777f` and run `golang/pbgen/gen.sh` and the Rust `buf generate` equivalents.

---

## Documentation Quality

Documentation quality is mixed, with a clear bifurcation between core/ECS layer files and platform-service layer files.

**Well-documented files:**
- `px_event.proto` (1,455 lines) is exhaustively commented: every field has a purpose comment, producer/consumer behaviour guidance, ECS mapping, and ClickHouse column references.
- `px_asset_instance.proto` includes a full package-level design rationale comment explaining the schema's five design principles, Kafka topic encoding, and multi-tenancy requirements.
- `px_agent.proto`, `px_notation.proto`, and the ECS layer files (host, process, file, network) carry meaningful field-level comments.

**Under-documented files:**
- 234 of 236 proto files have no package-level comment. Most platform-service files (`px_batch_action_data.proto`, `px_blocked_hash_data.proto`, `px_usage_report_data.proto`, `px_saved_query_data.proto`, etc.) have no file or message-level description.
- Field documentation coverage: 37.4% of 4,445 field definitions have at least one comment. The majority of uncommented fields are in the `phoenix/platform_store/`, `phoenix/commandcontrol/`, and `phoenix/organization_store/` namespaces.
- `pbdocgen.yaml` and `pbdocgen.env.sh` exist in the repo root, and a `docs/` directory is present, indicating auto-generated documentation tooling is set up. The quality of that output is entirely dependent on the comment coverage gaps above.
- Several proto files contain `TODO` and `FIXME` comments that are semantically confusing for downstream consumers: `EventCommon.integration` warns "not really a full-fledged integration" and `EventSeverityLevel.severity` contains a note that semantics are not clearly defined.

---

## Top Recommendations

**1. Bump the Phoenix submodule pin to phoenix-proto HEAD (Medium urgency)**

File: `projects/Phoenix/.gitmodules` and `git submodule update` command.

The Phoenix server's `modules/proto` submodule is pinned at `cd67b45e`, one commit behind HEAD (`446c777f`). The drift is additive, but any work involving RPC event telemetry will block until this is resolved. Run `git -C projects/Phoenix submodule update --remote modules/proto` and regenerate Go and Rust bindings.

**2. Add an `against` baseline to `buf.yaml` breaking-change config (High governance priority)**

File: `projects/phoenix-proto/buf.yaml`

The current configuration:
```yaml
breaking:
  use:
    - FILE
```
has no `against` field. Without it, `buf breaking` is a no-op unless `--against` is supplied at CI call time. Add:
```yaml
breaking:
  use:
    - FILE
  against:
    - '.git#branch=main'
```
or the equivalent BSR reference. This is critical for a wire protocol shared with deployed endpoint sensors.

**3. Migrate raw epoch integer timestamp fields to `cybereason.common.v1.Timestamp` (Medium)**

Files:
- `proto/cybereason/phoenix/platform_store/v1/px_policy_svc.proto:117` (`updated_at`)
- `proto/cybereason/phoenix/platform_store/v1/px_policy_data.proto:999` (`created_at`)
- `proto/cybereason/phoenix/sensor_gateway/v1/px_sensor_policy.proto:72` (`updated_at`)
- `proto/cybereason/phoenix/commandcontrol/v1/px_command_kill_process.proto:17` (`creation_time`)
- Multiple `uint64 *_ns` fields in `xdr_integration_store`, `integration_manager`, `integration_proxy`, and `vault` v2 files

Add new fields using `cybereason.common.v1.Timestamp`, reserve old field numbers, and deprecate old fields. This is a migration, not a same-number rename.

**4. Add `reserved` ranges for large field number gaps (Medium)**

Files:
- `proto/cybereason/phoenix/detection_store/v1/px_detection_event_svc.proto` — `DetectionEvent`: add `reserved 44 to 99;`
- `proto/cybereason/phoenix/asset/v1/px_asset_instance.proto` — `AssetInstance`: add `reserved 33 to 199;`
- `proto/cybereason/phoenix/platform_store/v1/px_policy_data.proto` — `Policy`: add reserved statements for all three gaps
- `proto/cybereason/ecs/agent/sensor/v1/px_sensor_status.proto` — `DocumentProtectionEngineStatus`: add `reserved 2 to 20;`

Without these reservations, a developer editing these messages could accidentally assign a field number already used by a previously deleted field, causing silent data corruption for deployed agents that still send the old field.

**5. Add `deprecated` service option to `SensorOnboardingService` (Low)**

File: `proto/cybereason/phoenix/auth/v1/px_sensor_onboarding_svc.proto`

The package-level deprecation comment is not machine-readable. Add:
```protobuf
service SensorOnboardingService {
  option deprecated = true;
  ...
}
```
so that buf lint and generated code stubs surface the deprecation to new consumers.

**6. Expand package-level and field-level documentation (Low)**

The 37% field documentation coverage and near-zero package-level comment coverage are technical debt. Priority targets should be the high-traffic service files: `px_batch_action_data.proto`, `px_asset_data.proto`, `px_sensor_data.proto`, `px_policy_data.proto`, and `px_detection_data.proto`. The existing `pbdocgen` tooling means documentation improvements will immediately improve the generated API reference.

**7. Resolve `CoreAgentInfo.group_id` type inconsistency (Low)**

File: `proto/cybereason/ecs/agent/v1/px_agent.proto:57`

`group_id` is typed `int64` while all other org/group IDs in the schema use `uint64`. The field also carries a FIXME noting it appears in two places. Standardise to `uint64`, using a new field number with a reserved statement on field 5.
