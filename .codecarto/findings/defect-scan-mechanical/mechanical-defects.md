# Mechanical Defects Report — deepseek-harness

## Scan Context

- **Source:** `../` (repository root)
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Pipeline:** full-with-deep-audit
- **Date:** 2026-08-19
- **Scope:** Mechanical passes only (1 logic, 2 error handling, 6 configuration). Semantic passes (3 concurrency, 4 security, 5 contract violations) deferred to `defect-scan-semantic` after protocols.

---

## Pass 1: Logic and Correctness

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 1 | `packages/util/timeout/src/index.ts:45-55` (`clampTimeout`) | The caller's `requested` hint is validated (`> 0`, finite), but the backend `def` and `max` bounds are **not** validated. `Math.min(requested ?? def, max)` returns a non-positive or negative value when `def` or `max` is itself non-positive (e.g. `clampTimeout(undefined, -5, 100)` → `-5`), silently producing a negative timeout that downstream `setTimeout` would treat as `0` (immediate fire). | medium | strong inference | fix before porting |
| 2 | `packages/core/agent-loop/src/tool-calls.ts:104-110` (`parseArguments`) | Invalid JSON is preserved as the raw string rather than coerced to `{}`. A tool whose schema expects an object receives a bare string in `exec.arguments`; any consumer that does `arguments.foo` on it gets `undefined` (or throws on a primitive). The behavior is documented, but it is a latent type-contract hazard for every tool that assumes an object. | low | observed fact | port differently |
| 3 | `packages/util/output-retention/src/index.ts:211-222` (`trimTrailingPartialUtf8`) | The walk-back over continuation bytes is bounded by `bytes.length - i <= 3`, and the `expected` length for a 4-byte lead (`0xF0`–`0xF7`) is `4`. A cut that lands immediately after a 4-byte lead with zero trailing continuation bytes is not walked back (the loop guard stops at the lead), so the lead is treated as complete and returned — correct — but the interaction between the `<= 3` guard and the `expected === 4` case is subtle enough that a boundary-spanning 4-byte sequence at the exact cut is worth a targeted test. | low | open question | leave behind |

---

## Pass 2: Error Handling and Resilience

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 1 | `packages/session/session-persistence-jsonl/src/index.ts:292-304` (`readStableFile`) | The revision-stable read loop is `for (;;)` with **no iteration bound or backoff**. Under a writer that appends continuously (a live session streaming chunks), `before !== after` can keep failing and the loop retries indefinitely, holding the read open and burning CPU. There is no max-attempt or yield. | medium | strong inference | fix before porting |
| 2 | `packages/subprocess/subprocess-local/src/spawn.ts:276-282` (`taskkillProcessTree`) | `spawnSync('taskkill', …)` outcome is deliberately unchecked: a missing `taskkill` binary, a nonzero status (already-absent tree), or a spawn failure are all silently ignored. On Windows this means tree termination can silently fail with no diagnostic, leaving orphaned descendants. Documented as intentional idempotent teardown, but the silent-failure surface is real. | low | observed fact | port differently |
| 3 | `packages/util/native-command/src/index.ts:25-44` (`runNativeCommand`) | On abort, `execFile`'s error carries `code: 'ABORT_ERR'` but `stdout`/`stderr` may be `undefined`; the wrapped failure object assigns `stdout`/`stderr` unconditionally, so the typed `{ stdout: string; stderr: string }` contract is violated on the abort path (fields are `undefined` at runtime). | low | strong inference | fix before porting |

---

## Pass 6: Configuration and Environment Hazards

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 1 | `packages/session/session-persistence-jsonl/src/index.ts:130` (`preparedSessionCacheSize`) | Schema bounds the cache size with `.min(1)` but **no upper bound**. A misconfigured (or maliciously patched) large value causes the coordinator to retain an unbounded number of cold `SessionPreparation` objects in memory. | low | observed fact | fix before porting |
| 2 | `packages/credentials/credentials-local/src/index.ts:103-122` (`assertOwnerOnly`) | The owner-only permission check is **POSIX-only**; on Windows it returns early because ACLs are not expressible as a mode. The credentials file's confidentiality on Windows therefore rests entirely on the create/replace APIs, with no equivalent verification that the file is not readable by other principals. Documented, but a genuine cross-platform security asymmetry. | low | observed fact | port differently |
| 3 | `packages/storage/storage-sqlite/src/schema.ts:43-50` (`createDatabaseFile`) | `open(path, 'wx', 0o600)` protects the freshly created file, but the code's own comment concedes it "does not protect confidentiality or integrity when another principal can replace the database entry in its parent directory." The parent is created `0o700`, so the exposure is limited to a hostile parent directory, but the limitation is load-bearing for the KV durability contract. | low | observed fact | leave behind |

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 0 |
| Medium | 2 |
| Low | 7 |
| **Total** | 9 |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| 1. Logic and correctness | 0 | 0 | 1 | 2 | 3 |
| 2. Error handling | 0 | 0 | 1 | 2 | 3 |
| 6. Config and environment | 0 | 0 | 0 | 3 | 3 |

### Top Findings

1. **Pass 1 — `clampTimeout` unvalidated `def`/`max`** (`packages/util/timeout/src/index.ts:45`): a non-positive backend default or cap silently yields a negative effective timeout. Medium. Fix before porting.
2. **Pass 2 — `readStableFile` unbounded retry loop** (`packages/session/session-persistence-jsonl/src/index.ts:292`): no iteration bound under a continuously-appending writer. Medium. Fix before porting.
3. **Pass 1 — `parseArguments` preserves invalid JSON as a raw string** (`packages/core/agent-loop/src/tool-calls.ts:104`): latent object-contract hazard for tools. Low. Port differently.
4. **Pass 2 — `runNativeCommand` abort path violates its own stdout/stderr string contract** (`packages/util/native-command/src/index.ts:25`). Low. Fix before porting.
5. **Pass 6 — `preparedSessionCacheSize` unbounded** (`packages/session/session-persistence-jsonl/src/index.ts:130`). Low. Fix before porting.

### Routed To Semantic Phase

| ID | Description | Why Routed |
|----|-------------|-----------|
| mech-CF1 | `executeToolCalls`/`runGroup` concurrency: `Promise.race(inFlight.values())` settlement, reclassification of later calls after ordered commits, and the `aborted`/`schedulerFailure` interplay are race-sensitive and need the concurrency pass (3) rubric. | Concurrency/race analysis is the semantic phase's job. |
| mech-CF2 | `OutputCollector` fd lifecycle (`discardSpill`/`seal`/`finalize`) and the `readStableFile` unbounded loop both involve concurrent writer/reader interaction; the resource-leak and race framing belongs to pass 3. | Resource/race framing is semantic. |
| mech-CF3 | Mechanical scan sampled ~10 of 233 packages (spine/util/persistence); the remaining packages (client UI, host, llm providers, subagent, workflow, etc.) were not read for mechanical defects. | Deeper, broader reading is the semantic phase's rubric. |

---

## Coverage and limits

- Inspected scope: `packages/util/*` (timeout, atomic-write, output-retention, native-command, launch-environment, brand, home-paths), `packages/core/agent-loop/src/tool-calls.ts`, `packages/fs/fs-local/src/fsio.ts`, `packages/subprocess/subprocess-local/src/spawn.ts`, `packages/credentials/credentials-local/src/index.ts`, `packages/storage/storage-sqlite/src/schema.ts`, `packages/session/session-persistence-jsonl/src/index.ts`, `packages/session/session-persistence-sqlite/src/schema.ts`.
- Skipped scope: the remaining ~220 packages' source bodies; the `client/*` UI group; `vendor/*` (already heavily documented in `vendor/README.md`); `python/` and `native/` sources.
- Evidence basis: source inspection only (no runtime verification, no test execution).
- Known blind spots: defects in packages not read; runtime-only defects (timing, resource exhaustion under load) are not observable from static reading.
- Coverage disposition: PARTIAL (mechanical scan sampled the highest-signal spine/util/persistence packages; the full 233-package surface was not exhaustively read).

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | At least two of the three mechanical passes (1, 2, 6) produced findings or documented "no defects found." | PASS | All three passes produced findings (3 + 3 + 3). |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | Every row carries location, severity, evidence level, and action. |
| 3 | Findings are organized by pass and sorted by severity. | PASS | Passes 1/2/6 sections; medium before low within each. |
| 4 | Summary tables are complete and counts match the detailed findings. | PASS | 9 total = 3+3+3; severity table 2 medium + 7 low = 9. |
| 5 | Findings are marked with evidence levels. | PASS | Each finding tagged observed fact / strong inference / open question. |
| 6 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PARTIAL | §Coverage and limits names the sampled scope and blind spots, but the scan read only the highest-signal spine/util/persistence packages (~10 of 233); the remaining packages are unread. Routed to `defect-scan-semantic` as `mech-CF3` (deeper reading is that phase's rubric). |

**Validated by:** defect-scan-mechanical phase, session 1 (2026-08-19)
**Overall:** PASS WITH GAPS
