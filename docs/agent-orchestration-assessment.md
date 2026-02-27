# Agent Orchestration Framework — Fitness Assessment

**Date**: 2026-02-11
**Branch**: `agent-orchestration` (on fork `chrisbodhi/kanban`)
**Assessed by**: Claude Code (Opus 4.6)

## Vision

Use this kanban project as the bedrock for an agent orchestration framework where:

1. Claude Code instances **accept work** (get assigned cards)
2. Agents **report status** (leave comments, ask questions)
3. Agents **move themselves through states** ("ready for review", "completed")
4. **One interface** manages different Claude instances across repos (or the same repo)
5. Mostly **git-backed**, but not always (e.g., local todo lists)

---

## What Already Works

### MCP Server (37 tools, full CLI/TUI parity)
Any Claude Code instance can already create cards, move them between columns, update metadata, and query board state. The subprocess-based architecture is robust (auto-retry on conflicts, 30s timeout per operation).

### `KanbanOperations` Trait — The Right Contract
This single trait (50+ methods) defines every board operation and is implemented by TUI, CLI, and MCP contexts. A new "agent orchestration" context would slot in cleanly. Any new capabilities added to the trait automatically propagate to all interfaces.

### File-based JSON + Conflict Detection
For git-backed repos, the atomic-write + metadata-based conflict detection is exactly right. Multiple Claude instances can work on the same board file. Optimistic concurrency with retry handles contention.

### Card Dependencies
`Blocks`, `RelatesTo`, and `ParentOf` edges with cycle detection model "this card can't start until that one finishes" — maps directly to agent task ordering.

### Sprint Management
Sprints group work into iterations. An orchestrator could create a sprint, assign cards to agents, activate it, and track completion.

### Export/Import
Moving work between boards (repos) is already a first-class operation.

---

## Gaps to Fill

### Gap 1: No Assignee Field on Cards
**Impact: Critical**

Cards have no `assigned_to` field. Can't assign a card to "claude-instance-repo-A" vs "claude-instance-repo-B."

**Fix**: Add `assigned_to: Option<String>` to `Card`. Simple, backward-compatible (`#[serde(default)]`). Propagate through `CardUpdate`, `CreateCardOptions`, MCP tools.

**Estimated scope**: ~200 lines across domain, CLI, MCP.

### Gap 2: No Comment/Activity System
**Impact: Critical**

Cards have `sprint_logs` but no general comment system. Agents need to leave notes like "blocked on upstream API, see error X" or ask questions like "should this use the v2 or v3 endpoint?"

The `Loggable` trait exists in kanban-core but is **not implemented on Card**. The infrastructure is there, just needs wiring.

**Fix**: Implement `Loggable` on `Card`, or add a richer `comments: Vec<Comment>` field with author/timestamp attribution. Add `add_comment` / `list_comments` to `KanbanOperations`.

**Estimated scope**: ~300 lines across core, domain, CLI, MCP.

### Gap 3: Only 4 Card States
**Impact: Moderate**

`CardStatus` is `Todo | InProgress | Blocked | Done`. Missing "Ready for Review" and potentially others ("Waiting on Human", "Failed").

**Fix — two approaches**:
- **Extend the enum** — simplest, but every new state requires a code change
- **Use columns as states** — columns already represent workflow stages; cards moving to a "Ready for Review" column is the kanban-native approach. The existing `move_card` + `completion_column_id` mechanics already support this.

**Recommendation**: Column-based approach. Create boards with columns like `Backlog -> Assigned -> In Progress -> Ready for Review -> Done`. Agents move cards between them. No code change needed — just a board setup convention.

### Gap 4: No Agent Identity / Multi-Board Orchestration
**Impact: High**

Each MCP server instance points to one `kanban.json` file. No concept of a "workspace" spanning multiple boards across repos, and no agent registry.

**Fix — options**:
1. **Hub board pattern**: One "orchestration board" tracking agents and their assigned boards. Cards on the hub reference cards on repo-specific boards via metadata/description.
2. **Multi-file MCP**: Extend MCP server to manage multiple board files, keyed by path or name.
3. **Lightweight agent registry**: New model in kanban-domain for `Agent { id, name, board_path, status, last_seen }`.

**Recommendation**: Start with the hub board pattern (zero code changes, pure convention), then evolve to multi-file MCP as the pattern solidifies.

### Gap 5: No Notification / Event System
**Impact: Moderate**

No way to push events to agents. Agents must poll for new assignments. The file watcher exists in TUI but is for detecting external changes to render, not for triggering agent actions.

**Fix**: For v1, polling via MCP is fine (agents check their assigned cards periodically). For v2, add webhook/callback mechanism or use file watcher with a lightweight event bus.

### Gap 6: No HTTP API
**Impact: Low-Medium (depends on deployment model)**

MCP over stdio requires the orchestrator and agents to be local processes. Remote agents (CI/CD, remote machines) would need HTTP.

**Fix**: The `KanbanOperations` trait makes this straightforward — implement as an HTTP handler (axum/actix-web slots in cleanly). The roadmap already lists this as planned.

---

## Architecture Fit Score

| Criterion | Score | Notes |
|-----------|-------|-------|
| Clean abstractions | 9/10 | `KanbanOperations` is exactly the right interface |
| Extensibility | 9/10 | Trait-based, SOLID, easy to add new fields |
| Persistence model | 8/10 | JSON files work great for git-backed; conflict detection is solid |
| Agent communication | 3/10 | No comments, no assignees, no events — all must be added |
| Multi-board orchestration | 2/10 | Single-board-per-instance; needs hub pattern or multi-file support |
| State management | 7/10 | Column-based workflow is flexible; explicit states are limited but columns compensate |
| Existing MCP integration | 9/10 | Already works with Claude Code; full parity with CLI/TUI |
| Non-git use case | 8/10 | Pure JSON file, no git dependency in domain layer |

**Overall: 7/10 — strong bedrock, focused investment needed**

---

## Recommended Build Order

### Phase 1: Minimum Viable Agent Loop (small, high-impact)
1. **Add `assigned_to` to Card** — unlocks "who's working on what"
2. **Add comments/activity log to Card** — unlocks agent <-> human communication
3. **Use columns as workflow states** — convention, no code change
4. **Add `assign_card` and `add_comment` to KanbanOperations + MCP** — agents can accept work and report status

### Phase 2: Multi-Board Orchestration
5. **Hub board pattern** — manage work across repos via convention
6. **Multi-file MCP** — single MCP server managing multiple board files
7. **Agent registry model** — track which instances are alive and what they're working on

### Phase 3: Production Hardening
8. **HTTP API** — enable remote agents
9. **Event/notification system** — push instead of poll
10. **Agent health checks** — detect stale/crashed agents, reassign work

---

## Why This Project Over Starting Fresh

- Would be reimplementing `KanbanOperations`, JSON persistence, conflict detection, TUI, CLI, and MCP from scratch
- The dependency graph system (blocking, parent-child) is non-trivial and already correct
- Undo/redo snapshot system gives safe rollback if an agent makes a mistake
- Codebase is well-tested and published on crates.io
- SOLID architecture means agent-specific extensions compose naturally without touching existing code

---

## Development Workflow

All agent orchestration work happens on a **fork** to keep the upstream repo clean until the feature set is ready.

### Repository Structure

| Remote | Repo | Purpose |
|--------|------|---------|
| `origin` | `fulsomenko/kanban` | Upstream source of truth |
| `fork` | `chrisbodhi/kanban` | Agent orchestration development |

### Branch Strategy

```
fulsomenko/kanban:develop          (upstream, stable)
    └── chrisbodhi/kanban:agent-orchestration   (long-lived integration branch)
            ├── add-card-assignee-field          (PR #1 - merged)
            ├── add-card-comments                (future)
            ├── multi-board-mcp                  (future)
            └── ...
```

- **`agent-orchestration`** on the fork is the integration branch for all agent work
- Feature branches are created off `agent-orchestration` and PR'd back into it
- When the full feature set is ready, one PR from `chrisbodhi:agent-orchestration` -> `fulsomenko:develop` brings it upstream

### Day-to-Day Commands

```bash
# Start new feature work
git checkout agent-orchestration
git pull fork agent-orchestration
git checkout -b my-feature-branch

# Push and PR against agent-orchestration
git push fork my-feature-branch
gh pr create --repo chrisbodhi/kanban --base agent-orchestration

# Sync with upstream periodically
git fetch origin develop
git checkout agent-orchestration
git merge origin/develop
git push fork agent-orchestration
```

### Progress Tracking

| Phase | Item | Status |
|-------|------|--------|
| 1 | `assigned_to` field on Card | Done (PR #1) |
| 1 | Comments/activity log on Card | Pending |
| 1 | Column-based workflow convention | Pending |
| 2 | Hub board pattern | Pending |
| 2 | Multi-file MCP | Pending |
| 2 | Agent registry | Pending |
| 3 | HTTP API | Pending |
| 3 | Event/notification system | Pending |
| 3 | Agent health checks | Pending |
