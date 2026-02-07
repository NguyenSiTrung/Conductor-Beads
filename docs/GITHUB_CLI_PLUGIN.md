# Conductor-Beads as GitHub CLI Plugin

This repository can be used as a GitHub CLI plugin (v5.4+) to extend your GitHub CLI with Conductor-Beads context-driven development capabilities.

## Installation

### From GitHub Repository (Latest)

```bash
gh https://github.com/NilusvanEdel/Conductor-Beads-Copilot
```

### From Local Directory

For local development or testing:

```bash
gh plugin install ./path/to/conductor-beads-copilot
```

## Available Commands & Skills

Once installed, the plugin exposes the following skills available as slash commands in GitHub CLI:

### Skills
- **conductor** - Orchestrates context-driven development workflow (planning, implementation, validation)
- **beads** - Persistent task memory system with cross-session tracking
- **skill-creator** - Guide for creating custom AI agent skills

### Conductor Commands
The plugin includes all 16 Conductor commands:
- `conductor-setup` - Initialize project with context files
- `conductor-newtrack` - Create feature/bug tracks with specs and plans
- `conductor-implement` - Execute implementation tasks (TDD workflow)
- `conductor-status` - Display progress overview
- `conductor-revert` - Git-aware revert of tracks/phases/tasks
- `conductor-validate` - Validate project integrity
- `conductor-block` - Mark tasks as blocked
- `conductor-skip` - Skip tasks with justification
- `conductor-revise` - Update specs when implementation reveals issues
- `conductor-archive` - Archive completed tracks
- `conductor-export` - Generate project summaries
- `conductor-handoff` - Create context handoff for section transfers
- `conductor-refresh` - Sync context with current codebase
- `conductor-formula` - List and manage track templates
- `conductor-wisp` - Create ephemeral exploration tracks
- `conductor-distill` - Extract reusable templates from completed tracks

## Usage

Access skills via GitHub CLI's command interface:

```bash
gh conductor-setup
gh conductor-newtrack "feature-name"
gh conductor-status
```

## Requirements

- **GitHub CLI v5.4.0+** - Plugin system support
- **Beads** (optional but recommended) - For persistent memory:
  ```bash
  npm install -g @beads/bd
  # or
  brew install steveyegge/beads/bd
  ```

## Supported Platforms

This plugin works alongside:
- **Gemini CLI** - via extension commands (`commands/conductor/`)
- **Claude Code** - via slash commands and skills (`.claude/`)
- **GitHub CLI** - via this plugin

All three systems coexist and share the same `conductor/` directory structure in user projects.

## Development

### Local Testing

```bash
# Install locally from current directory
gh plugin install .

# Use the plugin
gh conductor-setup

# Uninstall for cleanup
gh plugin uninstall conductor
```

### Repository Structure

```
.
├── plugin.json              # GitHub CLI plugin manifest
├── skills/                  # Skill definitions (symlinked to .claude/skills)
├── commands/                # Gemini CLI commands (TOML)
├── .claude/                 # Claude Code commands & skills
├── templates/               # Workflow templates
├── docs/                    # Documentation
└── README.md               # Main documentation
```

## Compatibility

✅ **Backward Compatible** - This plugin does not affect existing Gemini CLI extensions or Claude Code functionality.

## Documentation

For complete documentation on Conductor-Beads:
- See [README.md](README.md) for overview
- See [CLAUDE.md](CLAUDE.md) for Claude Code integration
- See [GEMINI.md](GEMINI.md) for Gemini CLI integration
- See [docs/](docs/) for detailed guides

## License

MIT
