# Project Workflow

## Guiding Principles

1. **The Plan is the Source of Truth:** All work must be tracked in `plan.md`
2. **The Tech Stack is Deliberate:** Changes to the tech stack must be documented in `tech-stack.md` *before* implementation
3. **Test-Driven Development:** Write unit tests before implementing functionality
4. **High Code Coverage:** Aim for >80% code coverage for all modules
5. **User Experience First:** Every decision should prioritize user experience
6. **Non-Interactive & CI-Aware:** Prefer non-interactive commands. Use `CI=true` for watch-mode tools (tests, linters) to ensure single execution.

## Status Markers

- `[ ]` - Pending/New
- `[~]` - In Progress  
- `[x]` - Completed
- `[!]` - Blocked (with reason)

### Blocker Format
When a task is blocked, use: `- [!] Task name [BLOCKED: reason]`

Example:
```markdown
- [!] Integrate payment API [BLOCKED: Waiting for API credentials from vendor]
```

## Task Workflow

All tasks follow a strict lifecycle:

### Standard Task Workflow

1. **Select Task:** Choose the next available task from `plan.md` in sequential order

2. **Mark In Progress:** Before beginning work, edit `plan.md` and change the task from `[ ]` to `[~]`

3. **Write Failing Tests (Red Phase):**
   - Create a new test file for the feature or bug fix.
   - Write one or more unit tests that clearly define the expected behavior and acceptance criteria for the task.
   - **CRITICAL:** Run the tests and confirm that they fail as expected. This is the "Red" phase of TDD. Do not proceed until you have failing tests.

4. **Implement to Pass Tests (Green Phase):**
   - Write the minimum amount of application code necessary to make the failing tests pass.
   - Run the test suite again and confirm that all tests now pass. This is the "Green" phase.

5. **Refactor (Optional but Recommended):**
   - With the safety of passing tests, refactor the implementation code and the test code to improve clarity, remove duplication, and enhance performance without changing the external behavior.
   - Rerun tests to ensure they still pass after refactoring.

6. **Verify Coverage:** Run coverage reports using the project's chosen tools. For example, in a Python project, this might look like:
   ```bash
   pytest --cov=app --cov-report=html
   ```
   Target: >80% coverage for new code. The specific tools and commands will vary by language and framework.

7. **Document Deviations:** If implementation differs from tech stack:
   - **STOP** implementation
   - Update `tech-stack.md` with new design
   - Add dated note explaining the change
   - Resume implementation

8. **Commit Code Changes:**
   - Stage all code changes related to the task.
   - Propose a clear, concise commit message e.g, `feat(ui): Create basic HTML structure for calculator`.
   - Perform the commit.

9. **Attach Task Summary with Git Notes:**
   - **Step 9.1: Get Commit Hash:** Obtain the hash of the *just-completed commit* (`git log -1 --format="%H"`).
   - **Step 9.2: Draft Note Content:** Create a detailed summary for the completed task. This should include the task name, a summary of changes, a list of all created/modified files, and the core "why" for the change.
   - **Step 9.3: Attach Note:** Use the `git notes` command to attach the summary to the commit.
     ```bash
     # The note content from the previous step is passed via the -m flag.
     git notes add -m "<note content>" <commit_hash>
     ```

10. **Get and Record Task Commit SHA:**
    - **Step 10.1: Update Plan:** Read `plan.md`, find the line for the completed task, update its status from `[~]` to `[x]`, and append the first 7 characters of the *just-completed commit's* commit hash.
    - **Step 10.2: Write Plan:** Write the updated content back to `plan.md`.

11. **Commit Plan Update:**
    - **Action:** Stage the modified `plan.md` file.
    - **Action:** Commit this change with a descriptive message (e.g., `conductor(plan): Mark task 'Create user model' as complete`).

### Phase Completion Verification and Checkpointing Protocol

**Trigger:** This protocol is executed immediately after a task is completed that also concludes a phase in `plan.md`.

1.  **Announce Protocol Start:** Inform the user that the phase is complete and the verification and checkpointing protocol has begun.

2.  **Ensure Test Coverage for Phase Changes:**
    -   **Step 2.1: Determine Phase Scope:** To identify the files changed in this phase, you must first find the starting point. Read `plan.md` to find the Git commit SHA of the *previous* phase's checkpoint. If no previous checkpoint exists, the scope is all changes since the first commit.
    -   **Step 2.2: List Changed Files:** Execute `git diff --name-only <previous_checkpoint_sha> HEAD` to get a precise list of all files modified during this phase.
    -   **Step 2.3: Verify and Create Tests:** For each file in the list:
        -   **CRITICAL:** First, check its extension. Exclude non-code files (e.g., `.json`, `.md`, `.yaml`).
        -   For each remaining code file, verify a corresponding test file exists.
        -   If a test file is missing, you **must** create one. Before writing the test, **first, analyze other test files in the repository to determine the correct naming convention and testing style.** The new tests **must** validate the functionality described in this phase's tasks (`plan.md`).

3.  **Execute Automated Tests with Proactive Debugging:**
    -   Before execution, you **must** announce the exact shell command you will use to run the tests.
    -   **Example Announcement:** "I will now run the automated test suite to verify the phase. **Command:** `CI=true npm test`"
    -   Execute the announced command.
    -   If tests fail, you **must** inform the user and begin debugging. You may attempt to propose a fix a **maximum of two times**. If the tests still fail after your second proposed fix, you **must stop**, report the persistent failure, and ask the user for guidance.

4.  **Propose a Detailed, Actionable Manual Verification Plan:**
    -   **CRITICAL:** To generate the plan, first analyze `product.md`, `product-guidelines.md`, and `plan.md` to determine the user-facing goals of the completed phase.
    -   You **must** generate a step-by-step plan that walks the user through the verification process, including any necessary commands and specific, expected outcomes.
    -   The plan you present to the user **must** follow this format:

        **For a Frontend Change:**
        ```
        The automated tests have passed. For manual verification, please follow these steps:

        **Manual Verification Steps:**
        1.  **Start the development server with the command:** `npm run dev`
        2.  **Open your browser to:** `http://localhost:3000`
        3.  **Confirm that you see:** The new user profile page, with the user's name and email displayed correctly.
        ```

        **For a Backend Change:**
        ```
        The automated tests have passed. For manual verification, please follow these steps:

        **Manual Verification Steps:**
        1.  **Ensure the server is running.**
        2.  **Execute the following command in your terminal:** `curl -X POST http://localhost:8080/api/v1/users -d '{"name": "test"}'`
        3.  **Confirm that you receive:** A JSON response with a status of `201 Created`.
        ```

5.  **Await Explicit User Feedback:**
    -   After presenting the detailed plan, ask the user for confirmation: "**Does this meet your expectations? Please confirm with yes or provide feedback on what needs to be changed.**"
    -   **PAUSE** and await the user's response. Do not proceed without an explicit yes or confirmation.

6.  **Create Checkpoint Commit:**
    -   Stage all changes. If no changes occurred in this step, proceed with an empty commit.
    -   Perform the commit with a clear and concise message (e.g., `conductor(checkpoint): Checkpoint end of Phase X`).

7.  **Attach Auditable Verification Report using Git Notes:**
    -   **Step 8.1: Draft Note Content:** Create a detailed verification report including the automated test command, the manual verification steps, and the user's confirmation.
    -   **Step 8.2: Attach Note:** Use the `git notes` command and the full commit hash from the previous step to attach the full report to the checkpoint commit.

8.  **Get and Record Phase Checkpoint SHA:**
    -   **Step 7.1: Get Commit Hash:** Obtain the hash of the *just-created checkpoint commit* (`git log -1 --format="%H"`).
    -   **Step 7.2: Update Plan:** Read `plan.md`, find the heading for the completed phase, and append the first 7 characters of the commit hash in the format `[checkpoint: <sha>]`.
    -   **Step 7.3: Write Plan:** Write the updated content back to `plan.md`.

9. **Commit Plan Update:**
    - **Action:** Stage the modified `plan.md` file.
    - **Action:** Commit this change with a descriptive message following the format `conductor(plan): Mark phase '<PHASE NAME>' as complete`.

10.  **Announce Completion:** Inform the user that the phase is complete and the checkpoint has been created, with the detailed verification report attached as a git note.

### Quality Gates

Before marking any task complete, verify:

- [ ] All tests pass
- [ ] Code coverage meets requirements (>80%)
- [ ] Code follows project's code style guidelines (as defined in `code_styleguides/`)
- [ ] All public functions/methods are documented (e.g., docstrings, JSDoc, GoDoc)
- [ ] Type safety is enforced (e.g., type hints, TypeScript types, Go types)
- [ ] No linting or static analysis errors (using the project's configured tools)
- [ ] Works correctly on mobile (if applicable)
- [ ] Documentation updated if needed
- [ ] No security vulnerabilities introduced

## Development Commands

**AI AGENT INSTRUCTION: This section should be adapted to the project's specific language, framework, and build tools.**

### Setup
```bash
# Example: Commands to set up the development environment (e.g., install dependencies, configure database)
# e.g., for a Node.js project: npm install
# e.g., for a Go project: go mod tidy
```

### Daily Development
```bash
# Example: Commands for common daily tasks (e.g., start dev server, run tests, lint, format)
# e.g., for a Node.js project: npm run dev, npm test, npm run lint
# e.g., for a Go project: go run main.go, go test ./..., go fmt ./...
```

### Before Committing
```bash
# Example: Commands to run all pre-commit checks (e.g., format, lint, type check, run tests)
# e.g., for a Node.js project: npm run check
# e.g., for a Go project: make check (if a Makefile exists)
```

## Testing Requirements

### Unit Testing
- Every module must have corresponding tests.
- Use appropriate test setup/teardown mechanisms (e.g., fixtures, beforeEach/afterEach).
- Mock external dependencies.
- Test both success and failure cases.

### Integration Testing
- Test complete user flows
- Verify database transactions
- Test authentication and authorization
- Check form submissions

### Mobile Testing
- Test on actual iPhone when possible
- Use Safari developer tools
- Test touch interactions
- Verify responsive layouts
- Check performance on 3G/4G

## Code Review Process

### Self-Review Checklist
Before requesting review:

1. **Functionality**
   - Feature works as specified
   - Edge cases handled
   - Error messages are user-friendly

2. **Code Quality**
   - Follows style guide
   - DRY principle applied
   - Clear variable/function names
   - Appropriate comments

3. **Testing**
   - Unit tests comprehensive
   - Integration tests pass
   - Coverage adequate (>80%)

4. **Security**
   - No hardcoded secrets
   - Input validation present
   - SQL injection prevented
   - XSS protection in place

5. **Performance**
   - Database queries optimized
   - Images optimized
   - Caching implemented where needed

6. **Mobile Experience**
   - Touch targets adequate (44x44px)
   - Text readable without zooming
   - Performance acceptable on mobile
   - Interactions feel native

## Commit Guidelines

### Message Format
```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

### Types
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation only
- `style`: Formatting, missing semicolons, etc.
- `refactor`: Code change that neither fixes a bug nor adds a feature
- `test`: Adding missing tests
- `chore`: Maintenance tasks

### Examples
```bash
git commit -m "feat(auth): Add remember me functionality"
git commit -m "fix(posts): Correct excerpt generation for short posts"
git commit -m "test(comments): Add tests for emoji reaction limits"
git commit -m "style(mobile): Improve button touch targets"
```

## Definition of Done

A task is complete when:

1. All code implemented to specification
2. Unit tests written and passing
3. Code coverage meets project requirements
4. Documentation complete (if applicable)
5. Code passes all configured linting and static analysis checks
6. Works beautifully on mobile (if applicable)
7. Implementation notes added to `plan.md`
8. Changes committed with proper message
9. Git note with task summary attached to the commit

## Parallel Task Workflow (Orchestrated Mode)

When using `/conductor-implement-parallel`, the workflow adapts for multi-agent execution:

### Prerequisites

| Tool | Purpose | Installation |
|------|---------|--------------|
| `bd` (Beads CLI) | Task tracking, dependencies | `go install github.com/steveyegge/beads/cmd/bd@latest` |
| `bv` (Beads Viewer) | Parallel track planning | See [beads_viewer](https://github.com/Dicklesworthstone/beads_viewer) |
| MCP Agent Mail | Agent coordination, file leases | `curl -fsSL https://raw.githubusercontent.com/Dicklesworthstone/mcp_agent_mail/main/install.sh \| bash` |

### Orchestrator Responsibilities

The main agent (Orchestrator) handles all coordination:

- **Task Selection**: Uses `bv --robot-plan` to identify independent parallel tracks
- **File Lease Management**: Requests exclusive leases via MCP Agent Mail before spawning agents
- **Plan Updates**: Only the Orchestrator modifies `plan.md` status markers
- **Beads Sync**: Only the Orchestrator runs `bd update` and `bd close` commands
- **Merge Operations**: Merges completed agent work into integration branch
- **Phase Checkpoints**: Creates checkpoint commits after batch completion

### Sub-Agent Responsibilities

Task agents work in isolated environments:

- **Execute TDD Workflow**: Standard Red → Green → Refactor within assigned task
- **Respect File Leases**: Only modify files explicitly leased to them
- **Work in Worktree**: All changes happen in isolated git worktree/branch
- **Report Results**: Return structured JSON with status, commit SHA, files modified
- **Never Modify Control Files**: Do NOT touch `plan.md`, `metadata.json`, or run `bd` commands

### File Access Rules

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         FILE LEASE PROTOCOL                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Orchestrator requests lease BEFORE spawning agent                   │
│     → file_reservation_paths(paths, exclusive=true, reason="bd-###")    │
│                                                                          │
│  2. If file already leased by another agent:                            │
│     → Task queued for next batch (no conflict)                          │
│                                                                          │
│  3. Agent receives lease list in spawn prompt                           │
│     → May ONLY modify files in that list                                │
│                                                                          │
│  4. Pre-commit hook enforces compliance                                 │
│     → Commit rejected if agent touches non-leased files                 │
│                                                                          │
│  5. Orchestrator releases lease after merge                             │
│     → release_file_reservations() on task completion                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Default Access:**
- Shared utilities (`/lib`, `/test/util`) are read-only unless explicitly leased
- Exclusive write requires explicit lease grant
- Leases have TTL (default: 1 hour) with renewal support

### Git Worktree Strategy

Each sub-agent works in an isolated git worktree:

```bash
# Orchestrator creates worktree for each task
git worktree add worktrees/<track_id>/<task_key> -b track/<track_id>/<task_key>

# Agent works in isolation
cd worktrees/<track_id>/<task_key>
# ... TDD workflow ...
git commit -m "feat(auth): Implement login [Task-Id: bd-123]"

# Orchestrator merges after task_complete
git checkout track/<track_id>/integration
git merge track/<track_id>/<task_key> --no-ff
```

### Parallel Execution Flow

```
                    ┌─────────────────────────────┐
                    │       ORCHESTRATOR          │
                    │  (owns plan.md & Beads)     │
                    └─────────────┬───────────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
    ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
    │  GreenCastle  │     │   BlueLake    │     │  RedMountain  │
    │  (Agent 1)    │     │  (Agent 2)    │     │  (Agent 3)    │
    │               │     │               │     │               │
    │  Task bd-123  │     │  Task bd-124  │     │  Task bd-125  │
    │  worktree/t1  │     │  worktree/t2  │     │  worktree/t3  │
    └───────┬───────┘     └───────┬───────┘     └───────┬───────┘
            │                     │                     │
            └─────────────────────┴─────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │     Integration Branch      │
                    │  (merged by Orchestrator)   │
                    └─────────────────────────────┘
```

### TDD in Parallel Mode

| Step | Sequential Mode | Parallel Mode |
|------|-----------------|---------------|
| Select Task | Agent reads plan.md | Orchestrator assigns via Task tool |
| Mark In Progress | Agent edits plan.md | Orchestrator edits after spawn |
| Write Tests (Red) | Direct file access | Within leased files only |
| Implement (Green) | Direct file access | Within leased files only |
| Commit | Direct to branch | To worktree branch |
| Update plan.md | Agent edits | Orchestrator edits after merge |
| Beads update | Agent calls `bd` | Orchestrator calls `bd` |

### Error Recovery

| Scenario | Detection | Recovery |
|----------|-----------|----------|
| Agent stall | No response 10 min | Rollback worktree, reassign |
| Merge conflict | Git merge fails | Create resolution task |
| Test failure | Agent reports | Retry (2x), then block |
| Lease violation | Pre-commit hook | Agent requests proper lease |
| MCP server down | Connection refused | Fallback to sequential |

### State Persistence

Parallel execution state is saved to `implement_parallel_state.json`:

```json
{
  "mode": "parallel",
  "current_batch": 2,
  "agents": {
    "GreenCastle": {"task_id": "bd-123", "status": "complete"},
    "BlueLake": {"task_id": "bd-124", "status": "running"}
  },
  "integration_branch": "track/auth_20241229/integration"
}
```

This enables resume after interruption with `/conductor-implement-parallel --resume`.

## Emergency Procedures

### Critical Bug in Production
1. Create hotfix branch from main
2. Write failing test for bug
3. Implement minimal fix
4. Test thoroughly including mobile
5. Deploy immediately
6. Document in plan.md

### Data Loss
1. Stop all write operations
2. Restore from latest backup
3. Verify data integrity
4. Document incident
5. Update backup procedures

### Security Breach
1. Rotate all secrets immediately
2. Review access logs
3. Patch vulnerability
4. Notify affected users (if any)
5. Document and update security procedures

## Deployment Workflow

### Pre-Deployment Checklist
- [ ] All tests passing
- [ ] Coverage >80%
- [ ] No linting errors
- [ ] Mobile testing complete
- [ ] Environment variables configured
- [ ] Database migrations ready
- [ ] Backup created

### Deployment Steps
1. Merge feature branch to main
2. Tag release with version
3. Push to deployment service
4. Run database migrations
5. Verify deployment
6. Test critical paths
7. Monitor for errors

### Post-Deployment
1. Monitor analytics
2. Check error logs
3. Gather user feedback
4. Plan next iteration

## Continuous Improvement

- Review workflow weekly
- Update based on pain points
- Document lessons learned
- Optimize for user happiness
- Keep things simple and maintainable
