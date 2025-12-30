# Conductor + Beads Integration

> **Status**: Implemented  
> **Version**: 0.1.0  
> **Date**: 2024-12-26

## Overview

This spec defines how Conductor's context-driven development methodology integrates with Beads' persistent task memory system, creating a unified workflow that combines:

- **Conductor**: Spec-first planning, human-readable context, TDD workflow
- **Beads**: Dependency-aware graph, cross-session memory, agent-optimized output

## Design Principles

1. **Conductor owns planning** - Specs, product vision, and phase organization
2. **Beads owns execution** - Task tracking, dependencies, and persistent memory
3. **Bidirectional sync** - Changes in either system reflect in both
4. **Graceful degradation** - If Beads unavailable, user can choose to continue without it

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     User / AI Agent                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Conductor Skill                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ product.md  │  │ tech-stack  │  │     workflow.md     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    tracks.md                             ││
│  │  [x] auth_20241226 ──────────┐                          ││
│  │  [~] api_20241226 ───────────┼──── Track references      ││
│  │  [ ] ui_20241227 ────────────┘                          ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                              │
                              │ Sync Layer
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Beads (.beads/)                         │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                  Dependency Graph                        ││
│  │                                                          ││
│  │    bd-a3f8 (Epic: auth_20241226)                        ││
│  │       ├── bd-a3f8.1 (Task: Write auth tests)            ││
│  │       │      └── bd-a3f8.1.1 (Subtask: JWT tests)       ││
│  │       └── bd-a3f8.2 (Task: Implement middleware)        ││
│  │              └── blocked-by: bd-a3f8.1                  ││
│  │                                                          ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

## Workflow Integration

### Phase 1: Setup (`/conductor-setup` + `bd init`)

When user runs `/conductor-setup`:

1. **Conductor creates** standard context files
2. **If Beads available**, also run:
   ```bash
   bd init              # Initialize .beads/ directory
   # OR for shared repos:
   bd init --stealth    # Keep Beads local-only
   ```
3. **Create linking config** in `conductor/beads.json`:
   ```json
   {
     "enabled": true,
     "mode": "stealth",
     "sync": "bidirectional",
     "epicPrefix": "conductor"
   }
   ```

### Phase 2: Track Creation (`/conductor-newtrack` + `bd create`)

When creating a new track:

```
User: /conductor-newtrack Add user authentication

Conductor Actions:
1. Create conductor/tracks/auth_20241226/
2. Generate spec.md with requirements
3. Generate plan.md with phased tasks

Beads Actions:
4. Create epic: bd create "auth_20241226: Add user authentication" -p 1
   → Returns: bd-a3f8

5. For each phase task in plan.md:
   bd create "Write failing auth tests" -P bd-a3f8 -p 1
   → Returns: bd-a3f8.1
   
   bd create "Implement JWT middleware" -P bd-a3f8 -p 1
   → Returns: bd-a3f8.2

6. Set dependencies based on phase order:
   bd dep add bd-a3f8.2 bd-a3f8.1  # .2 blocked by .1
```

**Metadata linkage** in `conductor/tracks/auth_20241226/metadata.json`:
```json
{
  "id": "auth_20241226",
  "beads_epic": "bd-a3f8",
  "beads_tasks": {
    "phase1_task1": "bd-a3f8.1",
    "phase1_task2": "bd-a3f8.2"
  }
}
```

### Phase 3: Implementation (`/conductor-implement` + `bd ready`)

When implementing:

```
User: /conductor-implement auth_20241226

Combined Workflow:
1. Conductor loads spec.md and plan.md for context
2. Query Beads for ready tasks:
   bd ready --epic bd-a3f8
   → Shows tasks with no blockers

3. Select task and mark in-progress:
   bd update bd-a3f8.1 --status in_progress

4. Execute TDD workflow (from Conductor):
   - Write failing tests
   - Implement to pass
   - Refactor

5. On completion:
   bd update bd-a3f8.1 --notes "Implemented JWT auth tests, 95% coverage. commit: abc123"
   bd close bd-a3f8.1 --reason "Task completed"
   
6. Conductor updates plan.md with commit SHA
```

### Phase 4: Status & Progress (`/conductor-status` + `bd show`)

```
User: /conductor-status

Output combines both sources:

# Project Status

## Active Tracks (from Conductor)
- [~] auth_20241226 - Add user authentication

## Task Progress (from Beads)
bd ready --epic bd-a3f8

Ready to work:
  bd-a3f8.2  P1  Implement JWT middleware

Completed:
  bd-a3f8.1  ✓   Write failing auth tests

## Blocked Tasks
bd show bd-a3f8.3 --deps
  Blocked by: bd-a3f8.2 (in progress)
```

### Phase 5: Blocking & Dependencies (`/conductor-block` + `bd dep`)

```
User: /conductor-block - External API not ready

Actions:
1. Conductor marks task [B] in plan.md
2. Beads records blocker:
   bd update bd-a3f8.2 --status blocked --note "External API not ready"
   
3. If blocking relationship to another task:
   bd dep add bd-a3f8.2 bd-external-api
```

### Phase 6: Session Resume (Beads shines here)

After context compaction or new session:

```
Agent Session Start:
1. bd ready                    # What can I work on?
2. bd show bd-a3f8.1           # Get full context: notes, design, acceptance
3. Read notes for: COMPLETED, IN PROGRESS, NEXT, KEY DECISIONS
4. Load conductor/tracks/auth_20241226/spec.md for context
5. Resume work
```

Beads' persistent notes survive conversation compaction, while Conductor's markdown provides human-readable context.

### Structured Notes Format

When updating notes for session resume, use this format:

```bash
bd update <epic_id> --notes "COMPLETED: Phase 1 - Auth tests
KEY DECISION: Using RS256 for JWT signing (enables key rotation)
IN PROGRESS: Phase 2 - Middleware implementation
NEXT: Implement token validation
BLOCKER: None
DISCOVERED: Found race condition in token refresh (created bd-xyz)"
```

| Field | Purpose |
|-------|---------|
| `COMPLETED:` | What was finished (past tense, specific) |
| `KEY DECISION:` | Important choices with rationale |
| `IN PROGRESS:` | Current work |
| `NEXT:` | Immediate next step (concrete action) |
| `BLOCKER:` | What's blocking (if any) |
| `DISCOVERED:` | New issues found during work |

### Session End Protocol

**CRITICAL**: Always run at session end or handoff:

```bash
bd update <epic_id> --notes "..." # Save context
bd sync                           # Force sync to remote
```

This bypasses the 30-second debounce and ensures changes persist immediately.

## Command Mapping

| Conductor Command | Beads Equivalent | Integration |
|-------------------|------------------|-------------|
| `/conductor-setup` | `bd init` | Run both |
| `/conductor-newtrack` | `bd create` (epic + tasks) | Create track + epic with `--design`, `--acceptance` |
| `/conductor-implement` | `bd ready`, `bd update`, `bd close` | Query ready, track progress, complete |
| `/conductor-implement-parallel` | `bv --robot-plan`, `bd ready`, `bd close` | Query parallel tracks, Orchestrator updates Beads |
| `/conductor-status` | `bd ready`, `bd show` | Combine outputs, read notes for context |
| `/conductor-block` | `bd update --status blocked` | Sync both with structured notes |
| `/conductor-skip` | `bd close` or `bd update` | Mark in both based on skip reason |
| `/conductor-handoff` | `bd update --notes`, `bd sync` | Save context + force sync |
| `/conductor-revert` | `bd reopen` | Sync status |
| `/conductor-archive` | `bd compact --auto` | Archive track + compact |

## Parallel Execution Integration

When using `/conductor-implement-parallel`, Beads plays a critical role:

### Pre-Execution
- `bv --robot-plan --json` queries the Beads dependency graph
- Returns tasks grouped by independent tracks (can run in parallel)
- Identifies "highest impact" tasks that unblock the most downstream work

### During Execution
- **Orchestrator ONLY** updates Beads - sub-agents never run `bd` commands
- Sub-agents report completion via JSON response
- Orchestrator runs `bd close` after merging each agent's work

### Parallel-Specific Commands
```bash
# Query parallel tracks
bv --robot-plan --json

# Example response structure:
{
  "plan": {
    "tracks": [
      {"track_id": "track-A", "items": [{"id": "bd-123", "unblocks": ["bd-124"]}]}
    ],
    "summary": {"highest_impact": "bd-123", "total_actionable": 10}
  }
}
```

### Cross-Phase Parallelism
The `parallel_analysis` in `metadata.json` enables optimized execution:
- Task 2.1 can start when only its specific dependency (1.1) completes
- Doesn't wait for all of Phase 1 if not required
- Beads tracks these fine-grained dependencies

## Data Synchronization

### Conductor → Beads (Plan Changes)

When `plan.md` is edited:
1. Detect new/removed/reordered tasks
2. Create/close Beads issues accordingly
3. Update dependency graph for new order

### Beads → Conductor (Status Changes)

When `bd close` or `bd update` runs:
1. Update corresponding task in `plan.md`
2. Add commit SHA if available
3. Update `tracks.md` status if epic complete

### Conflict Resolution

Priority: **Beads is source of truth for status**, Conductor is source of truth for specs.

```
Conflict: plan.md says [x], Beads says active
Resolution: Update plan.md to match Beads (agent likely working)

Conflict: Beads has task not in plan.md
Resolution: Add to plan.md under "Unplanned Tasks" section
```

## Configuration

### conductor/beads.json

```json
{
  "enabled": true,
  "mode": "stealth|normal",
  "sync": "bidirectional|conductor-to-beads|manual",
  "epicPrefix": "conductor",
  "autoCreateTasks": true,
  "autoSyncOnComplete": true,
  "compactOnArchive": true
}
```

### Detection Logic

Skill activation checks:
1. Does `conductor/` exist? → Load Conductor
2. Does `.beads/` exist? → Also load Beads integration
3. Both present? → Use combined workflow

## Implementation Phases

### Phase 1: Basic Integration (MVP)
- [ ] Add `bd init` to `/conductor-setup`
- [ ] Create epic on `/conductor-newtrack`
- [ ] Query `bd ready` in `/conductor-implement`
- [ ] Sync completion status

### Phase 2: Full Sync
- [ ] Bidirectional plan.md ↔ Beads sync
- [ ] Dependency graph from phase order
- [ ] Status aggregation in `/conductor-status`

### Phase 3: Advanced Features
- [ ] Beads compaction on archive
- [ ] Cross-track dependencies via Beads
- [ ] Team sync via git + Beads

## Benefits

| Capability | Conductor Only | With Beads |
|------------|----------------|------------|
| Cross-session memory | Git notes | Persistent graph |
| Dependency tracking | Phase order | Full DAG |
| Ready task detection | Manual | `bd ready` |
| Context after compaction | Re-read files | `bd show --notes` |
| Multi-agent coordination | File locks | Hash-based IDs |

## Open Questions

1. **Stealth by default?** - Should integration use `bd init --stealth` to avoid polluting shared repos?

2. **Sync frequency** - Real-time vs. on-command sync?

3. **Skill loading** - Load both skills separately or create unified `conductor-beads` skill?

4. **Fallback behavior** - ~~If `bd` not installed, silent skip or prompt to install?~~ **Resolved**: Always attempt Beads; prompt user to choose if unavailable.

## References

- [Conductor Skill](../.claude/skills/conductor/SKILL.md)
- [Beads Documentation](https://github.com/steveyegge/beads)
- [Beads Agent Instructions](https://github.com/steveyegge/beads/blob/main/AGENT_INSTRUCTIONS.md)
