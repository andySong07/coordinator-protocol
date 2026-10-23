# Codex–Claude Coordinator Protocol v0.3

## 1. Scope and authority

This is a shared contract: Codex and Claude have equal obligations. A user decides per project whether
coordination is enabled and which actor is coordinator. If disabled, none of this protocol is started.

Priority: user instructions; safety/authorization; pinned protocol commit; project record; accepted
decisions. Ordinary technical disagreement is not sent to the user.

Roles:

- coordinator creates the run, assigns roles, integrates, and asks for final confirmation;
- developer designs/implements, verifies, and responds to every review comment;
- reviewer independently reviews; only its author may resolve or withdraw a comment.

Developer and reviewer for a task must differ. A coordinator who is also a disputing party cannot decide
that dispute. A mutually defined objective test decides when possible; otherwise an open blocking review
stands. The coordinator cannot close it by decision record. If the same dispute has no new evidence for two
complete rounds and the coordinator is a disputing party, the coordinator sends the user one `ACTION REQUIRED`
notice containing both parties' evidence and the blocked outcome. This is a deadlock report, not permission for
the coordinator to self-decide; it is not repeated unless evidence or scope changes.

## 2. Project, run, and identity

A project is a long-lived repository/environment. A collaboration run is one user request. Sessions belong
to the current run and are resumed across rounds; they are archived after the user confirms completion.
The next request creates new sessions. v0.3 permits one active run per project; parallel work requires
separate subprojects/tasks inside that run.

The local registry is authoritative. Normalized project name and project ID must each be unique. An exact
name/ID/path/Git-root match is reused; missing entries are created; partial, multiple, or contradictory
matches stop without mutation. Product-side project lookup is optional enrichment.

`.collab/` retains project scope/roles, design, minimal task-to-transport mapping, authorizations, decisions,
confirmations, and generated review snapshots. In `pr`, GitHub owns live task/review state; committed files do
not maintain a second mutable copy. In `local`, committed task/review state remains authoritative.

Default project root is configurable as `projects_root`; this user may configure
`/Users/yuansong/Documents/projects`. It is never a portable protocol constant.

## 3. Bootstrap

Before acting, coordinator and peer read protocol files from the pinned snapshot, not the working tree:

```text
git -C <protocol_source> cat-file -e <protocol_commit>^{commit}
git -C <protocol_source> show <protocol_commit>:docs/protocol.md
```

The check means “commit exists and required files are readable,” not “commit equals HEAD.” Coordinator may
upgrade `protocol_commit` only between rounds with a migration commit; the active round retains its pin.

`.collab/BOOTSTRAP.md` carries project/run IDs, dedicated directory, repository, main ref, peer worktree,
roles, pinned protocol, trigger/target commits, enforced permissions, and expected response. Peer reuses an
exact existing product project or creates one, then uses a separate writable worktree. It never clones a
second divergent business repository.

## 4. Lifecycles

Project `status`: `active | waiting_authorization | ready_for_user_confirmation | done | cancelled`.
Project `phase`: `design | implementation | integration | delivery | closed`.
`PROJECT.md.resume_state` stores the exact prior status/phase while waiting for authorization and is cleared on
resume. It must be non-null exactly while `status: waiting_authorization`.

- `active/design -> active/implementation`: coordinator, design approved.
- `active/implementation -> active/integration`: coordinator, required tasks approved.
- `active/integration -> ready_for_user_confirmation/delivery`: coordinator, integration passed.
- any nonterminal -> `waiting_authorization`: coordinator, authorization record pending.
- waiting -> previous state: coordinator, authorization resolved or avoided.
- ready -> active design/implementation: coordinator, user requests changes.
- ready -> `done/closed`: coordinator, valid confirmation; same CLOSE commit.
- any nonterminal -> `cancelled/closed`: coordinator, only with explicit recorded user cancellation.

Only `done` or valid user-requested `cancelled` stops watchdogs. `Final confirmation.status` maps exactly:
`not_ready` for active/waiting, `awaiting_user` for ready, `confirmed` for done.

Design states: `draft -> in_review`; reviewer chooses `approved` or `changes_requested`; developer returns
changes_requested to in_review; reviewer may reopen approved only with new evidence.

Task states: `draft -> assigned -> in_progress -> review_requested`; reviewer chooses `approved` or
`changes_requested`; developer returns changes_requested to in_progress then review_requested. Any
nonterminal may enter `blocked` with a recorded dependency and return to its prior state, or `cancelled`
when the project/user cancellation record covers it. `TASK.md.resume_state` stores the prior state while
blocked and is cleared on resume. Reviewer may reopen `approved -> changes_requested` only with new evidence.
For `pr`, these semantic states are derived from Draft/Ready/review decision and are not manually mirrored in
the task manifest. Labels `blocked`, `awaiting-authorization`, `awaiting-user-confirmation`, and
`user-confirmed` are visible mirrors only; committed authorization/confirmation evidence remains authoritative.

## 5. Turn barrier, transports, and review

One task has one mutex independent of round:

```text
task/<project>/<subproject>/<task>
```

Each task manifest declares exactly one `transport_profile`: `pr` or `local`; validators never infer it.

For `pr`, one task has one GitHub PR. Draft means developer turn. The developer pushes, then marks the PR
ready to hand off and freezes the task. Ready means reviewer turn. A GitHub review and its `commit_id` define
the round. Reviewer ends with APPROVE, or REQUEST_CHANGES followed by conversion back to Draft. When shared
GitHub credentials prevent native decisions, a COMMENT review with `[actor/role]`, fixed SHA, and
`Actor-Decision: approved|changes_requested` is equivalent but cannot satisfy protected-branch approval.

For `local`, the developer makes one event handoff to a fixed SHA, then freezes the task. Reviewer makes one
response event. Only after that response and mutex release may developer act. Both profiles resume the current
run's sessions; neither creates a session per commit or round.
New requirements wait for the next developer turn. Emergency re-baselining uses the same task mutex and an
explicit `cancel_review` event: the active review round becomes `cancelled`, the task returns to `in_progress`,
and the developer may hand off a new fixed head. Whole-task cancellation instead uses `cancel_task`, requires
the covering user/project cancellation record, changes the task to `cancelled`, and sets both next actor and
next role to `none`.

Comment IDs never change. Status transitions:

- reviewer creates `open`;
- developer: `open -> accepted|disputed`, `accepted -> addressed`, `disputed -> accepted`;
- reviewer: `addressed -> resolved|open`, `disputed -> withdrawn|open`, `open -> withdrawn`;
- question: reviewer opens, developer marks answered, reviewer resolves/reopens.

In `pr`, review threads are the live comment authority: stable `[B-*]`, `[N-*]`, and `[Q-*]` prefixes;
developer replies `Accepted`, `Rejected`, or `Answer`; only reviewer resolves. Approval requires no unresolved
blocking/question thread. At integration, the coordinator generates (never hand-edits) a portable Markdown
snapshot containing generator actor/time plus every GitHub review/thread ID and `commit_id`; PR data wins on
disagreement. In `local`, the append-only Markdown timeline remains authoritative.

Two rounds “without new evidence” means no newly cited test result, requirement, specification, or artifact
fact. An interested coordinator still cannot close the comment.

## 6. Git and event envelope

`main` is the integration ref. A local-only task observes `main`; a GitHub-backed task may instead register one
Draft PR and observe its exact head ref plus submitted review decision until integration. Actors work in
`collab/<subproject>/<task>/<actor>` worktrees. Handoff/integration is fast-forward only. If the target moved,
the actor rebases/cherry-picks its own recoverable commit in its worktree, revalidates, and retries; it never
forces, discards unknown changes, or auto-resolves semantic conflicts.

Commit identity is declared exactly in project/registry for `codex`, `claude`, and `user`; validation is exact
name/email equality. Automation never changes global config. “Published” means reachable from observed main;
reachable commits are never amended.

Native GitHub approval as a merge gate requires distinct GitHub identities for developer and reviewer plus
branch protection. Shared credentials use the structured COMMENT fallback and must not claim native approval.

An actor never authors or commits with another actor's declared identity, even when apparent leftovers seem to
belong to that actor. The finder preserves unique leftovers on a recovery ref under the finder's own identity,
records their provenance, and reports them to the declared owner for acceptance or rejection.

Human summary/evidence precedes a final `git interpret-trailers` block. Required trailers depend only on the
task manifest profile:

| Profile | Required trailers |
|---|---|
| `pr` | `Task`, `Actor`, `Role`; also `Action` for authorization, confirmation, migration, cancellation, or close |
| `local` | every field in the full envelope below |

Full local envelope:

```text
Actor: codex|claude
Role: coordinator|developer|reviewer
Project: <id>
Subproject: <id|root>
Task: <globally-unique-id|PROJECT>
Entity: project|design|task|review|authorization
State: <entity state|unchanged>
Action: init|bootstrap|design|review|respond|rereview|integrate|protocol_migrate|authorization_hold|authorization_resolve|cancel_review|cancel_task|request_confirmation|close|none
Target-Commit: <sha|SELF|none>
Responds-To: <current trigger sha|none>
Next-Actor: codex|claude|user|none
Next-Role: coordinator|developer|reviewer|user|none
```

`SELF` means the commit carrying the envelope and avoids an impossible self-referential SHA. Fields are
ordered, single-valued, and mandatory. Project events use `Task: PROJECT`. `State` names the
`Entity`. One event has one recipient; parallel recipients get separate tasks/events. The local validator rejects and
alerts on duplicate response, non-current trigger, missing field, or envelope/file-state mismatch. Multiple
triggers require separate response commits; the local profile does not batch them.

The `Action` trailer is the mechanical event type. In particular, a CLOSE commit is exactly
`Entity: project`, `State: done`, `Action: close`; subject-line wording has no protocol meaning.

## 7. Wake adapter, permissions, and recovery

Wake mechanisms notify; they never own turn state. In `pr`, normal handoff actively resumes the peer session;
a light actor-specific poller recovers missed handoffs by reading only registered PR `isDraft`, head SHA, and
review decision. In `local`, the poller scans the registered integration ref from its local cursor and wakes
only on exact Next-Actor/Role. Completion is a descendant response whose `Responds-To` equals the trigger.
The multi-project dispatcher, fairness policy, and semaphores are a separate implementation task;
`docs/dispatcher.md` is non-normative design input until implemented and reviewed.

Registry defaults to `$XDG_CONFIG_HOME/coordinator/registry.yaml` or
`$HOME/.config/coordinator/registry.yaml`. Runtime defaults to `$XDG_STATE_HOME/coordinator` or
`$HOME/.local/state/coordinator`. Registry overrides are explicit. Light poller state contains only registered
project/task transport mapping, last observed SHA/decision, and failure count. It refuses to launch when the
same task already has a live job. Default execution timeout is 30 minutes; success resets failures and three
consecutive failures create one coordinator attention notice with backoff. Leases, global semaphores, and fair
multi-project scheduling belong to the deferred dispatcher task.

Unattended launch is fail-closed. Codex uses workspace-write sandbox with no automatic escalation. Claude
uses an explicit tool/path allowlist and noninteractive denial. Both prohibit bypass/dangerous modes. Network
is disabled unless bootstrap explicitly grants it. Interactive permission/auth prompts cause immediate failure,
an authorization hold event, and task blocking—never waiting forever or auto-approval. Bootstrap writable and
forbidden paths must be enforced by the launcher, not merely included in a prompt.

Jitter reduces simultaneous handoff contention; it is not a lock.

## 8. Authorization and final confirmation

Any action that can lose uncommitted data or commits without another ref, overwrite unknown changes, write
outside allowed roots, publish externally, send messages, spend money, or expand permission requires a
one-use authorization. Examples include destructive reset/clean/restore, deleting unmerged branches or
worktrees, stash drop, and immediate prune. Ordinary deletion is allowed only for files created by the agent
in a reachable commit and recoverable through Git.

An actor may clean or replace only its own abandoned, unpublished temporary output when the lease proves no
live process owns it and an immutable base plus deterministic replay path are recorded. It must first preserve
any unique diff on a recovery ref. Unknown ownership, unique data without a recovery ref, deletion of a ref or
worktree, or cleanup outside the actor's declared runtime scope requires authorization.

Authorization lives at `.collab/authorizations/A-NNN.md`: exact action/command, objects, impact, recovery,
request time, status, user quote/reference, channel, decision time, expiry. Status is
`pending|approved|denied|expired|consumed`; approval is exact-scope, single-use.

After integration, coordinator commits a delivery summary and asks the user explicitly. Confirmation record
contains user quote/reference, channel, time, summary path, and confirmed delivery commit. Silence is not
confirmation. Any later deliverable/design/task/review/decision change invalidates it. CLOSE atomically writes
the complete record plus `done/closed`. Every actor validates it before stopping or archiving sessions.

## 9. Completion

Completion requires approved design/tasks, no open blocking/question, verified integration, valid user
confirmation, and no unknown changes. Nonblocking comments are resolved/withdrawn or explicitly backlogged.
External publication follows the project authorization record; no authorization in one project is inherited
by another. Project completion still requires reviewer approval and the separate final user confirmation.
