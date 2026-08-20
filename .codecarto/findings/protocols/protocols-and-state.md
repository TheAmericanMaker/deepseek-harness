# Protocols and State

## Boundaries Identified

- **process ↔ process**: SDK runtime JSON-RPC over stdio (server plugin ↔ TS/Python clients); ACP server over JSON-RPC stdio; subprocess/PTY/LSP providers.
- **UI ↔ core**: Web UI ↔ host webserver ↔ API gateway (Typert RPC); client wire protocol.
- **core ↔ provider**: `ctx.llm` adapter seam (DeepSeek, pi-ai providers); `ctx.web` search/fetch providers; `ctx.shell`/`ctx.terminals`/`ctx.lsp`/`ctx.subprocess` providers.
- **tool layer ↔ runtime**: `ctx.tools` registry + guarded execution pipeline (`tools/pre-execute` → `tools/execute` → body → `tools/post-execute` → `tools/result`).
- **runtime ↔ persistence**: session log (JSONL/SQLite), storage (JSON/SQLite), credentials YAML.
- **local files ↔ exported artifacts**: session logs, generated catalogs, `cordis.yml`/`cordis.patch.yml`.

## Event Catalog

### SDK Runtime JSON-RPC (newline-delimited JSON-RPC 2.0 over stdio)

| Field | Value |
|---|---|
| **Producer** | `@deepseek-ai/dsh-sdk-jsonrpc-server` (runtime server plugin) |
| **Consumer** | `@deepseek-ai/dsh-sdk-client` (TS), Python SDK (`deepseek_harness`) |
| **Transport or carrier** | Newline-delimited JSON-RPC 2.0 over byte streams (`JsonRpcLineTransport`) |
| **Ordering guarantees** | Per-frame; requests carry a `req_<uuid>` id; responses matched by id |
| **Required fields** | Frame: `jsonrpc: '2.0'`; request: `id` + `method`; response: `id` + `result`/`error`; notification: `method` only |
| **Optional fields** | `params` (object; arrays/scalars collapse to `{}`) |
| **Identifiers and timestamps** | Request id `req_<uuid>` (dashes stripped); no timestamps on the wire |
| **Error cases** | `-32601` method not found; `-32603` handler failure; `JsonRpcResponseError` preserves wire `code`/`data`; malformed lines ignored |
| **Restart or resume behavior** | `close()` rejects pending requests; `shutdown` request returns `{}` |

**Request/result pairs** (`HarnessSdkRequestMap`):
- `initialize` → `{ cwd, provider, model, maxTokens? }` → `{ serverInfo: { name: 'deepseek-harness-sdk-runtime', version } }`
- `session/prompt` → `{ sessionId, contentBlocks }` → `{ messageId }` (unknown id lazily creates agent+session)
- `shutdown` → `undefined` → `{}`

**Notifications** (`HarnessSdkNotificationMap`):
- `session.event` → `{ sessionId, event: SessionEvent }` (streamed as recorded)
- `session.status` → `{ sessionId, status: 'idle' | 'running' }`
- `subagent.started` → `{ parentSessionId, childSessionId }`
- `subagent.finished` → `{ provider, agentId, parentSessionId, childSessionId, status: 'ok'|'error', stopReason, lastAssistantMessage? }`

### ACP (Agent Client Protocol) server

| Field | Value |
|---|---|
| **Producer** | `@deepseek-ai/dsh-acp` (automation-only server) |
| **Consumer** | ACP clients (`@agentclientprotocol/sdk` 0.25.1), e.g. `dsh-subagent-acp` |
| **Transport or carrier** | JSON-RPC stdio (`ndJsonStream` over stdin/stdout) |
| **Ordering guarantees** | Per-session ordered assistant-output chain (`outputTail`); one prompt in flight per session |
| **Required fields** | `initialize` → `{ protocolVersion, agentInfo, agentCapabilities, authMethods }`; `newSession` → `{ sessionId }`; `prompt` → `{ stopReason }` |
| **Optional fields** | `promptCapabilities.image` (provider-dependent), `audio: false`, `embeddedContext: false` |
| **Identifiers and timestamps** | `sessionId` = `SessionId(randomUUID())`; `agentInfo.name = 'deepseek-harness-acp'`, version `0.0.1` |
| **Error cases** | `RequestError.invalidParams` (unknown session, prompt already in flight), `RequestError.internalError` (turn failed, output delivery failed, disposed bridge) |
| **Restart or resume behavior** | `cancel` sets `cancelRequested` + aborts admission; `quiesce` drains continuable descendants child-first, then disposes agents |

**Stop reasons** (`turnEndToStopReason`): `completed` → `end_turn`; `aborted` → `cancelled`; `max-tokens` → `end_turn`; `error` → internal error; `blocked`/`interrupted` mapped per codec.

### Typert RPC (Remote decorators + Gateway bindings)

| Field | Value |
|---|---|
| **Producer** | `@deepseek-ai/dsh-typert-protocol` (Remote/RemoteScope decorators, `bindTypertRemote`) |
| **Consumer** | `@deepseek-ai/dsh-api-gateway` (Remote BFF), `@deepseek-ai/dsh-api-remotes` |
| **Transport or carrier** | Shared RPC carrier; endpoint segments validated by `isTypertRemoteSegment` (`/^[A-Za-z0-9_$.-]+$/`, not `.`/`..`) |
| **Ordering guarantees** | Remote methods marked in class declaration order (`remoteMethods`) |
| **Required fields** | `bindTypertRemote(service, serviceKey, { namespace? })`; `Remote`/`RemoteScope` markers on public instance methods |
| **Optional fields** | `exportName` (distinct endpoint method); `namespace` (defaults to service key) |
| **Identifiers and timestamps** | Service key = Cordis service key; namespace = wire namespace |
| **Error cases** | `TypertLookupFailure` (adapter-owned failure preserved); conflicting invocation markers throw; invalid segment chars throw `TypeError` |
| **Restart or resume behavior** | Markers are private module state (WeakMap); no persistence |

### Session event stream (`SessionEventMap`)

| Field | Value |
|---|---|
| **Producer** | `@deepseek-ai/dsh-session` (`Session.append`) |
| **Consumer** | Persistence backends, telemetry, UI, ACP bridge, SDK server |
| **Transport or carrier** | In-process `session/event` firehose + durable log (JSONL/SQLite) |
| **Ordering guarantees** | Monotonic `seq` per session; contiguous; append-only |
| **Required fields** | Envelope: `type`, `seq`, `time`, `data` |
| **Optional fields** | `surfaceOp`/`sourceEventSeqs` (surface events only), `ignorable: true` (informational records) |
| **Identifiers and timestamps** | `seq` (monotonic), `time` (Unix epoch ms) |
| **Error cases** | Non-serializable `meta` rejected at source (`isJsonValue`); unrecognized required event type → reader must refuse reconstruction |
| **Restart or resume behavior** | `session/end-seed` marks seed boundary; `firstLiveSeq` = first in-process seq; torn-tail repair |

**Event types** (core): `turn/start`, `turn/end`, `step/start`, `step/end`, `user/message`, `assistant/chunk`, `assistant/message`, `tool/call`, `tool/result`, `todo/write`, `request/header`, `request/context`, `session/end-seed`. Surface-eligible: `user/message`, `assistant/message`, `tool/result`.

## State Machine

### Agent turn/step loop

| Current State | Event / Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| (idle) | queued input claimed | — | turn/start | `turn/start` event appended |
| turn/start | `agent/pre-step` | reject or empty first claim | turn/end (no step) | durable turn with no step |
| turn/start | `agent/pre-step` enter(messages) | — | step/start | `step/start`; `user/message` appended |
| step/start | `agent/request` → `llm/stream` | — | assistant streaming | `assistant/chunk*` → `assistant/message` |
| assistant streaming | `tool/call*` | — | tool execution | `tools/pre-execute` → `tools/execute` → `tools/post-execute` → `tool/result*` |
| tool execution | tools owe another request / next-step input | — | next step | `step/end` → `step/start` |
| step/end | no more owed | — | `agent/turn-stopping` | serial, no `next()` |
| turn-stopping | — | — | turn/end | `turn/end` with `TurnEndReason` |

**Turn end reasons** (`TurnEndReasonMap`): `completed`, `aborted` (with `TurnEndCancelCause`), `blocked`, `error` (with `LlmFailure`), `max-tokens`, `interrupted` (persistence-closed crash-orphaned turn).

### Tool execution pipeline

| Current State | Event / Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| pending | `tools/pre-execute` | allow / deny / ask | dispatch or denied | waterfall; `next()` delegates to allow |
| dispatch | `tools/execute` | — | body | around-dispatch waterfall; may replace `exec.signal` |
| body | tool `execute()` | — | result | canonical lossless-JSON value |
| result | `tools/post-execute` | — | materialized | accept/replace/enrich/block |
| materialized | `tools/result` | — | done | deep-frozen snapshot; `finalizeContent` last-mile |

## Persistent Schema Notes

- **Session log (JSONL)**: append-only, one file per session; header line + contiguous events; zstd-compressed frames (checksummed) or plaintext; packed `text-chunks`/`reasoning-chunks`/`tool-call-chunks` rows (~60% smaller); torn-tail repair truncates + restores recovered events + appends synthetic closers. `SESSION_FORMAT_VERSION` 0 (no compatibility promise).
- **Session log (SQLite)**: `SCHEMA_VERSION` 17; application-id `0x44534850`; `trusted_schema=off`, `mmap_size=0`, `synchronous=FULL`; `user_version` stamp; schema ownership validated against a canonical in-memory reference; `validateSchemaForMutation` rechecks before each mutation.
- **Storage (SQLite)**: `STORAGE_SQLITE_SCHEMA_VERSION` 1; `units` + `unit_globals` tables; per-unit record tables; `user_version` stamp; journal modes `wal`/`delete`/`truncate`/`persist`.
- **Credentials**: `.credentials.yaml` strict `CredentialRef`→string mapping; 0600 owner-only (POSIX); atomic write + cross-process writer lock; hot-reload via chokidar.
- **Branching**: fork/resume derive from the append-only log; `parentSession`/`seedLength`/`delegationDepth` in the header record lineage.
- **Compaction**: `ctx.compaction` seam; surface `replace` op shadows source events (`sourceEventSeqs` must include every shadowed node).

## Compatibility Hazards

| Hazard | Where It Appears | Severity | Notes |
|---|---|---|---|
| JSON-RPC framing (newline-delimited) | `packages/sdk/protocol/src/transport.ts` | medium | Malformed lines silently ignored; a port must preserve the "ignore malformed, error on handler failure" contract. |
| `SESSION_FORMAT_VERSION` 0 with no migration | `packages/core/session/src/types.ts` | high | Backends reject any other version; a port must reproduce the reject-not-migrate behavior. |
| SQLite `trusted_schema`/`mmap_size`/`synchronous` pragmas | `packages/session/session-persistence-sqlite/src/schema.ts` | medium | Node's `node:sqlite` specifics; a port to another SQLite binding must re-verify these pragmas. |
| zstd frame format + torn-tail recovery | `packages/session/session-persistence-jsonl/src/zstd.ts` | high | Custom frame scanning + prefix decompression; not a standard zstd stream. |
| `AbortSignal.any` / `Symbol.dispose` / `Promise.withResolvers` | across `util/timeout`, `sdk/protocol`, `acp` | medium | Node 22+ / ES2024+ features; a port to an older runtime must polyfill. |
| Windows env case-folding + `taskkill /T /F` | `subprocess-local`, `launch-environment` | medium | OS-specific process-tree termination and env semantics. |
| POSIX process-group signalling (`kill(-pid)`) | `subprocess-local/src/spawn.ts` | medium | No Windows equivalent; port must branch. |
| `node:sqlite` `DatabaseSync` (synchronous API) | `storage-sqlite`, `session-persistence-sqlite` | medium | Synchronous SQLite; a port to async SQLite changes the concurrency model. |

## Coverage and limits

- Inspected scope: `packages/sdk/protocol/src/{index,types,transport}.ts` (full JSON-RPC wire format), `packages/typert/protocol/src/index.ts` (Remote/Gateway binding), `packages/acp/acp/src/index.ts` (ACP server), `packages/core/session/src/types.ts` (SessionEventMap + header + turn-end reasons), `packages/session/session-persistence-{jsonl,sqlite}/src/schema.ts`, `packages/storage/storage-sqlite/src/schema.ts`.
- Skipped scope: `packages/api/gateway` + `packages/api/remotes` source bodies (Typert RPC transport implementation beyond the protocol package); the client wire protocol (`packages/client/*`); the Python SDK's exact framing (mirrors the TS protocol).
- Evidence basis: source inspection (type definitions + transport implementation). No runtime verification.
- Known blind spots: the exact Typert Gateway transport framing (HTTP vs stdio) is in `packages/api/*`, not read in full; the client↔host wire protocol is not extracted.
- Coverage disposition: COMPLETE (for the SDK/ACP/Typert-protocol/session-event wire formats; the api-gateway transport detail is a minor gap).

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| q-api-gateway-transport | needs-runtime-test | The exact Typert Gateway transport framing (HTTP route shape vs stdio) lives in `packages/api/gateway` + `packages/api/remotes`, which were not read in full. | Requires reading the api group source or a runtime capture. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| pro-CF1 | porting | The api-gateway transport framing (HTTP vs stdio) and the client↔host wire protocol are not fully extracted; the porting bundle should note them as targeted deep-read triggers. | Synthesis-level resolution; the porting bundle's Source Index is the right place to flag deep-read triggers. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | An event catalog is documented. | PASS | §Event Catalog (SDK JSON-RPC, ACP, Typert RPC, session event stream). |
| 2 | A state machine is documented. | PASS | §State Machine (turn/step loop + tool execution pipeline). |
| 3 | Persistent schema notes are documented. | PASS | §Persistent Schema Notes (JSONL/SQLite/storage/credentials). |
| 4 | Compatibility hazards are documented. | PASS | §Compatibility Hazards (8 hazards). |
| 5 | Findings are marked with evidence levels. | PASS | Evidence basis in §Coverage and limits; wire formats are `observed fact` from type definitions. |
| 6 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | §Coverage and limits. |

**Validated by:** protocols phase, session 1 (2026-08-19)
**Overall:** PASS
