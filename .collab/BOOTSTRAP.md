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
peer_branch: "collab/root/root-T-001/claude-review"
protocol_source: "/Users/yuansong/Documents/coordinator"
protocol_commit: "7c73888d158ebd461c491ba3deb64467de556b3b"
task_id: "T-001"
transport_profile: "pr"
pull_request: "https://github.com/andySong07/coordinator-protocol/pull/2"
trigger_commit: "7c73888d158ebd461c491ba3deb64467de556b3b"
session_action: "resume"
---

# Peer bootstrap

Read the protocol from the pinned commit with `git show`; do not compare the pin to HEAD. Work as reviewer
for T-001 in a separate writable worktree. Unattended execution must fail closed and must not bypass approval
or sandbox controls. Respond to the current developer handoff with one review commit.

The current run predates the worktree rule and temporarily used the shared checkout. Starting with the Draft
PR handoff, each actor must use its declared worktree or an equivalent isolated checkout registered by the
coordinator. After task approval, coordinator makes a dedicated `protocol_migrate` commit that pins this
project and bootstrap to the approved protocol commit; the active review round retains the existing pin.
