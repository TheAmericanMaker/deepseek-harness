# State and Storage

## 2026-08-19 — architecture phase

### Durable state inventory
- **Harness home**: `~/.dsh` (override `DSH_HOME` env or explicit config). Single root for all user data.
- **Session log**: append-only `SessionEvent` log (`packages/core/session`, `ctx.sessions`). Model-visible-means-logged invariant: anything reaching a model request must be reconstructable from the log.
- **Session persistence**: JSONL (`session-persistence-jsonl`, `SESSION_FORMAT_VERSION` 0, no compatibility promise) and SQLite (`session-persistence-sqlite`, monotonic `SCHEMA_VERSION`).
- **Session projection**: `session-projection` + `session-projection-cache`.
- **Storage hub**: `packages/storage/*` (storage, storage-domain, storage-json, storage-sqlite) — non-session storage.
- **Settings**: `settings-file` (file-backed user settings).
- **Credentials**: `credentials-local` (env-over-`.env` provider).
- **Identity**: `anonymous-user-id`.
- **Attachment**: `attachment-local` (content-addressed storage).
- **Spill**: `spill-local` (tool-result spill storage).

### Config files
- `cordis.yml` (with `!!js` interpolation), `cordis.patch.yml` overlays, `dsh.profile`/`dsh.bundle` package.json fields.

### Environment variables
- `DEEPSEEK_API_KEY`, `DEEPSEEK_BASE_URL`, `DSH_HOME`, `DSH_SNAPSHOT`, `DSH_BUILD_FACE`.

### Logs / caches / generated artifacts
- Build outputs (`lib/`, `dist/`), generated catalogs (config, tool, persistence, cordis, module-graph), `tsconfig.*.tsbuildinfo`.

## 2026-08-19 — contracts phase

Contract-level state additions:
- Session header fields: `version` (SESSION_FORMAT_VERSION 0), `id`, `createdAt`, `cwd` (absolute), `parentSession`, `seedLength`, `origin` ('subagent'|null), `delegationDepth`, `agentPreset`.
- Session event envelope: `type`, `seq`, `time`, `data`, `surfaceOp`, `sourceEventSeqs`, `ignorable`.
- Credentials document: strict `CredentialRef`→string mapping at `$DSH_HOME/.credentials.yaml` (0600, owner-only enforced on POSIX).
- SQLite session DB: `SCHEMA_VERSION` 17, application-id `0x44534850`, `trusted_schema=off`, `mmap_size=0`, `synchronous=FULL`.

## 2026-08-19 — protocols phase

Protocol-level state additions:
- Session event envelope: `type`/`seq`/`time`/`data` + conditional `surfaceOp`/`sourceEventSeqs` (surface events) + `ignorable` (informational).
- Surface ops: `append` | `{ op: 'replace', start, end }` (compaction shadows source events).
- Turn end reasons: `completed`/`aborted`/`blocked`/`error`/`max-tokens`/`interrupted`.
- JSONL: zstd checksummed frames, packed chunk rows, torn-tail marker `{ truncateTo, recoveredEvents }`.
- SQLite session: `SCHEMA_VERSION` 17, application-id `0x44534850`, `user_version` stamp, canonical-schema validation.

## 2026-08-19 — porting phase

Synthesis-level state additions:
- The event-sourced session log is the single source of truth; all derived state (history, telemetry, UI) projects from it.
- `SESSION_FORMAT_VERSION` 0 (reject-not-migrate) and the custom zstd frame format are high portability hazards.
