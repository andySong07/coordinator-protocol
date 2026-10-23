# Polling and wake-up contract

The normative rules are protocol §§5–7. Normal handoff actively resumes the next actor's current session;
actor-specific Codex and Claude pollers scan every 2–5 minutes only as recovery. They notify but never own or
change turn state. A future dispatcher may optimize multi-project scheduling without changing this contract.

For each registered local-only task an actor reads `refs/heads/main`; a `pr` task reads only its registered PR's
`isDraft`, head SHA, review decision, and unresolved blocking/question threads. Draft wakes developer; Ready
wakes reviewer. It never scans arbitrary branches or PRs and never changes turn state itself.
Local traversal is:

```text
git rev-list --topo-order --reverse <cursor>..refs/heads/main
```

It parses the mandatory `git interpret-trailers` envelope, validates project/subproject/task, rejects events
from itself, and accepts only an exact Next-Actor/Next-Role match. A response is complete only when main has
a descendant on the registered transport ref with `Responds-To` equal to that trigger; a PR review turn also
requires the submitted review decision. Duplicate/non-current responses are errors.

Cursor is local, never committed. Task mutex excludes round. Actor capacity is a shared counting semaphore.
Default execution timeout/lease/renewal are 30/35/5 minutes. PID absent plus expired lease is required before
recovery. Success clears failures; three failures create coordinator attention and backoff.

Unattended CLI invocation is fail-closed: workspace-write or stricter, explicit tools/paths, no dangerous
bypass, no automatic elevation. Interactive permission or authentication exits and creates an authorization
hold. `done` or valid user-requested `cancelled` is required to stop a project watchdog.

Different projects may run concurrently. Independent tasks in one project need separate worktrees and
nonoverlapping ownership; one task remains strictly serial. Full cross-project scheduling, leases, fairness,
and semaphores are deferred to the dispatcher implementation task. Jitter only reduces handoff contention.
