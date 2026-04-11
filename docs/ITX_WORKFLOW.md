# ITX Workflow Guide

ITX (Issue To eXecution) provides a complete issue-to-PR workflow for GitHub projects.

## Overview

ITX implements a structured workflow:

```
Create Issue → Triage → Plan → Execute → Verify → Review → Merge
```

All skills are prefixed with `/itx:` and work with any GitHub repository.

## Workflow Pipeline

### 1. Issue Creation

**Create Feature/Improvement**:
```
/itx:issue-new
```
Creates a new GitHub issue with template for features, improvements, or chores.

**Create Bug Report**:
```
/itx:bug-new
```
Creates a bug report with structured template (steps to reproduce, environment, etc.).

**Quick Notes**:
```
/itx:note
```
Capture quick ideas to `NOTES.md` for later issue creation.

### 2. Triage

```
/itx:triage
```

Reviews unlabeled issues and applies appropriate labels:
- `bug` - Bugs and defects
- `feature` - New features
- `improvement` - Enhancements to existing functionality
- `chore` - Maintenance tasks

**When to use**: Daily review of new issues to ensure proper categorization.

### 3. Planning

**High-Level Plan**:
```
/itx:plan-create <issue-number>
```

Creates implementation plan with:
- Context analysis (codebase exploration)
- Technical approach
- Key considerations
- Plan saved to `.plan/issue-<N>/PLAN.md`

**Phased Execution Plan**:
```
/itx:plan-scaffold <issue-number>
```

Breaks plan into phases:
- Phase dependencies
- File changes per phase
- Verification steps
- Plan saved to `.plan/issue-<N>/EXECUTION.md`

**When to use**: Before starting implementation on any non-trivial issue.

### 4. Execution

```
/itx:execute <issue-number>
```

Implements the plan:
- Creates git worktree: `<repo>-issue-<N>`
- Creates feature branch: `issue-<N>`
- Executes phases with task tracking
- Updates GitHub project board (if configured)
- Runs in tmux session: `itx/exec`

**Task Tracking**:
- Uses Claude's task list for progress
- Each phase becomes a tracked task
- Updates status as work progresses

**GitHub Integration** (optional):
- Moves issue through project board
- Backlog → Ready → Executing → In Review → Done

### 5. Verification

```
/itx:verify
```

Runs validation in current worktree:
- Executes test suite
- Runs linter
- Checks for errors
- Configurable commands via `.claude/itx-config.json`

**Default commands**: `make test`, `make lint`

### 6. Code Review

**Check PR Status**:
```
/itx:pr-status
```
Lists all open PRs with status, reviewers, and checks.

**Request Review**:
```
/itx:review-pr
```
Initiates code review process:
- MCP-based automated review (if configured)
- Manual review checklist (fallback)
- Updates issue status

### 7. Issue Updates

**Comment on Issue**:
```
/itx:issue-update <issue-number>
```
Add updates, questions, or status to an issue.

**Comment on Bug**:
```
/itx:bug-update <issue-number>
```
Add updates to bug reports (specialized template).

## Complete Example Workflow

### Scenario: Fix Authentication Bug

1. **Create Bug Report**:
   ```
   /itx:bug-new
   ```
   - Title: "Login fails with OAuth providers"
   - Severity: High
   - Steps to reproduce documented

2. **Triage**:
   ```
   /itx:triage
   ```
   - Labels applied: `bug`, `auth`
   - Issue #42 created

3. **Plan**:
   ```
   /itx:plan-create 42
   /itx:plan-scaffold 42
   ```
   - Investigation plan created
   - 3-phase execution plan scaffolded

4. **Execute**:
   ```
   /itx:execute 42
   ```
   - Worktree created: `itx-issue-42/`
   - Branch: `issue-42`
   - Phases executed with task tracking
   - Tests added, bug fixed

5. **Verify**:
   ```
   /itx:verify
   ```
   - All tests pass
   - Linter clean

6. **Review**:
   ```
   /itx:review-pr
   ```
   - Review requested
   - PR checks passing

7. **Merge**:
   - Reviewer approves
   - Merge to main
   - Issue auto-closed

## Configuration

See [CONFIG.md](../.claude/CONFIG.md) for full configuration options.

### Zero-Config Usage

ITX works immediately without any configuration:
- Auto-detects repository from git remote
- Uses standard `make` commands
- Skips optional features (project board, MCP)

### Common Customizations

**JavaScript/Node Project**:
```json
{
  "build": {
    "test_command": "npm test",
    "lint_command": "npm run lint"
  }
}
```

**Python Project**:
```json
{
  "build": {
    "test_command": "pytest",
    "lint_command": "ruff check ."
  }
}
```

**Rust Project**:
```json
{
  "build": {
    "test_command": "cargo test",
    "lint_command": "cargo clippy"
  }
}
```

## Skill Reference

| Skill | Purpose | Arguments |
|-------|---------|-----------|
| `/itx:bug-new` | Create bug report | None |
| `/itx:bug-update` | Comment on bug | `<issue-number>` |
| `/itx:issue-new` | Create issue | None |
| `/itx:issue-update` | Comment on issue | `<issue-number>` |
| `/itx:note` | Quick note capture | None |
| `/itx:triage` | Label unlabeled issues | None |
| `/itx:plan-create` | Create high-level plan | `<issue-number>` |
| `/itx:plan-scaffold` | Create phased plan | `<issue-number>` |
| `/itx:execute` | Execute implementation | `<issue-number>` |
| `/itx:verify` | Run tests/lint | None |
| `/itx:pr-status` | Check PR status | None |
| `/itx:review-pr` | Request code review | None |

## Tips

**Daily Workflow**:
- Morning: `/itx:triage` - Review new issues
- Planning: `/itx:plan-create` + `/itx:plan-scaffold` - Design approach
- Implementation: `/itx:execute` - Build solution
- Validation: `/itx:verify` - Ensure quality
- Review: `/itx:review-pr` - Get feedback

**Best Practices**:
- Always plan before executing (non-trivial issues)
- Use `/itx:note` to capture ideas quickly
- Run `/itx:verify` before requesting review
- Keep plans updated if approach changes

**Worktree Management**:
- Worktrees are created in parent directory
- Clean up with: `git worktree remove <repo>-issue-<N>`
- List worktrees: `git worktree list`

**Tmux Session**:
- Execution runs in `itx/exec` session
- Attach: `tmux attach -t itx/exec`
- Kill session: `tmux kill-session -t itx/exec`

## Troubleshooting

**"Project board not configured"**:
- Normal behavior without config
- Add config to enable: See [CONFIG.md](../.claude/CONFIG.md)

**Build commands fail**:
- Check configured commands in `.claude/itx-config.json`
- Ensure commands work in your shell first

**Auto-detection issues**:
- Verify git remote: `git config --get remote.origin.url`
- Must be a GitHub repository

**Worktree already exists**:
- Remove old worktree: `git worktree remove <path>`
- Or use different issue number
