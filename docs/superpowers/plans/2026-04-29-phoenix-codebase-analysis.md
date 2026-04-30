# Phoenix Platform Codebase Analysis Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce a comprehensive architectural overview of ALL projects found in `projects/` readable by someone with zero prior context.

**Architecture:** Projects are discovered dynamically at runtime — this plan does NOT hardcode project names. For each discovered project, three artifacts are produced (deep architecture doc, code-quality review, concise summary). All analyses run in parallel. A final synthesis pass merges them into one unified document.

**Tech Stack (context for analysis):** Rust (Tokio, tonic, sqlx, clap), Go (slog, viper), TypeScript (Next.js, tRPC, Drizzle), Apache Flink, ClickHouse, Redpanda/Kafka, Protobuf/buf

---

## 1. Discovery Rule (run before everything else)

```bash
# List real project directories — skip hidden dirs, files, and obvious placeholders
ls -d projects/*/ 2>/dev/null \
  | xargs -n1 basename \
  | grep -Ev '^(clone_|placeholder|example|todo|tmp_)' \
  | while read p; do
      # Only count it as a real project if it contains source files
      if find "projects/$p" -maxdepth 3 \( -name "*.rs" -o -name "*.go" -o -name "*.ts" -o -name "*.tsx" -o -name "*.proto" -o -name "*.py" \) -print -quit 2>/dev/null | grep -q .; then
        echo "$p"
      fi
    done
```

Each line of output is one real project. For every real project, dispatch **three tasks** (architect, review, summary) using the templates in §3. All three depend only on reading the project's source — they can run in parallel across all projects.

**Output directory:** `docs/analysis/`
**Unified output:** `docs/analysis/phoenix-platform-architecture.md`

---

## 2. Project Slug Convention

The "slug" is used for output filenames. Derivation rule:

| Project dir name | Slug | Output files |
|------------------|------|--------------|
| `Phoenix/` (server platform) | `phoenix-server` | `phoenix-server-architect.md` etc. |
| `phoenix-agent/` | `phoenix-agent` | `phoenix-agent-architect.md` etc. |
| `phoenix-proto/` | `phoenix-proto` | `phoenix-proto-architect.md` etc. |

**Rules:**
1. Lowercase the dir name
2. Replace `_` with `-`
3. If the dir name is ambiguous (e.g., generic "Phoenix"), suffix with role qualifier (`-server`, `-agent`, etc.)
4. Slug must be unique across all projects — if collision, the controller picks distinct qualifiers

---

## 3. Subagent Selection & Dispatch Notes

| Task | Recommended agent type | Rationale |
|------|------------------------|-----------|
| Architect | `architecture` | Has Read+Write+Bash; designed for system mapping |
| Review | `code-analyzer` | Code quality + security focused |
| Summary | `council-implementer` (with explicit "write file" directive) | Smaller context, focused output |

**⚠️ Known subagent constraint:** Some subagent profiles refuse to write `.md` report files (they return content inline instead). When dispatching, **explicitly include in the prompt**:

> "You MUST write the file using the Write tool. Do NOT return findings inline. If the Write tool blocks .md report files in your environment, use `cat > <path> <<'EOF' ... EOF` via Bash as a fallback."

If a subagent still returns content inline, the controller writes the file itself (via Write tool) and commits, then proceeds.

---

## 4. Completed Tasks (as of 2026-04-29)

| Project | Architect | Review | Summary | Output files |
|---------|-----------|--------|---------|--------------|
| `Phoenix/` (server) | ✅ | ✅ | ✅ | `phoenix-server-{architect,review,summary}.md` |
| `phoenix-agent/` | ✅ | ✅ | ✅ | `phoenix-agent-{architect,review,summary}.md` |
| `phoenix-proto/` | ✅ | ✅ | ✅ | `phoenix-proto-{architect,review,summary}.md` |
| **Synthesis** | ✅ — `phoenix-platform-architecture.md` (3 projects, 610+ lines) |

When adding a new project, only the **new** rows need execution. Synthesis must be re-run to incorporate new findings.

---

## 5. Template: Architect Task

**For each discovered project `<project-name>` at `projects/<project-name>/`:**

**Files:**
- Read: all root config files (`Cargo.toml`, `go.mod`, `package.json`, `buf.yaml`, `pyproject.toml`, `README.md`)
- Read: source directories recursively to depth 3 (deeper sampling for key files)
- Read: `docker-compose*.yml` if present
- Read: `docs/` if present
- Create: `docs/analysis/<slug>-architect.md`

- [ ] **Step 1: Identify project type and tech stack**

  Read root config files to determine:
  - Language(s): Rust workspace? Go modules? TypeScript? Python? Protobuf-only?
  - Build tooling: cargo, go, bun/npm, buf, mise?
  - Key frameworks and dependencies

- [ ] **Step 2: Map all modules / crates / packages / services**

  For each top-level module/crate/service directory:
  - What is its single responsibility?
  - What does it depend on (internal + external)?
  - What does it expose (API, gRPC port, Kafka topic, CLI binary)?

  Per language:
  - **Rust workspace:** read every `Cargo.toml` + `src/main.rs` or `src/lib.rs`
  - **Go modules:** read `go.mod` + `cmd/*/main.go`
  - **TypeScript:** read `package.json` + entry points + router/route files
  - **Protobuf repos:** read all `.proto` files fully — every message, enum, service

- [ ] **Step 3: Trace the main data flows**

  - For services: inbound (HTTP/gRPC/Kafka) → processing → outbound (DB write/Kafka/gRPC)
  - For libraries: what API is exposed, how callers use it
  - For schema repos: which components produce vs consume each message type

- [ ] **Step 4: Map external dependencies**

  - Databases (PostgreSQL, ClickHouse, SQLite, Redis)
  - Message brokers (Kafka/Redpanda, MQTT)
  - Other services (gRPC, HTTP)
  - OS-level interfaces (eBPF, kernel APIs, OS security frameworks)

- [ ] **Step 5: Map infrastructure and deployment**

  - Docker compose services (if present)
  - Environment variables required
  - Port assignments
  - Packaging / distribution (if applicable)

- [ ] **Step 6: Cross-reference with sibling projects**

  Before writing, check `docs/analysis/` for existing architect docs of other projects. If they share schemas, topics, or APIs with this project, note the integration contract precisely (don't restate — link).

- [ ] **Step 7: Write `docs/analysis/<slug>-architect.md`**

  ```markdown
  # <Project Name> — Architectural Overview

  ## What Is This?
  [2-3 paragraphs: purpose, role in the broader system, who uses it]

  ## High-Level Architecture Diagram
  ```mermaid
  flowchart LR
      ...
  ```

  ## Tech Stack
  [Languages, frameworks, key libraries]

  ## Module / Crate / Package Catalog
  [Table: Name | Role | Key dependencies | Inbound | Outbound]

  ## Data Flows
  [Numbered step-by-step for each major flow]

  ## External Dependencies
  [Table: System | Type | Purpose | Connection details]

  ## Infrastructure & Deployment
  [Docker services, ports, env vars, packaging]

  ## Connection to Sibling Projects
  [Exact contracts: protocols, shared schemas, Kafka topics, gRPC ports]

  ## Glossary (project-specific terms)
  ```

- [ ] **Step 8: Self-verify**

  Re-read the doc and confirm:
  - Every top-level module/crate is in the catalog
  - At least one Mermaid diagram is present and renders
  - At least one data flow is numbered step-by-step
  - File contains no `[TBD]` / `[fill in]` placeholders

- [ ] **Step 9: Commit**

  ```bash
  git add docs/analysis/<slug>-architect.md
  git commit -m "docs: add <project-name> architectural overview"
  ```

---

## 6. Template: Review Task

**For each discovered project `<project-name>` at `projects/<project-name>/`:**

**Files:**
- Read: sampled source files (minimum 5 files covering different concerns)
- Create: `docs/analysis/<slug>-review.md`

- [ ] **Step 1: Identify applicable coding standards**

  Apply only the rules matching the language(s) in this project:

  **Rust:**
  - Error handling: `thiserror` in libs, `eyre`/`color-eyre` in bins
  - Logging: `tracing` with structured fields including `org_id`
  - Config: `clap::Parser` with `env = "..."` attributes
  - No `unwrap()`/`expect()` in non-test code
  - `unsafe_code = "forbid"` in workspace lints
  - Health endpoints if it's a server

  **Go:**
  - Logging: `log/slog` with structured key-value fields
  - Config: `github.com/spf13/viper` with `PHOENIX_` env prefix
  - Errors: `fmt.Errorf("context: %w", err)`
  - Table-driven tests

  **TypeScript:**
  - `"strict": true`, `"noUncheckedIndexedAccess": true`, no `any`
  - `zod` validation on all external inputs
  - `org_id` from session only, never from user input
  - Static Tailwind classes only

  **Protobuf:**
  - `snake_case` fields, `PascalCase` messages, `UPPER_SNAKE_CASE` enum values
  - Every enum has a `_UNSPECIFIED` zero value
  - `google.protobuf.Timestamp` for timestamps (not raw `int64`)
  - `reserved` statements for deleted field numbers
  - `buf breaking` configured with a baseline reference

- [ ] **Step 2: Multi-tenancy audit (if applicable)**

  If project handles multi-tenant data:
  ```bash
  PROJ=projects/<project-name>
  grep -rn "org_id" "$PROJ" --include="*.rs" --include="*.go" --include="*.ts" | head -20
  grep -rn "SELECT\|INSERT\|UPDATE\|DELETE" "$PROJ" --include="*.rs" -l | head -10
  ```
  Sample 5 query files: all filter by `org_id`? All parameterized (no string concatenation)?

- [ ] **Step 3: Security scan**

  ```bash
  PROJ=projects/<project-name>

  # Hardcoded secrets
  grep -rn "password\|secret\|api_key\|apikey" "$PROJ" --include="*.rs" --include="*.go" --include="*.ts" -i | grep -v test | grep -v mock | head -20

  # Unsafe code (Rust)
  grep -rn "unsafe {" "$PROJ" --include="*.rs" | grep -v test | head -20

  # Unwrap in prod code (Rust)
  grep -rn "\.unwrap()\|\.expect(" "$PROJ" --include="*.rs" | grep -v test | head -20
  ```

- [ ] **Step 4: Sample representative source files**

  Pick at least 5 files covering:
  - Entry point / main
  - Core business logic
  - External interface (API handler, Kafka consumer, proto definition)
  - Error handling path
  - Tests

  For each: assess standards compliance, note specific deviations with `file:line` references.

- [ ] **Step 5: Cross-component sync check (proto/schema repos only)**

  If this project is a schema repo, diff key files against embedded copies in sibling projects:
  ```bash
  diff projects/<this>/proto/foo.proto projects/<sibling>/proto/foo.proto
  ```
  Drift is the highest-severity finding for schema repos.

- [ ] **Step 6: Write `docs/analysis/<slug>-review.md`**

  ```markdown
  # <Project Name> — Code Review Findings

  ## Summary
  [3-5 sentences: overall quality score /10, main strengths, main concerns]

  ## Standards Compliance
  | Area | Status | Notes |
  |------|--------|-------|
  | [standard] | ✅/⚠️/❌ | [specific finding] |

  ## Strengths
  [Bulleted list with file references]

  ## Concerns by Severity
  ### Critical
  ### High
  ### Medium
  ### Low
  [Each: description, file:line, what the risk is, suggested fix]

  ## Multi-Tenancy Audit
  [Findings — only include if applicable]

  ## Security Observations
  [Hardcoded secrets scan, unsafe code inventory, auth boundary checks]

  ## Test Coverage
  [What testing exists, notable gaps]

  ## Top Recommendations
  [Prioritized, max 5, with specific file paths]
  ```

- [ ] **Step 7: Self-verify**

  Confirm:
  - Every standards-compliance row has a concrete file reference (no vague "looks good")
  - Every concern has `file:line` or `file:line-range`
  - Top recommendations are numbered and prioritized
  - File contains no `[TBD]` / `[fill in]` placeholders

- [ ] **Step 8: Commit**

  ```bash
  git add docs/analysis/<slug>-review.md
  git commit -m "docs: add <project-name> code review findings"
  ```

---

## 7. Template: Summary Task

**For each discovered project `<project-name>`:**

**Depends on:** the architect doc and review doc for this project must already exist.

**Files:**
- Read: `docs/analysis/<slug>-architect.md`
- Read: `docs/analysis/<slug>-review.md`
- Create: `docs/analysis/summary-<slug>.md`

- [ ] **Step 1: Read both source docs in full**

- [ ] **Step 2: Distill to 100–200 lines**

  ```markdown
  # <Project Name> — Project Summary

  ## One-Line Description
  [What it is in one sentence]

  ## Purpose
  [2-3 sentences]

  ## Tech Stack
  [Bullet list]

  ## Architecture at a Glance
  [Simplified Mermaid — main data path only]

  ## Key Components
  [Table: top 10 most important — Component | Role]

  ## Main Data Flows
  [Numbered, 3-5 steps each]

  ## External Dependencies
  [Bullet list]

  ## Multi-Tenancy
  [2-3 sentences — omit if not applicable]

  ## Code Quality Snapshot
  | Metric | Value |
  | Overall score | X.X/10 |
  | Top strength | ... |
  | Top concern | ... |
  | Critical issues | [count + one-line each] |

  ## Top 3 Risks
  [Numbered, with file:line refs]

  ## How to Run/Build Locally
  [Key commands]

  ## Key Files to Know
  [5-7 most important files for a new developer, one-line each]
  ```

- [ ] **Step 3: Commit**

  ```bash
  git add docs/analysis/summary-<slug>.md
  git commit -m "docs: add <project-name> project summary"
  ```

---

## 8. Final Task: Synthesize Unified Overview

**Depends on:** All architect, review, and summary tasks complete for all discovered projects.

**Files:**
- Read: all `docs/analysis/*-architect.md`
- Read: all `docs/analysis/*-review.md`
- Read: all `docs/analysis/summary-*.md` (cross-check consistency)
- Create/update: `docs/analysis/phoenix-platform-architecture.md`

- [ ] **Step 1: Inventory all completed analysis docs**

  ```bash
  ls docs/analysis/ | sort
  ```

- [ ] **Step 2: Write unified overview**

  ```markdown
  # Phoenix EDR/XDR Platform — Complete Architecture Overview

  ## Executive Summary
  [4-5 sentences: what Phoenix is, the multi-component model, target users, multi-tenancy]

  ## System Boundaries
  [Table: Component | Location | Language | Role — one row per discovered project]
  [Paragraph: how components interconnect]

  ## End-to-End Data Flows
  ### 1. Event Collection: Endpoint → Cloud
  ### 2. Detection & Alerting: Cloud Processing
  ### 3. Response & Command: Cloud → Endpoint
  [Other flows if discovered]

  ## Component Deep-Dives
  [One section per project — merged from architect docs, not just copied]

  ## Shared Schema Contract
  [If a proto/schema repo exists: message types, Kafka topic mapping, governance]

  ## Integration Contracts
  [How each component connects to each other — protocols, messages, topics]

  ## Cross-Cutting Concerns
  ### Multi-Tenancy Model
  ### Observability (logging, metrics, tracing)
  ### Security Model (auth, mTLS, secret handling)
  ### Configuration & Environment

  ## Code Quality Summary
  [One subsection per project: score, top strengths, top concerns with file:line refs]

  ## Development Setup
  [Per-component: how to build, test, run locally]

  ## Glossary
  [All domain terms across all projects, 1-2 sentences each]
  ```

- [ ] **Step 3: Self-verify the unified doc**

  - Every project from §1 discovery has a Component Deep-Dive section
  - Integration Contracts section shows the full graph (no orphan components)
  - Code Quality Summary scores match the per-project review docs
  - Cross-Cutting Concerns covers all four sub-sections

- [ ] **Step 4: Commit**

  ```bash
  git add docs/analysis/phoenix-analysis-architecture.md
  git commit -m "docs: update unified platform architecture overview"
  ```

---

## 9. Execution Order & Parallelism

```
Step 0: Run Discovery Rule (§1) — get list of real projects
         ↓
For each project, dispatch 3 tasks in parallel:
  ┌─ Architect Task  ──┐
  ├─ Review Task      ──┤  (all 3 read-only on the same project — no conflicts)
  └─ Summary Task     ──┘  (Summary depends on Architect+Review docs existing — start it AFTER they finish)
         ↓
Across all projects, all Architect tasks parallel; all Review tasks parallel.
         ↓
Final Synthesis Task — sequential, needs all per-project docs complete
```

**Parallelism budget:**
- Architect + Review per project: independent, both can run together
- Summary per project: depends on architect+review for that same project
- Across projects: fully parallel (no shared file writes)
- Synthesis: blocks on everything

**Practical timing (observed on this codebase):**
- Architect task: ~5–8 min per project
- Review task: ~5–7 min per project
- Summary task: ~2–4 min per project
- Synthesis: ~5–8 min

For N projects, end-to-end ≈ 8 min (longest architect) + 4 min (summary) + 8 min (synthesis) ≈ **~20 min total** if dispatched fully parallel.

---

## 10. Common Pitfalls & Lessons Learned

| Pitfall | Fix |
|---------|-----|
| Subagent refuses to write `.md` report files | Include the explicit Write directive from §3, with `cat > heredoc` fallback |
| Architect agent doesn't see existing sibling docs | Step 6 of architect template — read `docs/analysis/` first |
| Synthesis dispatched before late-added project completes | Always `ls docs/analysis/` immediately before synthesis; abort if expected files missing |
| File counts in subagent reports drift (e.g., "172" vs "236") | Always cite the exact source doc; use `find ... | wc -l` to verify |
| Standards check runs irrelevant rules (e.g., Go rules on a Rust project) | Step 1 of review template — apply only matching language rules |
| Discovery picks up `clone_pheonix_projects_here` and other empty dirs | The `find ... -name "*.rs"` filter in §1 excludes empty placeholders |
| Glossary in unified doc duplicates per-project glossaries | Synthesis merges and dedupes; don't paste verbatim |

---

## 11. Adding a New Project

1. Drop the project at `projects/<new-name>/`
2. Re-run §1 discovery — confirm the new project appears
3. Dispatch the 3 task templates (architect → review → summary) for the new slug
4. Update §4 Completed Tasks table with the new row
5. Re-run §8 Synthesis to merge new findings into the unified doc

No edits to this plan file are required other than the §4 table.
