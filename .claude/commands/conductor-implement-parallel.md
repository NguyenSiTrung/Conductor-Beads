---
description: Execute tasks from a track's plan using parallel sub-agents
argument-hint: [track_id] [--max-agents=N] [--dry-run]
---

<!-- 
SYSTEM DIRECTIVE: You are an AI agent orchestrator for the Conductor framework.
CRITICAL: You are the ORCHESTRATOR. Sub-agents NEVER modify plan.md or Beads directly.
You own all coordination: file leases, plan updates, Beads sync, merge operations.
-->

# Conductor Implement Parallel

Implement track in parallel mode: $ARGUMENTS

---

## 1.0 SETUP CHECK

**PROTOCOL: Verify Conductor environment and parallel execution prerequisites.**

1. **Check Required Files:** Verify existence of:
   - `conductor/product.md`
   - `conductor/tech-stack.md`
   - `conductor/workflow.md`

2. **Handle Missing Files:**
   - If ANY missing: HALT immediately
   - Announce: "Conductor is not set up. Please run `/conductor-setup` first."
   - Do NOT proceed.

3. **Parse Arguments:**
   - `track_id`: Optional track name to implement
   - `--max-agents=N`: Maximum concurrent agents (default: 3)
   - `--dry-run`: Show proposed parallel plan without executing

4. **Check Parallel Prerequisites:**

   a. **Beads Viewer (`bv`):** Execute `which bv`
      - **If NOT found:**
        > "⚠️ Parallel mode requires Beads Viewer (`bv`) for dependency analysis."
        > "Install from: https://github.com/Dicklesworthstone/beads_viewer"
        > "A) Continue with sequential mode instead"
        > "B) Stop - I'll install bv first"
        - If A: Redirect to `/conductor-implement`
        - If B: HALT

   b. **MCP Agent Mail:** Check for agent-mail MCP server
      - Run tool list to verify `agent-mail` MCP is available
      - **If NOT available:**
        > "⚠️ Parallel mode requires MCP Agent Mail for agent coordination."
        > "Start with: `am` (after installation)"
        > "A) Continue with sequential mode instead"
        > "B) Stop - I'll start MCP Agent Mail first"
        - If A: Redirect to `/conductor-implement`
        - If B: HALT

   c. **Beads CLI (`bd`):** Execute `which bd`
      - **If NOT found:**
        > "⚠️ Parallel mode requires Beads CLI (`bd`) for task tracking."
        - If A: Redirect to `/conductor-implement`
        - If B: HALT

---

## 2.0 TRACK SELECTION

**PROTOCOL: Identify and select the track to be implemented.**

*(Same as sequential mode - see conductor-implement.md sections 2.0)*

1. **Check for User Input:** Check if track name provided as argument.

2. **Parse Tracks File:** Read `conductor/tracks.md`
   - Split by `---` separator to identify track sections
   - Extract: status (`[ ]`, `[~]`, `[x]`), description, folder link
   - **CRITICAL:** If no track sections found: "The tracks file is empty or malformed." → HALT

3. **Select Track:**
   - If track name provided: Exact, case-insensitive match
   - If no track name: Find first track NOT marked `[x]`

4. **Check Dependencies:**
   - Read `conductor/tracks/<track_id>/metadata.json`
   - If `depends_on` array not empty, verify all are `[x]` completed

5. **Verify Beads Integration:**
   - Read `conductor/beads.json` - must have `"enabled": true`
   - Track must have `beads_epic` in metadata.json
   - If missing: "Parallel mode requires Beads epic. Run `/conductor-newtrack` to create proper integration."

---

## 3.0 PARALLEL PLANNING

**PROTOCOL: Analyze task dependencies and compute parallel execution groups.**

1. **Load Track Context:**
   - Read `conductor/tracks/<track_id>/plan.md`
   - Read `conductor/tracks/<track_id>/spec.md`
   - Read `conductor/tracks/<track_id>/metadata.json`
   - Extract `beads_epic`, `beads_tasks`, and `parallel_analysis`

2. **Determine Parallel Groups:**

   **Option A: Use `parallel_analysis` from metadata.json (if available)**
   - If `metadata.json` contains `parallel_analysis.enabled: true`:
     - Use pre-computed `can_parallel_with` and `depends_on_tasks` 
     - Use `estimated_files` for file lease planning
     - This was captured during `/conductor-newtrack`
   
   **Option B: Query Beads Viewer for dynamic analysis**
   - Run: `bv --robot-plan --json`
   - This analyzes the current Beads dependency graph
   - Returns tasks grouped by independent tracks
   
   **Option C: Combine both (recommended)**
   - Use `parallel_analysis` for file affinity estimates
   - Use `bv --robot-plan` for dependency verification
   - Cross-check: If discrepancies, prefer Beads (more up-to-date)

3. **If No Parallel Analysis Exists:**
   > "⚠️ No parallel analysis found for this track."
   > "Options:"
   > "A) Analyze now - I'll infer task dependencies and file affinity"
   > "B) Use Beads only - Rely on `bv --robot-plan` for parallel groups"
   > "C) Run sequential - Use `/conductor-implement` instead"
   
   If A: Run inline analysis (similar to newtrack 2.3a)

4. **Compute File Affinity:**
   - For each task, analyze spec.md and plan.md to estimate affected files
   - Group tasks by likely file overlap to minimize lease conflicts
   - Tasks touching same files should NOT run in parallel

5. **Build Execution Plan:**
   ```json
   {
     "parallel_groups": [
       {
         "group_id": "batch-1",
         "tasks": [
           {"plan_key": "phase1_task1", "beads_id": "bd-123", "estimated_files": ["src/auth/login.ts"]},
           {"plan_key": "phase1_task2", "beads_id": "bd-124", "estimated_files": ["src/api/users.ts"]}
         ]
       }
     ],
     "sequential_after": ["phase2_task1"],
     "highest_impact_first": "phase1_task1"
   }
   ```

6. **Display Plan:**
   > "📊 **Parallel Execution Plan:**"
   > "- **Batch 1:** 3 tasks can run simultaneously"
   > "  - Task 1.1: Implement login endpoint (bd-123)"
   > "  - Task 1.2: Create user model (bd-124)"
   > "  - Task 1.3: Add validation utils (bd-125)"
   > "- **Highest Impact:** Task 1.1 (unblocks 5 downstream tasks)"
   > "- **Sequential after:** Phase 2 tasks (depend on Batch 1)"

7. **Dry Run Check:**
   - If `--dry-run` flag present: Display plan and HALT
   - Otherwise: Ask "Proceed with parallel execution? (yes/no)"

---

## 4.0 ORCHESTRATOR INITIALIZATION

**PROTOCOL: Set up orchestrator state and MCP Agent Mail integration.**

1. **Create Parallel State File:**
   Create `conductor/tracks/<track_id>/implement_parallel_state.json`:
   ```json
   {
     "mode": "parallel",
     "started_at": "<timestamp>",
     "max_agents": 3,
     "current_batch": 0,
     "agents": {},
     "tasks": {},
     "integration_branch": "track/<track_id>/integration",
     "worktrees": []
   }
   ```

2. **Register as Orchestrator:**
   Use MCP Agent Mail tool:
   ```
   register_agent(
     project_key: "<absolute_project_path>",
     program: "amp",
     model: "<current_model>",
     name: "Orchestrator"
   )
   ```

3. **Create Integration Branch:**
   ```bash
   git checkout -b track/<track_id>/integration
   ```

4. **Update Track Status:**
   - In `conductor/tracks.md`, change `## [ ] Track:` to `## [~] Track:`

5. **Initialize Beads Epic:**
   ```bash
   bd update <beads_epic> --status in_progress --notes "PARALLEL MODE STARTED: <timestamp>
   BATCH 1: <task_count> tasks
   ORCHESTRATOR: Initialized"
   ```

---

## 5.0 PARALLEL EXECUTION LOOP

**PROTOCOL: Spawn and coordinate sub-agents for parallel task execution.**

### 5.1 For Each Parallel Batch:

1. **Select Ready Tasks:**
   - From current batch, select tasks up to `max_agents`
   - Verify no file overlap between selected tasks

2. **For Each Task in Selection:**

   a. **Request File Leases:**
      Use MCP Agent Mail:
      ```
      file_reservation_paths(
        project_key: "<project_path>",
        agent_name: "<agent_name>",  // Will be assigned
        paths: ["src/auth/login.ts", "src/auth/login.test.ts"],
        ttl_seconds: 3600,
        exclusive: true,
        reason: "Task bd-123: Implement login endpoint"
      )
      ```
      
      - **If conflict:** Skip task for this batch, try next available

   b. **Create Git Worktree:**
      ```bash
      git worktree add worktrees/<track_id>/<task_key> -b track/<track_id>/<task_key>
      ```
      
      - Record worktree path in parallel state

   c. **Spawn Sub-Agent:**
      Use the **Task** tool with comprehensive prompt:
      
      ```
      Task(
        description: "Execute task: <task_title>",
        prompt: """
        You are a Task Agent for Conductor parallel execution.
        
        ## Your Identity
        Agent Name: <assigned_name>  (e.g., GreenCastle)
        Task ID: <beads_id>
        
        ## Your Scope
        - You are working in worktree: worktrees/<track_id>/<task_key>
        - You may ONLY modify files you have leases for: <file_list>
        - You must NOT modify: plan.md, metadata.json, any Beads data
        
        ## Your Task
        <task_description from plan.md>
        
        ## Spec Context
        <relevant section from spec.md>
        
        ## TDD Workflow (from workflow.md)
        1. Write failing tests first (Red)
        2. Implement to pass tests (Green)
        3. Refactor
        4. Verify >80% coverage
        5. Commit with message: "<type>(<scope>): <description> [Task-Id: <beads_id>]"
        
        ## Communication Protocol
        You MUST report back with exactly this JSON structure:
        {
          "status": "complete" | "blocked" | "failed",
          "commit_sha": "<sha if complete>",
          "files_modified": ["list", "of", "files"],
          "tests_passed": true | false,
          "coverage_percent": <number>,
          "notes": "<summary of work done>",
          "blockers": "<reason if blocked>"
        }
        
        ## Constraints
        - Do NOT run `bd` commands - the Orchestrator handles Beads
        - Do NOT edit plan.md - the Orchestrator updates it
        - Stay within your leased files
        - Commit to your worktree branch only
        """
      )
      ```

   d. **Update State:**
      - Add agent to `agents` in parallel state
      - Set task status to `in_progress`

### 5.2 Monitor Agents:

1. **Wait for Agent Completion:**
   - Task tool returns when sub-agent finishes
   - Parse the returned JSON status

2. **Handle Agent Results:**

   **If status = "complete":**
   - Verify commit exists in worktree branch
   - Merge into integration branch:
     ```bash
     git checkout track/<track_id>/integration
     git merge track/<track_id>/<task_key> --no-ff -m "Merge: <task_title> [Task-Id: <beads_id>]"
     ```
   - Run integration tests
   - Update plan.md: Mark task `[x]` with commit SHA
   - Update Beads:
     ```bash
     bd update <beads_id> --notes "COMPLETED: commit <sha> | Parallel agent: <agent_name>"
     bd close <beads_id> --reason "Task completed in parallel mode"
     ```
   - Release file leases
   - Remove worktree:
     ```bash
     git worktree remove worktrees/<track_id>/<task_key>
     git branch -d track/<track_id>/<task_key>
     ```

   **If status = "blocked":**
   - Mark task `[!]` in plan.md with blocker reason
   - Update Beads: `bd update <beads_id> --status blocked --notes "<blocker>"`
   - Release leases, remove worktree
   - Log to `conductor/tracks/<track_id>/blockers.md`
   
   **If status = "failed":**
   - Attempt retry (max 2 retries)
   - If still failing: Mark blocked, escalate to user
   - Rollback worktree: `git worktree remove --force`

3. **Update Parallel State:**
   - Remove agent from active list
   - Update task status
   - Check if batch complete

### 5.3 Batch Completion:

1. **When All Tasks in Batch Complete:**
   - Run full test suite on integration branch
   - If tests fail: Halt, report failures, request user intervention
   
2. **Phase Checkpoint (if applicable):**
   - If batch completes a phase:
     ```bash
     git commit --allow-empty -m "conductor(checkpoint): Phase <N> complete [parallel]"
     bd update <beads_epic> --notes "PHASE <N> COMPLETE: <tasks_completed> tasks
     INTEGRATION TESTS: Passed
     NEXT: Phase <N+1>"
     ```
   - Update plan.md with checkpoint SHA

3. **Advance to Next Batch:**
   - Increment `current_batch` in parallel state
   - Re-query `bv --robot-plan` for newly unblocked tasks
   - Continue loop

---

## 6.0 CONFLICT RESOLUTION

**PROTOCOL: Handle merge conflicts and file contention.**

### 6.1 Proactive (Lease-based Prevention):

- Before spawning agent, verify lease can be obtained
- If file already leased: Queue task for next batch
- Display: "Task <X> queued - file conflict with <agent_name>"

### 6.2 Reactive (Merge Conflicts):

1. **Detect Conflict:**
   - Merge command returns non-zero exit
   - Git reports conflict markers

2. **Create Resolution Task:**
   ```bash
   bd create "Resolve merge conflict: <task_key>" -p 1 --type task
   ```

3. **Options:**
   > "⚠️ Merge conflict detected for <task>:"
   > "A) Assign resolution to original agent"
   > "B) I'll resolve manually"
   > "C) Abort this task, continue with others"

4. **If A:** Spawn conflict resolution sub-agent with both branches' context

---

## 7.0 ERROR HANDLING

**PROTOCOL: Handle failures gracefully with recovery options.**

### Error Categories:

| Category | Indicators | Action |
|----------|------------|--------|
| **MCP Agent Mail Down** | Connection refused | Fallback to sequential mode |
| **Beads Command Fail** | Non-zero exit | Offer: continue/retry/halt |
| **Agent Stall** | No response in 10 min | Rollback worktree, reassign |
| **Lease Violation** | Pre-commit rejects | Agent must request lease |
| **Git Worktree Fail** | Branch exists | Clean up, retry |

### Recovery:

1. **State Persistence:**
   - All progress saved to `implement_parallel_state.json`
   - Can resume from any point with `/conductor-implement-parallel --resume`

2. **Rollback:**
   - If catastrophic failure: Offer full rollback to pre-parallel state
   - Clean up all worktrees and temporary branches

---

## 8.0 FINALIZATION

**PROTOCOL: Complete parallel execution and clean up.**

1. **All Tasks Complete:**
   - Merge integration branch to main working branch
   - Run final test suite
   - Update `conductor/tracks.md`: `## [~]` → `## [x]`

2. **Clean Up:**
   - Remove all worktrees
   - Delete temporary branches
   - Delete `implement_parallel_state.json`
   - Release all leases

3. **Update Beads Epic:**
   ```bash
   bd update <beads_epic> --notes "TRACK COMPLETE (PARALLEL MODE)
   Total tasks: <count>
   Parallel batches: <batch_count>
   Time saved: ~<estimate>"
   bd close <beads_epic> --reason "Track completed in parallel mode"
   ```

4. **Summary Report:**
   > "🎉 **Parallel Execution Complete!**"
   > "- Tasks completed: X"
   > "- Parallel batches: Y"
   > "- Agents used: <names>"
   > "- Merge conflicts: Z (resolved)"
   > "- Final commit: <sha>"

5. **Continue to Documentation Sync:**
   - Proceed to Section 6.0 of standard `/conductor-implement` for doc sync
   - Then Section 7.0 for cleanup options

---

## Status Markers Reference

- `[ ]` - Pending
- `[~]` - In Progress
- `[x]` - Completed
- `[!]` - Blocked
- `[||]` - Running in parallel (visual indicator in progress display)

---

## Appendix: Resume Protocol

If `implement_parallel_state.json` exists when command starts:

1. **Load State:**
   - Read current batch, active agents, completed tasks

2. **Check Agent Status:**
   - For any agents marked "active", check if worktree has new commits
   - If worktree missing: Mark task for retry

3. **Resume Options:**
   > "🔄 **Parallel execution in progress detected:**"
   > "- Batch: <N>/<total>"
   > "- Completed: <X> tasks"
   > "- In progress: <Y> tasks"
   > "A) Resume from current state"
   > "B) Abort and restart"
   > "C) Abort and switch to sequential mode"

4. **If Resume:** Continue from saved state
