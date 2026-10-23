# Coordinator Protocol

一套让 Codex 与 Claude 通过本地 Git 仓库和 Markdown 文件协作的协议。

核心选择是 **Git + Markdown**：

- Git 是同步层、并发控制层和审计日志；
- Markdown 是任务、设计、review comment 和决策的可读载体；
- commit 只声明“发生了什么”，完整上下文保存在仓库文件中；
- 定时器只负责发现新 commit 并唤醒对应角色，不直接修改业务内容。

协议正文见 [docs/protocol.md](docs/protocol.md)，目录规范见
[docs/repository-layout.md](docs/repository-layout.md)，轮询约定见
[docs/polling.md](docs/polling.md)，延期实现的多项目调度设计输入见
[docs/dispatcher.md](docs/dispatcher.md)。

## 最小工作流

1. 用户创建项目并指定 coordinator。
2. coordinator 评估复杂度、确认 dedicated project dir、创建设计文档。
3. coordinator 指定 developer 与 reviewer；多模块项目创建 subproject，且每个
   subproject 的两个角色分别由 Codex 和 Claude 担任。
4. Task 明确选择 `pr` 或 `local` transport；validator 不猜测。
5. `pr` 模式中 Draft 是 developer 回合，Ready 是 reviewer 回合，GitHub review 的 `commit_id` 是固定快照；`local` 模式使用不可变 Git trigger/response。
6. developer 对每条 comment 标记 `accepted` 或 `disputed`。接受则修改；不接受则写明理由。
7. reviewer 对更新内容复审。循环直到所有 blocking comment 都为 `resolved`，双方记录同意。
8. coordinator 验证交付条件，将项目置为 `ready_for_user_confirmation`，并向用户发起最终确认。
9. 只有收到用户明确确认后，coordinator 才能将项目置为 `done` 并关闭。

正常交接由完成当前回合的一方主动唤醒另一方的 current session；轻量 poller 仅用于恢复遗漏
事件，不拥有状态。一个 run 内复用 session 和 PR，不为每轮新建 session 或 PR。PR review thread
是实时 comment 权威；集成时由 coordinator 自动生成带 GitHub ID/commit_id 的 Markdown 快照。

除有损操作、权限扩张、协议定义的强制升级事项和项目结束前的最终确认外，三个角色自行完成
决策，不向用户请求选择。最终确认用于接受整体交付，不用于把中间技术分歧交给用户裁决。

## 文件模板

- [templates/PROJECT.md](templates/PROJECT.md)
- [templates/DESIGN.md](templates/DESIGN.md)
- [templates/TASK.md](templates/TASK.md)
- [templates/REVIEW.md](templates/REVIEW.md)
- [templates/DECISION.md](templates/DECISION.md)
- [templates/BOOTSTRAP.md](templates/BOOTSTRAP.md)
- [templates/AUTHORIZATION.md](templates/AUTHORIZATION.md)

