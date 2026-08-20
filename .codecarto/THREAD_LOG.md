# Thread Log — Index

This file is an **index** of per-session closeouts. Each session writes a full closeout to
`closeouts/<YYYY-MM-DD>-<phase-or-module>.md` using `templates/closeout-template.md`, and
appends one line here pointing to it.

The body of each session lives in the closeout file, not in this index. This pattern scales
forever: per-session files are individually small and read-budget-cheap, and avoid the
heredoc-vs-edit sync risks that bite append-to-large-file workflows once the file grows past
~50 KB.

## Format

```
- YYYY-MM-DD — <phase-or-module> — <one-line-summary> — [closeout](closeouts/YYYY-MM-DD-phase-or-module.md)
```

## De-dup discipline

Before appending, scan the bottom 5 entries. If you see a line with the same date AND same
phase-or-module AND same summary, do not append — the prior session already wrote it. The
framework has no programmatic dedup gate; this is human-discipline. (See
`Apply 20 spec deltas to Thaumaturge.txt` for the incident that established this rule.)

A one-liner to surface duplicates from the shell:

```bash
grep -E '^- [0-9]{4}-[0-9]{2}-[0-9]{2}' .codecarto/THREAD_LOG.md | sort | uniq -d
```

## Entries

<!--
  Append one line per session below this marker.
  Example:
  - 2026-05-02 — framework-feedback-pass — applied 6 spec-blockers + 5 clarifications from FEEDBACK_INDEX.md — [closeout](closeouts/2026-05-02-framework-feedback-pass.md)
-->

- 2026-05-02 — framework-feedback-pass — applied 6 spec-blockers + 5 clarifications from FEEDBACK_INDEX.md; 14 deferred to BACKLOG.md — [closeout](closeouts/2026-05-02-framework-feedback-pass.md)
- 2026-08-20 — architecture — Architecture mapped: 233-package monorepo layered as vendored Cordis -> util -> core spine -> capability seams -> bundles/shells. Wire formats deferred to protocols; capability contracts deferred to contracts. — [closeout](closeouts/2026-08-20-architecture.md)
- 2026-08-20 — defect-scan-mechanical — Mechanical defect scan: 9 findings (0 critical, 0 high, 2 medium, 7 low) across passes 1/2/6 over the sampled spine/util/persistence surface. Concurrency/resource races and unread packages routed to defect-scan-semantic. — [closeout](closeouts/2026-08-20-defect-scan-mechanical.md)
- 2026-08-20 — contracts — Behavioral contracts recovered across CLI, tool surface, session, and credentials; arch-CF2 closed. Wire schemas deferred to protocols. — [closeout](closeouts/2026-08-20-contracts.md)
- 2026-08-20 — protocols — Wire formats extracted for SDK JSON-RPC, ACP, Typert RPC, and the session event stream; arch-CF1, con-CF1, and q-typert-rpc-schema closed. 8 compatibility hazards documented. — [closeout](closeouts/2026-08-20-protocols.md)
- 2026-08-20 — defect-scan-semantic — Semantic defect scan: 7 findings (0 critical, 0 high, 3 medium, 4 low) across passes 3/4/5. Closed mech-CF1/CF2/CF3. No critical or high semantic defects. — [closeout](closeouts/2026-08-20-defect-scan-semantic.md)
- 2026-08-20 — porting — Reverse-engineering bundle synthesized: 16 defects consolidated with dispositions, 10 portability hazards, 2 open questions. pro-CF1 closed; api-gateway transport routed to reimplementation-spec. — [closeout](closeouts/2026-08-20-porting.md)
- 2026-08-20 — reimplementation-spec — Final language-agnostic reimplementation spec produced: 8 modules, 12 acceptance scenarios, 3 known unknowns, 5 spikes. por-CF1 closed. Pipeline complete. — [closeout](closeouts/2026-08-20-reimplementation-spec.md)
- 2026-08-20 — amendment:post-pipeline-evidence-resolution — Resolved 3 open questions on evidence (approval default = deny; Typert Gateway transport = client connection RPC over /api, not stdio; semantic coverage limitation accepted) and retired the api-gateway transport spike. Recorded a contradiction correction: the workspace dependency graph has cycles (peer-dependency graph is acyclic). — [closeout](closeouts/2026-08-20-amendment-post-pipeline-evidence-resolution.md)
