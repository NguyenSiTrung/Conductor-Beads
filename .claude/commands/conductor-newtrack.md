---
description: Create a new feature or bug track with spec and plan
argument-hint: [description]
---

<!-- 
SYSTEM DIRECTIVE: You are an AI agent for the Conductor framework.
CRITICAL: Validate every tool call. If any fails, halt and announce the failure.
-->

# Conductor New Track

Create a new track for: $ARGUMENTS

---

## 1.0 SETUP CHECK

**PROTOCOL: Verify Conductor environment is properly set up.**

1. **Check Required Files:** Verify existence of:
   - `conductor/product.md`
   - `conductor/tech-stack.md`
   - `conductor/workflow.md`

2. **Handle Missing Files:**
   - If ANY missing: HALT immediately
   - Announce: "Conductor is not set up. Please run `/conductor-setup` first."
   - Do NOT proceed.

---

## 2.0 NEW TRACK INITIALIZATION

### 2.1 Get Track Description and Determine Type

1. **Load Project Context:** Read `conductor/` directory files.

2. **Get Track Description:**
   - **If `$ARGUMENTS` provided:** Use it as track description
   - **If empty:** Ask:
     > "Please provide a brief description of the track (feature, bug fix, chore, etc.) you wish to start."
     Wait for response.

3. **Infer Track Type:** Analyze description to classify as "Feature" or "Something Else" (Bug, Chore, Refactor). Do NOT ask user to classify.

---

### 2.2 Interactive Specification Generation (`spec.md`)

1. **Announce:**
   > "I'll now guide you through questions to build a comprehensive `spec.md` for this track."

2. **Questioning Phase (3-5 questions):**

   **Question Classification - CRITICAL:**
   - **1. Classify Question Type:** Before EACH question, classify as:
     - **Additive:** For brainstorming/scope (users, goals, features) - allows multiple answers
     - **Exclusive Choice:** For singular commitments (specific technology, workflow rule) - single answer

   - **2. Formulate Based on Classification:**
     - **If Additive:** Open-ended question + options + "(Select all that apply)"
     - **If Exclusive Choice:** Direct question, do NOT add multi-select

   - **3. Interaction Flow:**
     - **CRITICAL:** Ask ONE question at a time. Wait for response before next question.
     - Last option for every question MUST be "Type your own answer"
     - Confirm understanding by summarizing before moving on

   **If FEATURE (3-5 questions):**
   - Clarifying questions about the feature
   - Implementation approach, interactions, inputs/outputs
   - UI/UX considerations, data involved

   **If SOMETHING ELSE - Bug, Chore, etc. (2-3 questions):**
   - Reproduction steps for bugs
   - Specific scope for chores
   - Success criteria

3. **Draft `spec.md`:** Generate with sections:
   - Overview
   - Functional Requirements
   - Non-Functional Requirements (if any)
   - Acceptance Criteria
   - Out of Scope

4. **User Confirmation:**
   > "I've drafted the specification. Please review:"
   > ```markdown
   > [Drafted spec.md content]
   > ```
   > "Does this accurately capture the requirements? Suggest changes or confirm."

   Revise until confirmed.

---

### 2.3 Interactive Plan Generation (`plan.md`)

1. **Announce:**
   > "Now I will create an implementation plan (`plan.md`) based on the specification."

2. **Generate Plan:**
   - Read confirmed `spec.md` content
   - Read `conductor/workflow.md`
   - Generate hierarchical Phases, Tasks, Sub-tasks
   - **CRITICAL:** Plan structure MUST adhere to workflow methodology (e.g., TDD tasks)
   - Include `[ ]` status markers for each task/sub-task

   **CRITICAL: Inject Phase Completion Tasks**
   - Check if `conductor/workflow.md` defines "Phase Completion Verification and Checkpointing Protocol"
   - If YES, for EACH Phase, append final meta-task:
     ```
     - [ ] Task: Conductor - User Manual Verification '<Phase Name>' (Protocol in workflow.md)
     ```
   - Replace `<Phase Name>` with actual phase name

3. **User Confirmation:**
   > "I've drafted the implementation plan. Please review:"
   > ```markdown
   > [Drafted plan.md content]
   > ```
   > "Does this cover all necessary steps based on spec and workflow? Suggest changes or confirm."

   Revise until confirmed.

---

### 2.3a Parallel Execution Analysis (For `implement-parallel` support)

**PROTOCOL: Auto-analyze task dependencies and file affinity for parallel execution.**

1. **Auto-Analyze (Default):**
   
   The agent AUTOMATICALLY analyzes the plan to determine parallelization:

   a. **Infer File Affinity:**
      - For each task, analyze description + spec context
      - Use naming patterns to estimate files:
        - "Implement login endpoint" → `["src/auth/login.ts", "src/auth/login.test.ts"]`
        - "Create user model" → `["src/models/user.ts", "src/models/user.test.ts"]`
        - "Add validation utils" → `["src/utils/validation.ts"]`
      
   b. **Infer Dependencies (Two Levels):**
      
      **Intra-Phase (within same phase):**
      - **Default Rule:** Tasks in same phase are INDEPENDENT (can parallelize)
      - **Auto-detect sequential patterns:**
        - "Create X" before "Use X" → dependency
        - Shared file references → potential conflict
      
      **Cross-Phase (between phases):**
      - **Default Rule:** Each phase task depends on ALL previous phase tasks
      - **Optimized Rule (recommended):** Detect SPECIFIC cross-phase dependencies
        - If Phase 2 Task 1 only uses output from Phase 1 Task 1:
          - 2.1 depends on 1.1 only (not 1.2, 1.3)
          - 2.1 can START while 1.2 and 1.3 are still running!
      
   c. **Compute `can_parallel_with`:**
      - Tasks with no dependency (intra OR cross-phase) AND no file overlap → can parallelize
      - This can include tasks from DIFFERENT phases!

2. **Display Analysis for Confirmation:**
   > "📊 **Parallel Execution Analysis (auto-detected):**"
   > 
   > **Parallelization Model:** Cross-phase enabled
   > 
   > | Task | Phase | Depends On | Can Parallel With |
   > |------|-------|------------|-------------------|
   > | 1.1 Create user model | 1 | — | 1.2, 1.3 |
   > | 1.2 Implement login | 1 | — | 1.1, 1.3, 2.2 |
   > | 1.3 Add validation | 1 | — | 1.1, 1.2 |
   > | 2.1 Add auth middleware | 2 | 1.1 | 2.2 |
   > | 2.2 Create API routes | 2 | 1.2 | 1.3, 2.1 |
   > 
   > **Execution Timeline (3 agents):**
   > ```
   > Agent 1: [1.1] ──────> [2.1] ────>
   > Agent 2: [1.2] ──────> [2.2] ────>
   > Agent 3: [1.3] ──────>
   > ```
   > 
   > "Does this look correct? (yes / or describe corrections)"

3. **Handle User Corrections:**
   - If user says "yes" or skips: Save as-is
   - If user provides corrections (e.g., "2.1 actually needs both 1.1 and 1.2"):
     - Update `depends_on_tasks` for affected tasks
     - Recalculate `can_parallel_with`

4. **Store in Metadata:**
   Add to `conductor/tracks/<track_id>/metadata.json`:
   ```json
   {
     "parallel_analysis": {
       "enabled": true,
       "analyzed_at": "<timestamp>",
       "method": "auto",
       "cross_phase": true,
       "tasks": {
         "phase1_task1": {
           "estimated_files": ["src/models/user.ts"],
           "depends_on_tasks": [],
           "can_parallel_with": ["phase1_task2", "phase1_task3"]
         },
         "phase1_task2": {
           "estimated_files": ["src/auth/login.ts"],
           "depends_on_tasks": [],
           "can_parallel_with": ["phase1_task1", "phase1_task3", "phase2_task2"]
         },
         "phase2_task1": {
           "estimated_files": ["src/middleware/auth.ts"],
           "depends_on_tasks": ["phase1_task1"],
           "can_parallel_with": ["phase1_task2", "phase1_task3", "phase2_task2"]
         },
         "phase2_task2": {
           "estimated_files": ["src/routes/api.ts"],
           "depends_on_tasks": ["phase1_task2"],
           "can_parallel_with": ["phase1_task1", "phase1_task3", "phase2_task1"]
         }
       }
     }
   }
   ```

5. **Parallelization Modes:**
   User can choose:
   - **Conservative:** Phase-by-phase (Phase 2 waits for ALL of Phase 1)
   - **Optimized (default):** Cross-phase with specific dependencies
   - **Skip:** No parallel analysis

---

### 2.4 Create Track Artifacts and Update Main Plan

1. **Check for Duplicate Track Name:**
   - List existing directories in `conductor/tracks/`
   - Extract short names from track IDs (`shortname_YYYYMMDD` → `shortname`)
   - If proposed short name matches existing:
     - **HALT** creation
     - Explain track with that name exists
     - Suggest different name or resuming existing track

2. **Generate Track ID:** Create unique ID: `shortname_YYYYMMDD`

3. **Ask for Priority:**
   > "What priority should this track have?"
   > A) 🔴 Critical - Blocking other work
   > B) 🟠 High - Important, do soon
   > C) 🟡 Medium - Normal priority (default)
   > D) 🟢 Low - Nice to have

   Default to "medium" if skipped.

4. **Ask for Dependencies (Optional):**
   > "Does this track depend on any other tracks being completed first?"
   - If yes: List incomplete tracks from `conductor/tracks.md`, let user select
   - Store selected track_ids in `depends_on` array
   - Default to empty array if skipped or no incomplete tracks

5. **Ask for Time Estimate (Optional):**
   > "Estimated hours to complete? (Enter number or skip)"
   - Store in `estimated_hours` or null if skipped

6. **Create Directory:** `conductor/tracks/<track_id>/`

7. **Create `metadata.json`:**
   ```json
   {
     "track_id": "<track_id>",
     "type": "feature",
     "status": "new",
     "priority": "medium",
     "depends_on": [],
     "estimated_hours": null,
     "created_at": "YYYY-MM-DDTHH:MM:SSZ",
     "updated_at": "YYYY-MM-DDTHH:MM:SSZ",
     "description": "<Initial user description>"
   }
   ```
   Populate with actual values from steps 3-5.

8. **Write Files:**
   - `conductor/tracks/<track_id>/spec.md`
   - `conductor/tracks/<track_id>/plan.md`

9. **Update Tracks File:**
   - Announce: "Updating the tracks file."
   - Append to `conductor/tracks.md`:
     ```markdown

     ---

     ## [ ] Track: <Track Description>
     *Link: [./conductor/tracks/<track_id>/](./conductor/tracks/<track_id>/)*
     ```

10. **Announce Completion:**
    > "New track '<track_id>' has been created and added to the tracks file. Run `/conductor-implement` to start."

---

### 2.5 BEADS INTEGRATION

**PROTOCOL: Sync track with Beads for persistent task memory.**

1. **Check Beads Availability:**
   - Check if `bd` command exists: `which bd`
   - If command not found:
     > "⚠️ Beads CLI (`bd`) is not installed. Beads provides persistent task memory across sessions."
     > "A) Continue without Beads integration"
     > "B) Stop - I'll install Beads first (see: https://github.com/elimisteve/beads)"
     - If user chooses A: Skip remaining Beads steps, continue to completion
     - If user chooses B: HALT and wait for user to install

2. **Create Epic for Track with Full Context:**
   - Map priority: critical=0, high=1, medium=2, low=3
   - Extract technical approach from `spec.md` for design field
   - Extract acceptance criteria from `spec.md`
   - Run:
     ```bash
     bd create "<track_id>: <description>" \
       -t epic -p <priority_number> \
       --design "<technical approach from spec>" \
       --acceptance "<completion criteria from spec>" \
       --assignee conductor \
       --json
     ```
   - Store returned epic ID (e.g., `bd-a3f8`)

3. **Create Tasks for Each Phase with Context:**
   - Parse `plan.md` for phases and tasks
   - For each phase:
     ```bash
     bd create "<phase_name>" --parent <epic_id> --json
     ```
   - For each task in phase:
     ```bash
     bd create "<task_description>" \
       --parent <phase_id> \
       --design "<task technical notes>" \
       --acceptance "<task done criteria>" \
       --json
     ```

4. **Set Up Dependencies:**
   
   a. **Phase Dependencies (sequential by default):**
      - Phase 2 blocked by Phase 1: `bd dep add <phase2_id> <phase1_id>`
      - Continue for all phases
   
   b. **Intra-Phase Task Dependencies (from parallel analysis):**
      - If `parallel_analysis` exists in metadata.json:
        - For each task with `depends_on_tasks` not empty:
          ```bash
          # If phase1_task3 depends on phase1_task1:
          bd dep add <phase1_task3_beads_id> <phase1_task1_beads_id>
          ```
      - This enables `bv --robot-plan` to correctly identify parallel groups

5. **Update Metadata:**
   - Add to `conductor/tracks/<track_id>/metadata.json`:
     ```json
     {
       "beads_epic": "bd-a3f8",
       "beads_tasks": {
         "phase1": "bd-a3f8.1",
         "phase1_task1": "bd-a3f8.1.1",
         "phase1_task2": "bd-a3f8.1.2",
         "phase2": "bd-a3f8.2",
         "phase2_task1": "bd-a3f8.2.1",
         "phase2_task2": "bd-a3f8.2.2"
       }
     }
     ```
   - **Key naming convention:**
     - Phase keys: `phase{N}` (1-indexed, e.g., `phase1`, `phase2`)
     - Task keys: `phase{N}_task{M}` (both 1-indexed, e.g., `phase1_task1`, `phase2_task3`)
   - Store ALL phase and task IDs returned from `bd create --json` commands

7. **Announce:** "Track synced to Beads as epic <epic_id>."

**ERROR HANDLING:** If any `bd` command fails during steps 2-6:
- Announce the specific error
- Ask user:
  > "⚠️ Beads command failed: <error message>"
  > "A) Continue without Beads integration - track files are already created"
  > "B) Retry the failed command"
  > "C) Stop - I'll fix the issue first"
- If A: Skip remaining Beads steps, announce track created without Beads sync
- If B: Retry the failed command
- If C: HALT and wait for user
