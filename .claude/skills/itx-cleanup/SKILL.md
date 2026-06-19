---
name: itx:cleanup
description: Remove tmux window, worktree, and branch for a completed issue
argument-hint: "<issue-number> [--force]"
---
name: itx:cleanup

# Issue Cleanup

Tear down the per-issue infrastructure (tmux window, git worktree, feature branch) created by `/itx:execute`.

## Required Argument

`<issue-number>` is **required**. If not provided, exit with a non-zero error and print:

> Error: /itx:cleanup requires an issue number. Usage: /itx:cleanup <issue-number> [--force]

Do NOT attempt to infer the number from context, branch name, or current directory.

## Safety Model

By default, cleanup refuses to destroy work that hasn't been preserved:

- Worktree has uncommitted or untracked changes → refuse
- Local feature branch is not merged into `main` (or `origin/main` if a remote exists) → refuse

Pass `--force` to override both checks. `--force` is destructive and irreversible — only use it after confirming the PR is merged or the work is intentionally abandoned.

## Instructions

1. **Validate argument**:
   ```bash
   NUMBER="$1"
   FORCE="false"
   [ "$2" = "--force" ] && FORCE="true"

   if [ -z "$NUMBER" ]; then
     echo "Error: /itx:cleanup requires an issue number. Usage: /itx:cleanup <issue-number> [--force]" >&2
     exit 1
   fi
   ```

2. **Resolve paths and names** (run from inside the main repo, not the worktree being removed):
   ```bash
   REPO_ROOT=$(git rev-parse --show-toplevel)
   REPO_NAME=$(basename "$REPO_ROOT")
   REPO_PARENT=$(dirname "$REPO_ROOT")
   WORKTREE_PATH="${REPO_PARENT}/${REPO_NAME}-issue-${NUMBER}"
   TMUX_WINDOW="itx/exec:issue-${NUMBER}"
   # Branch slug varies; match by prefix
   BRANCH_PATTERN="issue-${NUMBER}-*"
   ```

   Detect a no-op early: if the worktree path does not exist AND no local branch matches the pattern AND the tmux window does not exist, print `Nothing to clean up for issue #${NUMBER}` and exit 0.

3. **Pre-flight safety checks** (skip entirely when `FORCE=true`):

   a. **Dirty worktree check**: if the worktree exists, ensure it has no uncommitted or untracked files.
      ```bash
      if [ -d "$WORKTREE_PATH" ]; then
        DIRTY=$(git -C "$WORKTREE_PATH" status --porcelain)
        if [ -n "$DIRTY" ]; then
          echo "Error: worktree $WORKTREE_PATH has uncommitted changes:" >&2
          echo "$DIRTY" >&2
          echo "Re-run with --force to discard." >&2
          exit 1
        fi
      fi
      ```

   b. **Unmerged branch check**: each matching local branch must be reachable from `origin/main` (preferred) or `main`.
      ```bash
      BASE_REF=$(git rev-parse --verify --quiet origin/main || git rev-parse --verify main)
      for BRANCH in $(git branch --list "$BRANCH_PATTERN" --format='%(refname:short)'); do
        if ! git merge-base --is-ancestor "$BRANCH" "$BASE_REF"; then
          echo "Error: branch $BRANCH is not merged into $BASE_REF." >&2
          echo "Re-run with --force to delete anyway." >&2
          exit 1
        fi
      done
      ```

4. **Kill tmux window**:
   ```bash
   if tmux has-session -t "itx/exec" 2>/dev/null; then
     tmux kill-window -t "$TMUX_WINDOW" 2>/dev/null || true
   fi
   ```
   No-op silently if the session or window is absent.

5. **Remove worktree**:
   ```bash
   if [ -d "$WORKTREE_PATH" ]; then
     if [ "$FORCE" = "true" ]; then
       git worktree remove --force "$WORKTREE_PATH"
     else
       git worktree remove "$WORKTREE_PATH"
     fi
   fi
   git worktree prune
   ```

6. **Delete local branch(es)**:
   ```bash
   for BRANCH in $(git branch --list "$BRANCH_PATTERN" --format='%(refname:short)'); do
     if [ "$FORCE" = "true" ]; then
       git branch -D "$BRANCH"
     else
       # -d refuses if not merged; second safety net even though step 3b already checked.
       git branch -d "$BRANCH"
     fi
   done
   ```

7. **Report**: Print a one-line summary of what was removed (tmux window, worktree path, branch names) and what was skipped because it didn't exist.

## Notes

- This skill is destructive. Verify the PR is merged before running without `--force`.
- Does NOT touch remote branches. Use `gh pr merge --delete-branch` or `git push origin --delete <branch>` separately if needed.
- Does NOT touch `.itx/<N>/` planning artifacts — those are committed history and are intentionally preserved.
- Does NOT close the GitHub issue. Issue state is managed by `/itx:execute` and PR merge.

## Prompt Logging

**REQUIRED**: Append prompt log to `.itx/<N>/02_VERIFY.md` (cleanup is a verification-phase action).

See [AGENTS.md](../../../AGENTS.md#prompt-logging-standard) for format specification.
