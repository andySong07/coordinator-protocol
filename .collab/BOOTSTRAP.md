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
protocol_source: "/Users/yuansong/Documents/coordinator"
protocol_commit: "7c73888d158ebd461c491ba3deb64467de556b3b"
task_id: "T-001"
trigger_commit: "7c73888d158ebd461c491ba3deb64467de556b3b"
session_action: "resume"
---

# Peer bootstrap

Read the protocol from the pinned commit with `git show`; do not compare the pin to HEAD. Work as reviewer
for T-001 in a separate writable worktree. Unattended execution must fail closed and must not bypass approval
or sandbox controls. Respond to the current developer handoff with one review commit.
