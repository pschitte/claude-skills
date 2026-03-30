---
name: ship
description: "Rebase, push, create PR (or use existing), wait for relevant checks, fix issues, and merge with --admin rebase into master. Use when the user wants to ship/merge a branch."
argument-hint: "[path-to-repo-or-branch-info]"
---

Ship the current branch to master on the Pinata-Consulting/ascenium repository. Follow these steps precisely and move as fast as possible. Run independent operations in parallel.

## Environment setup

All git commands in the ascenium repo must use:
```
PATH="/usr/lib/git-core:$PATH" GIT_CONFIG_NOSYSTEM=1 git ...
```
This is required because the system git is a Guix-installed version that lacks `git-remote-https`. The `/usr/lib/git-core` path provides the working version.

For `gh` commands, use `gh` directly (it works via snap).

## Step 1: Rebase on master

```bash
git fetch origin master
git rebase origin/master
```

If the rebase has conflicts, resolve them and continue.

## Step 2: Discover and apply commit message rules

The ascenium repo enforces commit format via CI. The rules are defined in `workflow/src/git_check.py` in the repo itself — **always read this file fresh** before validating commits, because the format rules may change over time.

Read the file and extract:
- The regexes for valid commit message headers (single-line and multiline)
- Any special rules for LLVM commits (files under `llvm/`)
- Email requirements for commit authors
- Filename conventions
- Line length limits
- Any whitelists or exceptions

Then check ALL commits from `origin/master..HEAD` against the discovered rules. If any commits don't conform, fix them using `GIT_SEQUENCE_EDITOR` with `git rebase` to reword non-interactively. The `Co-Authored-By` trailer is fine to keep.

If the author email doesn't match the required pattern, fix it with `git config user.email` before recommitting.

## Step 3: Force push

```bash
git push --force origin <branch-name>
```

If the branch doesn't exist on the remote yet, push with `-u` to create it.

## Step 4: Create or find PR

Check if a PR already exists for this branch:
```bash
gh pr list --repo Pinata-Consulting/ascenium --head <branch-name> --json number,url --jq '.[0]'
```

If no PR exists, create one:
```bash
gh pr create --repo Pinata-Consulting/ascenium --base master --head <branch-name> \
  --title "<short title under 70 chars>" \
  --body "$(cat <<'EOF'
## Summary
<1-3 bullet points>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

## Step 5: Wait for relevant checks and analyze results

Poll checks with `gh pr checks <number> --repo Pinata-Consulting/ascenium` every 30-60 seconds.

### Determine which checks are relevant

Get the list of changed files (`git diff origin/master...HEAD --name-only`) and map them to affected checks. Use common sense:

- **Git check** is ALWAYS relevant (it checks commit message format)
- **Docs build** is relevant if docs/, .rst, or documentation files changed
- **Validate GitHub workflows** is relevant if `.github/workflows/` changed
- **Toolchain build and test** is relevant if hardware/, llvm/, aptos-sim/, tests/, musl/, or benchmark/ code changed
- **Count cycles**, **tile-kpis**, **coremark**, **spec-2017** checks are only relevant if hardware/simulation code changed
- For review-only or plan-only changes, most build checks are irrelevant

### When checks finish

- If ALL relevant checks pass → proceed to merge
- If a relevant check fails → analyze the failure:
  1. Read the check logs: `gh run view <run-id> --repo Pinata-Consulting/ascenium --log-failed`
  2. If it's a git-check failure (commit message format), re-read `workflow/src/git_check.py`, fix the commit messages, force push, and wait again
  3. If it's a build/test failure caused by the changes, attempt to fix it, commit, force push
  4. If it's a flaky test or infrastructure issue unrelated to the changes, note it and proceed with admin merge
  5. Tell the user what you found and what you changed

## Step 6: Merge with admin rebase

```bash
gh pr merge <number> --repo Pinata-Consulting/ascenium --rebase --admin
```

Verify the merge:
```bash
gh pr view <number> --repo Pinata-Consulting/ascenium --json state,mergedAt
```

## Step 7: Report

Tell the user:
- PR URL
- Which checks passed/failed
- Any fixes you made
- Final merge status

$ARGUMENTS
