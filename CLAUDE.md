# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Conductor-Beads is a unified toolkit for **Context-Driven Development** that combines:

- **Conductor**: Spec-first planning, human-readable context, TDD workflow
- **Beads**: Dependency-aware task graph, cross-session memory, agent-optimized output

It works with both Gemini CLI (via extension) and Claude Code (via commands and skills).

## Architecture

### Repository Structure
```
Conductor-Beads/
├── .claude/
│   ├── commands/           # Claude Code slash commands (13 commands)
│   └── skills/             # Claude Code skills
│       ├── conductor/      # Context-driven development skill
│       ├── beads/          # Persistent task memory skill
│       └── skill-creator/  # Skill creation guide
├── commands/conductor/     # Gemini CLI TOML commands (13 commands)
├── templates/              # Workflow and styleguide templates
├── docs/                   # Documentation
├── CLAUDE.md               # This file
├── GEMINI.md               # Gemini CLI context
└── gemini-extension.json   # Gemini extension manifest
```

### Commands

| Gemini CLI | Claude Code | Purpose |
|------------|-------------|---------|
| `/conductor:setup` | `/conductor-setup` | Initialize project with context files and first track |
| `/conductor:newTrack` | `/conductor-newtrack` | Create feature/bug track with spec and plan |
| `/conductor:implement` | `/conductor-implement` | Execute tasks from track's plan (TDD workflow) |
| `/conductor:implementParallel` | `/conductor-implement-parallel` | Execute tasks in parallel with sub-agents |
| `/conductor:status` | `/conductor-status` | Display progress overview |
| `/conductor:revert` | `/conductor-revert` | Git-aware revert of tracks, phases, or tasks |
| `/conductor:validate` | `/conductor-validate` | Validate project integrity and fix issues |
| `/conductor:block` | `/conductor-block` | Mark task as blocked with reason |
| `/conductor:skip` | `/conductor-skip` | Skip current task with justification |
| `/conductor:revise` | `/conductor-revise` | Update spec/plan when implementation reveals issues |
| `/conductor:archive` | `/conductor-archive` | Archive completed tracks |
| `/conductor:export` | `/conductor-export` | Generate project summary export |
| `/conductor:handoff` | `/conductor-handoff` | Create context handoff for section transfer |
| `/conductor:refresh` | `/conductor-refresh` | Sync context docs with current codebase state |

### Skills

| Skill | Location | Purpose |
|-------|----------|---------|
| **conductor** | `.claude/skills/conductor/` | Auto-activates when `conductor/` exists. Provides intent mapping and proactive behaviors. |
| **beads** | `.claude/skills/beads/` | Auto-activates when `.beads/` exists. Provides persistent memory across sessions. |
| **skill-creator** | `.claude/skills/skill-creator/` | Guide for creating new AI agent skills. |

### Generated Artifacts (in user projects)

When users run Conductor-Beads, it creates:
```
project/
├── conductor/
│   ├── product.md           # Product vision and goals
│   ├── product-guidelines.md # Brand/style guidelines
│   ├── tech-stack.md        # Technology choices
│   ├── workflow.md          # Development workflow (TDD, commits)
│   ├── tracks.md            # Master track list with status
│   ├── beads.json           # Beads integration config
│   ├── setup_state.json     # Resume state for setup
│   ├── refresh_state.json   # Context refresh tracking
│   ├── code_styleguides/    # Language-specific style guides
│   └── tracks/
│       └── <track_id>/
│           ├── metadata.json     # Track config + Beads epic ID
│           ├── spec.md           # Requirements
│           ├── plan.md           # Phased task list
│           ├── implement_state.json # Resume state (if in progress)
│           ├── handoff_*.md      # Section handoff documents
│           ├── blockers.md       # Block history log
│           ├── skipped.md        # Skipped tasks log
│           └── revisions.md      # Revision history log
└── .beads/                  # Beads data (created by bd init)
```

## Key Concepts

### Tracks
A track is a logical unit of work (feature or bug fix). Each track has:
- Unique ID format: `shortname_YYYYMMDD` (e.g., `auth_20241226`)
- Status markers: `[ ]` new, `[~]` in progress, `[x]` completed, `[!]` blocked, `[-]` skipped
- Own directory with spec, plan, metadata, and state files

### Task Workflow (TDD)
1. Select task from plan.md (or use `bd ready` if Beads enabled)
2. Mark `[~]` in progress (sync to Beads: `bd update <id> --status in_progress`)
3. Write failing tests (Red)
4. Implement to pass (Green)
5. Refactor
6. Verify >80% coverage
7. Commit with message format: `<type>(<scope>): <description>`
8. Update plan.md with commit SHA
9. If Beads: `bd done <id> --note "commit: <sha>"`

### Beads Integration
When Beads is enabled (`conductor/beads.json` with `enabled: true`):
- Each track becomes a Beads epic
- Tasks sync to Beads for persistent memory
- `bd ready` finds tasks with no blockers
- Notes survive context compaction
- Graceful degradation if `bd` command fails

### Phase Checkpoints
At phase completion:
- Run full test suite
- Manual verification with user
- Create checkpoint commit

## Development Notes

- Commands are defined as Markdown (`.claude/commands/`) or TOML (`commands/conductor/`)
- Skills use SKILL.md format with references/ subdirectory
- State is tracked in JSON files (setup_state.json, implement_state.json, metadata.json)
- Git notes used for audit trails
- Commands validate setup before executing
- Both Gemini CLI and Claude Code use the same `conductor/` directory structure (interoperable)

## Documentation

- [Manual Workflow Guide](docs/manual-workflow-guide.md) - Step-by-step command reference
- [Beads Integration](docs/BEADS_INTEGRATION.md) - How Conductor and Beads work together
