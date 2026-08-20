# Runtime Lifecycle

## 2026-08-19 — architecture phase

### Boot sequence
1. `dsh` CLI parses args (`packages/boot/cmdline`), resolves profile + harness home (`packages/util/home-paths`, `packages/util/launch-environment`).
2. `packages/boot/app-boot` composes the plugin tree from ordered layers: each bundle in the profile's listed order → profile `cordis.patch.yml` → home-level patch → `--patch` overlays.
3. Vendored Loader/Include/HMR reconcile config transactionally (see `vendor/README.md` local modifications 8, 11, 12, 14, 15).
4. `dsh-base` bundle mounts model adapters, tools, persistence, sandbox/approval policy, settings, credentials, telemetry; `dsh-web-app` adds the browser app; `dsh-headless` adds a one-shot runner.

### Main event loop / request handling
The agent loop runs a turn/step state machine (from `docs/architecture.md` "Turn flow"):
```
turn/start → agent/pre-step → step/start → agent/request → llm/stream
  → assistant/chunk* → assistant/message → tool/call* → tools/pre-execute
  → tools/execute → tools/post-execute → tool/result* → step/end
  → agent/turn-stopping → turn/end
```
A **step** is one model request plus the tools it calls; a **turn** is zero or more steps. `turn/*`, `step/*`, `user/message`, `assistant/*`, `tool/*` are durable session events; the rest are live extension points.

### Shutdown and cleanup
Cordis fiber disposal unwinds effects in reverse registration order. The vendored `cordis/src/fiber.ts` hardening (local modification 6) closes reentrant disposal gaps: effect owner-list wrapper registered before setup, synchronous setup failure rolls back cleanup, async cleanup stays owner-visible until quiescence, effect creation rejected while owner is `UNLOADING`.

### Background tasks / scheduled work
- `packages/jobs/*` — generic background-job runtime + `job_*` control tools.
- `packages/schedule/*` — session-local scheduled follow-ups.
- `packages/workflow/*` — worker-thread workflow engine.
- `packages/code-runtime/*` — worker-thread code execution.

## 2026-08-19 — contracts phase

Contract-level lifecycle additions:
- Tool scheduling: `executeToolCalls` → `runGroup` (exclusive barrier vs bounded parallel pool); results commit in model order; abort drains started calls and records synthetic results for skipped calls.
- Session lifecycle: `ctx.sessions.create()` / `fork()` / `resume()`; `Session.create()` / `Session.fromRestore()`; `session/created` → `session/event` (append feed) → `session/flush` (awaited parallel durability checkpoint) → `session/disposed`.
- Credentials lifecycle: `[Service.init]` yields drain-then-settle disposal; chokidar watcher with `awaitWriteFinish` debounce; `ready` reconcile closes the initial-load race.

## 2026-08-19 — protocols phase

Protocol-level lifecycle additions:
- SDK runtime: `initialize` handshake → `session/prompt` (lazy agent+session creation) → `shutdown`; `session.status` idle/running transitions.
- ACP: one prompt in flight per session; `settleAfterQuiescence` waits admission → `whenIdle()` → `outputTail`; `quiesce` drains continuable descendants child-first.
- Session: `session/end-seed` marks the seed boundary; `firstLiveSeq` = first in-process seq; torn-tail repair on load.

## 2026-08-19 — porting phase

Synthesis-level lifecycle additions:
- The turn/step loop and tool pipeline are the two state machines a reimplementation must preserve (see `findings/porting/reverse-engineering-bundle.md` §Protocol and State Notes).
- Cordis fiber/effect lifecycle + waterfall events are a high portability hazard (reimplement or replace with an equivalent DI/effect system).
