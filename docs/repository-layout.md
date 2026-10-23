# Repository layout

协作控制文件统一放在业务仓库的 `.collab/` 下，避免和交付物混杂。

```text
<project>/
├── .git/
├── .collab/
│   ├── PROJECT.md
│   ├── design/
│   │   └── DESIGN.md
│   ├── tasks/
│   │   └── T-001.md
│   ├── reviews/
│   │   └── T-001.md
│   ├── decisions/
│   │   └── D-001.md
│   ├── authorizations/
│   │   └── A-001.md
│   ├── subprojects/
│   │   └── <subproject-id>/PROJECT.md
└── <deliverables>
```

约束：

- `PROJECT.md` 是角色、阶段、范围和完成条件的 source of truth。
- 每个 task 有最小 transport manifest。`local` review 使用 append-only 文件；`pr` review 文件仅在
  integration 时由 coordinator 从 GitHub API 生成，包含 review/thread ID 与 commit_id，禁止手改。
- Decision record 只追加，不回写历史结论；新决策用 `supersedes` 指向旧记录。
- 自动化不得把临时锁、PID、日志噪音或凭据提交到仓库。
- Cursor、lease、semaphore、失败计数等 runtime 状态只保存在本机 state 目录，永不提交。
- Subproject 的业务内容优先使用独立 worktree；根项目只保存映射和集成状态。

