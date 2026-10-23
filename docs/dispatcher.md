# Local dispatcher design

## 1. Process model

macOS 只安装一个通用 `launchd` job。它按固定周期启动一次 dispatcher；dispatcher 是短进程，
完成扫描和调度后退出。Codex 与 Claude CLI 只在出现可执行交接事件时按需启动。

```text
launchd
  -> dispatcher --once
       -> scan project registry
       -> select runnable events
       -> acquire scoped leases
       -> start Codex/Claude CLI jobs
       -> verify response commits
       -> update local cursors
       -> exit
```

Git hook 可以请求一次即时扫描以降低延迟，但不是可靠性基础；即使 hook 丢失，下一次 launchd
扫描仍会发现事件。

## 2. Project registry

Dispatcher 从本机 registry 发现项目。Registry 不提交到任何业务仓库，避免一个项目能够修改
另一个项目的调度设置。

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
    enabled: true
```

注册时 dispatcher 必须验证：路径为绝对路径、Git 根与声明路径匹配、project ID 与
`.collab/PROJECT.md` 一致、目录不是另一个已注册 project 的子目录。删除 registry entry 只停止
调度，不删除 project。

## 3. Parallelism boundaries

并发规则按从强到弱的顺序应用：

1. 同一 task/round 只允许一个 active actor，developer 与 reviewer 严格串行。
2. 同一 project 的 coordinator 状态、集成和最终确认一次只允许一个写者。
3. 同一 project 内只有交付物和控制文件都不重叠的 task 才能并行，并使用独立 branch/worktree。
4. 不同 project 默认互相独立，可以并行。
5. 所有 project 仍受全局和每个 Agent 的并发上限约束。

锁键：

```text
agent/<actor>/<job-id>
project/<project-id>/coordinator
task/<project-id>/<subproject-id>/<task-id>/<round>
```

Task 锁上限固定为 1。Project 锁只保护集成与 project 状态，不阻止两个互不重叠的 task worktree
并行。Agent 锁的总数受 `codex_max_jobs` 或 `claude_max_jobs` 限制。

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

不同 project 即使 task ID 相同也不会冲突，因为完整 identity 是：

```text
<project-id>/<subproject-id>/<task-id>/<round>/<actor>/<role>
```

## 6. Commit discovery

每个 project 独立维护扫描 cursor，只观察该仓库的 `refs/heads/main`。Dispatcher 不使用一个全局
Git cursor。一次扫描可以从多个 project 各选出事件，但同一 project 内仍要检查 task 依赖、
round barrier 和文件所有权。

事件开始后记录 trigger SHA。Agent 生成的交接 commit 必须包含 `Responds-To: <trigger-sha>`。
Dispatcher 只有在目标 project 的 main 包含该回应 commit 后才把事件标为完成。

## 7. Failure containment

- 一个 project 的 Git 错误、无效事件或 Agent 失败只更新该 project/job 的失败状态。
- 达到三次连续失败后创建 coordinator attention event，并对该 job 退避；其他项目继续运行。
- Dispatcher 崩溃后通过 lease、cursor 和 `Responds-To` 恢复，不重新执行已完成事件。
- Project 进入 `done` 或 `cancelled` 后停止扫描，但保留审计状态。
- Registry、runtime state 和业务仓库分别存储；dispatcher 不根据仓库内容执行任意 shell 命令。

## 8. GitHub Actions adapter

GitHub Actions 可作为未来的远端 adapter，但不是核心依赖。只有 project 明确启用远端模式、
已配置仓库与凭据，并且用户接受远端执行和可能费用时才能使用。远端 adapter 必须遵守相同的
事件、锁、round barrier、确认和授权协议。

