# ITX Skills Reference

ITX provides a complete workflow system for GitHub projects, from issue creation to code review.

## ITX Workflow Skills (Issue-to-PR Pipeline)

These skills implement a structured workflow: Create Issue → Triage → Plan → Execute → Verify → Review → Merge

### Issue Creation & Management

- `/itx:bug-new` - Create bug report with structured template
  - Asks for customer outcome
  - Creates issue with bug label
  - Requires: steps to reproduce, expected/actual behavior, environment

- `/itx:issue-new` - Create feature or improvement issue
  - Asks for customer outcome
  - Creates structured issue with motivation and acceptance criteria

- `/itx:note` - Quick capture ideas to NOTES.md
  - Fast note taking during development
  - Appends with timestamp
  - No GitHub interaction (local only)

- `/itx:triage` - Review and label unlabeled issues
  - Finds issues without workflow labels
  - Applies: needs-triage, planning, bug labels
  - Adds comment with rationale

- `/itx:issue-update <N>` - Comment on existing issue
  - Adds comment with prompt log
  - Works for any issue type

- `/itx:bug-update <N>` - Comment on bug issue
  - Specialized for bug updates
  - Includes technical details

### Planning

- `/itx:plan-create <N>` - Create high-level implementation plan
  - Explores codebase for context
  - Identifies affected files
  - Saves to `.itx/<N>/00_PLAN.md`
  - Creates subtasks if needed
  - Updates issue labels: planning → planned

- `/itx:plan-scaffold <N>` - Break plan into executable phases
  - Reads plan-create output
  - Defines phases with entry/exit criteria
  - Saves to `.itx/<N>/01_EXECUTION.md`
  - Creates subtask issues for complex work
  - Updates issue labels: planned → ready

### Execution

- `/itx:execute <N>` - Execute implementation plan
  - Creates git worktree: `<repo>-issue-<N>/`
  - Creates feature branch: `issue-<N>`
  - Runs in tmux session: `itx/exec`
  - Uses task tracking for progress
  - Runs configured build commands
  - Optional: Updates GitHub project board
  - Creates PR when complete

### Verification & Review

- `/itx:verify` - Run tests and linting
  - Executes configured test command (default: make test)
  - Runs configured lint command (default: make lint)
  - Optional: Coverage check
  - Reports results

- `/itx:pr-status` - Check status of open PRs
  - Lists all open PRs with details
  - Shows review status and CI results
  - Highlights PRs needing attention

- `/itx:review-pr [N]` - Request code review
  - Optional: MCP-based automated review
  - Default: Manual review with checklist
  - Posts review summary as comment
  - Optional: Updates project board to "in review"

## Configuration

All skills work with **zero configuration**. Optionally customize via `.claude/itx-config.json`:

### Build Commands
```json
{
  "build": {
    "test_command": "npm test",
    "lint_command": "npm run lint"
  }
}
```

### GitHub Project Board (Optional)
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

### MCP Integration (Optional)
```json
{
  "mcp": {
    "review_enabled": true,
    "review_tool": "mcp__atx__request_review"
  }
}
```

See `.claude/CONFIG.md` for complete configuration reference.

## Additional Skills

### Blog Writing

- `/blog-writing` - Technical blog post creation using Socratic method
  - Drafts in Obsidian vault
  - Publishes to Hugo blog (devashish.me)
  - Workflows: brainwave, draft, review, publish
  - Follows Will Larson's style guide
  - See `.claude/skills/blog-writing/SKILL.md` for details

### Diagram Creation

- `/excalidraw-diagram` - Create visual diagrams that argue, not just display
  - Generates `.excalidraw` JSON files
  - Evidence-based diagrams for technical architectures
  - Multi-zoom architecture (summary, sections, details)
  - Requires render-view-fix validation loop
  - See `.claude/skills/excalidraw-diagram/SKILL.md` for details

### Meeting Management

- `/meeting-to-gdoc` - Convert meeting notes to formatted Google Docs
  - Creates structured docs with sections
  - Preserves raw notes in appendix
  - Uploads to appropriate Drive folder
  - Supports VC, customer, and general meetings
  - See `.claude/skills/meeting-to-gdoc/SKILL.md` for details

### JIRA Sprint Helper

- `/jira-sprint-helper` - Vanguards team JIRA management
  - Validates epic/story/bug fields
  - Generates work type reports
  - Auto-calculates DevRel from sprint name
  - Ensures compliance with sprint guidelines
  - **Company-specific**: Esper/Vanguards team
  - See `.claude/skills/jira-sprint-helper/SKILL.md` for details

### Company Journal

- `/i7n-journal` - Iteration Inc. company journal management
  - Logs wins/learnings/help-requests to monthly syslog
  - Drafts investor updates from syslog
  - Google Drive integration via gws CLI
  - **Company-specific**: Iteration Inc. only
  - **Access control**: Only for i7n-cos agent
  - See `.claude/skills/i7n-journal/SKILL.md` for details

## Workflow Examples

### Bug Fix Workflow
```
/itx:bug-new              # Create bug report
/itx:triage               # Label it
/itx:plan-create 42       # Create fix plan
/itx:plan-scaffold 42     # Break into phases
/itx:execute 42           # Implement fix
/itx:verify               # Run tests
/itx:review-pr            # Request review
```

### Feature Development
```
/itx:note                 # Capture initial idea
/itx:issue-new            # Create formal issue
/itx:triage               # Label it
/itx:plan-create 15       # Design approach
/itx:plan-scaffold 15     # Create phases
/itx:execute 15           # Implement
/itx:verify               # Validate
/itx:review-pr            # Review
```

### Blog Post Creation
```
/blog-writing             # Start Socratic process
# Or for quick capture:
/blog-writing brainwave   # Save idea to pipeline
```

### Technical Diagram
```
/excalidraw-diagram       # Create architecture diagram
# Renders to PNG for validation
```

## Prompt Logging Standard

ALL itx skills MUST capture user intent by appending prompt log entries to local files.

### File Structure

```
.itx/
  <N>/
    00_PLAN.md        # Planning phase
    01_EXECUTION.md   # Execution phase
    02_VERIFY.md      # Verification phase
```

### Prompt Log Location by Skill

| Skill | Phase | Prompt Log Location |
|-------|-------|---------------------|
| `itx:issue-new` | Planning | `.itx/<N>/00_PLAN.md` |
| `itx:bug-new` | Planning | `.itx/<N>/00_PLAN.md` |
| `itx:issue-update` | Planning | `.itx/<N>/00_PLAN.md` |
| `itx:bug-update` | Planning | `.itx/<N>/00_PLAN.md` |
| `itx:triage` | Planning | `.itx/<N>/00_PLAN.md` |
| `itx:plan-create` | Planning | `.itx/<N>/00_PLAN.md` |
| `itx:plan-scaffold` | Execution | `.itx/<N>/01_EXECUTION.md` |
| `itx:execute` | Execution | `.itx/<N>/01_EXECUTION.md` |
| `itx:verify` | Verify | `.itx/<N>/02_VERIFY.md` |
| `itx:pr-status` | Verify | `.itx/<N>/02_VERIFY.md` |
| `itx:review-pr` | Verify | `.itx/<N>/02_VERIFY.md` |
| `itx:note` | Planning | `NOTES.md` |

### Format

Append to end of the appropriate file:

```yaml
---
# PROMPT_LOG (append new entries, do not overwrite)
prompt_log:
  - timestamp: "<ISO8601>"
    model: "<model-id>"
    skill: "<skill-name>"
    input: |
      <verbatim user input>
---
```

### Rules

1. **Append only** - Do not overwrite existing entries
2. **Existing section** - If `prompt_log` section exists, append to the YAML array
3. **Verbatim capture** - Record exact user input including arguments
4. **ISO8601 timestamp** - Use format `2026-04-30T12:00:00Z`
5. **Model ID** - Record the model used (e.g., `claude-opus-4-5-20251101`)

### Example: Multiple Rounds

```yaml
---
# PROMPT_LOG
prompt_log:
  - timestamp: "2026-04-30T12:00:00Z"
    model: "claude-opus-4-5-20251101"
    skill: "itx:plan-create"
    input: |
      /itx:plan-create 42
  - timestamp: "2026-04-30T14:30:00Z"
    model: "claude-opus-4-5-20251101"
    skill: "itx:plan-create"
    input: |
      update the plan to add error handling for edge cases
  - timestamp: "2026-04-30T16:00:00Z"
    model: "claude-sonnet-4-20250514"
    skill: "itx:issue-update"
    input: |
      /itx:issue-update 42 added caching layer to the design
---
```

## Best Practices

1. **Always plan before executing** non-trivial issues
2. **Use `/itx:note`** to capture ideas quickly
3. **Run `/itx:verify`** before requesting review
4. **Keep plans updated** if approach changes
5. **Use worktree mode** for parallel work on multiple issues
6. **Review task list frequently** during execution with `TaskList()`

## Documentation

- `README.md` - ITX overview and quick start
- `docs/ITX_WORKFLOW.md` - Complete workflow guide
- `.claude/CONFIG.md` - Configuration options
- Individual skill files in `.claude/skills/*/SKILL.md`

## Portability

ITX skills work in **any GitHub repository**:
1. Copy `.claude/skills/` to your project
2. Optionally add `.claude/itx-config.json` for customization
3. Start using skills immediately

Zero configuration required - skills auto-detect:
- Repository from git remote
- Project name from repository
- Sensible defaults for build commands
