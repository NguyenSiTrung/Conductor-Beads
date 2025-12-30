# Parallel Task Execution Design

## Overview

This document describes the architecture for enabling parallel task execution in Conductor using MCP Agent Mail for agent coordination, Beads Viewer (`bv`) for dependency analysis, and Claude's Task tool for sub-agent spawning.

## Problem Statement

Current Conductor implementation executes tasks **sequentially** (one-by-one from `plan.md`). This is inefficient when:
- Multiple independent tasks exist within a phase
- Tasks don't share file dependencies
- Human oversight bottleneck is minimized

## Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    conductor-implement --parallel                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      ORCHESTRATOR                             │   │
│  │  • Owns plan.md, Beads updates, state management             │   │
│  │  • Computes parallel tracks via bv --robot-plan              │   │
│  │  • Manages file leases via MCP Agent Mail                    │   │
│  │  • Spawns/monitors sub-agents                                │   │
│  │  • Handles merging and conflict resolution                   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│              ┌───────────────┼───────────────┐                      │
│              ▼               ▼               ▼                      │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐       │
│  │  Task Agent 1   │ │  Task Agent 2   │ │  Task Agent 3   │       │
│  │  (GreenCastle)  │ │  (BlueLake)     │ │  (RedMountain)  │       │
│  │                 │ │                 │ │                 │       │
│  │  • TDD workflow │ │  • TDD workflow │ │  • TDD workflow │       │
│  │  • File leases  │ │  • File leases  │ │  • File leases  │       │
│  │  • Heartbeats   │ │  • Heartbeats   │ │  • Heartbeats   │       │
│  │  • Git worktree │ │  • Git worktree │ │  • Git worktree │       │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘       │
│              │               │               │                      │
│              └───────────────┴───────────────┘                      │
│                              │                                       │
│                              ▼                                       │
│                    MCP Agent Mail Server                            │
│                    • Agent identities                               │
│                    • File reservations                              │
│                    • Inbox/outbox messaging                         │
│                    • Pre-commit hooks                               │
└─────────────────────────────────────────────────────────────────────┘
```

## Dependencies

### Required Tools

| Tool | Purpose | Installation |
|------|---------|--------------|
| `bd` (Beads CLI) | Task management, dependency tracking | `go install github.com/steveyegge/beads/cmd/bd@latest` |
| `bv` (Beads Viewer) | Graph analytics, parallel track planning | See [beads_viewer](https://github.com/Dicklesworthstone/beads_viewer) |
| MCP Agent Mail | Agent coordination, file leases | See below |

### MCP Agent Mail Setup

MCP Agent Mail is an **HTTP-based MCP server** that runs locally and integrates with coding agents.

#### One-Line Installation

```bash
curl -fsSL https://raw.githubusercontent.com/Dicklesworthstone/mcp_agent_mail/main/install.sh | bash
```

This installer:
1. Installs Python 3.14 venv with `uv`
2. Starts MCP HTTP server on port 8765
3. Auto-detects and configures Claude Code, Codex CLI, Gemini CLI
4. Installs `bd` and `bv` (unless `--skip-beads` / `--skip-bv`)
5. Creates `am` shell alias for quick server startup

#### Starting the Server

```bash
# After installation, just type:
am

# Or with custom port:
am --port 8766
```

#### Manual Client Integration

For Claude Code, the integration script updates `.claude/settings.json`:

```bash
scripts/integrate_claude_code.sh
```

For other tools:
- **Codex CLI**: `scripts/integrate_codex_cli.sh` → updates `~/.codex/config.toml`
- **Gemini CLI**: `scripts/integrate_gemini_cli.sh` → updates `~/.gemini/settings.json`

#### Amp Code Integration

Amp supports HTTP-based MCP servers natively. After starting the MCP Agent Mail server:

**Option 1: CLI**
```bash
# Add to Amp (no auth needed for localhost by default)
amp mcp add agent-mail http://127.0.0.1:8765/mcp/

# Or with bearer token if required:
amp mcp add agent-mail --header "Authorization=Bearer YOUR_TOKEN" http://127.0.0.1:8765/mcp/
```

**Option 2: Workspace Settings** (`.amp/settings.json` in project root)
```json
{
  "amp.mcpServers": {
    "agent-mail": {
      "url": "http://127.0.0.1:8765/mcp/"
    }
  }
}
```

**Option 3: Global Settings** (`~/.amp/settings.json`)
```json
{
  "amp.mcpServers": {
    "agent-mail": {
      "url": "http://127.0.0.1:8765/mcp/",
      "headers": {
        "Authorization": "Bearer ${AGENT_MAIL_TOKEN}"
      }
    }
  }
}
```

**Option 4: VS Code Settings** (with Amp extension)
```json
{
  "amp.mcpServers": {
    "agent-mail": {
      "url": "http://127.0.0.1:8765/mcp/"
    }
  }
}
```

**Verify Connection:**
```bash
amp -x "List the MCP tools available from agent-mail"
```

#### Server Configuration

The server runs at `http://127.0.0.1:8765/mcp/` by default.

Key environment variables (in `.env`):

| Variable | Default | Purpose |
|----------|---------|---------|
| `HTTP_HOST` | `127.0.0.1` | Bind host |
| `HTTP_PORT` | `8765` | Bind port |
| `HTTP_PATH` | `/mcp/` | MCP endpoint path |
| `HTTP_BEARER_TOKEN` | (generated) | Auth token |
| `HTTP_ALLOW_LOCALHOST_UNAUTHENTICATED` | `true` | Skip auth for localhost |
| `STORAGE_ROOT` | `~/.mcp_agent_mail_git_mailbox_repo` | Data directory |

#### Verifying Installation

```bash
# Check server is running
curl http://127.0.0.1:8765/health/liveness

# Test MCP endpoint (requires token if auth enabled)
curl -X POST http://127.0.0.1:8765/mcp/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

### Key MCP Agent Mail Features Used

1. **Agent Identities**: Memorable names (e.g., `GreenCastle`) for sub-agents
2. **File Reservations (Leases)**: Advisory locks to prevent concurrent edits
3. **Inbox/Outbox Messaging**: Structured communication between orchestrator and agents
4. **Pre-commit Hook**: Enforces lease compliance before commits

---

## Command: `conductor-implement --parallel`

### Usage

```bash
/conductor-implement --parallel [track_id]

Options:
  --max-agents=N      Maximum concurrent agents (default: 3)
  --max-tasks=N       Maximum tasks per batch (optional)
  --strategy=beads    Use bv --robot-plan for parallel detection (default)
  --dry-run           Show proposed parallel plan without executing
```

### Execution Flow

```
1. SETUP CHECK (existing)
   └─ Verify conductor/ structure exists

2. TRACK SELECTION (existing)
   └─ Select track to implement

3. PARALLEL PLANNING (new)
   ├─ Run: bv --robot-plan (JSON output)
   ├─ Identify independent task tracks
   ├─ Get highest_impact task from summary
   └─ Compute file affinity for each task

4. ORCHESTRATION LOOP (new)
   ├─ While tasks remain:
   │   ├─ Select ready tasks (deps satisfied, files available)
   │   ├─ Request file leases via MCP Agent Mail
   │   ├─ Spawn Task Agents (up to max-agents)
   │   ├─ Monitor agent messages:
   │   │   ├─ heartbeat → refresh leases, track TDD phase
   │   │   ├─ lease_request → evaluate and grant/deny
   │   │   ├─ task_complete → merge, update plan.md, close Beads
   │   │   ├─ task_blocked → mark [!], log blocker
   │   │   └─ task_failed → retry or escalate
   │   └─ Merge completed work into integration branch
   └─ Phase completion triggers checkpoint protocol

5. FINALIZATION (existing, adapted)
   └─ Update tracks.md, cleanup state
```

---

## Sub-Agent Protocol

### Identity Registration

Each sub-agent registers with MCP Agent Mail:

```python
register_agent(
    project_key="/path/to/project",
    program="claude-code",
    model="claude-sonnet",
    name="GreenCastle",  # Memorable identity
    task_description="Implementing phase1_task3: Create user auth module"
)
```

### File Reservation Protocol

Before starting work, orchestrator reserves files for agent:

```python
file_reservation_paths(
    project_key="/path/to/project",
    agent_name="GreenCastle",
    paths=[
        "src/auth/login.ts",
        "src/auth/login.test.ts",
        "src/auth/types.ts"
    ],
    ttl_seconds=1800,  # 30 minutes, refreshed by heartbeat
    exclusive=True,
    reason="bd-a3f8.1.1: Implement login endpoint"
)
```

### Message Types

| Direction | Message Type | Purpose |
|-----------|--------------|---------|
| Orch → Agent | `task_assigned` | Initial task payload with leases |
| Orch → Agent | `lease_updated` | Extended/modified lease |
| Orch → Agent | `suspend` | Pause work (conflict detected) |
| Orch → Agent | `cancel` | Abort task |
| Agent → Orch | `heartbeat` | TDD phase, files touched, health |
| Agent → Orch | `lease_request` | Request additional files |
| Agent → Orch | `task_complete` | Commit SHA, summary, coverage |
| Agent → Orch | `task_blocked` | External blocker detected |
| Agent → Orch | `task_failed` | Implementation or test failure |

### Heartbeat Structure

```json
{
  "agent_id": "conductor/auth-track/task/phase1_task3",
  "task_id": "bd-a3f8.1.1",
  "tdd_phase": "green",
  "files_touched": ["src/auth/login.ts"],
  "tests_status": "passing",
  "coverage": 85.2,
  "timestamp": "2024-12-29T10:30:00Z"
}
```

---

## Parallel State Management

### State File: `implement_parallel_state.json`

```json
{
  "mode": "parallel",
  "track_id": "auth_20241229",
  "max_agents": 3,
  "status": "running",
  "agents": {
    "GreenCastle": {
      "task_id": "bd-a3f8.1.1",
      "plan_key": "phase1_task3",
      "status": "running-green",
      "leases": ["src/auth/login.ts", "src/auth/login.test.ts"],
      "last_heartbeat": "2024-12-29T10:30:00Z",
      "tdd_phase": "green",
      "worktree": "worktrees/auth_20241229/phase1_task3"
    }
  },
  "tasks": {
    "phase1_task1": {"status": "completed", "commit_sha": "abc1234"},
    "phase1_task2": {"status": "completed", "commit_sha": "def5678"},
    "phase1_task3": {"status": "in_progress", "agent_id": "GreenCastle"},
    "phase1_task4": {"status": "pending", "deps": ["phase1_task3"]}
  },
  "phases": [
    {"name": "Phase 1", "status": "in_progress", "checkpoint": null}
  ],
  "last_updated": "2024-12-29T10:30:00Z"
}
```

### Task Status Transitions

```
pending → assigned → running-red → running-green → verifying → completed
                                                             ↘ blocked
                                                             ↘ failed
```

---

## Git Workflow for Parallel Execution

### Branch Strategy

```
main
 └── track/auth_20241229/integration    # Orchestrator's integration branch
      ├── track/auth_20241229/phase1_task1  # Agent 1's branch
      ├── track/auth_20241229/phase1_task2  # Agent 2's branch
      └── track/auth_20241229/phase1_task3  # Agent 3's branch
```

### Git Worktrees

Each agent works in isolated worktree:

```bash
# Orchestrator creates worktrees
git worktree add worktrees/auth_20241229/phase1_task3 \
    -b track/auth_20241229/phase1_task3

# Agent works in worktree
cd worktrees/auth_20241229/phase1_task3
# ... TDD workflow ...
git commit -m "feat(auth): Implement login endpoint [Task-Id: bd-a3f8.1.1]"

# Orchestrator merges after task_complete
git checkout track/auth_20241229/integration
git merge track/auth_20241229/phase1_task3 --no-ff
```

### Pre-commit Hook (MCP Agent Mail)

```bash
#!/bin/bash
# Installed by MCP Agent Mail in .git/hooks/pre-commit

# Check lease compliance
agent_mail check-leases --staged-files

# Verify Task-Id in commit message
if ! grep -q "Task-Id:" "$1"; then
    echo "ERROR: Commit must include Task-Id"
    exit 1
fi

# Run tests for modified files
npm test -- --findRelatedTests $(git diff --cached --name-only)
```

---

## Conflict Resolution

### Proactive: Lease-based Prevention

```
Agent A requests lease for src/shared/utils.ts
Agent B already has exclusive lease
→ Orchestrator denies Agent A's request
→ Agent A must:
   1. Wait for Agent B to release, OR
   2. Refactor to avoid that file, OR
   3. Report blocker
```

### Reactive: Merge Conflicts

```
1. Agent completes task, sends task_complete
2. Orchestrator attempts merge into integration branch
3. If conflict:
   a. Create new Beads task: "Resolve merge conflict: phase1_task3"
   b. Assign to original agent (preferred) or merge-specialist agent
   c. Agent resolves conflict, runs tests, commits
   d. Retry merge
```

---

## TDD Workflow Adaptation

Each sub-agent follows the standard TDD workflow from `workflow.md`, with these adaptations:

| Step | Sequential Mode | Parallel Mode |
|------|-----------------|---------------|
| Select Task | Agent reads plan.md | Orchestrator assigns task |
| Mark In Progress | Agent edits plan.md | Agent signals; Orchestrator edits |
| Write Tests (Red) | Direct file access | Within leased files only |
| Implement (Green) | Direct file access | Within leased files only |
| Commit | Direct to branch | To worktree branch |
| Update plan.md | Agent edits | Orchestrator edits |
| Beads update | Agent calls `bd` | Orchestrator calls `bd` |

---

## Error Handling

### Agent Failure Categories

| Category | Indicators | Action |
|----------|------------|--------|
| Tool Error | Command fails to execute | Retry once; mark blocked if persists |
| Spec/Plan Issue | Requirement mismatch | Trigger Revise workflow |
| Implementation Bug | Tests fail | Allow 2 fix attempts; then block |
| Lease Violation | Pre-commit rejects | Agent must request lease or revert |
| Stalled Agent | Missing heartbeats | Rollback worktree; reassign task |

### Heartbeat Monitoring

```python
# Orchestrator monitoring loop
for agent in active_agents:
    if now - agent.last_heartbeat > HEARTBEAT_TIMEOUT:
        if agent.missed_heartbeats >= MAX_MISSED:
            mark_agent_stalled(agent)
            rollback_worktree(agent.worktree)
            requeue_task(agent.task_id)
        else:
            agent.missed_heartbeats += 1
```

---

## Workflow.md Additions

New section to add to `templates/workflow.md`:

```markdown
## Parallel Task Workflow (Orchestrated Mode)

When using `conductor-implement --parallel`, the workflow adapts:

### Orchestrator Responsibilities
- Task selection based on `bv --robot-plan` dependencies
- File lease management via MCP Agent Mail
- `plan.md` updates (status markers, commit SHAs)
- Beads status updates (`bd update`, `bd close`)
- Phase checkpoint coordination

### Agent Responsibilities
- Execute TDD workflow within assigned task scope
- Respect file leases (only modify leased files)
- Send heartbeats with TDD phase and progress
- Report completion, blockers, or failures via messaging
- Never modify `plan.md`, Beads data, or Conductor control files

### File Access Rules
- Agents MUST only modify files for which they hold an active lease
- Shared utilities (`/lib`, `/test/util`) default to read-only
- Exclusive write requires explicit lease grant
- Pre-commit hook enforces compliance

### Integration Protocol
- Each agent works in isolated git worktree/branch
- Orchestrator merges completed work into integration branch
- Full test suite runs after each merge
- Phase checkpoint created only after all phase tasks integrated
```

---

## Implementation Phases

### Phase 1: Foundation
- [ ] Create `conductor-implement-parallel.md` command
- [ ] Implement parallel state file management
- [ ] Add `bv --robot-plan` integration
- [ ] Set up basic orchestrator loop

### Phase 2: Agent Coordination
- [ ] Integrate MCP Agent Mail for identities
- [ ] Implement file lease request/release
- [ ] Create message protocol handlers
- [ ] Add heartbeat monitoring

### Phase 3: Git Workflow
- [ ] Implement worktree creation/cleanup
- [ ] Add branch merge logic
- [ ] Install pre-commit hook integration
- [ ] Handle merge conflict detection

### Phase 4: Integration
- [ ] Update `workflow.md` with parallel mode section
- [ ] Add parallel mode to Conductor skill
- [ ] Create Gemini CLI TOML command variant
- [ ] Write documentation and examples

### Phase 5: Testing & Hardening
- [ ] Error handling for all failure modes
- [ ] Resume from interrupted parallel execution
- [ ] Performance optimization
- [ ] User documentation

---

## References

- [bd (Beads CLI)](https://github.com/steveyegge/beads) - Task CRUD, dependencies, sync
- [bv (Beads Viewer)](https://github.com/Dicklesworthstone/beads_viewer) - Graph analysis, parallel planning
- [MCP Agent Mail](https://github.com/Dicklesworthstone/mcp_agent_mail) - Agent coordination
- [conductor-implement.md](../.claude/commands/conductor-implement.md)
- [workflow.md](../templates/workflow.md)

---

## Appendix: bv Robot Flags Reference

### bv --robot-plan
**Purpose:** Groups actionable work into parallel tracks with unblocks info

**Algorithm:**
1. Find actionable issues (no open blockers)
2. Compute what each item unblocks when completed
3. Find connected components via union-find
4. Build parallel tracks from components
5. Identify highest-impact item (most unblocks)

**Key Output Fields:**
```json
{
  "plan": {
    "tracks": [{"track_id": "...", "items": [...]}],
    "summary": {
      "highest_impact": "bd-123",
      "unblocks_count": 5
    }
  }
}
```

### bv --robot-priority
**Purpose:** Priority misalignment detection with recommendations

**Scoring Weights:**
- PageRank: 0.22 (foundational dependencies)
- Betweenness: 0.20 (bottlenecks)
- Blocker ratio: 0.13 (direct blocking count)
- Time-to-impact: 0.10 (critical path depth)
- Urgency: 0.10 (urgent labels + decay)
- Risk: 0.10 (volatility signals)
- Staleness: 0.05 (age-based surfacing)
- Priority boost: 0.10 (explicit priority)

### bv --robot-insights
**Purpose:** Full graph metrics (9 analysis types)

**Metrics:**
- **PageRank**: Recursive importance (foundational blockers)
- **Betweenness**: Shortest-path traffic (bottlenecks)
- **HITS**: Hub/Authority duality (epics vs. utilities)
- **Critical Path**: Longest chain (zero-slack keystones)
- **Eigenvector**: Influence via neighbors
- **K-core**: Structural cohesion
- **Articulation**: Cut vertices (disconnection points)
- **Slack**: Longest-path slack (parallel flexibility)
- **Cycles**: Circular dependency detection

### bv --robot-diff --diff-since <ref>
**Purpose:** Compare current state to historical point

**Accepts:**
- Git commit SHA: `--diff-since abc123`
- Tag: `--diff-since v1.0.0`
- Relative ref: `--diff-since HEAD~30`
- Date: `--diff-since 2024-01-01`

**Output:**
- new_issues, closed_issues, modified_issues
- new_cycles, resolved_cycles
- health_trend: "improving" | "degrading" | "stable"
