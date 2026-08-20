# Closeout — protocols

## Summary

- SDK JSON-RPC: initialize/session/prompt/shutdown + 4 notifications over newline-delimited JSON-RPC 2.0.
- ACP: initialize/newSession/prompt/cancel; one prompt in flight per session; quiesce drains continuable descendants.
- Typert RPC: Remote/RemoteScope decorators + bindTypertRemote; endpoint segment grammar.
- Session event stream: SessionEventMap (13 core event types), surface ops, turn-end reasons.
- Closed arch-CF1, con-CF1, q-typert-rpc-schema, oq-defect-scan-mechanical-1.
- Routed pro-CF1 (api-gateway transport) to porting.
