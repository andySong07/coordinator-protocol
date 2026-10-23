---
design_id: "DESIGN-001"
status: "in_review"
author: "codex"
reviewer: "claude"
---

# Design: Codex–Claude Coordinator Protocol

## Context and goals

Codex 和 Claude 需要在一个项目内自主轮换 coordinator、developer、reviewer 三种角色，
通过可审计、可恢复、适合异步消费的载体协作。

## Non-goals

- 不把两个模型封装成一个不可观察的长驻黑盒。
- 不依赖共享对话上下文。
- 不允许通过缺少记录的口头状态判定完成。

## Constraints and assumptions

- 两个 Agent 能访问同一本地 Git repository 或其 worktree。
- 每个 Agent 都能被外部 timer 唤醒。
- 同一 subproject 的 developer 与 reviewer 不同。
- Git commit 是不可变交接点，Markdown 是完整语义载体。
- Codex、Claude 和用户必须使用可区分的 Git identity，自动化不得依赖系统自动推断身份。

## Proposed approach

采用混合架构：Git 管理版本、顺序和并发；`.collab/` 下的 Markdown 管理项目状态、设计、
任务、review comment 和 decision。每个交接动作形成 commit，commit body 提供机器可解析的
元数据及人可读 summary。每个 Agent 的 timer 根据 cursor 扫描 commit，并仅在 `Next`
指向自己时唤醒。

任务和 review 使用显式状态机。每条 review comment 有稳定 ID 和独立状态，developer 可以
接受或反驳，reviewer 负责最终 resolved/withdrawn。连续两轮无新证据时 coordinator 用实验、
可逆方案或 decision record 收敛，而不是让用户裁决普通技术问题。

所有交付条件满足后只进入 `ready_for_user_confirmation`。Coordinator 必须向用户提供最终
交付摘要并请求确认；只有用户明确同意后，才能提交 `CLOSE` commit 并结束 project。

## Interfaces / output format

- Project control surface: `.collab/PROJECT.md`
- Design: `.collab/design/*.md`
- Tasks: `.collab/tasks/<task-id>.md`
- Reviews: `.collab/reviews/<task-id>.md`
- Decisions: `.collab/decisions/<decision-id>.md`
- Event envelope: structured Git commit body
- Delivery acknowledgement: per-actor cursor containing a commit SHA

## Alternatives considered

### 只用共享 Markdown

简单，但无法可靠表达消息顺序、消费点、并发修改和历史审计，容易出现覆盖或重复执行。

### 只用 Git commit message

顺序清晰，但长设计、逐条 comment 和多轮回应难以阅读、diff 和长期维护。

### 外部数据库或消息队列

并发能力更强，但增加服务依赖、权限面和运维成本，不符合本地优先的要求。

## Risks and mitigations

- 两个 Agent 同时写 checkout：使用单写者规则和 subproject worktree。
- Timer 重复唤醒：使用 commit cursor 和状态检查实现幂等。
- Review 无限循环：两轮无新证据触发 coordinator 收敛规则。
- Commit body 漂移：固定字段并在后续实现校验器。
- CLI 差异：把唤醒方式隔离为 timer adapter。

## Verification plan

1. 对模板与协议字段做静态校验。
2. 构造 design → review → disputed → revised → approved 的 dry run。
3. 重复投递同一 commit，确认不会产生重复工作。
4. 构造两个 subproject，确认角色交叉和 worktree 隔离规则可用。
5. 确认未收到用户最终确认时，项目不能进入 `done` 或停止 timer。

## Rollback / reversibility

协议文件可以通过普通 revert 回退；已被消费的 commit 不 amend。Schema 破坏性变化提高
`protocol_version` 并提供迁移说明。

## Open questions

- Codex 与 Claude 各自采用哪一种实际 scheduler/CLI adapter，需要在运行环境确认后实现。

## Approval

- Developer: `agreed`
- Reviewer: `pending`

