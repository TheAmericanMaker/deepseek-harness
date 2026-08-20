# Public Surfaces

## 2026-08-19 — architecture phase

Enumerated public surfaces (summary; wire schemas deferred to protocols phase).

### Binaries and CLI commands
- `dsh` bin (`apps/cli/lib/bin.js`, package `@deepseek-ai/dsh`). Subcommands: `web` (Web UI at `http://127.0.0.1:3080`), `headless` (one-shot runner), `--profile <name>`, `--dump-config`, `--patch <overlay>`, `--no-open`, plugin management. Source launch: `node --import tsx/esm apps/cli/src/bin.ts`.

### Exported libraries and public types
- Every `@deepseek-ai/dsh-*` package exposes an ESM `exports` map (runtime `lib/` + declarations `lib/types`).
- Vendored framework: `@deepseek-ai/cordis` (Context, Service, Fiber, EventsService, RegistryService, ReflectService, LoggerService), `@deepseek-ai/cosmokit`, `@deepseek-ai/schemastery`, `@deepseek-ai/cordis-plugin-{loader,include,group,timer,hmr,logger-console}`.
- Core spine ctx keys: `ctx.sessions`, `ctx.systemPrompt`, `ctx.tools`, `ctx.agents`, `ctx.agentLoop`, `ctx.agentDefaultModel`, `ctx.llm`, `ctx.storage`, `ctx.sessionPersistence`, `ctx.sessionProjection`, `ctx.sessionTitle`, `ctx.settings`, `ctx.credentials`, `ctx.sandbox`, `ctx.fs`, `ctx.shell`, `ctx.terminals`, `ctx.subprocess`, `ctx.jobs`, `ctx.commands`, `ctx.goals`, `ctx.agentTeams` (experimental).

### Network / RPC interfaces
- Web UI HTTP server (`packages/host/webserver` + `packages/host/apiproxy`).
- ACP (Agent Client Protocol) server (`packages/acp/acp`, `@agentclientprotocol/sdk` 0.25.1).
- JSON-RPC over stdio (`packages/sdk/*`: protocol, server, client).
- Typert RPC gateway (`packages/api/gateway`, `packages/api/remotes`).
- MCP client (`packages/mcp/mcp-client`).
- Hook bridges: Claude Code + Codex (`packages/hooks/*`).

### File formats and persistent artifacts
- `cordis.yml` (with `!!js` interpolation), `cordis.patch.yml` overlays.
- `dsh.profile` / `dsh.bundle` package.json fields.
- Session logs: JSONL (`SESSION_FORMAT_VERSION` 0) and SQLite (`SCHEMA_VERSION` monotonic).
- Storage: JSON + SQLite backends.
- Generated catalogs: config-catalog, tool-catalog, persistence-catalog, cordis-catalog, module-graph.

### User-facing screens / workflows
- Web UI (browser app): conversation, settings, plugin inventory, agent presets, subagent, workflow-run, trajectory, deliverables, goal, plan, skill, jobs, theme, sidebar, workspace.
- Headless one-shot runner.
- Python SDK turns API (`deepseek_harness`).

## 2026-08-19 — contracts phase

Contract-level additions (behavioral contracts recovered in `findings/contracts/behavioral-contracts.md`):
- Model-facing tool surface: 20+ tool packages enumerated in `docs/tool-catalog.md` (ask_user_question, run_code, bash, pwsh, str_replace_editor, edit/read/read_image/write, glob/grep, terminal_*, create_goal/get_goal/update_goal, schedule_*, lsp, ralph, skill, session_event_*, subagent, interrupt_agent/list_agents/send_message, report, job_*, todo_write, workflow, web_fetch/web_search, plus experimental agent-team tools).
- Tool execution pipeline: `tools/pre-execute` → `tools/execute` → body → `tools/post-execute` → `tools/result`; `tools/code-dispatch-log` for Code Mode.
- Session events: `session/created`, `session/event`, `session/flush`, `session/disposed`.

## 2026-08-19 — protocols phase

Wire-format additions (full schemas in `findings/protocols/protocols-and-state.md`):
- SDK JSON-RPC: `initialize`, `session/prompt`, `shutdown` requests; `session.event`, `session.status`, `subagent.started`, `subagent.finished` notifications; newline-delimited JSON-RPC 2.0 over stdio.
- ACP: `initialize`/`newSession`/`prompt`/`cancel`; `agent_message_chunk` session updates; `requestPermission` one-shot allow-once/reject-once.
- Typert RPC: `Remote`/`RemoteScope` decorators + `bindTypertRemote` Gateway bindings; endpoint segment grammar `/^[A-Za-z0-9_$.-]+$/`.

## 2026-08-19 — porting phase

Synthesis-level additions (consolidated in `findings/porting/reverse-engineering-bundle.md`):
- Delivery surfaces ranked for porting: CLI + headless (core), SDK JSON-RPC + ACP (important), Web UI + Python SDK + native launcher (optional).
- The api-gateway transport framing and client↔host wire protocol remain unextracted (deep-read triggers).
