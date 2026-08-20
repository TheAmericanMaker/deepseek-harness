# Amendment — post-pipeline-evidence-resolution

## Resolved open questions

- **q-approval-default** → DENY (fail-closed). `serviceAsk` degrades an `ask` to deny when no approval service is composed.
- **q-api-gateway-transport** → client↔host connection RPC (`/api/<namespace>/<method>`, WebSocket downlink + HTTP bridge), not stdio.
- **oq-defect-scan-semantic-1** → documented coverage limitation, accepted; no action required.

## Retired post-pipeline item

- **post-spike-1** (api-gateway transport prototype) → unnecessary; resolved from source.

## Contradiction correction

- The architecture map's "no cycles" claim holds only for the peer-dependency graph. pnpm reports cyclic workspace dependencies (5 cycle groups). Corrected here; the architecture map's Layer Map section should be read with this caveat.

## Runtime verification

- `CI=true pnpm dsh --profile web --dump-config` → exit 0, 504 lines of composed config.
