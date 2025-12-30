# Conductor Context

If a user mentions a "plan" or asks about the plan, and they have used the conductor extension in the current session, they are likely referring to the `conductor/tracks.md` file or one of the track plans (`conductor/tracks/<track_id>/plan.md`).

## Beads Integration

If `.beads/` directory exists alongside `conductor/`, this project uses Beads for persistent task memory. Check `conductor/beads.json` for integration config.

When Beads is enabled:
- Use `bd ready` to find tasks with no blockers
- Each Conductor track maps to a Beads epic
- Notes in Beads survive context compaction

## Parallel Execution

For tracks with multiple independent tasks, use `/conductor:implementParallel` for parallel sub-agent execution.

**Prerequisites:**
- `bv` (Beads Viewer) - for `--robot-plan` dependency analysis
- MCP Agent Mail - for file leases and agent coordination (`am` to start)
- `bd` (Beads CLI) - for task tracking

**Key concepts:**
- Orchestrator owns plan.md and Beads updates
- Sub-agents work in isolated git worktrees
- File leases prevent concurrent edit conflicts
- Merges happen into integration branch
