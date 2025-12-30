# Beads Integration Reference

## Overview

Conductor integrates with [Beads](https://github.com/lostinspiration/beads) to provide persistent task memory that survives context compaction. This creates a hybrid system:

- **Conductor**: Human-readable specs and plans
- **Beads**: Agent-optimized task state with dependency tracking

**IMPORTANT: Beads integration is always attempted.** If `bd` CLI is unavailable or fails, the user is prompted to choose whether to continue without persistent task memory. Conductor still functions fully, just without cross-session state.

## CRITICAL: Availability Check

**You MUST run this check before using ANY `bd` command:**

```bash
# Check if Beads is available and enabled
BEADS_AVAILABLE=false
if which bd > /dev/null 2>&1; then
  if [ -f conductor/beads.json ]; then
    if grep -q '"enabled"[[:space:]]*:[[:space:]]*true' conductor/beads.json 2>/dev/null; then
      BEADS_AVAILABLE=true
    fi
  fi
fi

# ONLY use bd commands if BEADS_AVAILABLE is true
# Otherwise, skip all bd commands and use plan.md markers only
```

**If BEADS_AVAILABLE is false:**
- Do NOT run any `bd` commands
- Use `plan.md` markers for all task tracking
- Conductor workflows work identically without Beads

## How It Works (when Beads is enabled)

### Ownership Model

| Component | Conductor Owns | Beads Owns |
|-----------|----------------|------------|
| Specs | `spec.md` (requirements) | — |
| Plans | `plan.md` (task list) | — |
| Task State | Status markers `[ ]` `[~]` `[x]` | Full state + dependencies |
| Memory | Session-bound | Cross-session persistent |

### Track → Epic Mapping

Each Conductor track becomes a Beads epic:

```
conductor/tracks/auth_20250115/
├── spec.md          # Conductor: requirements
├── plan.md          # Conductor: task breakdown
└── metadata.json    # Links to Beads epic ID + task mapping
```

```
.beads/epics/auth_20250115.md   # Beads: persistent state
```

### Task ID Mapping

The `metadata.json` contains a `beads_tasks` mapping that links plan tasks to Beads IDs:

```json
{
  "track_id": "auth_20250115",
  "beads_epic": "bd-a3f8",
  "beads_tasks": {
    "phase1": "bd-a3f8.1",
    "phase1_task1": "bd-a3f8.1.1",
    "phase1_task2": "bd-a3f8.1.2",
    "phase2": "bd-a3f8.2",
    "phase2_task1": "bd-a3f8.2.1"
  }
}
```

**Key naming convention:**
- Phase keys: `phase{N}` (1-indexed)
- Task keys: `phase{N}_task{M}` (both 1-indexed)

**Usage during implementation:**
1. Store `beads_enabled=true` flag when Beads is detected
2. Load `beads_tasks` mapping from metadata.json
3. Generate task key from current phase/task index
4. Look up Beads ID from mapping
5. Use Beads ID for `bd update` and `bd close` commands

### Bidirectional Sync

1. **plan.md → Beads**: Tasks created/updated in plan sync to Beads
2. **Beads → plan.md**: Status changes reflect back to plan markers
3. **Conflict resolution**: Beads state is authoritative for status

## What Conductor READS from Beads

| What | Command | Used For |
|------|---------|----------|
| **Ready tasks** | `bd ready --epic <id>` | Task selection (dependency-aware, no blockers) |
| **Epic notes** | `bd show <epic_id>` | Session resume, context recovery after compaction |
| **Task status** | `bd show <task_id>` | Verify current state, check blockers |
| **Blocked info** | `bd show <task_id>` | Understand what's blocking and why |

### Context Recovery Flow (after compaction)

```bash
# 1. Read epic to get last session context
bd show <epic_id>
# Returns notes: "COMPLETED: Phase 1, IN PROGRESS: auth middleware, NEXT: add rate limiting"

# 2. Get ready tasks
bd ready --epic <epic_id>
# Returns: tasks with no blockers, sorted by priority

# 3. Resume with full context even if conversation history is gone
```

## What Conductor WRITES to Beads

| When | Command | Data Written |
|------|---------|--------------|
| **Task start** | `bd update --status in_progress` | Status change |
| **Task complete** | `bd close --reason` | Completion + commit SHA |
| **Task blocked** | `bd update --status blocked` | Block reason |
| **Phase complete** | `bd update --notes` | COMPLETED/IN PROGRESS/NEXT summary |
| **Handoff** | `bd update --notes` | Full session context for recovery |

## Configuration

Enable integration via `conductor/beads.json`:

```json
{
  "enabled": true,
  "auto_sync": true,
  "epic_prefix": "conductor",
  "sync_on_implement": true
}
```

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | boolean | Enable/disable Beads integration |
| `auto_sync` | boolean | Sync on every task completion |
| `epic_prefix` | string | Prefix for Beads epic IDs |
| `sync_on_implement` | boolean | Sync before `/conductor-implement` |

## Command Mapping

| Conductor Command | Beads Operations | Description |
|-------------------|------------------|-------------|
| `/conductor-setup` | `bd init` | Initialize Beads in project |
| `/conductor-newtrack` | `bd create` (epic + tasks) | Create epic with linked tasks |
| `/conductor-implement` | `bd ready` → `bd update` → `bd done` | Get next task, track progress, complete |
| `/conductor-implement-parallel` | `bv --robot-plan` → `bd close` | Query parallel tracks, Orchestrator updates Beads |
| `/conductor-status` | `bd ready`, `bd show` | Show available tasks, epic status |
| `/conductor-block` | `bd update --status blocked` | Mark task blocked with reason |
| `/conductor-skip` | `bd update --status skipped` | Skip task with justification |
| `/conductor-archive` | `bd compact --auto` | Archive completed epics |

### Example Flow

```bash
# User runs /conductor-implement

# 1. Conductor checks Beads for ready tasks
bd ready --epic auth_20250115

# 2. Agent works on task, updates status
bd update TASK-001 --status in_progress

# 3. On completion
bd close TASK-001 --reason "Implemented JWT validation"

# 4. Conductor updates plan.md markers
[x] Task 1: Implement JWT validation <!-- SHA: abc123 -->
```

## Benefits

### Cross-Session Memory
- Task state persists across context compaction
- Agent resumes exactly where it left off
- No re-reading of entire plan.md needed

### Compaction Survival Notes

When updating task status, use structured notes for recovery after compaction:

```bash
bd update TASK-001 --notes "COMPLETED: JWT validation with RS256
KEY DECISION: RS256 over HS256 for key rotation
IN PROGRESS: Password reset flow
NEXT: Implement rate limiting
BLOCKER: None
DISCOVERED: Found race condition in token refresh (created bd-xyz)"
```

**Notes format (use ALL relevant fields):**
- `COMPLETED:` - What was finished (past tense, specific)
- `KEY DECISION:` - Important choices made (with rationale)
- `IN PROGRESS:` - Current work
- `NEXT:` - Immediate next step (concrete action)
- `BLOCKER:` - What's blocking (if any)
- `DISCOVERED:` - New issues found during work (with beads ID)

**Best practices:**
- Write notes as if explaining to someone with zero context
- Include technical specifics, not vague progress
- Update notes BEFORE session end or handoff
- Use `bd sync` after updating notes to ensure persistence

This enables full context recovery after compaction with zero conversation history.

### Discovered-From Tracking

When discovering new work during implementation, link it:

```bash
# Create discovered issue with automatic linking
bd create "Found race condition in token refresh" \
  -t bug -p 2 \
  --deps discovered-from:<current_task_id> \
  --json
```

**Benefits:**
- Builds graph of agent thinking and discovery
- Reconstructs context: "Why was this created?"
- Tracks work trail: "What else was found?"

### Dependency-Aware Selection
- `bd ready` returns only unblocked tasks
- Respects task dependencies defined in plan
- Prevents out-of-order execution

### Audit Trail
- Every status change logged in Beads
- Notes attached to task completions
- Full history survives compaction

## Fallback Behavior (IMPORTANT)

### Decision Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Is `bd` command installed?                          │
│   Run: which bd > /dev/null 2>&1                            │
│   NO  → STOP. Do not use any bd commands.                   │
│   YES → Continue to Step 2                                  │
├─────────────────────────────────────────────────────────────┤
│ Step 2: Does conductor/beads.json exist?                    │
│   NO  → STOP. Do not use any bd commands.                   │
│   YES → Continue to Step 3                                  │
├─────────────────────────────────────────────────────────────┤
│ Step 3: Is "enabled": true in conductor/beads.json?         │
│   NO  → STOP. Do not use any bd commands.                   │
│   YES → Use Beads integration                               │
└─────────────────────────────────────────────────────────────┘
```

### Failure Handling

| Scenario | Behavior |
|----------|----------|
| `bd` not installed | Conductor operates normally without Beads |
| `bd` command fails | Log warning, continue with plan.md only |
| Sync conflict | Beads state wins, plan.md updated |
| `.beads/` missing | Auto-initialize on next command |

### Detection Logic

```python
# Pseudo-code for integration check
def should_use_beads():
    if not shutil.which("bd"):
        return False
    if not Path("conductor/beads.json").exists():
        return False
    config = load_json("conductor/beads.json")
    return config.get("enabled", False)
```

## Quick Reference

```bash
# Check if Beads is available
which bd

# Initialize integration
bd init
echo '{"enabled": true}' > conductor/beads.json

# View current state
bd show --epic conductor_<track_id>

# See what's ready to work on
bd ready
```
