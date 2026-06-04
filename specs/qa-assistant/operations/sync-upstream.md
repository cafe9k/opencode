# Upstream Sync Log

Record each synchronization from the official OpenCode `dev` branch here.

## Template

```md
# Sync upstream YYYY-MM-DD

- Upstream base: <commit>
- Internal base: <commit>
- Merge branch: qa/sync-YYYYMMDD
- Conflict files:
  - path: reason / resolution
- Verification:
  - bun typecheck: pass/fail
  - cli smoke: pass/fail
  - desktop smoke: pass/fail
- Follow-ups:
  - ...
```

