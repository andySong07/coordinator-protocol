# Codex–Claude Coordinator Protocol v0.4

## 1. Authority and roles

Order of authority: user instructions; safety and authorization; protocol at the pinned commit; project
record; accepted decisions.

- **Coordinator** creates the run, assigns roles, integrates approved work, and requests final confirmation.
- **Developer** designs or implements, verifies, and responds to every review finding.
- **Reviewer** independently reviews. Only the reviewer may resolve or withdraw its finding.

Developer and reviewer must differ for a task. A coordinator involved in a dispute cannot decide it. Use an
objective test when possible. After two complete rounds without new evidence, send the user one
`ACTION REQUIRED` report containing both positions; do not silently deadlock or repeatedly notify.

## 2. Cold start and records

The user decides whether coordination is enabled and names the coordinator. If enabled, every actor first
reads `.collab/BOOTSTRAP.md`, then reads this protocol with:

```text
git -C <protocol_source> show <protocol_commit>:docs/protocol.md
```

The pin must exist and be readable; it need not equal HEAD. `BOOTSTRAP.md` binds the run ID, project path,
protocol pin, roles, declared Git identities, permissions, and peer worktree. Exact existing projects are
reused; ambiguous name/ID/path matches stop without mutation.

Minimal committed records:

```text
.collab/BOOTSTRAP.md
.collab/PROJECT.md
.collab/design/DESIGN.md
.collab/tasks/T-NNN.md
.collab/reviews/T-NNN.snapshot.md       # PR profile only; generated at integration
.collab/authorizations/A-NNN.md         # only when authorization is needed
.collab/decisions/D-NNN.md              # only when a decision is needed
```

`PROJECT.md` is the project lifecycle and confirmation authority. `DESIGN.md` is the approved design record.
The task manifest declares `transport_profile: pr|local`; validators never infer it.

## 3. Lifecycle

Project status is `active | waiting_authorization | ready_for_user_confirmation | done | cancelled`; phase is
`design | implementation | integration | delivery | closed`. `resume_state` is non-null exactly while waiting
for authorization. Only `done` or user-recorded `cancelled` stops recovery polling.

Task states are `draft | assigned | in_progress | review_requested | changes_requested | approved | blocked |
cancelled`. PR tasks derive their live state from GitHub and do not mirror it in Markdown. Local tasks store it
in `TASK.md`. `cancel_review` cancels only the stale review and returns the task to `in_progress`;
`cancel_task` requires a covering user/project cancellation record.

Every project begins with design. Implementation starts only after design approval; integration starts only
after required task approval; delivery starts only after integration passes.

## 4. One turn barrier, two transports

One task has one mutex: `task/<project>/<subproject>/<task>`.

| Question | `pr` profile authority | `local` profile authority |
|---|---|---|
| Registered transport | `TASK.md.pull_request` | registered task ref |
| Whose turn? | Draft = developer; Ready = reviewer | last event `Next-Actor`/`Next-Role` |
| Fixed review target | review `commit_id`; COMMENT fallback `Reviewed-Head` | handoff `Target-Commit` |
| Open findings | unresolved GitHub review threads | append-only timeline in `TASK.md` |
| Turn end | submitted review decision | one response commit |

The developer freezes the task after handoff. New requirements wait for the next developer turn. Sessions are
resumed across rounds; a run does not create a session or PR per round. Active wake-up is normal; a poller is
recovery only and never changes turn state.

With distinct GitHub identities, use native APPROVE/REQUEST_CHANGES and branch protection. With shared
credentials, use one COMMENT review containing `[actor/role]`, `Actor-Decision: approved|changes_requested`,
and `Reviewed-Head: <sha>`. Those names are fixed. After changes requested, return the PR to Draft.

## 5. Findings and agreement

Finding IDs are stable. Severity is `blocking | non_blocking | question`.

- Reviewer opens; developer replies `Accepted`, `Rejected`, or `Answer` with evidence.
- Accepted fixes cite their commit. Rejected findings cite counter-evidence.
- Only reviewer resolves, reopens, or withdraws.
- Approval requires no unresolved blocking finding or question.

PR threads are the only live PR review record. At integration, coordinator generates—not hand-edits—a
snapshot containing generator/time, every review/thread ID, and every `commit_id`; GitHub wins on disagreement.
For local tasks, `TASK.md` contains the single append-only finding timeline and review decision.

## 6. Git identity and trailers

Actors use declared name/email pairs without changing global Git config. Never commit as another actor.
Apparent leftovers are preserved on a recovery ref under the finder's identity and reported to the owner.
Published commits are never amended. Integration is fast-forward only; never force or discard unknown work.

Required commit trailers are selected only from `TASK.md.transport_profile`:

| Field | `pr` | `local` |
|---|---:|---:|
| `Task`, `Actor`, `Role` | required | required |
| `Action` | authorization, confirmation, migration, cancellation, close | required |
| `Project`, `Subproject`, `Entity`, `State` | — | required |
| `Target-Commit`, `Responds-To`, `Next-Actor`, `Next-Role` | — | required |

Local `Action` is one of `init | bootstrap | design | review | respond | integrate | migrate |
authorization_hold | authorization_resolve | cancel_review | cancel_task | request_confirmation | close`.
`Action: close`, project `State: done`, identifies the atomic CLOSE event.

## 7. Minimal lease and unattended execution

The wake adapter keeps one per-task lease. Refuse launch while a non-stale lease exists. A lease is stale only
when its PID is absent **and** its expiry has passed. Release it only after validating the turn-ending event.
Durations, renewal cadence, backoff, launchd/cron, registry capacity, fairness, and multi-project scheduling
are implementation-guide concerns, not protocol rules.

Unattended execution is fail-closed: enforce writable/forbidden paths and tool permissions; never use bypass
modes or automatic privilege escalation. Interactive permission/authentication creates an authorization hold
and exits rather than waiting or approving itself.

## 8. Authorization

A one-use user grant is required before losing uncommitted/unique data, overwriting unknown work, writing
outside allowed roots, publishing externally, messaging third parties, spending money, or expanding permission.
Record exact action, objects, impact, recovery, user quote/reference, channel/time, expiry, and consumption in
`.collab/authorizations/A-NNN.md`. A consumed v0.4 authorization names exactly one reachable `action_commit`
whose `Action` trailer matches the grant.

An actor may recover only its own abandoned temporary output after the lease proves no live owner and any
unique diff is preserved on a recovery ref. Unknown ownership, unrecoverable data, or ref/worktree deletion
requires authorization.

## 9. Completion

Coordinator may request final confirmation only when design/tasks are approved, integration passed, no
blocking/question remains, nonblocking findings are resolved/withdrawn/backlogged, and no unknown changes
exist. The request identifies the delivery commit and review snapshot.

Silence is not confirmation. Any later deliverable/design/task/review change invalidates confirmation. After
the user's explicit reply, coordinator records quote/reference, channel/time, and confirmed delivery commit in
`PROJECT.md`, then makes one `Action: close` commit that atomically sets `done/closed`. Until then the project
remains open.
