---
description: Run full AI code review cycle on current PR branch
---

# Rami Code Review

Automated code review cycle: find PR, get issues, fix or rebutt, repeat until clean.

**IMPORTANT:** All Rami operations MUST use the `mcp__rami__*` MCP tools. Do NOT use gh CLI or direct API calls for review operations.

## Additional Instructions

$ARGUMENTS

---

## Execution

### Phase 1: Find PR

**If PR URL provided in arguments:** Skip to Phase 2.

**Otherwise:** Get git info locally and find PR:
```bash
git remote get-url origin
git branch --show-current
```

Then call the MCP tool:
```
mcp__rami__get_current_branch_pr(remote_url, branch)
```

| Result | Action |
|--------|--------|
| `status: success` | Continue to Phase 2 with `pr_url` |
| `status: not_found` | Stop. Tell user: "No PR found. Push branch and create PR first." |
| `status: error` | Stop. Report error. |

### Phase 2: Get Review

```
mcp__rami__get_review_results(pr_url)
```

This is a blocking call. Wait for completion.

| Result | Action |
|--------|--------|
| `issue_count == 0` | Done. Report: "No issues found." |
| `issue_count > 0` | Continue to Phase 3 |

### Phase 3: Fix Issues

Process issues in priority order: **Blocking → High → Medium → Low**

For each issue:

1. Get fix instructions:
   ```
   mcp__rami__get_fix_prompt(pr_url, issue_index)
   ```

2. Implement the fix in code

3. Run project tests and lint (if available):
   ```bash
   make test && make lint
   # or: npm test && npm run lint
   # or: go test ./... && golangci-lint run
   ```

4. If issue is a false positive, rebutt using the MCP tool:
   ```
   mcp__rami__rebutt(pr_url, issue_index, author_reply="<specific evidence>")
   ```
   - `verdict: valid` → Issue dismissed, continue to next issue
   - `verdict: invalid` → Must fix the issue
   - `verdict: partial` → Fix the valid part

After all issues addressed:

```bash
git add -A
git commit -m "fix: address rami review feedback"
git push
```

### Phase 4: Re-check

Return to Phase 2. Repeat until:

- `issue_count == 0` (clean review)
- Max 5 iterations reached (prevent infinite loops)
- User interrupts

### Phase 5: Report

Summarize:
- Total iterations
- Issues fixed
- Issues rebutted (with verdicts)
- Final status

---

## MCP Tool Reference

All tools are accessed via the `mcp__rami__` prefix:

| MCP Tool | Parameters | Purpose |
|----------|------------|---------|
| `mcp__rami__get_current_branch_pr` | `remote_url`, `branch` | Find PR URL from git remote and branch |
| `mcp__rami__get_review_results` | `pr_url` | Trigger/retrieve code review (blocking) |
| `mcp__rami__get_fix_prompt` | `pr_url`, `issue_index` | Get detailed fix instructions |
| `mcp__rami__rebutt` | `pr_url`, `issue_index`, `author_reply` | Challenge a finding with evidence |
| `mcp__rami__login` | `base_url` (optional) | Authenticate via GitHub device flow |

---

## When to Rebutt

Use `mcp__rami__rebutt` only with concrete evidence:
- False positive (code is actually safe)
- Framework guarantees handle the concern
- Intentional design decision with clear justification
- Duplicate of another issue

Do NOT rebutt:
- To avoid work
- Style preferences (just fix them)
- When unsure
