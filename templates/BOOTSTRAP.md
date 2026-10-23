---
project_id: "<project-id>"
project_name: "<globally-unique-project-name>"
run_id: "<run-id>"
coordination_enabled: true
coordinator: "codex|claude"
peer_actor: "codex|claude"
peer_initial_role: "developer|reviewer"
project_dir: "<absolute-path>"
repository: "<absolute-path>"
main_ref: "refs/heads/main"
peer_worktree: "<absolute-path>"
peer_branch: "collab/<scope>/<actor>"
protocol_source: "<absolute-path>"
protocol_commit: "<sha>"
task_id: "<task-id>"
transport_profile: "pr|local"
pull_request: null
trigger_commit: "<sha>"
target_commit: "<sha>"
session_action: "create|resume"
session_id: null
---

# Peer bootstrap

## Objective

<Current collaboration run objective.>

Before creating anything, look up `project_name`. Reuse an existing project only when its project ID,
dedicated directory and Git root all match this bootstrap. Treat any partial or ambiguous match as a
conflict and stop without changing either project.

## Required protocol reading

- `README.md`
- `docs/protocol.md`
- implementation guides only when working on the corresponding adapter
- `docs/polling.md`
- relevant files under `templates/`

Do not act if `protocol_source` is unavailable, the pinned commit does not exist, or required files cannot be
read with `git show`. The pin is not required to equal HEAD.

## Scope and permissions

- readable: <paths>
- writable: <paths>
- forbidden: <paths/actions>
- authorization holds: <none or IDs>

## Expected response

- action (`pr`, only when required): `<authorization|request_confirmation|migrate|cancel_review|cancel_task|close>`
- action (`local`): `<init|bootstrap|design|review|respond|integrate|migrate|authorization_hold|authorization_resolve|cancel_review|cancel_task|request_confirmation|close>`
- required trailers: for `pr`, `Task`/`Actor`/`Role` plus conditional `Action`; for `local`, the full protocol envelope including `Responds-To: <trigger_commit>`
- next actor/role: `<actor>/<role>`

## Validation

- <command or document check>
