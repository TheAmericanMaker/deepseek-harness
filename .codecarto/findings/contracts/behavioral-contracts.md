# Behavioral Contracts

## Surfaces Covered

- **CLI** (`dsh` binary)
- **Web UI** (browser app served by the host)
- **API/SDK** (JSON-RPC SDK, Python SDK, ACP server, Typert RPC gateway)
- **Model-facing tool surface** (`ctx.tools` registry — the tools the agent is offered)
- **Storage/export formats** (session logs JSONL/SQLite, storage JSON/SQLite, credentials YAML)
- **Headless runner** (one-shot)

## Feature Contracts

### Surface Type: CLI

#### `dsh web`

| Field | Value |
|---|---|
| **Feature** | Start the Web UI server |
| **Trigger or input** | `dsh web` (optionally `--no-open`, `--profile <name>`, `--patch <overlay>`) |
| **Defaults** | Serves at `http://127.0.0.1:3080`; opens the default browser on local launch |
| **Observable output** | HTTP server; browser opens (local) or host URL printed (SSH) |
| **Side effects** | Boots the plugin tree; mounts `dsh-web-app` bundle |
| **Persisted state** | Session logs, settings, credentials under `~/.dsh` |
| **Error behavior** | Boot failure exits nonzero with diagnostics |
| **Retry or recovery behavior** | None (fresh boot each run) |
| **Owner (layer/package)** | `apps/cli` + `packages/boot/app-boot` + `packages/host/webserver` |

#### `dsh headless`

| Field | Value |
|---|---|
| **Feature** | One-shot runner with no server |
| **Trigger or input** | `dsh --profile headless "task"` |
| **Defaults** | `dsh-headless` bundle |
| **Observable output** | Task result on stdout |
| **Side effects** | Runs one agent turn/step loop to completion |
| **Persisted state** | Session log |
| **Error behavior** | Nonzero exit on failure |
| **Retry or recovery behavior** | None |
| **Owner (layer/package)** | `apps/cli` + `packages/bundle/headless` |

#### `dsh --dump-config`

| Field | Value |
|---|---|
| **Feature** | Print the composed plugin tree without booting |
| **Trigger or input** | `dsh --profile <name> --dump-config` |
| **Defaults** | Composes empty profile root + each bundle's patch layer + profile/home patches + `--patch` overlays |
| **Observable output** | The exact entry list the include would mount |
| **Side effects** | None (pure composition) |
| **Persisted state** | None |
| **Error behavior** | Invalid config fails |
| **Retry or recovery behavior** | None |
| **Owner (layer/package)** | `apps/cli` + vendored `include` (`applyEntryPatches`) |

### Surface Type: Model-facing tool surface (`ctx.tools`)

#### Tool registration and execution pipeline

| Field | Value |
|---|---|
| **Feature** | Scoped tool registry + guarded execution pipeline |
| **Trigger or input** | A model emits `tool/call` blocks; the agent loop schedules them via `executeToolCalls` |
| **Defaults** | `defineTool` requires a name, description, parameters schema, and a mandatory canonical `output` (JSON Schema + render projection) |
| **Observable output** | `tool/result` events with model-facing content + optional `meta` |
| **Side effects** | `tools/pre-execute` (allow/deny/ask) → `tools/execute` (around-dispatch) → body → `tools/post-execute` → `tools/result`; durable `tool/call`/`tool/result` session events |
| **Persisted state** | `tool/call` and `tool/result` events in the session log |
| **Error behavior** | `UNKNOWN_TOOL` (unregistered), `ABORTED` (cancelled after body invoked), `ABORTED_BEFORE_DISPATCH` (cancelled before body); structured `ToolFailure` with `message` + `info{name,code}` |
| **Retry or recovery behavior** | Abort records synthetic error results for skipped calls so replay stays valid |
| **Owner (layer/package)** | `packages/core/tools` |

#### Concurrency modes

| Field | Value |
|---|---|
| **Feature** | Parallel vs exclusive tool-call scheduling |
| **Trigger or input** | `isConcurrencySafe?(args)` returns `true` to opt into parallel; omission/exceptions/non-`true` are exclusive |
| **Defaults** | Exclusive (ordering barrier) |
| **Observable output** | Parallel calls overlap up to `maxParallelToolCalls`; exclusive calls form barriers |
| **Side effects** | Results commit in model order regardless of dispatch overlap |
| **Persisted state** | Ordered `tool/call`/`tool/result` pairs |
| **Error behavior** | Scheduler failure stops new dispatches, drains started calls, rejects without fabricating results |
| **Retry or recovery behavior** | Abort drains started calls, records synthetic results for unstarted |
| **Owner (layer/package)** | `packages/core/agent-loop` (`tool-calls.ts`) + `packages/core/tools` |

#### Code Mode (`run_code`)

| Field | Value |
|---|---|
| **Feature** | Reserved transport tool: the model writes a program that calls SDK-declared tools |
| **Trigger or input** | `run_code` tool call under `mode: 'code'` |
| **Defaults** | `run_code` is the only directly-callable tool; other tools reachable only from inside the program |
| **Observable output** | `tool/code-dispatch-start` + `tool/code-dispatch` pairs per bridged sub-call |
| **Side effects** | Nested sub-dispatches re-enter the full guarded tool pipeline |
| **Persisted state** | `tool/code-dispatch` events |
| **Error behavior** | A model-direct call (no parent) to a native tool name is denied as `UNKNOWN_TOOL` |
| **Retry or recovery behavior** | N/A |
| **Owner (layer/package)** | `packages/core/tools` (`code-mode.ts`) |

### Surface Type: Session service (`ctx.sessions`)

#### Session lifecycle

| Field | Value |
|---|---|
| **Feature** | Event-sourced append-only session log + in-memory store |
| **Trigger or input** | `ctx.sessions.create()`, `fork()`, `resume()`, `Session.create()`, `Session.fromRestore()` |
| **Defaults** | `SESSION_FORMAT_VERSION` 0; minimal header synthesized when store-owned header absent |
| **Observable output** | `session/created`, `session/event`, `session/flush`, `session/disposed` events |
| **Side effects** | Append-only log growth; `deriveMessages()` projects model history from the log |
| **Persisted state** | Session log (JSONL or SQLite backend); header (format version, cwd, lineage, seed boundary) |
| **Error behavior** | Header/event validation rejects malformed seeds (version mismatch, non-absolute cwd, invalid origin, bad message shape) |
| **Retry or recovery behavior** | Torn-tail repair (`interruptedTurnClosers`, `TOOL_NOT_STARTED`, `TOOL_OUTCOME_UNKNOWN`) |
| **Owner (layer/package)** | `packages/core/session` |

#### Model-visible-means-logged invariant

| Field | Value |
|---|---|
| **Feature** | Anything reaching a model request must be reconstructable from the log |
| **Trigger or input** | Any new model-visible input |
| **Defaults** | New model-visible input requires a new session event |
| **Observable output** | Runtime invariant asserts it |
| **Side effects** | Extending `SessionEventMap` + rendering from the log |
| **Persisted state** | The event itself |
| **Error behavior** | Invariant violation fails loudly |
| **Retry or recovery behavior** | N/A |
| **Owner (layer/package)** | `packages/core/session` + `packages/runtime-diagnostics/invariants` |

### Surface Type: Credentials

#### Credential resolution

| Field | Value |
|---|---|
| **Feature** | Layered credential resolution |
| **Trigger or input** | `ctx.credentials.resolve(ref)` |
| **Defaults** | Precedence: inherited process env (read-only, wins) > `$DSH_HOME/.credentials.yaml` (writable) > `<cwd>/.env` > `$DSH_HOME/.env` |
| **Observable output** | `{ value, source }` or `undefined` |
| **Side effects** | None on read |
| **Persisted state** | `.credentials.yaml` (strict `CredentialRef`→string mapping) |
| **Error behavior** | Owner-only mode violation (POSIX) fails boot; invalid YAML fails activation; empty value rejected |
| **Retry or recovery behavior** | Hot-reload: external edits publish through the seam; reload failure keeps last good snapshot |
| **Owner (layer/package)** | `packages/credentials/credentials-local` |

## High-Value Behaviors

### Cancellation and abort handling
- Tool pipeline: `ABORTED` (after body) vs `ABORTED_BEFORE_DISPATCH` (before body); abort records synthetic results for skipped calls so replay stays valid.
- Subprocess: SIGTERM→SIGKILL escalation bounded by `graceMs`; tree-scoped signalling (POSIX groups, Windows taskkill); abort listener triggers `terminate()`.
- Session reads: `AbortSignal` checked between chunks; `readStableFile` retries while the stat revision changes.

### Streaming and partial output
- `assistant/chunk` delta events preserve replay/UI fidelity; packed `text-chunks`/`reasoning-chunks`/`tool-call-chunks` rows (~60% smaller).
- `OutputCollector` keeps a bounded in-memory tail + optional spill file; `TextRetainer`/`ItemRetainer` bound model-facing output with exact omission metadata.

### Queueing and follow-up behavior
- `ctx.jobs` generic background-job runtime; `job_*` tools collect/stop background bash, PTY sends, and subagents.
- `ctx.schedule` session-local scheduled follow-ups (`after_seconds`, absolute `at`, bounded `every_seconds`).
- Subagent `run_in_background` with automatic settlement delivery.

### Compaction and summarization
- `ctx.compaction` seam + `compaction-basic` provider + `command-compact` consumer; `compaction-tool-result-pruner`.

### Persistence and resume flows
- JSONL (append-only, zstd-compressed, torn-tail repair) and SQLite (monotonic `SCHEMA_VERSION` 17, application-id ownership check) backends.
- Fork/resume/transcripts/telemetry all derive from the session log.

### Tool execution and validation
- `defineTool` validates args against a JSON Schema; canonical output enforced against `output.schema`; `finalizeContent` last-mile transform; `presentCall`/`presentResult` pure replayable UI projections.

## Security and Authorization

- **Authentication**: API-key based (`DEEPSEEK_API_KEY`), resolved through the layered credential seam. No OAuth/SSO/session-token auth in the harness itself.
- **Authorization model**: capability-based — a tool/plugin is authorized by being registered in the composition; `tools/pre-execute` waterfall allows deny/ask; approval seam (`ctx.approval`) gates dispatch; `permission-presets` and `user-approval` packages.
- **Trust boundaries**: model output is untrusted (validated against tool schemas); the inherited process environment is the most-trusted credential layer; the invoking project's `.env` is trusted below the managed store.
- **Permission checks**: enforced in the tool pipeline (not just UI); `fs/observation-policy` gates read-before-write/edit; sandbox seam confines spawned processes (bwrap/Landlock/Seatbelt/ACL).
- **Secret management**: `.credentials.yaml` at `0600` (owner-only enforced on POSIX); YAML parse errors never quote the secret value; env-over-`.env` precedence.
- **Session security**: SQLite `trusted_schema=off`, `mmap_size=0`, `synchronous=FULL`, application-id ownership check; JSONL zstd checksummed frames.

## Configuration Model

- **Sources and precedence**: bundles (profile order) → profile `cordis.patch.yml` → home-level patch → `--patch` overlays. Credentials: process env > `.credentials.yaml` > project `.env` > home `.env`. Harness home: explicit config > `$DSH_HOME` > `~/.dsh`.
- **Format/location**: `cordis.yml` (with `!!js` interpolation), `cordis.patch.yml`, `dsh.profile`/`dsh.bundle` package.json fields; `.credentials.yaml` under harness home.
- **Feature flags**: `disabled: !!js` per entry; `enableRunInBackground`, `packChunks`, `compression`, `watch`, `debounceMs` etc. per plugin.
- **Validation**: Schemastery schemas validate plugin config; malformed config fails boot; SQLite schema-version mismatch rejects rather than migrates.
- **Detail**: see `findings/config-model/config-model.md`.

## Doc/Test Conflicts

None identified in the sampled surface. The generated `docs/tool-catalog.md` is boot-verified (`verify-tool-catalog` boots each tool plugin and reads `ctx.tools.schemas()`), so doc/code drift on tool schemas is gated in CI. The `docs/module-graph.md` is similarly freshness-gated.

## Black-Box Acceptance List

| # | Scenario | Precondition | Action | Expected Outcome |
|---|----------|--------------|--------|------------------|
| 1 | Boot web UI | Node ≥ 22.19, `pnpm install` | `dsh web` | HTTP server on `127.0.0.1:3080`; browser opens (local) |
| 2 | Dump config | A profile with bundles | `dsh --profile web --dump-config` | Prints the composed entry list without booting |
| 3 | Unknown tool | A model requests an unregistered tool | tool call | `UNKNOWN_TOOL` error result |
| 4 | Tool abort before dispatch | A step is aborted before a tool starts | abort | Synthetic `ABORTED_BEFORE_DISPATCH` result recorded; replay valid |
| 5 | Session fork | A live session | `ctx.sessions.fork(source)` | Child session with lineage header; log replayable |
| 6 | Credential precedence | `DEEPSEEK_API_KEY` set in env AND in `.credentials.yaml` | `resolve('DEEPSEEK_API_KEY')` | Env value wins, `source: 'env'`, `writable: false` |
| 7 | Credential write shadowed | `DEEPSEEK_API_KEY` in inherited env | `set('DEEPSEEK_API_KEY', v)` | Throws (shadowed by read-only env) |
| 8 | SQLite schema mismatch | A DB with `user_version` ≠ 17 | open | Rejects with version-mismatch error |
| 9 | JSONL torn tail | A session log with an incomplete final zstd frame | load | Recovers complete prior frames; torn tail marked for repair |
| 10 | Parallel tool calls | Two `isConcurrencySafe` tools | one step | Both dispatch concurrently; results commit in model order |

## Coverage and limits

- Inspected scope: `packages/core/tools/src/index.ts` (registry + pipeline + events), `packages/core/session/src/index.ts` (event-sourced store + validation), `docs/tool-catalog.md` (all model-facing tool contracts), `packages/credentials/credentials-local/src/index.ts`, `docs/architecture.md`, `packages/README.md`, `vendor/README.md`, `apps/cli` + `apps/web` + `python/sdk-runtime` manifests.
- Skipped scope: per-tool source bodies beyond the catalog's contract summaries; the `client/*` UI group's user-facing workflows; the ACP/JSON-RPC/Typert wire schemas (deferred to protocols); the Python SDK's turns API surface.
- Evidence basis: source inspection + generated catalogs + docs. No runtime verification.
- Known blind spots: exact wire schemas of RPC surfaces; UI workflow details; per-tool edge-case behavior beyond the catalog.
- Coverage disposition: COMPLETE (for the contract-level surface; wire-format detail deferred to protocols).

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| q-approval-default | needs-runtime-test | Whether the approval seam defaults to allow or ask when no approval provider is loaded (the `tools/pre-execute` JSDoc says "missing approval support turns `ask` into denial", but the default posture for a bare composition is not pinned from source alone). | Requires a runtime test against a minimal composition. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| con-CF1 | protocols | ACP, JSON-RPC, and Typert RPC wire schemas (request/response shapes, framing) not extracted. | Wire-format extraction is the protocols phase's rubric. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | User-facing surfaces are split by surface type. | PASS | §Surfaces Covered + §Feature Contracts grouped by CLI / model-facing tool surface / session / credentials. |
| 2 | Feature contracts record trigger, defaults, outputs, side effects, persisted state, error behavior, and recovery behavior. | PASS | Each contract table carries all seven fields. |
| 3 | Security and authorization model is documented (if applicable). | PASS | §Security and Authorization (API-key auth, capability-based authz, trust boundaries, secret management, session security). |
| 4 | Contract ownership is mapped back to a layer or package. | PASS | Each contract's "Owner (layer/package)" row. |
| 5 | A black-box acceptance list is included. | PASS | §Black-Box Acceptance List (10 scenarios). |
| 6 | Findings are marked with evidence levels. | PASS | Evidence basis stated in §Coverage and limits; contracts are `observed fact` from source/catalog/docs. |
| 7 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | §Coverage and limits. |

**Validated by:** contracts phase, session 1 (2026-08-19)
**Overall:** PASS
