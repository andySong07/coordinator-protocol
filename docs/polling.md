# Polling and wake-up contract

The normative rules are protocol §§5–7. Normal handoff calls `dispatcher kick-next`; actor-specific Codex
and Claude launchd watchdogs scan every 2–5 minutes only as recovery.

For each registered project an actor reads only `refs/heads/main`:

```text
git rev-list --topo-order --reverse <cursor>..refs/heads/main
```

It parses the mandatory `git interpret-trailers` envelope, validates project/subproject/task, rejects events
from itself, and accepts only an exact Next-Actor/Next-Role match. A response is complete only when main has
a descendant with `Responds-To` equal to that trigger. Duplicate/non-current responses are errors.

Cursor is local, never committed. Task mutex excludes round. Actor capacity is a shared counting semaphore.
Default execution timeout/lease/renewal are 30/35/5 minutes. PID absent plus expired lease is required before
recovery. Success clears failures; three failures create coordinator attention and backoff.

Unattended CLI invocation is fail-closed: workspace-write or stricter, explicit tools/paths, no dangerous
bypass, no automatic elevation. Interactive permission or authentication exits and creates an authorization
hold. `done` or valid user-requested `cancelled` is required to stop a project watchdog.

Different projects may run concurrently within global/actor/project limits. Independent tasks in one project
need separate worktrees and nonoverlapping ownership; one task remains strictly serial. Jitter only reduces
fast-forward/semaphore contention.
