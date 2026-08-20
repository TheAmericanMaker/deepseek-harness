# Architecture Map

## System Intent

DeepSeek Harness (`dsh`) is an open-source agent harness developed by DeepSeek AI. It is a plugin-based runtime for building and running LLM agents, built on a source-vendored copy of the Cordis framework. The core design principle is **"everything is a plugin"**: the model adapter, the tool registry, the session log, the agent loop, persistence, sandboxing, and the UI are all plugins that contribute services, typed events, and reversible effects to a shared context. There is no privileged core to patch — a running `dsh` is a plugin tree composed at boot from ordered layers (bundles → profile patches → home patches → `--patch` overlays). The system targets developers who want a composable, auditable, patchable agent runtime, and ships through multiple delivery surfaces: a `dsh` CLI with a Web UI, a headless one-shot runner, an ACP (Agent Client Protocol) automation server, a JSON-RPC SDK (TypeScript + Python), and a Python SDK that drives the harness as a subprocess. It is in developer preview (v0.1.0-rc.8) with explicit compatibility-breaking-change posture.

## Layer Map

The repository is a pnpm monorepo with 233 npm packages organized as `packages/<group>/<pkg>`, plus a vendored framework layer, two application assemblies, a Python SDK, a native launcher, and a docs site. The dependency direction is strictly layered: **vendored framework → zero-dependency utilities → product API spine → capability seams → product shells**.

### Package Inventory

| Package / Module | Role | Public Entrypoints | Key Dependencies | Runtime Surface |
|---|---|---|---|---|
| `vendor/cordis` (`@deepseek-ai/cordis`) | core semantics (framework) | `Context`, `Service`, `Fiber`, `EventsService`, `RegistryService` | none (base) | Node ESM |
| `vendor/cosmokit`, `schemastery`, `loader`, `include`, `group`, `timer`, `hmr`, `logger-console` | core semantics (framework plugins) | Loader/Include/HMR/Group/Timer services | cordis | Node ESM |
| `packages/util/*` (brand, home-paths, timeout, atomic-write, native-command, launch-environment, output-retention) | core semantics (zero-dep utilities) | `Branded<B>`, `resolveDshHome`, `withTimeout` | none (harness-dep-free) | Node ESM |
| `packages/core/*` (scope, session, system-prompt, tools, agent, agent-default-model, agent-loop, agent-tool-presentation) | core semantics (product API spine) | `ctx.sessions`, `ctx.systemPrompt`, `ctx.tools`, `ctx.agents`, `ctx.agentLoop` | cordis, util, llm, typert-protocol | Node ESM |
| `packages/typert/*` (protocol, generator, loader, registry) | protocol / normalization layer | type-graph generator, runtime registry | cordis, util | Node ESM |
| `packages/llm/*` (llm, llm-deepseek, llm-pi-ai, llm-retry, token-meter) | integration adapter (LLM seam) | `ctx.llm` adapter seam | core, credentials, settings | Node ESM |
| `packages/session/*` (persistence, persistence-jsonl, persistence-sqlite, projection, projection-cache, stats, telemetry, telemetry-otel, title*) | persistence / state | `ctx.sessionPersistence`, `ctx.sessionProjection` | core/session, storage | Node ESM |
| `packages/storage/*` (storage, storage-domain, storage-json, storage-sqlite) | persistence / state | `ctx.storage` | cordis, util | Node ESM |
| `packages/fs/*`, `shell/*`, `subprocess/*`, `terminal/*`, `lsp/*`, `web/*`, `skill/*`, `compaction/*`, `subagent/*`, `workflow/*`, `code-runtime/*`, `sandbox/*`, `e2b/*`, `spill/*`, `attachment/*`, `jobs/*`, `schedule/*`, `goal/*`, `context/*`, `credentials/*`, `settings/*`, `identity/*`, `workspace/*`, `plan/*`, `preset/*`, `todo/*`, `guard/*`, `feedback/*`, `interaction/*`, `hooks/*`, `extensions/*`, `mcp/*`, `acp/*`, `sdk/*`, `api/*`, `session-query/*` | integration adapter (capability seams) | Service Definition / Provider / Consumer per seam | core, llm, session | Node ESM |
| `packages/bundle/*` (base, headless, web-app) | product shell (composition) | `dsh.profile` / `dsh.bundle` patch layers | all spine + seams | Node ESM |
| `packages/boot/*` (app-boot, cmdline) | product shell (boot glue) | profile boot, CLI arg parsing | cordis, util | Node ESM |
| `packages/host/*` (webserver, apiproxy, frontend-static, plugin-inventory, directory-picker*) | product shell (web host) | HTTP route server, API gateway | core, api | Node ESM |
| `packages/client/*` (web, web-react, runtime, modules, connection, hmr, locale, schema-form, ui-*) | UI / rendering (browser half) | web shell, `ui-*` plugins | host, core | Browser (Vite) |
| `apps/cli` (`@deepseek-ai/dsh`) | product shell | `dsh` bin (`lib/bin.js`) | app-boot, bundles, cmdline | Node ESM |
| `apps/web` (`@deepseek-ai/dsh-web-frontend`) | UI / rendering | Vite build → `dist/` | client-web | Browser |
| `python/sdk` (`deepseek-harness-sdk`) | integration adapter | `deepseek_harness` module | JSON-RPC over stdio | Python |
| `python/sdk-runtime` (`deepseek-harness-runtime-bin`) | product shell (deploy root) | bundled runtime exe | dsh-jsonrpc-agent-pkg closure | Python + Node exe |
| `native/landlock-run/*` (entry, linux-x64, linux-arm64) | integration adapter (native) | Landlock launcher npm family | none (native) | Linux native |
| `examples/*` (acp-agent, headless-agent, jsonrpc-agent, web-cordis, web-schedule, mcp-memory) | product shell (demo leaves) | `cordis.yml` leaves | bundles | Node ESM |
| `website/` | UI / rendering (docs) | VitePress site | none | Static |

### Dependency Direction

The stable base is the **vendored Cordis framework** (`vendor/cordis` + its plugins), which nothing inside the repo depends on except through the `@deepseek-ai/cordis` peer dependency that every harness package declares. Above it sit the **zero-dependency utilities** (`packages/util/*`), which are harness-dep-free. The **product API spine** (`packages/core/*`) depends on the framework, utilities, the LLM seam, and the Typert protocol. **Capability seams** (fs, shell, subprocess, terminal, lsp, web, skill, compaction, subagent, workflow, code-runtime, sandbox, etc.) depend on the spine and on each other only through Service Definitions (never concrete providers). **Composition bundles** (`packages/bundle/*`) and **product shells** (`apps/cli`, `apps/web`, `packages/host/*`, `packages/client/*`) sit at the top.

The generated `docs/module-graph.md` (from `scripts/gen-module-graph.ts`) is the canonical inter-package dependency graph, derived from each package's `peerDependencies`. The `runtime-diagnostics/invariants` package is a near-universal leaf dependency (every package declares it) — it is the runtime-invariant assertion library, not a semantic dependency.

**No dependency cycles** are evident in the peer-dependency graph; the layering is acyclic. The `client` group (browser half) depends on `host` (server half) only through the wire protocol, not through direct imports of server code. The `api` group (Remote BFF + Typert RPC gateway) bridges host and client.

## Public Surfaces

- **CLI**: `dsh` binary (`apps/cli/lib/bin.js`) — subcommands `web`, `headless`, `--profile`, `--dump-config`, `--patch`, plugin management. Source launch via `node --import tsx/esm apps/cli/src/bin.ts`.
- **Web UI**: HTTP server at `http://127.0.0.1:3080` (default), served by `packages/host/webserver` + `apps/web` Vite build.
- **ACP server**: automation-only Agent Client Protocol server (`packages/acp/acp`), `@agentclientprotocol/sdk` 0.25.1.
- **JSON-RPC SDK**: `packages/sdk/*` (protocol, server, client) — newline-delimited JSON-RPC over stdio; consumed by the Python SDK and the `jsonrpc-agent` example.
- **Python SDK**: `deepseek_harness` module — high-level turns API + low-level JSON-RPC client; drives the harness as a subprocess.
- **Native launcher**: `@deepseek-ai/node-addon-landlock-run` — Landlock self-restrict-then-exec launcher (Linux).
- **Hook bridges**: Claude Code and Codex hook bridges (`packages/hooks/*`) over a shared wire-protocol library.
- **MCP client**: `packages/mcp/mcp-client` — MCP client capability.
- **File formats / persistent artifacts**: session logs (JSONL + SQLite), storage (JSON + SQLite), `cordis.yml` config, `cordis.patch.yml` overlays, `dsh.profile`/`dsh.bundle` package.json fields.
- **Exported libraries**: every `@deepseek-ai/dsh-*` package's `exports` map (ESM, `lib/` runtime + `lib/types` declarations).

## Runtime Lifecycle

Boot composes a plugin tree from ordered layers: each bundle in the profile's listed order → the profile's `cordis.patch.yml` → the home-level patch → any `--patch` overlay. `dsh-base` is the first layer of every profile (model adapters, tools, persistence, sandbox/approval policy, settings, credentials, telemetry); `dsh-web-app` adds the browser app; `dsh-headless` adds a one-shot runner. The Loader/Include/HMR vendored plugins reconcile config changes transactionally at runtime (see `vendor/README.md` local modifications 8, 11, 12, 14, 15). The agent loop runs a turn/step state machine (see `docs/architecture.md` "Turn flow"): `turn/start` → `agent/pre-step` → `step/start` → `agent/request` → `llm/stream` → `assistant/chunk*` → `tool/call*` → `tools/pre-execute` → `tools/execute` → `tools/post-execute` → `tool/result*` → `step/end` → `agent/turn-stopping` → `turn/end`. Shutdown unwinds effects in reverse registration order (Cordis fiber disposal).

## Concurrency Model

- **Async/await single-threaded event loop** on Node, with Cordis fibers/effects providing reversible registration and disposal.
- **Worker threads** for isolated execution: `code-runtime-worker-thread` (code execution) and `workflow-worker-thread` (workflow engine).
- **Subprocess providers** for shell/terminal/LSP: `subprocess-local` (process-tree provider), `node-pty` (persistent PTY, patched), `bash-local`/`pwsh-local`/`bash-sandbox`/`pwsh-sandbox`.
- **Sandboxing** via `sandbox` seam: bwrap (bubblewrap), Landlock (native launcher), Seatbelt (macOS), Windows ACL.
- **Waterfall events** (`agent/pre-step`, `agent/request`, `llm/stream`, `tools/*`) require listeners to call `next()` to delegate; `agent/turn-stopping` is serial with no `next()`.
- **Portability hazards**: the Cordis fiber/effect lifecycle, the waterfall event semantics, and the PTY/process-tree providers are Node-specific and do not translate 1:1 to other runtimes. The Landlock/bwrap/Seatbelt/ACL sandbox backends are OS-specific by design.

## Build and Packaging

- **Package manager**: pnpm 11.7.0 workspaces; `linkWorkspacePackages: true`; `allowBuilds` allowlist (esbuild, lefthook, node-pty, koffi); `patchedDependencies` (node-pty).
- **Build**: `tsc -b` (host + client project references) emits `lib/types`; `tsdown` bundles runtime to `lib/`; `vite build` for the web frontend. `pnpm run build` orchestrates via `scripts/build.ts`.
- **Test**: vitest (unit, coverage, e2e, snapshot, web, web-perf, web-stress); `test:coverage` is the CI coverage gate (per-file 100% on `packages/*/*/src`).
- **Hygiene gates**: knip, publint, workspace constraints, NodeNext consumer check, plus ~40 `verify-*` scripts (doc budgets, package invariants, cordis config, vendored links, type-equiv, etc.).
- **CI/CD**: GitHub Actions (ci.yml, release.yml, sandbox.yml, e2e.yml, e2b-e2e.yml, pi-ai-provider-e2e.yml, landlock-run.yml, landlock-run-release.yml, python-release.yml, release-vendor.yml, docs-pages.yml, issue-*.yml) + GitLab CI (`.gitlab-ci.yml`) for Python wheel builds/publish (manylinux_2_28, glibc ≤ 2.28).
- **Release**: tag-driven (`v*` tags for npm, `python-v*` for Python wheels); `scripts/release/*` (bump, verify, pack, verify-packed-install, publish).
- **Distribution**: npm packages (`@deepseek-ai/dsh-*`), Python wheels (`deepseek-harness-sdk`, `deepseek-harness-runtime-bin`), single-exe build for the Python runtime (`scripts/build-exe-for-python-sdk.ts`).

## Porting Priorities

| Component | Priority | Rationale |
|---|---|---|
| Cordis framework (vendored) | core | The entire plugin/effect/event model depends on it; must be reimplemented or replaced with an equivalent DI/effect system. |
| `core/*` spine (session, tools, agent, agent-loop, system-prompt) | core | The product API spine; the turn/step loop and event-sourced session log are the heart of the harness. |
| Session log + persistence (JSONL/SQLite) | core | Model-visible-means-logged invariant; fork/resume/telemetry all derive from it. |
| Capability seams (fs, shell, subprocess, terminal, lsp, web, skill, subagent, workflow) | important | Required for parity on major workflows; each is a three-role seam. |
| Typert type graph + RPC gateway | important | Powers the API BFF and remote surfaces. |
| Sandbox backends (bwrap/Landlock/Seatbelt/ACL) | important | Security-critical; OS-specific. |
| Web UI (`client/*`, `host/*`, `apps/web`) | optional | Valuable but not needed for a first viable headless port. |
| Python SDK + runtime | optional | A separate delivery surface; can be ported independently. |
| Native Landlock launcher | optional | Linux-specific; only needed for the sandbox story on Linux. |
| Docs site (`website/`) | incidental | Source-specific ergonomics. |

## Durable State

- **Harness home**: `~/.dsh` (override via `DSH_HOME` env or explicit config), single-root for all user data.
- **Config**: `cordis.yml` (with `!!js` interpolation), `cordis.patch.yml` overlays, `dsh.profile`/`dsh.bundle` package.json fields.
- **Session data**: append-only `SessionEvent` log; persistence backends JSONL (`session-persistence-jsonl`, `SESSION_FORMAT_VERSION` 0) and SQLite (`session-persistence-sqlite`, monotonic `SCHEMA_VERSION`).
- **Storage**: non-session storage hub with JSON and SQLite backends.
- **Settings**: file-backed user settings (`settings-file`).
- **Credentials**: credential-reference seam with env-over-`.env` provider (`credentials-local`).
- **Identity**: anonymous user id (`anonymous-user-id`).
- **Env vars**: `DEEPSEEK_API_KEY`, `DEEPSEEK_BASE_URL`, `DSH_HOME`, `DSH_SNAPSHOT`, `DSH_BUILD_FACE`.
- **Logs/caches/generated**: build outputs (`lib/`, `dist/`), generated catalogs (config, tool, persistence, cordis, module-graph), `tsconfig.*.tsbuildinfo`.

## Coverage and limits

- Inspected scope: root manifests (package.json, pnpm-workspace.yaml), README/AGENTS/CLAUDE docs, `docs/architecture.md`, `docs/module-graph.md`, `packages/README.md` (group hierarchy), `vendor/README.md` (vendored framework + local modifications), `packages/core/README.md`, `python/README.md`, `native/README.md`, `apps/cli` + `apps/web` + `python/sdk-runtime` package.json, `.github/workflows/` + `.gitlab-ci.yml`, `packages/util/home-paths/src/index.ts`, package-group file counts.
- Skipped scope: individual package source bodies (233 packages, ~483K LOC) — deferred to contracts/protocols/defect phases; per-package READMEs beyond the group-level tables; the full 1682-line module-graph edge list (read the first 500 lines).
- Evidence basis: source inspection (manifests, docs, one utility source file) | generated module graph | upstream docs. No runtime verification performed.
- Known blind spots: exact wire schemas of the JSON-RPC/ACP/Typert RPC protocols (deferred to protocols phase); precise event payload shapes; per-package invariant details.
- Coverage disposition: COMPLETE (for the architecture map's structural scope; wire-format detail is intentionally deferred).

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| q-typert-rpc-schema | needs-runtime-test | Exact Typert RPC wire schema (request/response shapes) not extracted from source. | Requires reading `packages/typert/*` and `packages/api/*` source; deferred to protocols phase. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| arch-CF1 | protocols | JSON-RPC, ACP, and Typert RPC wire formats listed by name only; schemas not extracted. | Wire-format extraction is the protocols phase's rubric. |
| arch-CF2 | contracts | Per-capability Service Definition contracts (trigger/defaults/outputs/side-effects) not enumerated. | Behavioral contract recovery is the contracts phase's rubric. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | The system intent is documented. | PASS | §System Intent. |
| 2 | The layer map and dependency direction are documented. | PASS | §Layer Map (package inventory + dependency direction). |
| 3 | Public surfaces are identified. | PASS | §Public Surfaces (CLI, Web UI, ACP, JSON-RPC, Python SDK, native launcher, hooks, MCP, file formats). |
| 4 | Runtime lifecycle, concurrency model, and porting priorities are summarized. | PASS | §Runtime Lifecycle, §Concurrency Model, §Porting Priorities. |
| 5 | Findings are marked with evidence levels. | PASS | Evidence levels noted in §Coverage and limits; structural claims are `observed fact` from manifests/docs, layering is `strong inference` from the generated module graph. |
| 6 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | §Coverage and limits. |

**Validated by:** architecture phase, session 1 (2026-08-19)
**Overall:** PASS
