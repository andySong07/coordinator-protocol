# Commit polling and wake-up contract

## 职责拆分

正常路径由提交方调用 dispatcher 即时启动下一 actor。Codex 和 Claude 仍各自运行一个独立
watchdog timer，恢复没有被即时交接成功的事件。Timer 只做四件事：

1. 读取该角色的 cursor；
2. 查询 cursor 之后的新 commit；
3. 判断 commit 的 `Next` 与 task ownership；
4. 有可执行事件时唤醒对应 Agent，并传入 project path、task ID 和 commit SHA。

Agent 才能修改交付物、Markdown、状态和 Git。这样轮询器可以保持无状态、可替换和幂等。
Timer adapter 在启动 Agent 时还必须提供或验证与 actor 对应的 Git identity，且不得修改用户
的 global Git config。

## 推荐周期

- 活跃项目：30–60 秒；
- 等待用户授权：5 分钟，仅检查授权结果，不重复提醒；
- `done` 或 `paused`：停止 timer。`ready_for_user_confirmation` 期间不得停止 timer，但仅处理
  用户确认或修改请求，不重复发送确认提醒。

轮询应加入最多 10% 随机抖动，避免两个 Agent 永远在同一瞬间抢锁。

## 事件判定

对每个未消费 commit，从 commit body 解析 `Role`、`Actor`、`Task`、`State` 和 `Next`。
`Next` 形式必须是：

```text
Next: <actor>/<role> <action>
```

例如：

```text
Next: claude/reviewer review T-001
```

Timer 必须按祖先顺序处理 commit。一次唤醒可合并同一 task 的连续 commit，但必须把所有 SHA
传给 Agent。Agent 在确认动作已提交后才能推进 cursor。

## 互斥与幂等

- 每个 `<actor>/<role>/<project>` 最多一个执行实例；
- timer 获得本机互斥锁后再唤醒 Agent；锁不进入 Git；
- Agent 启动后先检查 task 当前状态，若动作已完成则只推进 cursor；
- 失败时不推进 cursor，下次 timer 可重试；
- 连续失败三次时由 coordinator 记录故障并尝试替代路径，不自动询问用户，除非涉及授权。

## 本地 Git 边界

如果两个 Agent 位于同一台机器，使用共享 bare repo 或主 repo + worktree。不要通过同一个可写
checkout 交互。若 Agent 在不同机器，本协议需要额外 transport；该 transport 不属于“本地
Git”范围，必须由 coordinator 明确配置。

## Timer 适配器

实际唤醒方式依运行环境选择：Codex heartbeat、Claude scheduler、launchd、cron 或常驻进程。
适配器必须满足：

- 运行命令和日志不包含凭据；
- 能传入绝对 project path；
- 能区分 actor 与 role；
- 能停止已完成项目；
- 不把“没有新 commit”当成需要通知用户的事件。

## 多 project 调度

本机运行两个 actor-specific launchd watchdog，共享同一个 dispatcher 程序。Dispatcher 读取
本机 project registry，分别扫描每个 project 的 Git main 和独立 cursor。不同 project 可以并行；
同一 project 只有文件范围不重叠、使用独立 worktree 的 task 可以并行；同一 task/round 固定串行。

全局、每 actor、每 project 和每 task 的并发限制、锁键、公平队列、隔离与失败传播规则见
`docs/dispatcher.md`。

