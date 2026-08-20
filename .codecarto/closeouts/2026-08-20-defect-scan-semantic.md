# Closeout — defect-scan-semantic

## Summary

- Passes 3 (concurrency), 4 (security), 5 (contract violations).
- 7 findings: 3 medium (readStableFile unbounded retry, runGroup deferred scheduler-failure, JSON-RPC params not runtime-validated), 4 low.
- No critical or high semantic defects.
- Closed mech-CF1, mech-CF2, mech-CF3.
