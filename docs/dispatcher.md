# Local dispatcher design (deferred, non-normative)

This document is design input for a separately reviewed implementation task. Protocol v0.3 requires only the
light per-actor wake adapters in `docs/polling.md`; none of the registry fairness, lease, or semaphore behavior
below is a core compliance dependency until an implementation is approved.

## 1. Process model

正常交接采用主动模式：Agent 成功提交后调用 `dispatcher kick-next`，由 dispatcher 校验并立即
启动下一方。恢复路径安装两个 actor-specific `launchd` watchdog，分别以 `--actor codex` 和
`--actor claude` 扫描遗漏事件。Dispatcher 是短进程，完成扫描或调度后退出；Codex 与 Claude
CLI 只在出现可执行交接事件时按需启动。

```text
Agent commit -> dispatcher kick-next

Codex launchd watchdog              Claude launchd watchdog
  -> dispatcher scan --actor codex    -> dispatcher scan --actor claude
       -> scan project registry
       -> select runnable events
       -> acquire scoped leases
       -> start Codex/Claude CLI jobs
       -> verify response commits
       -> update local cursors
       -> exit
```

即时启动失败不会消费事件；相应 actor 的下一次 watchdog 扫描会恢复。Hook 可作为额外通知，但
不是必需路径。两个 watchdog 使用各自 cursor/失败状态，共享原子并发 semaphore 和 task lease。

## 2. Project registry

Dispatcher 从本机 registry 发现项目。Registry 不提交到任何业务仓库，避免一个项目能够修改
另一个项目的调度设置。

默认 registry 为 `$XDG_CONFIG_HOME/coordinator/registry.yaml`，未设置 XDG 时使用
`$HOME/.config/coordinator/registry.yaml`；runtime 对应 `$XDG_STATE_HOME/coordinator` 或
`$HOME/.local/state/coordinator`。其他位置必须由本机配置显式指定。

```yaml
version: 1
limits:
  global_max_jobs: 3
  codex_max_jobs: 2
  claude_max_jobs: 2
  per_project_max_jobs: 2
projects:
  - id: coordinator-protocol
    path: /Users/yuansong/Documents/coordinator
    coordinator: codex
    coordination_enabled: true
    protocol_source: /Users/yuansong/Documents/coordinator
    protocol_commit: <sha>
    current_run:
      id: run-001
      codex_session_id: <id-or-null>
      claude_session_id: <id-or-null>
    enabled: true
```

注册时 dispatcher 必须验证：路径为绝对路径、Git 根与声明路径匹配、project ID 与
`.collab/PROJECT.md` 一致、目录不是另一个已注册 project 的子目录、规范化后的 project name
和 project ID 在整个 registry 中分别唯一。已有匹配 project 直接复用；同名不同路径、同路径
不同 ID 或重复匹配时拒绝注册，不自动创建、合并或覆盖。删除 registry entry 只停止调度，不
删除 project。

Project 是长期环境；session 属于一次 current collaboration run。一个 run 内 Codex/Claude 的
多轮交接始终 resume 各自 current session。用户确认 run 完成后归档 session 并清空
`current_run`；同一 project 后续新任务创建新的 run 和 session。默认每个 project 同时只有一个
current run，复杂项目由 coordinator 明确开启独立 workstream 才可拥有多个 active run。

Registry 和 bootstrap 对 Codex/Claude 对称。最先接到用户请求的一方不必自动成为 coordinator；
它先读取用户指定值。Peer 首次参与 run 时，dispatcher 根据 `.collab/BOOTSTRAP.md` 查找对应的
本地 project：存在且 identity 一致则复用，不存在才创建；随后创建或 resume 当前 session，
不为每个 commit 新建 session。

## 3. Parallelism boundaries

并发规则按从强到弱的顺序应用：

1. 同一 task 只允许一个 active actor，developer 与 reviewer 严格串行。
2. 同一 project 的 coordinator 状态、集成和最终确认一次只允许一个写者。
3. 同一 project 内只有交付物和控制文件都不重叠的 task 才能并行，并使用独立 branch/worktree。
4. 不同 project 默认互相独立，可以并行。
5. 所有 project 仍受全局和每个 Agent 的并发上限约束。

锁键：

```text
semaphore/<actor>
project/<project-id>/coordinator
task/<project-id>/<subproject-id>/<task-id>
```

Task 锁不含 round 且上限固定为 1。Project 锁只保护集成与 project 状态，不阻止两个互不重叠
的 task worktree 并行。Agent 维度是受 `codex_max_jobs` 或 `claude_max_jobs` 限制的计数信号量，
不是每个 job 一个互不冲突的锁。

## 4. Fair scheduling

每轮扫描建立 runnable event 列表，然后采用 aging round-robin：

- 先按事件第一次变为 runnable 的时间排序；
- 每个 project 每轮最多先取得一个槽位，再分配剩余槽位；
- 等待用户授权、等待用户最终确认的项目不占模型执行槽；
- coordinator integration 事件可高于普通开发，但不能抢占正在运行的 task；
- 失败项目按指数退避，不阻塞其他 project；
- 同一个 project 不得连续占用所有全局槽位。

默认限制为：

```text
global_max_jobs = 3
codex_max_jobs = 2
claude_max_jobs = 2
per_project_max_jobs = 2
per_task_max_jobs = 1
```

如果某个 CLI 或账户不支持并发 session，将相应 actor 上限设为 1，不改变项目级协议。

## 5. Isolation

每个 job 使用唯一 worktree：

```text
<runtime>/worktrees/<project-id>/<subproject-id>/<task-id>/<actor>/
```

Prompt 必须包含 project 的绝对路径、worktree、project/subproject/task、角色、触发 SHA、固定
target SHA 和允许的动作。不得复用另一个 project 的会话上下文。日志、cursor、lease、失败计数
也必须以 project ID 开头，禁止跨项目共享。

首次启动 peer 时 prompt 还必须要求先读取 bootstrap 指定的 protocol source/commit；读取失败
则不执行项目动作。Coordinator 从 Codex 或 Claude 发起时使用同一字段和校验流程。

不同 project 即使 task ID 相同也不会冲突，因为完整 identity 是：

```text
<project-id>/<subproject-id>/<task-id>/<actor>/<role>
```

## 6. Pull-request turn transport and commit discovery

For GitHub-backed projects, one task normally uses one Draft PR. The developer pushes one fixed turn and
requests review, then freezes the task. The reviewer reviews that head SHA and submits one complete review
decision before the developer pushes the next turn. New commits remain in the same PR. PR comments are the
discussion surface; committed Markdown and trailers remain the portable source of truth. A PR does not
replace the task mutex or authorize concurrent edits. Local-only projects use the same barrier without a PR.

每个 project 独立维护扫描 cursor。本地 task 观察 `refs/heads/main`；GitHub-backed task 在集成前
观察 registry 中精确登记的 Draft PR head 和 review decision。Dispatcher 不使用一个全局 Git
cursor，也不扫描任意 PR。一次扫描可以从多个 project 各选出事件，但同一 project 内仍要检查
task 依赖、round barrier 和文件所有权。

When a task is transported by Draft PR, the dispatcher also observes that registered PR's head and review
decision, but never scans arbitrary repository PRs. It wakes the reviewer only after a developer handoff and
wakes the developer only after the review is submitted. Actor sessions are resumed for the run; a new session
is not created per commit or per review round. Watchdog scanning is recovery, while an explicit kick from the
actor completing the turn is the normal path.

事件开始后记录 trigger SHA。Agent 生成的交接 commit 必须包含 `Responds-To: <trigger-sha>`。
Dispatcher 只有在目标 project 的 main 包含该回应 commit 后才把事件标为完成。

## 7. Failure containment

- 一个 project 的 Git 错误、无效事件或 Agent 失败只更新该 project/job 的失败状态。
- 达到三次连续失败后创建 coordinator attention event，并对该 job 退避；其他项目继续运行。
- Dispatcher 崩溃后通过 lease、cursor 和 `Responds-To` 恢复，不重新执行已完成事件。
- Project 进入 `done`，或具有用户明确取消记录的 `cancelled` 后停止扫描，但保留审计状态。
- Registry、runtime state 和业务仓库分别存储；dispatcher 不根据仓库内容执行任意 shell 命令。

## 8. GitHub Actions adapter

GitHub Actions 可作为未来的远端 adapter，但不是核心依赖。只有 project 明确启用远端模式、
已配置仓库与凭据，并且用户接受远端执行和可能费用时才能使用。远端 adapter 必须遵守相同的
事件、锁、round barrier、确认和授权协议。

