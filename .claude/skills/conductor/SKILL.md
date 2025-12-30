---
name: conductor
description: |
  Context-driven development methodology for organized, spec-first coding. Use when:
  - Project has a `conductor/` directory
  - User mentions specs, plans, tracks, or context-driven development
  - Files like `conductor/tracks.md`, `conductor/product.md`, `conductor/workflow.md` exist
  - User asks about project status, implementation progress, or track management
  - User wants to organize development work with TDD practices
  - User invokes `/conductor-*` commands (setup, newtrack, implement, implement-parallel, status, revert, validate, block, skip, revise, archive, export, refresh)
  - User mentions documentation is outdated or wants to sync context with codebase changes
  - User wants to run tasks in parallel or mentions multi-agent execution
  
  Interoperable with Gemini CLI extension and Claude Code commands.
  Integrates with Beads for persistent task memory across sessions.
  Supports parallel execution with MCP Agent Mail for agent coordination.
---

# Conductor: Context-Driven Development

Measure twice, code once.

## Overview

Conductor enables context-driven development by:
1. Establishing project context (product vision, tech stack, workflow)
2. Organizing work into "tracks" (features, bugs, improvements)
3. Creating specs and phased implementation plans
4. Executing with TDD practices and progress tracking

## Context Loading

When this skill activates, load these files to understand the project:
1. `conductor/product.md` - Product vision and goals
2. `conductor/tech-stack.md` - Technology constraints
3. `conductor/workflow.md` - Development methodology (TDD, commits)
4. `conductor/tracks.md` - Current work status

For active tracks, also load:
- `conductor/tracks/<track_id>/spec.md`
- `conductor/tracks/<track_id>/plan.md`

## Beads Integration

Beads integration is **always attempted** for persistent task memory. If `bd` CLI is unavailable or fails, the user can choose to continue without it.

### Detection (MUST check before using bd commands)

Before using ANY `bd` command, you MUST verify:
1. `bd` CLI is installed: `which bd` returns a path
2. `conductor/beads.json` exists AND has `"enabled": true`

```bash
# Check availability - run this before any bd command
if which bd > /dev/null 2>&1 && [ -f conductor/beads.json ]; then
  BEADS_ENABLED=$(cat conductor/beads.json | grep -o '"enabled"[[:space:]]*:[[:space:]]*true' || echo "")
  if [ -n "$BEADS_ENABLED" ]; then
    # Beads is available and enabled - use bd commands
  fi
fi
```

### If Beads is NOT available:
- **DO NOT** run any `bd` commands
- Use only plan.md markers for task tracking
- All conductor commands work normally without Beads

### If Beads IS available:
- Tracks become Beads epics
- Tasks sync to Beads for persistent memory
- Use `bd ready` instead of manual task selection
- Notes survive context compaction

## Proactive Behaviors

1. **On new session**: Check for in-progress tracks, offer to resume
2. **On task completion**: Suggest next task or phase verification
3. **On blocked detection**: Alert user and suggest alternatives
4. **On all tasks complete**: Congratulate and offer archive/cleanup
5. **On stale context detected**: If setup >2 days old or significant codebase changes detected, suggest `/conductor-refresh`
6. **On Beads available**: If `bd` CLI detected during setup, offer integration

## Intent Mapping

| User Intent | Command |
|-------------|---------|
| "Set up this project" | `/conductor-setup` |
| "Create a new feature" | `/conductor-newtrack [desc]` |
| "Start working" / "Implement" | `/conductor-implement [id]` |
| "Run in parallel" / "Multi-agent" | `/conductor-implement-parallel [id]` |
| "What's the status?" | `/conductor-status` |
| "Undo that" / "Revert" | `/conductor-revert` |
| "Check for issues" | `/conductor-validate` |
| "This is blocked" | `/conductor-block` |
| "Skip this task" | `/conductor-skip` |
| "This needs revision" / "Spec is wrong" | `/conductor-revise` |
| "Save context" / "Handoff" / "Transfer to next section" | `/conductor-handoff` |
| "Archive completed" | `/conductor-archive` |
| "Export summary" | `/conductor-export` |
| "Docs are outdated" / "Sync with codebase" | `/conductor-refresh` |

## Parallel Mode

For tracks with multiple independent tasks, use `/conductor-implement-parallel`:

### When to Use Parallel Mode
- Track has 3+ tasks that don't depend on each other
- Tasks touch different files (no overlap)
- You want faster execution with multiple sub-agents

### Prerequisites
- `bv` (Beads Viewer) for parallel track detection
- MCP Agent Mail running (`am` command)
- `bd` (Beads CLI) for task tracking

### Quick Start
```bash
# Start MCP Agent Mail server
am

# Run parallel implementation
/conductor-implement-parallel auth_20241229 --max-agents=3
```

See [references/parallel-execution.md](references/parallel-execution.md) for details.

## References

- **Detailed workflows**: [references/workflows.md](references/workflows.md) - Step-by-step command execution
- **Directory structure**: [references/structure.md](references/structure.md) - File layout and status markers
- **Beads integration**: [references/beads-integration.md](references/beads-integration.md)
