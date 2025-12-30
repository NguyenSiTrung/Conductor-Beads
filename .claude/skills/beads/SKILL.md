---
name: beads
description: >
  Tracks complex, multi-session work using the Beads issue tracker and dependency graphs, and provides
  persistent memory that survives conversation compaction. Use when work spans multiple sessions, has
  complex dependencies, or needs persistent context across compaction cycles. Trigger with phrases like
  "create task for", "what's ready to work on", "show task", "track this work", "what's blocking", or
  "update status".
allowed-tools: "Read,Bash(bd:*)"
version: "0.34.0"
author: "Steve Yegge <https://github.com/steveyegge>"
license: "MIT"
---

# Beads - Persistent Task Memory for AI Agents

Graph-based issue tracker that survives conversation compaction. Provides persistent memory for multi-session work with complex dependencies.

## Overview

**bd (beads)** replaces markdown task lists with a dependency-aware graph stored in git. Unlike TodoWrite (session-scoped), bd persists across compactions and tracks complex dependencies.

**Key Distinction**:
- **bd**: Multi-session work, dependencies, survives compaction, git-backed
- **TodoWrite**: Single-session tasks, linear execution, conversation-scoped

**Core Capabilities**:
- 📊 **Dependency Graphs**: Track what blocks what (blocks, parent-child, discovered-from, related)
- 💾 **Compaction Survival**: Tasks persist when conversation history is compacted
- 🐙 **Git Integration**: Issues versioned in `.beads/issues.jsonl`, sync with `bd sync`
- 🔍 **Smart Discovery**: Auto-finds ready work (`bd ready`), blocked work (`bd blocked`)
- 📝 **Audit Trails**: Complete history of status changes, notes, and decisions
- 🏷️ **Rich Metadata**: Priority (P0-P4), types (bug/feature/task/epic), labels, assignees

**When to Use bd vs TodoWrite**:
- ❓ "Will I need this context in 2 weeks?" → **YES** = bd
- ❓ "Could conversation history get compacted?" → **YES** = bd
- ❓ "Does this have blockers/dependencies?" → **YES** = bd
- ❓ "Is this fuzzy/exploratory work?" → **YES** = bd
- ❓ "Will this be done in this session?" → **YES** = TodoWrite
- ❓ "Is this just a task list for me right now?" → **YES** = TodoWrite

**Decision Rule**: If resuming in 2 weeks would be hard without bd, use bd.

## Prerequisites

**Required**:
- **bd CLI**: Version 0.34.0 or later installed and in PATH
- **Git Repository**: Current directory must be a git repo
- **Initialization**: `bd init` must be run once (humans do this, not agents)

**Verify Installation**:
```bash
bd --version  # Should return 0.34.0 or later
```

**First-Time Setup** (humans run once):
```bash
cd /path/to/your/repo
bd init  # Creates .beads/ directory with database
```

## Instructions

### Session Start Protocol

**Every session, start here:**

1. **Check for Ready Work**: `bd ready` - shows tasks with no open blockers
2. **Pick Highest Priority**: Choose P0 > P1 > P2 > P3 > P4
3. **Get Full Context**: `bd show <task-id>` - view details and dependencies
4. **Start Working**: `bd update <task-id> --status in_progress`
5. **Add Notes as You Work**: `bd update <task-id> --notes "Progress..."` (critical for compaction survival)

### Task Creation

```bash
bd create "Task title" -p 1 --type task
```

**Arguments**:
- **Priority**: 0-4 where 0=critical, 1=high, 2=medium, 3=low, 4=backlog
- **Type**: bug, feature, task, epic, chore

**Epic with Children**:
```bash
bd create "Epic: OAuth Implementation" -p 0 --type epic
# Returns: myproject-abc

bd create "Research OAuth providers" -p 1 --parent myproject-abc
bd create "Implement auth endpoints" -p 1 --parent myproject-abc
```

### Dependency Management

```bash
bd dep add <child-id> <parent-id>  # parent blocks child
```

**View Dependencies**: `bd dep list <task-id>`

### Completion

```bash
bd close <task-id> --reason "Completion summary"
bd ready  # Check newly unblocked work
```

## Essential Commands

| Command | Purpose |
|---------|---------|
| `bd ready` | Show tasks ready to work on |
| `bd create "Title" -p 1` | Create new task |
| `bd show <id>` | View task details |
| `bd update <id> --status in_progress` | Start working |
| `bd update <id> --notes "Progress"` | Add progress notes |
| `bd close <id> --reason "Done"` | Complete task |
| `bd dep add <child> <parent>` | Add dependency |
| `bd list` | See all tasks |
| `bd search <query>` | Find tasks by keyword |
| `bd sync` | Sync with git remote |

## Conductor Integration

When used with Conductor, Beads provides persistent task memory:

- Each Conductor track becomes a Beads epic
- Tasks in plan.md sync to Beads subtasks
- `bd ready` helps select next task
- Notes survive context compaction
- Phase dependencies map to Beads graph

### Parallel Execution Support

For `/conductor-implement-parallel`:
- **Beads Viewer (`bv`)** queries dependency graph with `bv --robot-plan --json`
- Returns independent task tracks that can run simultaneously
- **Orchestrator ONLY** runs `bd` commands - sub-agents never touch Beads
- Cross-phase dependencies enable fine-grained parallelism

See `conductor/beads.json` for integration config.

## Resources

- Full documentation: https://github.com/steveyegge/beads
- Conductor integration: See [conductor skill](../conductor/SKILL.md)
