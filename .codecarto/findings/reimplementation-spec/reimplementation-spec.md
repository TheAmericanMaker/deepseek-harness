# Reimplementation Spec

<!-- selection: auto-default (language-agnostic; the Strategic Alignment Hook was suppressed for the unattended full-pipeline run) -->

## System Summary

DeepSeek Harness (`dsh`) is a plugin-based agent harness: a runtime for building and running LLM agents where **everything is a plugin**. The model adapter, tool registry, session log, agent loop, persistence, sandboxing, and UI are all plugins contributing services, typed events, and reversible effects to a shared context. A running instance is a plugin tree composed at boot from ordered layers (bundles → profile patches → home patches → `--patch` overlays). The heart is an **event-sourced session log** with the invariant *model-visible means logged*; the agent loop runs a turn/step state machine; tools execute through a guarded pipeline with parallel/exclusive scheduling. The system ships through a CLI + Web UI, a headless runner, an ACP automation server, a JSON-RPC SDK (TypeScript + Python), and a Python SDK that drives the harness as a subprocess.

## Conceptual Module Model

### Plugin Runtime (framework)

| Field | Value |
|---|---|
| **Responsibility** | Provide the DI/effect substrate: services, typed events, reversible effects, and lifecycle (load/unload/dispose) |
| **Public inputs** | Plugin registrations (service classes, function plugins), config rows |
| **Public outputs** | A shared context (`ctx`) exposing services and event dispatch |
| **Owned state** | The plugin tree, effect registrations, service store |
| **Invariants** | Registrations are effects that unwind on unload; disposal is reentrant-safe |
| **Collaborators** | Everything (the base layer) |

### Session Store

| Field | Value |
|---|---|
| **Responsibility** | Event-sourced append-only session log + in-memory store + derived model history |
| **Public inputs** | `create`/`fork`/`resume`/`append`/`deriveMessages` |
| **Public outputs** | `session/created`/`session/event`/`session/flush`/`session/disposed` events; `Session` objects |
| **Owned state** | The event log, surface manager, header |
| **Invariants** | Model-visible means logged; monotonic contiguous `seq`; `SESSION_FORMAT_VERSION` 0 reject-not-migrate |
| **Collaborators** | Persistence backends, telemetry, UI, ACP/SDK bridges |

### Tool Registry + Execution Pipeline

| Field | Value |
|---|---|
| **Responsibility** | Scoped tool registry + guarded execution (pre/execute/post/result waterfalls) |
| **Public inputs** | `defineTool` registrations; `tool/call` blocks |
| **Public outputs** | `tool/result` events; canonical lossless-JSON values |
| **Owned state** | Tool definitions, execution modes, scheduler state |
| **Invariants** | Results commit in model order; abort records synthetic results for skipped calls |
| **Collaborators** | Agent loop, system-prompt assembly, approval seam |

### Agent Loop

| Field | Value |
|---|---|
| **Responsibility** | Drive the turn/step state machine (one model request + its tool calls per step) |
| **Public inputs** | Queued input, `agent/pre-step` decisions |
| **Public outputs** | `turn/*`, `step/*`, `assistant/*`, `tool/*` events |
| **Owned state** | Turn/step counters, inbox, in-flight dispatch |
| **Invariants** | Waterfall events require `next()`; `agent/turn-stopping` is serial |
| **Collaborators** | LLM seam, tool registry, session store |

### LLM Adapter Seam

| Field | Value |
|---|---|
| **Responsibility** | Provider-agnostic message/stream vocabulary + adapter registration |
| **Public inputs** | `agent/request`; provider config |
| **Public outputs** | `llm/stream` chunks; `assistant/message` |
| **Owned state** | Registered adapters, retry policy |
| **Invariants** | Adapters are swappable; retry is durable before its cancellable wait |
| **Collaborators** | Agent loop, credentials, settings |

### Persistence Backends

| Field | Value |
|---|---|
| **Responsibility** | Durable session/storage persistence (JSONL + SQLite) |
| **Public inputs** | `append`/`load`/`inspect`/`readFrom` |
| **Public outputs** | Session logs, storage records |
| **Owned state** | Files, DB handles, torn-tail markers |
| **Invariants** | Append-only; zstd checksummed frames; `SCHEMA_VERSION` 17; torn-tail repair |
| **Collaborators** | Session store, storage hub |

### Capability Seams (fs, shell, subprocess, terminal, lsp, web, skill, compaction, subagent, workflow, code-runtime, sandbox, credentials, settings, …)

| Field | Value |
|---|---|
| **Responsibility** | Each is a three-role seam: Service Definition + Provider + Consumer |
| **Public inputs** | Provider registrations; tool calls |
| **Public outputs** | Tool results; capability events |
| **Owned state** | Per-seam provider state |
| **Invariants** | Extension plugins depend on Service Definitions, never concrete providers |
| **Collaborators** | Tool registry, agent loop, session store |

### Delivery Surfaces (CLI, Web UI, headless, ACP, SDK, Python)

| Field | Value |
|---|---|
| **Responsibility** | Expose the harness to users/programs |
| **Public inputs** | CLI args, HTTP requests, JSON-RPC/ACP frames |
| **Public outputs** | Web UI, task results, wire responses |
| **Owned state** | Per-surface connection/session state |
| **Invariants** | ACP: one prompt in flight per session; SDK: newline-delimited JSON-RPC 2.0 |
| **Collaborators** | Plugin runtime, session store, agent loop |

## Layer Split

| Module | Layer | Notes |
|---|---|---|
| Plugin Runtime (framework) | core semantics | Must survive the port unchanged in behavior |
| Session Store | core semantics | Event-sourced log is the source of truth |
| Tool Registry + Pipeline | core semantics | Guarded execution + ordered commit |
| Agent Loop | core semantics | Turn/step state machine |
| LLM Adapter Seam | core semantics | Provider-agnostic vocabulary |
| Persistence Backends | core semantics | JSONL/SQLite formats + repair |
| Capability Seams | adapters | OS/cloud/SDK integrations |
| Delivery Surfaces | delivery surfaces | CLI/Web/ACP/SDK/Python wrappers |

## Required Behaviors

- **Plugin composition**: boot a plugin tree from ordered layers (bundles → profile patches → home patches → `--patch` overlays); patch-by-id replace/insert; `!!js` interpolation; `disabled: !!js` per-entry evaluation.
- **Session lifecycle**: create/fork/resume; append-only log; `deriveMessages()` projects model history; model-visible-means-logged invariant.
- **Turn/step loop**: `turn/start → agent/pre-step → step/start → agent/request → llm/stream → assistant/chunk* → tool/call* → tools/* → step/end → agent/turn-stopping → turn/end`.
- **Tool execution**: `tools/pre-execute` (allow/deny/ask) → `tools/execute` (around-dispatch) → body → `tools/post-execute` → `tools/result`; parallel/exclusive scheduling; ordered result commit; abort records synthetic results.
- **Code Mode**: `run_code` reserved transport; model-direct native-tool calls denied as `UNKNOWN_TOOL`.
- **Credential resolution**: process env > `.credentials.yaml` > project `.env` > home `.env`; owner-only 0600 (POSIX).
- **Sandboxing**: process confinement via bwrap/Landlock/Seatbelt/ACL.
- **Subagent delegation**: provider registry; continuable vs one-shot.

## Protocols and Persisted State

- **SDK JSON-RPC**: newline-delimited JSON-RPC 2.0 over stdio; `initialize`/`session/prompt`/`shutdown` requests; `session.event`/`session.status`/`subagent.started`/`subagent.finished` notifications; `-32601`/`-32603` errors; malformed lines ignored.
- **ACP**: `initialize`/`newSession`/`prompt`/`cancel`; one prompt in flight; `agent_message_chunk` updates; one-shot permission decisions.
- **Typert RPC**: `Remote`/`RemoteScope` decorators + `bindTypertRemote`; endpoint segment grammar `/^[A-Za-z0-9_$.-]+$/`.
- **Session events**: `SessionEventMap` (13 core types); envelope `type`/`seq`/`time`/`data` + `surfaceOp`/`sourceEventSeqs`/`ignorable`; surface ops `append`/`replace`; turn-end reasons `completed`/`aborted`/`blocked`/`error`/`max-tokens`/`interrupted`.
- **Persistence**: JSONL (zstd checksummed frames, packed chunk rows, torn-tail repair) and SQLite (`SCHEMA_VERSION` 17, application-id `0x44534850`, `trusted_schema=off`, `mmap_size=0`, `synchronous=FULL`).

## External Dependencies

| Dependency | Stance | Rationale |
|---|---|---|
| Cordis framework (vendored) | replace | Reimplement or substitute an equivalent DI/effect system in the target language |
| zstd | wrap | Use the target language's zstd library; reimplement the custom frame scanning + prefix decompression |
| SQLite | wrap | Use the target language's SQLite binding; re-verify `trusted_schema`/`mmap_size`/`synchronous` pragmas |
| node-pty / ConPTY | replace | OS-specific PTY; use the target's terminal library |
| bwrap / Landlock / Seatbelt / ACL | replace | OS-specific sandbox; use the target's confinement primitives |
| `@agentclientprotocol/sdk` | wrap | ACP wire protocol; reimplement the JSON-RPC framing |
| ripgrep (`@vscode/ripgrep`) | wrap | glob/grep discovery; use the target's search primitive |
| chokidar (file watching) | replace | Use the target's file-watch primitive |
| `AbortSignal.any`/`Symbol.dispose`/`Promise.withResolvers` | emulate | Polyfill or target a modern runtime |

## Portability Hazards

- **Cordis fiber/effect lifecycle + waterfall events** — high; reimplement or replace with an equivalent DI/effect system.
- **`SESSION_FORMAT_VERSION` 0 reject-not-migrate** — high; reproduce reject-not-migrate, no silent skip.
- **Custom zstd frame format + torn-tail recovery** — high; reimplement frame scanning + prefix decompression.
- **JSON-RPC newline framing (malformed lines silently ignored)** — medium; preserve the ignore-malformed/error-on-handler-failure contract.
- **SQLite pragmas** — medium; re-verify on the target binding.
- **ES2024+ features** — medium; polyfill or target a modern runtime.
- **Windows env case-folding + `taskkill /T /F`** — medium; branch on OS.
- **POSIX process-group signalling** — medium; no Windows equivalent.
- **`node:sqlite` synchronous API** — medium; async SQLite changes the concurrency model.

## Implementation Sequence

1. **Plugin runtime** (DI/effect substrate) — the base everything else builds on.
2. **Session store** (event-sourced log + header + surface) — the source of truth.
3. **LLM adapter seam** (message/stream vocabulary + one provider).
4. **Tool registry + execution pipeline** (defineTool, waterfalls, scheduling).
5. **Agent loop** (turn/step state machine).
6. **Persistence backends** (JSONL first, then SQLite).
7. **Capability seams** (fs, shell, subprocess first; then terminal, lsp, web, skill, subagent, workflow).
8. **Delivery surfaces** (headless CLI first, then SDK JSON-RPC, then ACP, then Web UI).

### Scope Tiers

**Minimum viable port:** plugin runtime + session store + LLM seam + tool registry + agent loop + JSONL persistence + headless CLI. This is a working agent harness with no UI.

**Major-workflow parity:** + capability seams (fs, shell, subprocess, terminal, lsp, web, skill, subagent, workflow) + SQLite persistence + SDK JSON-RPC + ACP server + credential/settings/sandbox.

**Full parity:** + Web UI + Python SDK + native Landlock launcher + experimental agent-team + all remaining seams.

## Acceptance Scenarios

| # | Scenario | Input | Expected Output / Side Effect |
|---|----------|-------|-------------------------------|
| 1 | Boot headless | `dsh --profile headless "task"` | Task result on stdout; session log written |
| 2 | Dump config | `dsh --profile web --dump-config` | Composed entry list printed without booting |
| 3 | Unknown tool | Model requests unregistered tool | `UNKNOWN_TOOL` error result |
| 4 | Tool abort before dispatch | Abort a step before a tool starts | Synthetic `ABORTED_BEFORE_DISPATCH` result; replay valid |
| 5 | Session fork | Fork a live session | Child session with lineage header; log replayable |
| 6 | Credential precedence | `DEEPSEEK_API_KEY` in env AND `.credentials.yaml` | Env value wins, `source: 'env'`, `writable: false` |
| 7 | Credential write shadowed | `DEEPSEEK_API_KEY` in inherited env | `set()` throws (shadowed by read-only env) |
| 8 | SQLite schema mismatch | DB with `user_version` ≠ 17 | Rejects with version-mismatch error |
| 9 | JSONL torn tail | Log with incomplete final zstd frame | Recovers complete prior frames; torn tail marked for repair |
| 10 | Parallel tool calls | Two concurrency-safe tools in one step | Both dispatch concurrently; results commit in model order |
| 11 | SDK round-trip | `initialize` → `session/prompt` → `shutdown` | `serverInfo` returned; `messageId` returned; `{}` on shutdown |
| 12 | ACP one-prompt | Two concurrent `prompt` on one session | Second rejected with invalid-params |

## Deliberate Non-Goals

- **Web UI** — deferred to full parity; not needed for a first viable headless port.
- **Python SDK + runtime** — deferred; independent surface.
- **Native Landlock launcher** — deferred; Linux-only.
- **Experimental agent-team** — deferred; private prototype.
- **Source-specific build tooling** (pnpm/tsdown/vite) — incidental; the port's build is its own concern.
- **Exact source folder layout** — the port should follow capability seams, not the source tree.

## Coverage and limits

- Inspected scope: the porting bundle (`findings/porting/reverse-engineering-bundle.md`) as the compression boundary; no lower-level reports deep-read (the bundle was self-sufficient).
- Skipped scope: the api-gateway transport framing and client↔host wire protocol (named as known unknowns); the Python SDK and native launcher sources.
- Evidence basis: source inspection + generated catalogs + docs (via the bundle). No runtime verification.
- Known blind spots: runtime-only races; the api-gateway transport; the full 233-package surface not exhaustively read.
- Coverage disposition: COMPLETE (the bundle was self-sufficient; residual gaps are named as known unknowns).

## Known Unknowns

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| q-approval-default | needs-runtime-test | Whether the approval seam defaults to allow or ask when no approval provider is loaded. | Requires a runtime test against a minimal composition. |
| q-api-gateway-transport | needs-runtime-test | The exact Typert Gateway transport framing (HTTP vs stdio) in `packages/api/*`. | Requires reading the api group source or a runtime capture. |
| q-spec-variant | needs-maintainer-decision | The spec defaulted to language-agnostic (auto-default); an opinionated target stack was not locked. | The Strategic Alignment Hook was suppressed for the unattended run. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| post-spike-1 | spike | Prototype the api-gateway transport framing (HTTP vs stdio) to resolve q-api-gateway-transport. | Post-pipeline spike; the pipeline cannot close it without executing code. |

## Spike List

- **api-gateway transport**: prototype the Typert Gateway transport (HTTP route shape vs stdio) to resolve q-api-gateway-transport.
- **zstd frame format**: prototype the custom zstd frame scanning + torn-tail prefix decompression against a real session log.
- **Cordis replacement**: spike an equivalent DI/effect system in the target language to validate the fiber/effect lifecycle + waterfall semantics.
- **SQLite pragmas**: verify `trusted_schema`/`mmap_size`/`synchronous` behavior on the target SQLite binding.
- **Process-tree termination**: spike POSIX group signalling vs Windows taskkill parity.

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | Concept-level modules are defined. | PASS | §Conceptual Module Model (8 modules). |
| 2 | Required behaviors are stated. | PASS | §Required Behaviors. |
| 3 | Protocol and persisted state expectations are stated. | PASS | §Protocols and Persisted State. |
| 4 | Acceptance scenarios and known unknowns are included. | PASS | §Acceptance Scenarios (12) + §Known Unknowns (3). |
| 5 | Findings are marked with evidence levels. | PASS | Evidence basis in §Coverage and limits; hazards marked in §Portability Hazards. |
| 6 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | §Coverage and limits. |
| 7 | Lower-level findings are deep-read only when the porting bundle identifies a gap, conflict, missing acceptance detail, or defect rationale. | PASS | Only the bundle was read; no lower-level deep reads were needed. |

**Validated by:** reimplementation-spec phase, session 1 (2026-08-19)
**Overall:** PASS
