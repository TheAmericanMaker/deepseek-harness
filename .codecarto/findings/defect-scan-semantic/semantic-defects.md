# Semantic Defects Report — deepseek-harness

## Scan Context

- **Source:** `../` (repository root)
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Contracts reference:** `findings/contracts/behavioral-contracts.md`
- **Protocols reference:** `findings/protocols/protocols-and-state.md`
- **Mechanical defects reference:** `findings/defect-scan-mechanical/mechanical-defects.md`
- **Pipeline:** full-with-deep-audit
- **Date:** 2026-08-19
- **Scope:** Semantic passes only (3 concurrency, 4 security, 5 contract violations). Mechanical passes (1 logic, 2 error handling, 6 configuration) were covered earlier in `defect-scan-mechanical`.

---

## Pass 3: Concurrency and Resource Management

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 1 | `packages/session/session-persistence-jsonl/src/index.ts:292-304` (`readStableFile`) | The revision-stable read loop is `for (;;)` with no iteration bound or backoff. Under a continuously-appending writer (a live session streaming chunks), `before !== after` can keep failing and the reader retries indefinitely, holding the read open and burning CPU. This is the concurrency framing of the mechanical finding: a reader/writer race with no liveness guarantee. | medium | strong inference | fix before porting |
| 2 | `packages/core/agent-loop/src/tool-calls.ts:164-235` (`runGroup`) | A dispatch rejection sets `schedulerFailure` inside the `.then` rejection handler, but the failure is only surfaced via `throwSchedulerFailure()` after the next `commitReady()` or `Promise.race` settlement. A dispatch that rejects while other calls are still in flight is not propagated until a later settlement point, so the loop can continue dispatching/committing sibling calls after a terminal scheduler failure has already occurred. | medium | strong inference | fix before porting |
| 3 | `packages/sdk/protocol/src/transport.ts:180-189` (`drainLines` → `handleLine`) | `handleLine` is invoked fire-and-forget (`void this.handleLine(line)`). `handleIncomingRequest` wraps the handler in try/catch, but if `writeError`/`write` itself throws (an output-stream write failure), the rejection of the async `handleLine` is unhandled — no `.catch` on the fire-and-forget call. | low | strong inference | fix before porting |

---

## Pass 4: Security and Trust Boundaries

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 1 | `packages/storage/storage-sqlite/src/schema.ts:43-50` (`createDatabaseFile`) | The code's own comment concedes the freshly-created DB file's confidentiality/integrity is "not protected when another principal can replace the database entry in its parent directory." The parent is created `0o700`, so the exposure is limited to a hostile parent directory, but the KV durability contract's confidentiality clause rests on that parent being trusted. | low | observed fact | port differently |
| 2 | `packages/acp/acp/src/index.ts:304-306` (`authenticate`) | The ACP server's `authenticate` is a no-op and `initialize` returns `authMethods: []` — no authentication on the automation wire. This is documented as "automation-only" and stdio-bound, but the trust boundary is entirely "whoever can write to the server's stdin." A port must preserve (or explicitly tighten) this boundary rather than silently adding auth assumptions. | low | observed fact | port differently |

No critical or high security findings. The codebase is security-conscious: owner-only file modes (`0600`/`0700`), env scrubbing in subprocess spawn, no-shell `execFile` argv execution, exclusive-create (`wx`) file writes that refuse symlink following, and secret-value redaction in YAML parse diagnostics.

---

## Pass 5: API Contract Violations

| # | Location | Defect | Severity | Evidence Level | Action | Spec Reference |
|---|----------|--------|----------|----------------|--------|----------------|
| 1 | `packages/sdk/protocol/src/transport.ts:121-160` (`request`/`notify`) | The transport's `request(method, params: object)` and `notify` are untyped at runtime; `HarnessSdkRequestMap`/`HarnessSdkNotificationMap` declare typed param/result shapes but are type-only. `objectParams` collapses arrays/scalars to `{}` and no runtime schema validation of wire params occurs at the transport layer. Whether the server plugin re-validates is unread here — if it does not, a malformed `session/prompt` params object is accepted and fails late. | medium | strong inference | fix before porting | `findings/protocols/protocols-and-state.md` §SDK Runtime JSON-RPC |
| 2 | `packages/util/native-command/src/index.ts:25-44` (`runNativeCommand`) | The declared return type is `Promise<{ stdout: string; stderr: string }>`, but on abort `execFile`'s error carries `stdout`/`stderr` as `undefined`, and the wrapped failure object assigns them unconditionally — the runtime value violates the declared type contract on the abort path. (Cross-referenced from mechanical pass 2; the return-type-inconsistency framing is this pass's rubric.) | low | strong inference | fix before porting | `runNativeCommand` type signature |

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 0 |
| Medium | 3 |
| Low | 4 |
| **Total** | 7 |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| 3. Concurrency and resources | 0 | 0 | 2 | 1 | 3 |
| 4. Security and trust | 0 | 0 | 0 | 2 | 2 |
| 5. API contract violations | 0 | 0 | 1 | 1 | 2 |

### Top Findings

1. **Pass 3 — `readStableFile` unbounded retry** (`session-persistence-jsonl/src/index.ts:292`): reader/writer race with no liveness bound. Medium. Fix before porting.
2. **Pass 3 — `runGroup` deferred scheduler-failure surfacing** (`core/agent-loop/src/tool-calls.ts:164`): a terminal dispatch failure is not propagated until a later settlement point. Medium. Fix before porting.
3. **Pass 5 — JSON-RPC wire params not runtime-validated** (`sdk/protocol/src/transport.ts:121`): typed map is type-only; malformed params accepted at the transport layer. Medium. Fix before porting.
4. **Pass 3 — fire-and-forget `handleLine`** (`sdk/protocol/src/transport.ts:180`): unhandled rejection on output write failure. Low. Fix before porting.
5. **Pass 4 — ACP no-auth trust boundary** (`acp/acp/src/index.ts:304`): automation wire trusts stdin. Low. Port differently.

### Carry-Forward Closure

| ID | Source Phase | Closed Because |
|----|--------------|---------------|
| mech-CF1 | defect-scan-mechanical | Addressed as Pass 3 finding 2 (`runGroup` scheduler-failure surfacing). |
| mech-CF2 | defect-scan-mechanical | Addressed as Pass 3 finding 1 (`readStableFile` unbounded retry, concurrency framing). |
| mech-CF3 | defect-scan-mechanical | Partially closed: this phase read additional packages (llm-retry, acp, sdk/protocol, typert/protocol, session types) beyond the mechanical sample. The full 233-package surface remains not exhaustively read; residual noted in Coverage and limits. |

---

## Coverage and limits

- Inspected scope: `packages/session/session-persistence-jsonl/src/index.ts`, `packages/core/agent-loop/src/tool-calls.ts`, `packages/sdk/protocol/src/{transport,types}.ts`, `packages/acp/acp/src/index.ts`, `packages/llm/llm-retry/src/index.ts`, `packages/storage/storage-sqlite/src/schema.ts`, `packages/util/native-command/src/index.ts`, plus the contracts/protocols/mechanical findings as spec.
- Skipped scope: the remaining ~220 packages (client UI, host, llm providers, subagent, workflow, sandbox backends, etc.); the `packages/api/*` gateway transport; the Python SDK and native launcher sources.
- Evidence basis: source inspection + upstream findings (contracts/protocols/mechanical). No runtime verification.
- Known blind spots: runtime-only races (timing, resource exhaustion under load) are not observable from static reading; the api-gateway transport framing remains unread.
- Coverage disposition: PARTIAL (semantic scan sampled the highest-signal concurrency/security/contract surfaces; the full package surface was not exhaustively read).

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | All three semantic passes (3, 4, 5) produced findings or documented "no defects found." | PASS | Pass 3 (3), pass 4 (2), pass 5 (2). |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | Every row carries all four fields. |
| 3 | Pass 5 findings cite the contract or protocol reference they violate. | PASS | Spec Reference column cites `protocols-and-state.md` and the `runNativeCommand` type signature. |
| 4 | Findings are organized by pass and sorted by severity; summary tables match the detailed findings. | PASS | 7 total = 3+2+2; severity table 3 medium + 4 low = 7. |
| 5 | Findings are marked with evidence levels. | PASS | Each finding tagged observed fact / strong inference. |
| 6 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PARTIAL | §Coverage and limits names the sampled scope and blind spots, but the full 233-package surface was not exhaustively read (residual of mech-CF3). |

**Validated by:** defect-scan-semantic phase, session 1 (2026-08-19)
**Overall:** PASS WITH GAPS
