---
description: Run full AI code review cycle on current PR branch
---

# Rami Code Review

Automated code review cycle: find PR, get issues, fix or rebutt, repeat until clean.

## Additional Instructions

$ARGUMENTS

---

## Execution

### Phase 1: Find PR

```
get_current_branch_pr()
```

| Result | Action |
|--------|--------|
| `status: found` | Continue to Phase 2 |
| `status: not_found` | Stop. Tell user: "No PR found. Push branch and create PR first." |
| `status: error` | Stop. Report error. |

### Phase 2: Get Review

```
get_review_results(pr_url)
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
   get_fix_prompt(pr_url, issue_index)
   ```

2. Implement the fix in code

3. Run project tests and lint (if available):
   ```bash
   make test && make lint
   # or: npm test && npm run lint
   # or: go test ./... && golangci-lint run
   ```

4. If user disagrees with a finding, rebutt with evidence:
   ```
   rebutt(pr_url, issue_index, author_reply="<specific evidence>")
   ```
   - `verdict: valid` → Issue dismissed, continue
   - `verdict: invalid` → Must fix
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
- Issues rebutted
- Final status

---

## Tool Reference

| Tool | Purpose |
|------|---------|
| `get_current_branch_pr()` | Find PR URL from git branch |
| `get_review_results(pr_url)` | Trigger/retrieve code review |
| `get_fix_prompt(pr_url, issue_index)` | Get detailed fix instructions |
| `rebutt(pr_url, issue_index, author_reply)` | Challenge a finding |

---

## When to Rebutt

Use rebuttal only with concrete evidence:
- False positive (code is actually safe)
- Framework guarantees handle the concern
- Intentional trade-off with documentation

Do NOT rebutt:
- To avoid work
- Style preferences (just fix them)
- When unsure
