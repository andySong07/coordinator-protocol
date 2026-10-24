---
project_id: "coordinator-protocol"
project_name: "coordinator-protocol"
run_id: "run-001"
coordination_enabled: true
coordinator: "codex"
peer_actor: "claude"
peer_initial_role: "reviewer"
project_dir: "/Users/yuansong/Documents/coordinator"
repository: "/Users/yuansong/Documents/coordinator"
main_ref: "refs/heads/main"
peer_worktree: "/Users/yuansong/Documents/coordinator-claude-review"
peer_branch: "protocol/v0.4-simplify"
protocol_source: "/Users/yuansong/Documents/coordinator"
protocol_commit: "70ba541756be24c4f561a7419fd89514fc2a312f"
task_id: "T-002"
transport_profile: "pr"
pull_request: "https://github.com/andySong07/coordinator-protocol/pull/4"
trigger_commit: "30ffcc84e6a026d0b628b62c215257ee118d9b1b"
session_action: "resume"
---

# Peer bootstrap

Read the protocol from the pinned commit with `git show`; do not compare the pin to HEAD. Work as reviewer
for T-002 in a separate writable worktree. Unattended execution must fail closed and must not bypass approval
or sandbox controls. Review the fixed PR head and submit one complete review decision.

The pinned commit is the approved v0.4 operating protocol. PR #4 is merged; its generated audit snapshot is
`.collab/reviews/T-002.snapshot.md`.

The current run predates the worktree rule and temporarily used the shared checkout. Starting with the Draft
PR handoff, each actor must use its declared worktree or an equivalent isolated checkout registered by the
coordinator. After task approval, coordinator makes a dedicated migration commit that pins this project and
bootstrap to the approved protocol commit; the active review round retains the existing pin.
