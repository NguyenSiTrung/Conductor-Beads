# Parallel Execution Reference

## Overview

Parallel mode enables multiple sub-agents to work on independent tasks simultaneously, coordinated by an Orchestrator agent.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    conductor-implement-parallel                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      ORCHESTRATOR                             │   │
│  │  • Owns plan.md, Beads updates, state management             │   │
│  │  • Computes parallel tracks via bv --robot-plan              │   │
│  │  • Manages file leases via MCP Agent Mail                    │   │
│  │  • Spawns/monitors sub-agents                                │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│              ┌───────────────┼───────────────┐                      │
│              ▼               ▼               ▼                      │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐       │
│  │  Task Agent 1   │ │  Task Agent 2   │ │  Task Agent 3   │       │
│  │  (GreenCastle)  │ │  (BlueLake)     │ │  (RedMountain)  │       │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘       │
│                              │                                       │
│                    MCP Agent Mail Server                            │
└─────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

| Tool | Purpose | Check | Installation |
|------|---------|-------|--------------|
| `bd` | Task tracking | `which bd` | `go install github.com/steveyegge/beads/cmd/bd@latest` |
| `bv` | Parallel planning | `which bv` | See [beads_viewer](https://github.com/Dicklesworthstone/beads_viewer) |
| MCP Agent Mail | Coordination | `curl http://127.0.0.1:8765/health/liveness` | See below |

### MCP Agent Mail Setup

```bash
# One-line install
curl -fsSL https://raw.githubusercontent.com/Dicklesworthstone/mcp_agent_mail/main/install.sh | bash

# Start server
am

# Verify
curl http://127.0.0.1:8765/health/liveness
```

### Tool Integration

**For Amp Code:**
```bash
amp mcp add agent-mail http://127.0.0.1:8765/mcp/
```

**For Claude Code:**
```bash
scripts/integrate_claude_code.sh
```

## Command Usage

```bash
/conductor-implement-parallel [track_id] [options]

Options:
  --max-agents=N      Maximum concurrent agents (default: 3)
  --dry-run           Show proposed plan without executing
  --resume            Resume from saved state
```

## How It Works

### 1. Parallel Planning

The Orchestrator queries `bv --robot-plan` to identify independent task tracks:

```json
{
  "plan": {
    "tracks": [
      {"track_id": "track-A", "items": [{"id": "bd-123", "unblocks": ["bd-124"]}]},
      {"track_id": "track-B", "items": [{"id": "bd-200", "unblocks": []}]}
    ],
    "summary": {
      "highest_impact": "bd-123",
      "total_actionable": 10
    }
  }
}
```

Tasks within the same track can run in parallel (no mutual dependencies).

### 2. File Lease Management

Before spawning an agent, the Orchestrator requests exclusive file leases:

```python
file_reservation_paths(
    project_key="/path/to/project",
    agent_name="GreenCastle",
    paths=["src/auth/login.ts", "src/auth/login.test.ts"],
    ttl_seconds=3600,
    exclusive=True,
    reason="Task bd-123: Implement login"
)
```

If files conflict with another agent, the task is queued for the next batch.

### 3. Sub-Agent Spawning

Each agent receives:
- Task description and context
- List of leased files (only files they may modify)
- Worktree path for isolated work
- JSON response format for completion report

### 4. Merge Protocol

After agent completion:
1. Orchestrator verifies commit in worktree branch
2. Merges into integration branch with `--no-ff`
3. Runs integration tests
4. Updates `plan.md` with commit SHA
5. Closes Beads task with `bd close`
6. Releases file leases
7. Removes worktree

## State Management

Parallel state is persisted to `implement_parallel_state.json`:

```json
{
  "mode": "parallel",
  "started_at": "2024-12-29T10:00:00Z",
  "max_agents": 3,
  "current_batch": 1,
  "agents": {
    "GreenCastle": {
      "task_id": "bd-123",
      "plan_key": "phase1_task1",
      "status": "running",
      "leases": ["src/auth/login.ts"],
      "worktree": "worktrees/auth_20241229/phase1_task1"
    }
  },
  "integration_branch": "track/auth_20241229/integration"
}
```

Resume with `--resume` flag after interruption.

## Ownership Rules

| Resource | Orchestrator | Sub-Agent |
|----------|:------------:|:---------:|
| `plan.md` | ✅ Write | ❌ Read-only |
| `metadata.json` | ✅ Write | ❌ Read-only |
| `bd` commands | ✅ Run | ❌ Never |
| Leased files | Grant/Revoke | ✅ Write |
| Worktree branch | Merge/Delete | ✅ Commit |

**Critical:** Sub-agents NEVER modify `plan.md` or run Beads commands.

## Error Handling

| Error | Detection | Recovery |
|-------|-----------|----------|
| Agent stall | No response 10 min | Rollback worktree, reassign |
| Merge conflict | Git merge fails | Create resolution task |
| Test failure | Agent reports | Retry 2x, then block |
| Lease conflict | MCP returns conflict | Queue for next batch |
| MCP server down | Connection refused | Fallback to sequential |

## Comparison: Sequential vs Parallel

| Aspect | Sequential | Parallel |
|--------|------------|----------|
| Task selection | Agent reads plan.md | Orchestrator assigns |
| File access | Direct | Via leases only |
| Plan updates | Agent edits | Orchestrator edits |
| Beads sync | Agent runs `bd` | Orchestrator runs `bd` |
| Commits | Direct to branch | To worktree, then merge |
| Speed | 1x | Up to Nx (N = agents) |

## Quick Reference

```bash
# Start MCP Agent Mail
am

# Dry run to see plan
/conductor-implement-parallel auth_20241229 --dry-run

# Run with 3 agents
/conductor-implement-parallel auth_20241229 --max-agents=3

# Resume interrupted execution
/conductor-implement-parallel auth_20241229 --resume
```

## Related

- [beads-integration.md](beads-integration.md) - Beads CLI details
- [workflows.md](workflows.md) - TDD workflow
- [../../commands/conductor-implement-parallel.md](../../commands/conductor-implement-parallel.md) - Full command spec
