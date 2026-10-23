---
protocol_version: "0.3"
project_id: "coordinator-protocol"
title: "Codex–Claude Coordinator Protocol"
unique_name: "coordinator-protocol"
phase: "integration"
status: "active"
complexity: "standard"
coordinator: "codex"
developer: "codex"
reviewer: "claude"
created_at: "2026-10-23T10:00:00-07:00"
coordination_enabled: true
protocol_source: "/Users/yuansong/Documents/coordinator"
protocol_commit: "34a7c08410c687f469f28210a348d537bc672dbf"
current_run_id: "run-001"
resume_state: null
---

# Project: Codex–Claude Coordinator Protocol

## Objective

定义并实现一套基于本地 Git 与 Markdown 的 Codex–Claude 自主协作机制，覆盖角色指派、
设计、开发、复审、分歧闭环、commit 交接、轮询唤醒和有损操作授权。

## Scope

### In scope

- 协议与仓库目录规范
- Markdown 模板
- Commit message 与轮询事件约定
- 多 subproject 的角色和隔离规则
- 自主决策与用户授权边界

### Out of scope

- 绑定到某个未指定版本的 Codex 或 Claude CLI
- 跨机器 Git transport
- 自动安装系统级 timer

## Constraints

- Developer 和 reviewer 必须是不同 Agent。
- 每个可消费动作必须有 commit summary。
- 未经用户授权不能执行有损操作或扩大权限。

## Deliverables

- `docs/protocol.md`
- `docs/repository-layout.md`
- `docs/polling.md`
- `templates/*.md`
- `examples/commit-message.txt`

## Completion criteria

- [x] Design approved by Claude reviewer
- [x] Protocol draft exists
- [x] Templates cover project, design, task, review and decision
- [x] Wake/poll adapter contract exists (implementation is a separate task)
- [x] End-to-end v0.3 PR dry run passes through merge and generated review snapshot (`.collab/reviews/T-001.snapshot.md`)
- [ ] User explicitly confirms project completion

## Role assignment rationale

用户当前在 Codex 中发起项目，因此 Codex 担任 coordinator 和首轮 developer；Claude 担任
独立 reviewer。后续实际执行需要在 Claude 环境中启动 reviewer timer。

## Subprojects

本项目评估为 `standard`：文件较多但共享同一协议边界，当前拆分不会带来足够并行收益。

## Authorization holds

None.

## Peer environment

- bootstrap: `.collab/BOOTSTRAP.md`
- codex session: current Codex project session
- claude session: current Claude review session
- shared repository: `/Users/yuansong/Documents/coordinator`
- writable checkout policy: `separate-worktree`

## Git identities

- codex: `Codex Agent <codex-agent@localhost>`
- claude: `Claude Agent <claude-agent@localhost>`
- user: `andySong07 <15460992+andySong07@users.noreply.github.com>`

## Final confirmation

- status: `not_ready`
- requested_at: null
- confirmed_at: null
- confirmed_delivery_commit: null
- user_quote_or_reference: null
- channel: null
- delivery_summary_path: ".collab/DELIVERY.md"

