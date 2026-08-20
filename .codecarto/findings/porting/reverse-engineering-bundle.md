# Reverse-Engineering Bundle

## System Summary

DeepSeek Harness (`dsh`) is a plugin-based agent harness built on a source-vendored copy of the Cordis framework. Its central design principle is **"everything is a plugin"**: the model adapter, tool registry, session log, agent loop, persistence, sandboxing, and UI are all plugins that contribute services, typed events, and reversible effects to a shared context. A running `dsh` is a plugin tree composed at boot from ordered layers (bundles → profile patches → home patches → `--patch` overlays); there is no privileged core to patch.

The system's heart is an **event-sourced session log** with a hard invariant: *model-visible means logged* — anything reaching a model request must be reconstructable from the log. The agent loop runs a turn/step state machine (`turn/start → agent/pre-step → step/start → agent/request → llm/stream → assistant/chunk* → tool/call* → tools/* → step/end → agent/turn-stopping → turn/end`). Tools execute through a guarded pipeline (`tools/pre-execute → tools/execute → body → tools/post-execute → tools/result`) with parallel/exclusive scheduling and ordered result commit.

The system ships through multiple delivery surfaces: a `dsh` CLI with a Web UI, a headless one-shot runner, an ACP automation server, a JSON-RPC SDK (TypeScript + Python), and a Python SDK that drives the harness as a subprocess. It is a 233-package pnpm monorepo (~483K LOC TypeScript) in developer preview (v0.1.0-rc.8) with explicit compatibility-breaking-change posture.

## Source Index

| Area | Canonical upstream section | Summary carried forward | Deep-read trigger |
|---|---|---|---|
| Architecture | `findings/architecture/architecture-map.md` §Layer Map, §Porting Priorities | Layering: vendored Cordis → zero-dep util → core spine → capability seams → bundles/shells; acyclic peer-dependency graph | A package's role is ambiguous or a new layer is proposed |
| Contracts | `findings/contracts/behavioral-contracts.md` §Feature Contracts, §Security and Authorization | CLI/tool/session/credential contracts; capability-based authz; layered credential precedence | A feature's trigger/defaults/error behavior needs exact detail |
| Protocols and state | `findings/protocols/protocols-and-state.md` §Event Catalog, §State Machine, §Persistent Schema Notes | SDK JSON-RPC + ACP + Typert RPC wire formats; turn/step + tool-pipeline state machines; JSONL/SQLite schemas | A wire schema or state transition needs exact shape |
| Defects | `findings/defect-scan-mechanical/mechanical-defects.md` + `findings/defect-scan-semantic/semantic-defects.md` | 16 findings (0 critical, 0 high, 5 medium, 11 low); dispositions below | A defect's rationale or fix detail is needed |

## Layer Map With Ownership

| Layer / Module | Role | Owns |
|---|---|---|
| Vendored Cordis framework (`vendor/*`) | core semantics (framework) | `Context`, `Service`, `Fiber`, `EventsService`, `RegistryService`; Loader/Include/HMR/Group/Timer plugins |
| Zero-dep utilities (`packages/util/*`) | core semantics (utilities) | `Branded<B>`, `resolveDshHome`, `withTimeout`, `writeFileAtomic`, `withFileLock`, retention, launch-environment snapshot |
| Product API spine (`packages/core/*`) | core semantics (spine) | `ctx.sessions`, `ctx.systemPrompt`, `ctx.tools`, `ctx.agents`, `ctx.agentLoop`, `ctx.agentDefaultModel` |
| Typert (`packages/typert/*`) | protocol / normalization | type-graph generator, loader, runtime registry, Remote/Gateway protocol |
| LLM seam (`packages/llm/*`) | integration adapter | `ctx.llm` adapter seam; DeepSeek/pi-ai providers; retry policy |
| Capability seams (fs, shell, subprocess, terminal, lsp, web, skill, compaction, subagent, workflow, code-runtime, sandbox, e2b, spill, attachment, jobs, schedule, goal, context, credentials, settings, identity, workspace, plan, preset, todo, guard, feedback, interaction, hooks, extensions, mcp, acp, sdk, api, session-query, storage, session) | integration adapter | Service Definition / Provider / Consumer per seam |
| Composition bundles (`packages/bundle/*`) | product shell | `dsh.profile`/`dsh.bundle` patch layers (base, headless, web-app) |
| Boot glue (`packages/boot/*`) | product shell | profile boot, CLI arg parsing |
| Web host (`packages/host/*`) | product shell | HTTP route server, API gateway, frontend static |
| Web client (`packages/client/*`) | UI / rendering | browser shell, `ui-*` plugins |
| App assemblies (`apps/cli`, `apps/web`) | product shell | `dsh` bin; Vite web build |
| Python SDK + runtime (`python/*`) | integration adapter | `deepseek_harness` module; bundled runtime exe |
| Native launcher (`native/landlock-run/*`) | integration adapter | Landlock self-restrict-then-exec launcher |

## Feature Contract Table

| Feature | Surface | Priority | Key Contracts | Notes |
|---|---|---|---|---|
| Plugin tree composition (bundles → patches → overlays) | boot | core | `dsh.profile`/`dsh.bundle`; patch-by-id replace/insert; `!!js` interpolation | Vendored Include/Loader/HMR reconcile transactionally |
| Event-sourced session log | core | core | append-only; `SESSION_FORMAT_VERSION` 0; model-visible-means-logged | Fork/resume/telemetry derive from it |
| Agent turn/step loop | core | core | turn/step state machine; waterfall events | `agent/pre-step`, `agent/request`, `llm/stream`, `tools/*` |
| Tool registry + execution pipeline | core | core | `defineTool`; pre/execute/post/result waterfalls; parallel/exclusive | Defect D3.2 (scheduler-failure surfacing) |
| Code Mode (`run_code`) | tool | important | reserved transport; SDK-declared tools | Nested sub-dispatches re-enter the pipeline |
| Session persistence (JSONL/SQLite) | persistence | core | zstd frames; torn-tail repair; `SCHEMA_VERSION` 17 | Defect D1.2/D3.1 (unbounded retry) |
| Credential resolution | credentials | important | env > `.credentials.yaml` > project `.env` > home `.env` | Owner-only 0600 (POSIX) |
| Sandbox seam (bwrap/Landlock/Seatbelt/ACL) | sandbox | important | process confinement | OS-specific |
| Subagent delegation | subagent | important | provider registry; continuable vs one-shot | `subagent`/`subagent_fork` |
| Web UI | web | optional | browser app; host webserver | Not needed for first headless port |
| SDK JSON-RPC | sdk | important | newline-delimited JSON-RPC 2.0; 3 requests + 4 notifications | Defect D5.1 (params not runtime-validated) |
| ACP server | acp | important | automation-only; one prompt in flight | No auth (stdio trust boundary) |
| Python SDK | python | optional | turns API + JSON-RPC client | Independent surface |
| Native Landlock launcher | native | optional | Linux-only | — |

## Protocol and State Notes

- **SDK JSON-RPC**: newline-delimited JSON-RPC 2.0 over stdio. Requests `initialize`/`session/prompt`/`shutdown`; notifications `session.event`/`session.status`/`subagent.started`/`subagent.finished`. Malformed lines ignored; `-32601`/`-32603` error codes.
- **ACP**: `initialize`/`newSession`/`prompt`/`cancel`; one prompt in flight per session; `settleAfterQuiescence` waits admission → `whenIdle()` → `outputTail`; `quiesce` drains continuable descendants child-first.
- **Typert RPC**: `Remote`/`RemoteScope` decorators + `bindTypertRemote`; endpoint segment grammar `/^[A-Za-z0-9_$.-]+$/`.
- **Session event stream**: `SessionEventMap` (13 core types); envelope `type`/`seq`/`time`/`data` + conditional `surfaceOp`/`sourceEventSeqs`/`ignorable`; surface ops `append`/`replace`; turn-end reasons `completed`/`aborted`/`blocked`/`error`/`max-tokens`/`interrupted`.
- **Persistence**: JSONL (zstd checksummed frames, packed chunk rows, torn-tail marker) and SQLite (`SCHEMA_VERSION` 17, application-id `0x44534850`, `trusted_schema=off`, `mmap_size=0`, `synchronous=FULL`).

## Portability Hazards

| Hazard | Source Phase | Impact | Mitigation |
|---|---|---|---|
| JSON-RPC newline framing (malformed lines silently ignored) | protocols | medium | Preserve the ignore-malformed/error-on-handler-failure contract |
| `SESSION_FORMAT_VERSION` 0, reject-not-migrate | protocols | high | Reproduce reject-not-migrate; no silent skip |
| SQLite `trusted_schema`/`mmap_size`/`synchronous` pragmas | protocols | medium | Re-verify pragmas on the target SQLite binding |
| zstd frame format + torn-tail recovery (custom, not standard stream) | protocols | high | Reimplement frame scanning + prefix decompression |
| `AbortSignal.any`/`Symbol.dispose`/`Promise.withResolvers` (ES2024+) | protocols | medium | Polyfill or target a modern runtime |
| Windows env case-folding + `taskkill /T /F` | protocols | medium | Branch on OS for process-tree termination |
| POSIX process-group signalling (`kill(-pid)`) | protocols | medium | No Windows equivalent; branch |
| `node:sqlite` `DatabaseSync` (synchronous API) | protocols | medium | Async SQLite changes the concurrency model |
| Cordis fiber/effect lifecycle + waterfall events | architecture | high | Reimplement or replace with an equivalent DI/effect system |
| PTY/process-tree providers (node-pty, ConPTY) | architecture | medium | OS-specific; not 1:1 portable |

## Defect Synthesis

| Defect ID | Source Report | One-line Description | Severity | Disposition | Required design consequence |
|-----------|---------------|----------------------|----------|-------------|-----------------------------|
| D1.1 | mechanical | `clampTimeout` unvalidated `def`/`max` → negative timeout | medium | fix before porting | Validate backend defaults/caps; never emit non-positive timeouts |
| D1.2 | mechanical | `readStableFile` unbounded retry loop | medium | fix before porting | Bound retries + backoff on revision-stable reads |
| D1.3 | mechanical | `parseArguments` preserves invalid JSON as raw string | low | port differently | Coerce invalid JSON to `{}` or a typed error |
| D1.4 | mechanical | `trimTrailingPartialUtf8` 4-byte-lead boundary subtlety | low | leave behind | Add a targeted UTF-8 boundary test |
| D2.1 | mechanical | `taskkillProcessTree` silent failure | low | port differently | Surface tree-termination failure diagnostics |
| D2.2 | mechanical | `runNativeCommand` abort path violates stdout/stderr contract | low | fix before porting | Make stdout/stderr optional on the abort path |
| D6.1 | mechanical | `preparedSessionCacheSize` unbounded | low | fix before porting | Add an upper bound |
| D6.2 | mechanical | credentials owner-only check POSIX-only | low | port differently | Document/verify Windows ACL equivalent |
| D6.3 | mechanical | storage DB confidentiality rests on trusted parent dir | low | leave behind | Document the parent-dir trust assumption |
| D3.1 | semantic | `readStableFile` unbounded retry (concurrency framing) | medium | fix before porting | Same as D1.2 — bound + backoff |
| D3.2 | semantic | `runGroup` deferred scheduler-failure surfacing | medium | fix before porting | Propagate terminal dispatch failure immediately |
| D3.3 | semantic | fire-and-forget `handleLine` unhandled rejection | low | fix before porting | `.catch` on the fire-and-forget call |
| D4.1 | semantic | storage DB parent-dir trust (security framing) | low | port differently | Same as D6.3 |
| D4.2 | semantic | ACP no-auth trust boundary | low | port differently | Preserve or explicitly tighten the stdio trust boundary |
| D5.1 | semantic | JSON-RPC wire params not runtime-validated | medium | fix before porting | Runtime-validate wire params against the typed map |
| D5.2 | semantic | `runNativeCommand` return-type inconsistency | low | fix before porting | Same as D2.2 |

**Net**: 16 findings, 0 critical, 0 high, 5 medium, 11 low. The reimplementation must design around the 5 medium findings (timeout validation, bounded retry, scheduler-failure propagation, wire-param validation) and the `SESSION_FORMAT_VERSION`/zstd compatibility hazards.

## Observed Facts vs. Inferred Structure

### Observed Facts

- 233 npm packages in `packages/<group>/<pkg>`; ~483K LOC TypeScript; pnpm 11.7.0 workspaces.
- `SESSION_FORMAT_VERSION` 0; SQLite `SCHEMA_VERSION` 17; application-id `0x44534850`.
- SDK JSON-RPC: 3 requests (`initialize`, `session/prompt`, `shutdown`) + 4 notifications.
- ACP `authenticate` is a no-op; `authMethods: []`.
- Credential precedence: process env > `.credentials.yaml` > project `.env` > home `.env`.
- Tool pipeline: `tools/pre-execute` → `tools/execute` → body → `tools/post-execute` → `tools/result`.
- `dsh` CLI serves Web UI at `127.0.0.1:3080` by default.

### Inferred Structure

- The layering (vendored framework → util → spine → seams → shells) is acyclic (from the generated peer-dependency graph).
- The "everything is a plugin" design means the reimplementation's module boundaries should follow capability seams, not the source folder layout.
- The event-sourced session log is the single source of truth; all derived state (history, telemetry, UI) projects from it.

## Domain Glossary

| Term | Definition | Where Used |
|---|---|---|
| Cordis | The vendored framework: plugins contribute services, typed events, reversible effects to a shared context | `vendor/cordis` |
| Seam | A swappable capability with three roles: Service Definition, Service Provider, Consumer | `packages/*` |
| Bundle | A distribution format for Cordis config rows + code (`dsh.bundle`) | `packages/bundle/*` |
| Profile | A named composition of bundles (`dsh.profile`) | `packages/bundle/*` |
| Turn / Step | A turn is zero+ steps; a step is one model request + its tool calls | `packages/core/agent-loop` |
| Surface | The ordered model-visible message surface over the event log | `packages/core/session` |
| SurfaceOp | How an event entered the surface: `append` or `replace` | `packages/core/session` |
| Torn tail | An incomplete final zstd frame in a JSONL log, repaired on load | `packages/session/session-persistence-jsonl` |
| Code Mode | `run_code` reserved transport: model writes a program calling SDK tools | `packages/core/tools` |
| Typert | Type-graph generator + runtime registry + Remote/Gateway RPC | `packages/typert/*` |
| Landlock | Linux self-restrict-then-exec sandbox launcher | `native/landlock-run` |

## Coverage and limits

- Inspected scope: all five upstream reports (architecture, mechanical defects, contracts, protocols, semantic defects) synthesized into this bundle.
- Skipped scope: the `packages/api/*` gateway transport framing and the client↔host wire protocol (flagged as deep-read triggers); the Python SDK and native launcher sources.
- Evidence basis: source inspection + generated catalogs + docs. No runtime verification.
- Known blind spots: runtime-only races; the api-gateway transport; the full 233-package surface not exhaustively read.
- Coverage disposition: COMPLETE (the bundle is a self-contained compression boundary; residual gaps are named as deep-read triggers).

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| q-approval-default | needs-runtime-test | Whether the approval seam defaults to allow or ask when no approval provider is loaded. | Requires a runtime test against a minimal composition. |
| q-api-gateway-transport | needs-runtime-test | The exact Typert Gateway transport framing (HTTP vs stdio) in `packages/api/*`. | Requires reading the api group source or a runtime capture. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| por-CF1 | reimplementation-spec | The api-gateway transport framing and client↔host wire protocol remain unextracted; the spec should treat them as known unknowns rather than pinning a transport. | The reimplementation-spec's acceptance scenarios should name these as open rather than assume a transport. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | The system summary, layer map, contract table, protocol notes, and porting findings are synthesized. | PASS | §System Summary, §Layer Map, §Feature Contract Table, §Protocol and State Notes. |
| 2 | Portability hazards and open questions are separated from facts. | PASS | §Portability Hazards + §Open Questions vs §Observed Facts. |
| 3 | Feature importance is sorted for porting. | PASS | §Feature Contract Table (core/important/optional/incidental). |
| 4 | Known defects are referenced in the Defect Synthesis with porting recommendations. | PASS | §Defect Synthesis (16 findings with fix-before-porting/port-differently/leave-behind). |
| 5 | Findings are marked with evidence levels. | PASS | §Observed Facts vs. Inferred Structure. |
| 6 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | §Coverage and limits. |
| 7 | The Source Index makes the bundle a self-contained compression boundary and identifies targeted deep-read triggers. | PASS | §Source Index (4 areas with deep-read triggers). |

**Validated by:** porting phase, session 1 (2026-08-19)
**Overall:** PASS
