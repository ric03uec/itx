# Implementation Plan: Standardize Skill I/O Formats

## Overview

Implement Unix-style input/output contracts for all ITX skills to enable workflow composition. This requires:
1. Defining a standard envelope format for skill outputs
2. Documenting input/output schemas per skill
3. Updating skills to emit structured output
4. Creating a compatibility matrix

## Complexity Assessment

**Complex Issue** - 12 skill files + 3 documentation files + schema definitions

## Files to Modify

### Documentation (Create)
- `.claude/SCHEMAS.md` - Central schema definitions
- `.claude/COMPATIBILITY.md` - Skill compatibility matrix

### Skill Files (Modify - Add I/O Sections)
All 12 SKILL.md files need Input/Output sections added:

1. `.claude/skills/itx-issue-new/SKILL.md`
2. `.claude/skills/itx-bug-new/SKILL.md`
3. `.claude/skills/itx-issue-update/SKILL.md`
4. `.claude/skills/itx-bug-update/SKILL.md`
5. `.claude/skills/itx-note/SKILL.md`
6. `.claude/skills/itx-triage/SKILL.md`
7. `.claude/skills/itx-plan-create/SKILL.md`
8. `.claude/skills/itx-plan-scaffold/SKILL.md`
9. `.claude/skills/itx-execute/SKILL.md`
10. `.claude/skills/itx-verify/SKILL.md`
11. `.claude/skills/itx-pr-status/SKILL.md`
12. `.claude/skills/itx-review-pr/SKILL.md`

### Update Existing
- `.claude/AGENTS.md` - Add workflow composition examples

## Schema Design

### Standard Envelope Format

```yaml
---
skill: <skill-name>
version: 1
status: success | error
timestamp: <ISO8601>
---
<payload>
```

### Defined Types (Primitives)

| Type | Format | Example |
|------|--------|---------|
| `issue-ref` | `{number, url}` | `{number: 42, url: "https://..."}` |
| `pr-ref` | `{number, url, branch}` | `{number: 15, url: "...", branch: "issue-42"}` |
| `plan-ref` | `{issue: issue-ref, file: path}` | `{issue: {...}, file: ".itx/42/00_PLAN.md"}` |
| `verification-result` | `{tests: pass/fail, lint: pass/fail}` | `{tests: "pass", lint: "pass"}` |
| `review-result` | `{verdict: ready/needs-changes, issues: [...]}` | `{verdict: "ready", issues: []}` |

### Skill I/O Map

| Skill | Input Type | Output Type |
|-------|------------|-------------|
| `itx:issue-new` | `context` (conversation) | `issue-ref` |
| `itx:bug-new` | `context` (conversation) | `issue-ref` |
| `itx:triage` | `issue-ref[]` or `none` | `issue-ref[]` |
| `itx:issue-update` | `issue-ref` + `text` | `issue-ref` |
| `itx:bug-update` | `issue-ref` + `text` | `issue-ref` |
| `itx:note` | `text` | `file-ref` |
| `itx:plan-create` | `issue-ref` | `plan-ref` |
| `itx:plan-scaffold` | `plan-ref` | `plan-ref` + `issue-ref[]` |
| `itx:execute` | `issue-ref` or `plan-ref` | `pr-ref` |
| `itx:verify` | `none` (working dir) | `verification-result` |
| `itx:pr-status` | `none` | `pr-ref[]` |
| `itx:review-pr` | `pr-ref` | `review-result` |

## Composition Patterns

### Chain Compatibility

```
issue-new → triage → plan-create → plan-scaffold → execute → review-pr
    ↓           ↓          ↓              ↓           ↓         ↓
issue-ref  issue-ref   plan-ref      plan-ref     pr-ref   review-result
```

### Valid Chains

1. **Full Pipeline**: `issue-new | triage | plan-create | execute | review-pr`
2. **Quick Fix**: `bug-new | plan-create | execute`
3. **Review Flow**: `pr-status | review-pr`
4. **Planning Only**: `issue-new | plan-create | plan-scaffold`

## Steps

### Phase 1: Schema Foundation
1. Create `.claude/SCHEMAS.md` with type definitions
2. Create `.claude/COMPATIBILITY.md` with skill chains

### Phase 2: Update Issue Creation Skills
3. Update `itx-issue-new/SKILL.md` with I/O sections
4. Update `itx-bug-new/SKILL.md` with I/O sections

### Phase 3: Update Issue Management Skills
5. Update `itx-issue-update/SKILL.md`
6. Update `itx-bug-update/SKILL.md`
7. Update `itx-note/SKILL.md`
8. Update `itx-triage/SKILL.md`

### Phase 4: Update Planning Skills
9. Update `itx-plan-create/SKILL.md`
10. Update `itx-plan-scaffold/SKILL.md`

### Phase 5: Update Execution Skills
11. Update `itx-execute/SKILL.md`
12. Update `itx-verify/SKILL.md`

### Phase 6: Update Review Skills
13. Update `itx-pr-status/SKILL.md`
14. Update `itx-review-pr/SKILL.md`

### Phase 7: Documentation
15. Update `AGENTS.md` with composition examples

## Test Strategy

- Verify each skill document has valid I/O sections
- Validate schema examples parse correctly
- Check compatibility matrix covers all chains
- Manual test: run a sample chain and verify output formats

## Risks

1. **Breaking existing behavior**: Skills must maintain backward compatibility
2. **Over-engineering**: Keep schemas simple, don't add fields that aren't used
3. **Enforcement**: Schemas are documentation, not runtime validation

## Subtask Recommendation

Given complexity (12+ files), recommend creating subtasks:
- Phase 1-2: Schema + Issue Creation (subtask 1)
- Phase 3-4: Issue Management + Planning (subtask 2)
- Phase 5-7: Execution + Review + Docs (subtask 3)
