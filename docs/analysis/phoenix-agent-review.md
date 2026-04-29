# Phoenix Agent (Sunbird) — Code Review Findings

## Summary

The Sunbird agent is a mature, well-structured Rust codebase with strong architectural foundations: the build-time `panic = "abort"` profile, a structured enrollment/authentication protocol using ephemeral Ed25519 signing and mTLS short-term certificates, and a well-layered detection pipeline. However, the workspace is missing the mandatory `unsafe_code = "forbid"` lint, there are numerous `panic!` and `.unwrap()` / `.expect()` calls in initialization-critical production paths, and a compile-time environment variable (`SUNBIRD_FORCE_DISABLE_MQTTS`) can silently disable mTLS at build time. The Linux/macOS keystore falls back to plaintext filesystem key storage (file-based, no OS keychain integration), and the static HKDF key is embedded in the binary in an obfuscated-but-recoverable form. These combine to present meaningful security risk on non-Windows endpoints. Test coverage exists in the auth and keystore crates but is gated behind a CI environment variable, and the secret/DPAPI crates have good unit tests.

---

## Standards Compliance

| Area | Status | Notes |
|------|--------|-------|
| Workspace `unsafe_code = "forbid"` | ❌ | Not set in workspace `Cargo.toml`. `[workspace.lints.rust]` block is absent. 40+ `unsafe` blocks exist in `bitdefender-cst` and `sunbird-detection-pipeline`, with no workspace-level guardrail. |
| Release `panic = "abort"` | ✅ | Correctly set in `[profile.release]` in the workspace `Cargo.toml`. |
| No `unwrap()`/`expect()` in prod code | ⚠️ | Multiple `.unwrap()` and `.expect()` calls in `app.rs` initialization paths, `challenge.rs`, `keystore/file/mod.rs`, and `sigma-rust/src/basevalue.rs`. See detailed list below. |
| Error handling (thiserror/eyre) | ⚠️ | `thiserror` is used correctly in library crates. However, the main entry points (`sunbird-agent-linux/src/bin/agent.rs`) do not use `color-eyre`; they use `unwrap_or_else` with `std::process::exit`. Partially compliant. |
| Structured logging (tracing) | ✅ | `tracing` with `#[instrument]` is consistently used throughout. `org_id` is logged at key boundaries. |
| Security-critical crate test coverage | ⚠️ | `sunbird-auth` has integration tests but they are gated behind `if std::env::var("CI").is_err() { return; }`, meaning they do not run on developer machines. `sunbird-keystore` has file-keystore tests. `sunbird-secret` (DPAPI and file providers) has unit tests. `sunbird-identity` has basic unit tests. No fuzzing or property-based tests exist anywhere. |

---

## Core Agent — Findings

### Strengths

- The `initialize()` function in `/Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/phoenix-sunbird/src/app.rs` installs a custom panic hook (line 141–143) that routes panics through `tracing::error!`, ensuring panics are captured in structured logs before `abort` occurs.
- The network initialization is isolated in `spawn_network_init()` running on a dedicated thread with exponential backoff (2s → 300s max) for both discovery and enrollment failures. The agent continues running locally even if the backend is unreachable.
- Shutdown sequencing is well-designed with `ShutdownManager` and per-component `ShutdownHandle` drop guards; a 20-second timeout prevents indefinite hang.
- The mTLS client configuration (`ClientConfig`) is created at enrollment and threaded explicitly through the CnC handler — no global mutable TLS state.
- Directory permissions on Unix are enforced at startup as `0o750` (line 159), providing defense-in-depth against local privilege escalation via file manipulation.

### Concerns

- **[High] Multiple `panic!` calls during startup in production code** (`app.rs` lines 157–161, 167–168, 268, 284). For example, failure to create the agent data directory panics the process. With `panic = "abort"` in release, these are instant process deaths with no recovery. On a locked-down system (e.g., write-protected filesystem), the agent silently dies.

  ```
  /Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/phoenix-sunbird/src/app.rs:157–161
  /Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/phoenix-sunbird/src/app.rs:268
  /Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/phoenix-sunbird/src/app.rs:284
  ```

- **[High] Compile-time mTLS bypass**: The env var `SUNBIRD_FORCE_DISABLE_MQTTS` checked at `option_env!()` (compile-time, not runtime) can disable MQTT mTLS. This disables the MQTT transport security layer silently. A build produced with this flag set will never use mTLS for C2 communication, with no runtime indication beyond a single `info!` log line. This must be enforced to never appear in production release builds.

  ```
  /Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/phoenix-sunbird/src/app.rs:1071–1076
  ```

- **[Medium] Hardcoded `HostType::Desktop`**: `build_agent_identity()` line 1169 has `//FIXME: Hardcoded Desktop`. All enrolled sensors will report as Desktop regardless of host type.

- **[Medium] Unbounded channels throughout**: The detection pipeline (`lib.rs:401–402`), high/low priority client senders (`client.rs:97–98`), BitDefender command/event channels (`bitdefender/mod.rs:379–383`), and quarantine channel (`app.rs:325`) are all `unbounded()`. If the backend is slow or the network is unavailable, these channels will grow without bound, consuming heap memory proportional to the event rate. Under high telemetry load this is a denial-of-service risk.

- **[Low] `SigmaEngine::from_embedded()` and YARA engines use `.expect()` at startup** (`app.rs:881, 907, 933`). A corrupted embedded bundle will crash the agent rather than degrade gracefully.

- **[Low] `SUNBIRD_DISABLE_SLOW_BUNDLES` compile-time flag**: If accidentally left enabled in a build, VFP and VPP detection bundles are silently skipped. There is a code comment warning about this but no build-time assertion.

---

## Security-Critical Crates — Findings

### sunbird-auth (mTLS)

**Strengths:**
- Enrollment uses ephemeral Ed25519 key pairs (generated per enrollment session with `OsRng`) to sign the challenge initiation request, limiting replay risk.
- The challenge uses a proof-of-work with BLAKE3 hashing, making replay attacks computationally expensive.
- Long-term and short-term certificates are maintained separately; the short-term certificate (used for mTLS connections) is renewed within a 5-minute leeway window, limiting exposure from a compromised short-term cert.
- The server-facing mTLS uses a `rustls::ClientConfig` with system root certificates, providing full certificate chain validation against the OS trust store.
- `#[instrument(skip_all)]` is consistently applied to enrollment functions, preventing sensitive parameters from appearing in logs.
- The auth tests in `test.rs` are comprehensive integration tests covering the full enrollment flow against a mock server.

**Concerns:**
- **[High] No server-side signature verification on the challenge response**: The `verify_and_solve_challenge()` function (enrollment.rs:304–337) decodes the challenge description and solves the proof-of-work, but **does not verify a server signature on the challenge token itself**. An attacker with network access (MITM position despite TLS) could serve a crafted challenge. The mTLS layer for the non-enrollment endpoint provides server authentication, but the initial enrollment (unenrolled state) uses plain TLS against the system root store, not a pinned server certificate. This means the chain of trust at initial enrollment relies entirely on the system CA store.
- **[Medium] `AuthMode::NoAuth` is a supported production mode**: `EnrollmentSecurity::Unsecured` maps to `AuthMode::NoAuth`, which allows sensor enrollment without any proof of authorization. This is labelled as intentional but has no additional guardrails (e.g., IP allowlist, rate limiting visible to the agent) to prevent unauthorized sensor registration at the backend level from the agent side.
- **[Low] `expect()` in `verifying_key_to_jwk()`** (enrollment.rs:541–542): Panics if the P-256 encoded point has no x or y coordinate. This should be structurally impossible for a valid key but there is no comment justifying the `.expect()`.

**Risk:** Medium

---

### sunbird-keystore (Key Storage)

**Strengths:**
- Windows implementation uses Windows CNG (NCrypt) through `WinKeystoreProvider`, which stores the private key in the Windows software key storage provider. Private keys are not directly exportable from the process.
- macOS implementation is in `macos/mod.rs` and `macos/tls.rs` (see directory structure).
- The file-based implementation (`file/mod.rs`) checks for existing key files before overwriting, preventing accidental key rotation.

**Concerns:**
- **[High] Linux and macOS production fallback uses plaintext private key files**: The `load_keystore()` function in `lib.rs:51–61` routes non-Windows platforms to `FileBasedKeystore`. The private key is written as a raw DER file at `<data_dir>/keystore/<name>/private-key.der` with **no encryption**. The file permissions depend on the containing directory (set to `0o750` at startup), not on the file itself. There is no per-file permission enforcement in `FileBasedKeystore::create_keypair()`. A local attacker with read access to the data directory (e.g., another process running as the same user) can extract the long-term mTLS private key.

  ```
  /Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/sunbird-keystore/src/lib.rs:58–61
  /Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/sunbird-keystore/src/file/mod.rs:39–43
  ```

- **[Medium] `FIXME: Switch to real implementation for MacOS and Linux`**: The comment on `lib.rs:58` explicitly acknowledges the file-based keystore is a stopgap. The production deployment for Linux/macOS endpoints is using an incomplete security implementation.
- **[Medium] `rustls_native_certs::load_native_certs().unwrap()`** (`file/mod.rs:133`): Panics if the system cert store cannot be read. This would crash the TLS setup during enrollment.

**Risk:** High (for Linux/macOS deployments)

---

### sunbird-identity (Device Identity)

**Strengths:**
- Agent IDs are 128-bit random values generated using `rand::rng().random()` (which uses the OS CSPRNG), with the top nibble used as a type discriminator.
- IDs are persisted to the runtime config (`keys::AgentId`) and reloaded across restarts, providing continuity.
- If the stored ID is corrupt or invalid, the error is logged and a new ID is generated rather than crashing.
- Crockford Base32 encoding is used for human-readable display, preventing confusion with other ID formats.

**Concerns:**
- **[Medium] Agent ID can be spoofed by a local admin/root**: The ID is stored in the runtime config (a simple file on Linux). A local administrator can modify this file, causing the agent to register under a different identity and masquerade as a different sensor. There is no cryptographic binding between the agent ID and the mTLS key pair — the server's trust comes from the enrollment certificate, not the agent ID field.
- **[Low] No validation of `IdType` at load time**: `build_agent_identity()` in `app.rs` calls `sunbird_identity::Id::try_from(id.as_str())` which validates the Crockford encoding and length but does not check `id_type()`, so an ephemeral ID persisted accidentally would be silently accepted as the agent's static ID.

**Risk:** Medium

---

### sunbird-secret (Secret Management)

**Strengths:**
- Windows uses DPAPI with `CRYPTPROTECT_LOCAL_MACHINE` scope, wrapping an HKDF-derived ChaCha20-Poly1305 encryption layer on top. This provides two layers of protection: platform credential binding and symmetric encryption.
- The `PlatformKey` provides a derived encryption key (DEK) used for database encryption, so the database is encrypted at rest.
- Linux/macOS uses HKDF(STATIC_KEY, agent_id) + ChaCha20-Poly1305, providing encryption-at-rest for the database even without a platform keychain.
- The static key is XOR-obfuscated with mask `0xBD` (`0xFF ^ 0x42`) in the binary — the code explicitly documents this is NOT a cryptographic secret but provides defense-in-depth.
- `unsafe` blocks in `dpapi.rs` are minimal, well-scoped, and each carries a `// SAFETY:` comment explaining the invariant.
- `FileSecretProvider::write_file()` sets `0o600` permissions on Linux/macOS, providing strong file-level isolation.

**Concerns:**
- **[Medium] Static HKDF key is hardcoded in the binary**: `LONG_TERM_STATIC_KEY_OBFUSCATED` in `static_key/long_term.rs:16–19` is a 32-byte XOR-obfuscated key. Anyone who can extract the binary can recover the key with a single XOR operation, then combined with the agent ID (stored in plaintext), can decrypt the platform parameters file on Linux/macOS endpoints. The code acknowledges this ("This is NOT a cryptographic secret on its own"), but the defense-in-depth argument weakens significantly for Linux where there is no DPAPI and no OS keychain integration.
- **[Low] The `AlreadyExists` guard in `create()` is a check-then-act pattern**: A TOCTOU race exists between `read_encrypted_blob()?.is_some()` check and the subsequent write. On Linux filesystems without atomic file operations, concurrent agent instances starting simultaneously could both pass the check. In practice the agent is a singleton service but this is worth noting.

**Risk:** Medium

---

## Platform Collector Safety — Linux

The Linux agent (`sunbird-agent-linux`) is a thin wrapper that delegates collection to **owLSM** (OwlSM, the kernel-level security monitor), which is a separate binary spawned as a subprocess. The `owlsm-rs` crate provides the Rust wrapper.

### Strengths
- owLSM is spawned as a separate process, isolating kernel-interface code from the main agent process. A crash in owLSM does not kill the agent.
- Event forwarding uses a bounded channel of 256 entries (`owlsm-rs/src/pipeline.rs:87`), providing backpressure from the owLSM subprocess into the detection pipeline.
- owLSM pipeline restarts are handled gracefully on policy change (app.rs:454–500): the old handle is dropped before spawning the new instance, preventing duplicate event emission.
- Failure to start owLSM (e.g., binary not found) is handled gracefully: `spawn_owlsm_pipeline` returns `Ok(None)` and the agent continues in degraded-collection mode with a `warn!` log.
- The `stop_tx` channel is used for graceful shutdown signaling, and `OwlsmPipelineJoin::drop()` waits for the forwarder thread to exit.

### Concerns
- **[Medium] No privilege-dropping logic visible in the agent layer**: The Linux agent binary itself does not call `setuid`/`setgid` or use capabilities APIs to drop privileges after initializing. Whether the owLSM binary drops privileges after loading its kernel module is opaque from this review (it is a subprocess, not part of this codebase). The agent runs with whatever privileges the service manager grants it.
- **[Medium] No error recovery if owLSM exits unexpectedly mid-run**: If the owLSM process crashes after initial startup, the channel sender will close and the forwarder thread will exit silently. There is no watchdog or restart mechanism for owLSM outside of a policy change event. Linux endpoint monitoring would be silently degraded until the agent itself restarts.
- **[Low] The agent data directory permission setup** (`0o750`) is done before the startup sequence, but there is no verification that existing files within the data directory have the correct permissions. On a system that was previously running a misconfigured agent, old files with permissive permissions would not be corrected.

### Risk Assessment
**Medium** — The subprocess isolation is a good design, but the lack of visible privilege-dropping and absent owLSM restart watchdog are operational concerns.

---

## Detection Pipeline Safety

### Strengths
- The `verify_and_solve_challenge()` function validates `challenge_version != 1` before processing (enrollment.rs:320), providing a version-guard against future protocol additions.
- Sigma rule loading from filesystem (`loader.rs`) handles individual rule parse failures gracefully: failed rules are counted and warned but do not abort rule loading (`warn!` not `error!`/`panic!`).
- Rule bundles are loaded from PCP encrypted bundles (`load_embedded_bundle` calls `LoadedPCPBundle::load_encrypted(raw)`), providing integrity protection for the embedded detection rules.
- The stateful detection pool has a hard capacity cap of 1,000 instances (`STATEFUL_POOL_CAPACITY = 1000`), preventing unbounded memory growth from process injection chain tracking.
- There is no `eval` or `exec` of rule content — rules are parsed into typed IR (`CompiledRule`) and evaluated against structured events via the sigma-rust engine.
- The inline detection pipeline is multi-threaded at `max(cpus/4, 4)` threads, balancing throughput with CPU resource limits.
- The async pipeline uses `try_send` for event delivery, dropping events with a `tracing::debug!` rather than blocking or panicking.

### Concerns
- **[Medium] Unbounded async/inline pipeline channels**: Both `async_sender` and `inline_sender` are `crossbeam::channel::unbounded()` (lib.rs:401–402). Under heavy endpoint activity (e.g., a process generating thousands of file events per second), these queues can grow without limit. The `pipeline_metrics` module samples queue depths every 30 seconds, enabling alerting, but there is no automatic back-pressure or event dropping at the channel level.
- **[Medium] `std::thread::sleep()` in test paths of stateful engine** (`engine/stateful/engine.rs:241, 557, 573, 577`): While these appear to be in test-only paths, they are in non-`#[cfg(test)]` blocks. Should be audited to confirm they do not appear in the production event processing hot path.
- **[Low] Case-insensitive path exclusions use `eq_ignore_ascii_case()`** (lib.rs:1258, 1265): On Linux with a case-sensitive filesystem, an attacker could bypass path exclusions by changing the case of a process name (e.g., `MalWaRe.Exe` vs `malware.exe` on a case-sensitive Linux FS). This is currently noted as a TODO comment (lib.rs:1244).

### Risk Assessment
**Medium** — The detection logic is well-designed, but the unbounded channels are a resource management concern under adversarial high-event-rate conditions.

---

## Resource & Performance Observations

- **Unbounded channels are the primary resource concern**: Detection pipeline async/inline channels (lib.rs:401–402), Phoenix client high/low priority queues (`client.rs:97–98`), and BitDefender command channels (`bitdefender/mod.rs:379–383`) are all `unbounded()`. Under sustained high load or backend unavailability, heap memory usage can grow proportionally to the event backlog.

- **`reqwest::blocking::Client`** is used throughout `sensor-gateway-client/src/client.rs`, which is correct for non-async contexts (the agent uses OS threads, not Tokio). No blocking inside async context concern applies here.

- **Policy callbacks are synchronous and fire on the policy controller's thread**: `add_on_change()` callbacks (app.rs:386–680) send commands to BitDefender synchronously via `crossbeam::channel::Sender::send()`. If any channel is full or disconnected, the error is logged and ignored, which is acceptable.

- **Backfill cache threads** (`spawn_named("backfill_cache")` and `spawn_named("backfill_cache_linux")`) enumerate running processes at startup and may be slow on systems with many processes. These run detached and do not block agent startup.

- **`metrics` flushing interval is 30 seconds** with `sunbird_metrics::spawn_file_writer()`, which uses `std::thread::sleep(interval)` in a dedicated thread. This is appropriate for a background thread.

---

## Test Coverage

| Crate | Test Type | Coverage Notes |
|-------|-----------|----------------|
| `sunbird-auth` | Integration tests with mock server (`test.rs`) | Full enrollment flow, but gated behind `CI` env var — skipped on developer machines |
| `sunbird-keystore` | Unit tests in `file/mod.rs` | Covers create/save/get certificate round-trip; no Windows CNG tests in this repository |
| `sunbird-secret` | Unit tests in `platform/file.rs` and `platform/dpapi.rs` | Round-trip, double-create error, delete tested |
| `sunbird-identity` | Unit tests in `identity.rs` | ID generation, type discrimination, byte conversion |
| `sunbird-detection-pipeline` | Unit tests in `lib.rs` | Extensive: resolve_decision variants, path exclusions, inline tagging |
| `sigma-rust` | Unit and benchmark tests | `field.rs` and `basevalue.rs` have extensive `#[test]` blocks with `unwrap()` (test-appropriate) |
| `sunbird-cloud` | Unit tests | AWS/Azure metadata parsing round-trips |
| `sunbird-auth` challenge | Unit tests in `challenge.rs` | PoW solver and difficulty mask tested |

**Gaps:**
- No integration tests for the full encryption/decryption cycle of the agent database (secret + storage layers together).
- No chaos/fault injection tests for backend unavailability (the retry/backoff logic in `spawn_network_init()` is untested).
- No fuzzing for the sigma rule compiler (`sigma-rust/src/`) or YARA rule loader — both parse external/server-provided content.
- No tests for `WindowsKeystore` (CNG-backed) outside Windows CI.
- `sunbird-auth` enrollment integration tests are CI-only, creating a gap in developer feedback loops.

---

## Unsafe Code Inventory

All `unsafe` blocks found are in two crates:

### `crates/bitdefender/windows/bitdefender-cst/`
**Count: ~30 blocks** across `ffi.rs`, `events.rs`, `macros.rs`, `loader.rs`, `sdk.rs`.

**Assessment:** Legitimate FFI to the BitDefender SDK Windows DLL. The patterns are standard Rust FFI: casting C pointers to Rust references, calling Windows API functions through loaded function pointers, and managing C string lifetimes. The `sdk.rs` raw pointer casts (e.g., `context as *const CallbackWrapper<T, F>` at line 53) follow the standard callback context pattern for C APIs. No assessment of soundness issues without the full BitDefender SDK headers, but the patterns are idiomatic for Windows DLL interop.

### `crates/sunbird-detection-pipeline/src/path_canonicalizer.rs`
**Count: 4 blocks** (lines 137, 140, 199, 207, 474).

**Assessment:** Legitimate Windows API calls to `GetLogicalDriveStringsW` and `GetLastError`. The pattern (calling Win32 API, checking return value, using `GetLastError` on failure) is standard and necessary for interacting with the Windows filesystem API. No safety concerns identified.

### `crates/sunbird-secret/src/platform/dpapi.rs`
**Count: 6 blocks** (lines 148–170, 186–201).

**Assessment:** Correct and well-commented DPAPI wrapping. Each block has a `// SAFETY:` comment explaining the invariant. The `LocalFree` calls correctly free the `CRYPT_INTEGER_BLOB.pbData` pointer allocated by the Windows API. No concerns.

**Summary**: All `unsafe` blocks are legitimate OS-level bindings. The workspace-level `unsafe_code = "forbid"` lint is absent but all existing uses are justified. The risk is that new `unsafe` code could be introduced without workspace-level review enforcement.

---

## Top 5 Recommendations

### 1. Add `unsafe_code = "forbid"` to the workspace lints — CRITICAL

**File:** `/Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/Cargo.toml`

Add to `[workspace.lints.rust]`:
```toml
[workspace.lints.rust]
unsafe_code = "forbid"
```

Then add `#[allow(unsafe_code)]` crate-level attributes to the three crates that legitimately use unsafe (`bitdefender-cst`, `sunbird-detection-pipeline`, `sunbird-secret`). This ensures any new `unsafe` usage in other crates triggers a compile error, enforcing the policy stated in the coding standards.

---

### 2. Implement OS keychain integration for Linux/macOS private key storage — HIGH

**File:** `/Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/sunbird-keystore/src/lib.rs:58–61`

The `FileBasedKeystore` stores the mTLS long-term private key as an unencrypted DER file. On Linux, use the kernel keyring (`keyutils`) or a file with `0o600` permissions plus encryption using the `PlatformKey` from `sunbird-secret` (which already applies ChaCha20-Poly1305 at-rest encryption). On macOS, integrate with the Keychain Services API (analogous to the Windows CNG implementation). Until this is addressed, any local user with filesystem read access to `<data_dir>/keystore/` can impersonate the sensor.

---

### 3. Gate `SUNBIRD_FORCE_DISABLE_MQTTS` behind a build-time assertion against production features — HIGH

**File:** `/Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/phoenix-sunbird/src/app.rs:1071–1076`

A build compiled with `SUNBIRD_FORCE_DISABLE_MQTTS=true` in the environment will silently disable mTLS for all MQTT connections. Add a compile-time assertion:

```rust
#[cfg(all(feature = "production", env = "SUNBIRD_FORCE_DISABLE_MQTTS"))]
compile_error!("SUNBIRD_FORCE_DISABLE_MQTTS must not be set in production builds");
```

Alternatively, remove the flag entirely and use a runtime config key for development environments. The current approach creates a category of production builds that are indistinguishable from secure builds except by inspecting the binary.

---

### 4. Convert startup `panic!` and `.expect()` calls to recoverable errors — HIGH

**Files:**
- `/Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/phoenix-sunbird/src/app.rs:157–161, 167–168, 268, 284, 881, 907, 933`
- `/Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/sunbird-keystore/src/file/mod.rs:133`

With `panic = "abort"` in release builds, any panic is an immediate process termination with no cleanup. Initialization errors such as "cannot create data directory" or "cannot load sigma bundle" should be propagated as `Result<Application, StartupError>` to the binary's `main()` function, which can log the error and exit cleanly with an appropriate exit code. This also enables the agent to emit a final error metric before exiting.

---

### 5. Add bounded channels or explicit backpressure to the detection pipeline — MEDIUM

**Files:**
- `/Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/sunbird-detection-pipeline/src/lib.rs:401–402`
- `/Users/aviad.cohen/Code/phoenix_review/projects/phoenix-agent/crates/phoenix-sunbird/src/client/client.rs:97–98`

Under high event rates (e.g., a process generating 50,000 file events/second on a busy build server), the unbounded async and inline pipeline channels will accumulate events in memory without bound. Consider:

- Switching the async detection channel to a bounded channel (e.g., 10,000 events) and dropping or sampling events when full, with a metric increment.
- Adding a memory-usage guard: if the total pending event count exceeds a threshold, apply sampling to low-priority events.
- The owLSM forwarder already uses a bounded channel of 256; the same approach should apply to the main pipeline channels.
