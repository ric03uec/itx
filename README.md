# ITX

Agent-first workflow for GitHub projects. Complete issue-to-PR pipeline.

## Overview

ITX provides 12 skills that implement a structured workflow from issue creation to code review:

```
Create Issue → Triage → Plan → Execute → Verify → Review → Merge
```

**Zero configuration required** - Works with any GitHub repository out of the box.

## Features

- **Issue Management**: Create, triage, and update issues and bugs
- **Planning**: Generate implementation plans with context analysis
- **Execution**: Automated implementation with task tracking
- **Verification**: Run tests and linting
- **Code Review**: Automated or manual review workflows
- **Git Worktrees**: Isolated development environments per issue
- **GitHub Integration**: Optional project board automation
- **Portable**: Works with any GitHub repository

## Quick Start

### Installation

1. Clone this repository to access the skills:
   ```bash
   git clone https://github.com/ric03uec/itx.git
   cd itx
   ```

2. Copy skills to your project:
   ```bash
   # From your project directory
   cp -r /path/to/itx/.claude/skills .claude/
   ```

3. Start using skills immediately:
   ```bash
   claude code
   # In Claude: /itx:triage
   ```

### Your First Workflow

1. **Create an issue**:
   ```
   /itx:issue-new
   ```

2. **Triage it**:
   ```
   /itx:triage
   ```

3. **Create a plan**:
   ```
   /itx:plan-create 1
   /itx:plan-scaffold 1
   ```

4. **Execute the plan**:
   ```
   /itx:execute 1
   ```

5. **Verify the changes**:
   ```
   /itx:verify
   ```

6. **Request review**:
   ```
   /itx:review-pr
   ```

## Skills Reference

### Issue Creation
- `/itx:bug-new` - Create bug report with template
- `/itx:issue-new` - Create feature/improvement issue
- `/itx:note` - Quick note capture to NOTES.md

### Issue Management
- `/itx:triage` - Review and label unlabeled issues
- `/itx:issue-update <N>` - Comment on issue
- `/itx:bug-update <N>` - Comment on bug

### Planning
- `/itx:plan-create <N>` - Create high-level implementation plan
- `/itx:plan-scaffold <N>` - Break plan into phases

### Development
- `/itx:execute <N>` - Execute implementation with task tracking
- `/itx:verify` - Run tests and linting

### Code Review
- `/itx:pr-status` - Check status of open PRs
- `/itx:review-pr` - Request code review

## Configuration (Optional)

ITX works with zero configuration, but you can customize:

### Build Commands

Create `.claude/itx-config.json`:

```json
{
  "build": {
    "test_command": "npm test",
    "lint_command": "npm run lint"
  }
}
```

### GitHub Project Board

Enable automated issue status updates:

```json
{
  "github": {
    "project_board": {
      "enabled": true,
      "project_id": "PVT_...",
      "status_field_id": "PVTSSF_...",
      "status_options": {
        "backlog": "...",
        "ready": "...",
        "executing": "...",
        "in_review": "...",
        "done": "..."
      }
    }
  }
}
```

See [CONFIG.md](.claude/CONFIG.md) for full configuration options.

## Documentation

- [Workflow Guide](docs/ITX_WORKFLOW.md) - Complete workflow documentation
- [Configuration Guide](.claude/CONFIG.md) - Configuration options

## How It Works

### Execution Flow

1. **Planning**: Analyzes codebase and creates detailed implementation plan
2. **Worktree**: Creates isolated git worktree for issue (e.g., `myrepo-issue-42/`)
3. **Branch**: Creates feature branch (`issue-42`)
4. **Task Tracking**: Breaks work into tracked tasks
5. **Implementation**: Executes phases with progress tracking
6. **Validation**: Runs tests and linting
7. **Review**: Initiates code review process

### Git Worktrees

Each issue gets an isolated worktree:
- Main repo: `/project/`
- Issue 42 worktree: `/project-issue-42/`
- Clean separation, no branch switching

### Task Management

ITX uses Claude's task tracking:
- Each phase becomes a task
- Real-time progress updates
- Clear completion criteria

## Example Workflows

### Bug Fix Workflow

```bash
# Create bug report
/itx:bug-new

# Triage and label
/itx:triage

# Plan the fix
/itx:plan-create 42
/itx:plan-scaffold 42

# Implement
/itx:execute 42

# Verify
/itx:verify

# Review
/itx:review-pr
```

### Feature Development Workflow

```bash
# Capture initial idea
/itx:note

# Create formal issue
/itx:issue-new

# Triage
/itx:triage

# Design and plan
/itx:plan-create 15
/itx:plan-scaffold 15

# Implement
/itx:execute 15

# Validate
/itx:verify

# Code review
/itx:review-pr
```

## Requirements

- Claude Code or OpenCode CLI
- Git repository with GitHub remote
- GitHub CLI (`gh`) installed and authenticated

## Project Structure

```
.claude/
├── skills/           # 12 ITX skills
│   ├── itx-bug-new/
│   ├── itx-execute/
│   ├── itx-plan-create/
│   └── ...
├── itx-config.json.example  # Configuration template
└── CONFIG.md         # Configuration documentation

.itx/                 # Generated plans and execution files (auto-created)
└── <N>/              # Issue number
    ├── 00_PLAN.md    # High-level plan
    └── 01_EXECUTION.md  # Phased execution plan

docs/
└── ITX_WORKFLOW.md   # Workflow guide
```

## Why ITX?

**Structured**: Enforces consistent workflow across all issues

**Automated**: Reduces manual steps in issue-to-PR pipeline

**Traceable**: Clear audit trail from issue to implementation

**Portable**: Works with any GitHub project, any language

**Flexible**: Optional features, sensible defaults

## License

MIT

## Contributing

Contributions welcome! See issues for planned enhancements.
