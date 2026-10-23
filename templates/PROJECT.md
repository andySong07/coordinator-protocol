---
protocol_version: "0.1"
project_id: "<project-id>"
title: "<title>"
unique_name: "<globally-unique-project-name>"
phase: "design"
status: "active"
complexity: "simple|standard|complex"
coordinator: "codex|claude"
developer: "codex|claude"
reviewer: "codex|claude"
created_at: "<ISO-8601>"
coordination_enabled: true
protocol_source: "<absolute-path>"
protocol_commit: "<sha>"
current_run_id: "<run-id|null>"
---

# Project: <title>

## Objective

<Measurable project outcome.>

## Scope

### In scope

- <item>

### Out of scope

- <item>

## Constraints

- <constraint>

## Deliverables

- <path or artifact>

## Completion criteria

- [ ] Design approved
- [ ] All tasks approved
- [ ] Integration verification passed
- [ ] No open blocking comments
- [ ] User explicitly confirmed project completion

## Role assignment rationale

<Why these actors and roles were selected.>

## Peer environment

- bootstrap: `.collab/BOOTSTRAP.md`
- codex session: `<current-session-id|null>`
- claude session: `<current-session-id|null>`
- shared repository: `<absolute-path>`
- writable checkout policy: `separate-worktree`

## Subprojects

| ID | Owner | Developer | Reviewer | Depends on | Status |
|---|---|---|---|---|---|
| — | — | — | — | — | — |

## Authorization holds

None.

## Final confirmation

- status: `not_ready|awaiting_user|confirmed`
- requested_at: null
- confirmed_at: null
- confirmed_delivery_commit: null

