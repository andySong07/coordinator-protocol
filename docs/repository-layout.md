# Minimal repository layout

```text
<project>/.collab/
├── BOOTSTRAP.md
├── PROJECT.md
├── design/
│   └── DESIGN.md
├── tasks/
│   └── T-NNN.md
├── reviews/                       # PR profile only
│   └── T-NNN.snapshot.md
├── authorizations/                # only when needed
│   └── A-NNN.md
└── decisions/                     # only when needed
    └── D-NNN.md
```

- `BOOTSTRAP.md` is the cold-start trust record: run, protocol pin, roles, identities, permissions, worktree.
- `PROJECT.md` alone answers project lifecycle and final-confirmation status.
- `TASK.md` is a minimal PR binding in `pr`; it also contains the append-only review timeline in `local`.
- `reviews/*.snapshot.md` is generated once at PR integration and never hand-edited.
- Authorization and decision files are created only when the event exists.
- Cursor, lease, PID, logs, credentials, and other runtime state stay outside Git and under the local state root.
- Subprojects are an optional extension using the same records and barriers, not another core file hierarchy.
