# Codex–Claude 协作协议 v0.1

## 1. 目标与原则

本协议让 Codex 和 Claude 在没有用户逐步介入的情况下，完成“设计 → 开发 → review →
修订 → 一致 → 交付”的闭环。

规则优先级如下：

1. 用户的明确要求；
2. 安全、权限和有损操作约束；
3. 项目 `PROJECT.md`；
4. 已接受的 decision record；
5. 本协议；
6. 各 Agent 自己的默认习惯。

Agent 不得把普通技术分歧升级给用户。分歧必须先通过证据、实验、最小实现或 reviewer
复审解决。只有第 10 节所列事项需要用户授权。

## 2. 角色

### Coordinator

- 由用户在创建 project 时指定为 Codex 或 Claude；
- 评估复杂度、建立项目目录和 Git 仓库、指定其余角色；
- 先推动设计，设计未达到 `approved` 前不得进入实现阶段；
- 把复杂项目拆成边界清晰的 subproject，并管理依赖和集成顺序；
- 不代替 developer 自审，也不代替 reviewer 批准；
- 当循环停滞时选择实验、缩小范围或作出项目内决策；
- 验证完成条件并关闭项目。

### Developer

- 产出可以是代码、设计、研究、文档或其他约定成果；
- 在任务文件中记录假设、验证方法和完成证据；
- 逐条处理 review comment：接受则修改并给证据，不接受则写出反驳依据；
- 不得自行把 blocking comment 标为 `resolved`。

### Reviewer

- 独立检查设计和交付物；
- comment 必须指向具体文件、段落、需求或验证证据；
- 对 developer 的回应逐条复审；
- 只有 reviewer 可以把自己提出的 comment 标为 `resolved` 或 `withdrawn`；
- 不以风格偏好阻塞交付，blocking comment 必须对应正确性、安全性、需求、可维护性或可验证性风险。

同一个 project 中 developer 与 reviewer 必须由不同 Agent 担任。Coordinator 可以兼任其中
一个角色，但优先保持三职分离。对于多 subproject，Codex 和 Claude 必须交叉担任 developer
和 reviewer；不得让同一个 Agent 同时开发和 review 同一 subproject。

## 3. 启动协议

Coordinator 按以下顺序启动项目：

1. 检查是否已有明确的 dedicated project dir。
2. 若没有，在 `/Users/yuansong/Documents/projects/<project-id>` 创建目录；`project-id`
   使用小写短横线格式并避免覆盖已有目录。
3. 若目录不是 Git repo，则初始化本地 Git 仓库。
4. 创建 `.collab/PROJECT.md` 与 `.collab/design/DESIGN.md`。
5. 评估复杂度并写入 `PROJECT.md`：
   - `simple`：单一交付物、边界清楚、低集成风险；
   - `standard`：多个任务或存在明显取舍，但可由一个 project 管理；
   - `complex`：多个可独立开发模块、并行收益明确或集成风险高。
6. 指定 developer 和 reviewer，并记录到 `PROJECT.md`。
7. 复杂项目按模块创建 `.collab/subprojects/<id>/PROJECT.md`，声明输入、输出、依赖、
   接口和 owner。只有真正可独立验收的模块才拆分。
8. 提交 `INIT` commit。

## 4. 设计门禁

所有项目都必须先有设计。设计至少包含：目标、非目标、约束、方案、备选方案、接口或交付
格式、风险、验证计划、回滚策略。

设计采用与实现相同的 developer/reviewer 循环。Reviewer 写下 `approved` 后，coordinator
才把 project phase 从 `design` 改为 `implementation`。紧急修复也必须有轻量设计，允许和
实现放在同一轮，但不能省略。

## 5. 任务状态机

```text
draft -> assigned -> in_progress -> review_requested
      -> changes_requested -> in_progress -> review_requested
      -> approved -> integrated -> done
```

允许的写入者：

| 状态 | 写入者 | 必要条件 |
|---|---|---|
| `draft` | coordinator | 已定义目标和验收标准 |
| `assigned` | coordinator | 已指定 developer/reviewer |
| `in_progress` | developer | 已确认任务边界 |
| `review_requested` | developer | 有交付物、验证证据和 commit |
| `changes_requested` | reviewer | 至少一个 open blocking comment |
| `approved` | reviewer | 无 open blocking comment |
| `integrated` | coordinator | 依赖和集成验证通过 |
| `done` | coordinator | 项目完成条件满足 |

状态只能前进到表中允许的相邻状态；重新打开时由 reviewer 将 `approved` 改为
`changes_requested` 并创建新 comment。

## 6. Review comment 协议

每条 comment 使用稳定 ID：`R-<round>-<sequence>`，例如 `R-2-03`。禁止删除历史
comment；状态变化通过编辑该 comment 的字段并保留时间线完成。

字段：

- `severity`: `blocking | non_blocking | question`
- `status`: `open | accepted | disputed | addressed | resolved | withdrawn`
- `location`: 文件路径、标题或行号；非代码成果可写交付物章节
- `claim`: reviewer 认为存在的问题
- `required_outcome`: 可验证的期望结果，不强制指定实现手段
- `evidence`: reviewer 的依据
- `developer_response`: 接受、反驳或请求澄清的说明
- `resolution`: reviewer 的复审结论

流转规则：

```text
open -> accepted -> addressed -> resolved
open -> disputed -> resolved
open -> disputed -> open          # reviewer 提供新证据或收窄问题
open -> withdrawn                 # reviewer 接受反驳或认为问题不再适用
```

Developer 对 blocking comment 不得只回复“不同意”。必须给出需求、测试、实验、规范、
代码路径或明确取舍作为依据。Reviewer 不接受反驳时也必须补充新理由，不能原样重复 claim。

双方连续两轮没有引入新证据且仍不一致时，coordinator 必须选择以下一种处理：

1. 定义一个可判定的实验或验收测试；
2. 选风险更低、可回滚的方案；
3. 记录 decision record，明确取舍和后果。

这不是用户升级条件。

## 7. Git 交互规则

### 单写者原则

共享仓库可以被多个 Agent 读取，但同一工作目录在任一时刻只有一个写者。并行 subproject
使用独立 branch + worktree，避免两个 Agent 同时写一个 checkout。

推荐分支：

- `main`：coordinator 管理的集成分支；
- `collab/<subproject>/<role-or-task>`：开发分支；
- review 默认通过 commit 和 Markdown 进行，不要求 reviewer 修改开发分支。

### 一次交接一个 commit

每个可被另一角色消费的动作必须形成独立 commit。Commit subject：

```text
<TYPE>(<scope>): <imperative summary>
```

`TYPE` 为 `INIT | DESIGN | ASSIGN | WORK | REVIEW | RESPOND | REVISE | DECIDE | INTEGRATE | CLOSE`。

Commit body 必须包含：

```text
Role: coordinator|developer|reviewer
Actor: codex|claude
Task: <task-id>
State: <new-state>
Summary:
- <what changed>
- <why>
Evidence:
- <tests, checks, or document paths>
Next: <expected role and action>
```

`Summary` 是用户所要求的每一步摘要；不依赖 subject 单独承载。不得把秘密、token 或大量
生成内容写进 commit message。

### 消费 commit

Agent 只处理满足以下条件的 commit：

- commit 晚于自己记录的 cursor；
- `Next` 指向自己的角色，或修改了自己负责的任务；
- commit 属于当前 project/subproject；
- commit 已存在于稳定可读取的本地 ref。

Agent 完成动作后更新 `.collab/runtime/cursors/<actor>-<role>.txt`。Runtime cursor 默认可提交，
以便两个独立进程共享进度；它只能包含最后处理的 commit SHA，不包含锁或 PID。

## 8. 一致与完成

“达成一致”必须是显式状态，而不是没有新消息：

- 所有 blocking comment 均为 `resolved` 或 `withdrawn`；
- developer 在任务中声明交付物和验证结果完整；
- reviewer 将任务标为 `approved`；
- developer 没有未回答的 `question`；
- coordinator 完成集成验证。

Non-blocking comment 可以带入 backlog，但必须由 coordinator 明确接受为后续事项。

## 9. Coordinator 的自主决策边界

Coordinator 可以自主：选择技术方案、拆分任务、调整顺序、定义实验、接受可回滚取舍、
把非阻塞项移入 backlog，以及在项目范围内增删生成文件。

Coordinator 不应仅因不确定就询问用户；应先收集证据并选择最可逆的合理方案。

## 10. 必须向用户请求授权的事项

只有以下动作需要暂停并请求用户：

- 删除或不可恢复地覆盖用户数据；
- 强制改写共享 Git 历史、删除远端分支或覆盖未知改动；
- 发布到生产、公开发布、发送外部消息、付款或产生费用；
- 获取新的账户权限、凭据或扩大已授权范围；
- 需求之间存在无法通过证据消除的冲突，且任何选择都会造成不可逆业务后果；
- 平台本身要求交互式授权。

请求必须写明：准确动作、影响对象、为什么不可逆、恢复方式（如有）。等待授权期间可以继续
不依赖该动作的工作。

## 11. 异常与恢复

- **脏工作树**：停止覆盖，记录发现，隔离自己的工作；未知改动归用户所有。
- **并发冲突**：coordinator 决定集成顺序；developer 负责自己分支的冲突修复，reviewer
  复查语义变化。
- **Agent 超时**：轮询器不重做动作，只再次唤醒；以 task state 和 commit SHA 保证幂等。
- **坏 commit**：追加修复 commit，禁止静默 amend 已被另一角色消费的 commit。
- **协议违规**：coordinator 创建 `DECIDE` commit 记录纠正方式，然后从最后有效状态继续。

