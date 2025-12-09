---
description: Quick check of PR review status without triggering fixes
---

# Rami Status Check

Get current review status for your PR branch. Does not trigger fix loop.

## Arguments

$ARGUMENTS

---

## Process

1. **Find PR**

```bash
REMOTE=$(git remote get-url origin)
BRANCH=$(git branch --show-current)
```

Then: `mcp__plugin_rami-code-review_rami__get_current_branch_pr(remote_url=$REMOTE, branch=$BRANCH)`

| Result | Action |
|--------|--------|
| `status: success` | Continue with PR URL |
| `status: not_found` | Report: "No PR found for branch. Create a PR first." |
| `status: error` | Report error |

2. **Get Review Status**

```
mcp__plugin_rami-code-review_rami__get_review_status(pr_url)
```

Note: Uses `get_review_status` (not `get_review_results`) to avoid triggering a new review.

3. **Report**

| Status | Report |
|--------|--------|
| `not_found` | "No review exists. Run `/rami` to trigger a review." |
| `completed` with `issue_count == 0` | "Clean review. No issues found." |
| `completed` with `issue_count > 0` | List issues by severity (blocking → high → medium → low) |
| `in_progress` / `queued` | "Review in progress. Check again shortly." |
| `failed` | Report error message |

Include `pending_history_count` if > 0: "Also {N} unresolved issues from previous reviews."
