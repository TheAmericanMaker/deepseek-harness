# CodeCartographer Run Summary — deepseek-harness

- **Repository:** `/home/jamessesler/Documents/Github/experiments/deepseek-harness`
- **Pipeline:** `pipeline-full-with-deep-audit` (`workflow/pipeline-full-with-deep-audit.yaml`)
- **Final state:** `complete` — 7/7 phases complete, 0 carry-forward outstanding, 1 post-pipeline item pending
- **Run date:** 2026-08-19 (EDT) / closeouts stamped 2026-08-20 (framework clock)

## Phases that ran

| # | Phase | Validation | Primary output |
|---|---|---|---|
| 1 | architecture | PASS | `findings/architecture/architecture-map.md` |
| 2 | defect-scan-mechanical | PASS WITH GAPS | `findings/defect-scan-mechanical/mechanical-defects.md` |
| 3 | contracts | PASS | `findings/contracts/behavioral-contracts.md` |
| 4 | protocols | PASS | `findings/protocols/protocols-and-state.md` |
| 5 | defect-scan-semantic | PASS WITH GAPS | `findings/defect-scan-semantic/semantic-defects.md` |
| 6 | porting | PASS | `findings/porting/reverse-engineering-bundle.md` |
| 7 | reimplementation-spec | PASS | `findings/reimplementation-spec/reimplementation-spec.md` |

## Artifacts produced per phase

| Phase | Primary | Secondary (append) | Handoff | Closeout |
|---|---|---|---|---|
| architecture | 1 | 5 (public-surfaces, runtime-lifecycle, state-and-storage, build-and-deploy, config-model) | 1 | 1 |
| defect-scan-mechanical | 1 | 0 | 1 | 1 |
| contracts | 1 | 4 (public-surfaces, runtime-lifecycle, state-and-storage, config-model) | 1 | 1 |
| protocols | 1 | 4 (public-surfaces, runtime-lifecycle, state-and-storage, config-model) | 1 | 1 |
| defect-scan-semantic | 1 | 0 | 1 | 1 |
| porting | 1 | 5 (all five secondary outputs) | 1 | 1 |
| reimplementation-spec | 1 | 0 | 1 | 1 |
| **Total** | **7** | **18 secondary sections** (across 5 files) | **7** | **7** |

Plus framework-owned: `THREAD_LOG.md` (8 index entries), `dashboard.html`, `workflow/status.yaml` (canonical state).

## Key findings

### Top 3 architectural patterns
1. **"Everything is a plugin"** on vendored Cordis — model adapter, tool registry, session log, agent loop, persistence, and UI are all plugins contributing services/typed events/reversible effects to a shared context; no privileged core to patch.
2. **Event-sourced session log as the single source of truth** — the *model-visible-means-logged* invariant; fork/resume/telemetry/UI all project from the append-only log.
3. **Three-role capability seams** (Service Definition / Provider / Consumer) — one provider swap (e.g. filesystem + subprocess → remote sandbox) moves Bash/PTY/LSP together with no provider forks.

### Top 3 defects (16 total: 0 critical, 0 high, 5 medium, 11 low)
1. `readStableFile` unbounded retry loop under a continuously-appending writer (mechanical D1.2 / semantic D3.1) — medium.
2. `runGroup` defers scheduler-failure surfacing until a later settlement point (semantic D3.2) — medium.
3. JSON-RPC wire params not runtime-validated at the transport layer (semantic D5.1) — medium.

### Top 3 contracts
1. **Tool execution pipeline** — `tools/pre-execute` (allow/deny/ask) → `tools/execute` → body → `tools/post-execute` → `tools/result`, with ordered result commit and synthetic abort results for replay validity.
2. **Credential precedence** — inherited process env (read-only, wins) > `$DSH_HOME/.credentials.yaml` > project `.env` > home `.env`, with owner-only 0600 enforcement.
3. **Session lifecycle** — `SESSION_FORMAT_VERSION` 0 reject-not-migrate; torn-tail repair; `session/end-seed` boundary.

## Total tokens used

**Not measurable by the framework.** `codecarto_usage` reports `0 in / 0 out / 0 cache-write` with the explicit note: *"7 run(s) recorded via codecarto_complete carry no token or activity data (MCP hosts execute phases in their own context) — zeros above are unknowns, not free runs."*

The phases were executed inline by the driving Hermes session (the MCP host), so the framework's per-phase usage log was never populated. The token cost is real but lives in the host's own context accounting, which is not exposed to the CodeCartographer usage tool. No fabricated number is reported here.

## Total wall-clock time

**Not precisely measurable.** The framework records `0ms` for the same reason (host-executed phases carry no duration data). From filesystem evidence, the run spanned roughly **8 minutes** of wall-clock: the `.codecarto/` template was copied at 20:17 and the final closeout (`reimplementation-spec`) was written at 20:25 (framework clock, 2026-08-20). This is an approximation from file mtimes, not an instrumented measurement.

## Outstanding items (post-pipeline)

- **4 terminal open questions** (resolvable only by runtime test / maintainer decision / source read outside the pipeline):
  - `q-approval-default` — approval seam default posture with no provider loaded.
  - `q-api-gateway-transport` — Typert Gateway transport framing (HTTP vs stdio).
  - `q-spec-variant` — spec defaulted to language-agnostic (auto-default); opinionated stack not locked.
  - `oq-defect-scan-semantic-1` — full 233-package surface not exhaustively read.
- **1 post-pipeline item**: `post-spike-1` — prototype the api-gateway transport framing.

These are applied via `codecarto_amend` (write `scratch/amendments/<slug>.yaml`), not by editing `status.yaml` directly.
