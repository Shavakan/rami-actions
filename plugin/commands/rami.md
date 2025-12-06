---
description: Run full AI code review cycle on current PR branch
---

# Rami Code Review

Fix or rebutt all review issues until clean. Max 5 iterations.

## Constraints

- Use Rami MCP tools for all review operations (get issues, get fix prompts, rebutt)
- Use `gh` CLI only for PR metadata (branch name lookup)
- Rebutt only with concrete evidence (false positive, framework guarantees, intentional design, duplicate)
- Never rebutt to avoid work or for style preferences

## Arguments

$ARGUMENTS

---

## Phase 1: Setup

**With PR URL argument:**
```bash
BRANCH=$(gh pr view <PR_URL> --json headRefName -q '.headRefName')
git fetch origin "$BRANCH" && git checkout "$BRANCH" && git pull origin "$BRANCH"
```

**Without PR URL:**
```bash
REMOTE=$(git remote get-url origin)
BRANCH=$(git branch --show-current)
```
Then: `mcp__plugin_rami-code-review_rami__get_current_branch_pr(remote_url=$REMOTE, branch=$BRANCH)`

| Result | Action |
|--------|--------|
| `status: success` | Use returned `pr_url` |
| `status: not_found` | Stop: "No PR found. Push branch and create PR first." |
| `status: error` | Stop: Report error |

---

## Phase 2: Review Loop

```
mcp__plugin_rami-code-review_rami__get_review_results(pr_url)
```

Exit if `issue_count == 0`.

### For each issue (Blocking → High → Medium → Low):

1. `mcp__plugin_rami-code-review_rami__get_fix_prompt(pr_url, issue_index)` → Get instructions
2. Implement fix OR rebutt with evidence:
   ```
   mcp__plugin_rami-code-review_rami__rebutt(pr_url, issue_index, author_reply="<evidence>")
   ```
   - `verdict: valid` → Dismissed
   - `verdict: invalid` → Must fix
   - `verdict: partial` → Fix valid part
3. Run tests if project has them (check for Makefile, package.json, go.mod)

### After all issues:
```bash
git add -A && git commit -m "fix: address rami review feedback" && git push
```

Repeat Phase 2. Stop at 5 iterations or clean review.

---

## Phase 3: Report

Summarize: iterations, issues fixed, rebuttals (with verdicts), final status.
