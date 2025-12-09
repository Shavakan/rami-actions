# rami-actions

GitHub Action for AI code reviews using [Rami](https://github.com/shavakan/rami-code-review).

## Usage

```yaml
name: Rami Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  pull-requests: write
  contents: read

jobs:
  review:
    if: github.event.pull_request.draft != true
    runs-on: ubuntu-latest
    steps:
      - uses: Shavakan/rami-actions@v1
        with:
          api_endpoint: ${{ secrets.RAMI_API_ENDPOINT }}
```

**Required**: Add `permissions` block or API returns 500 errors.

## Inputs

| Input | Required |
|-------|----------|
| `api_endpoint` | Yes |
| `openrouter_api_key` | No |

## Outputs

- `issues_found`: Number of issues found

---

## Claude Code Plugin

This repository also serves as a Claude Code marketplace plugin.

### Installation

```bash
/plugin marketplace add Shavakan/rami-actions
/plugin install rami-code-review@rami
```

### /rami-code-review:rami Command

Run a full code review cycle on your current PR branch:

```
/rami-code-review:rami
```

This will:
1. Find the PR for your current branch
2. Trigger AI code review
3. For each issue: get fix instructions and implement
4. Commit and push fixes
5. Re-check until clean (max 5 iterations)

### MCP Tools

| Tool | Description |
|------|-------------|
| `login` | Authenticate via GitHub device flow (required on first use) |
| `get_current_branch_pr` | Find PR URL from current git branch |
| `get_review_results` | Trigger or retrieve code review (triggers new review if none exists) |
| `get_review_status` | Check review status without triggering new review |
| `get_fix_prompt` | Get detailed fix instructions |
| `rebutt` | Challenge a finding with evidence |
| `invalidate_cache` | Clear cached review results (enterprise tier only) |

### Authentication

On first use, run the `login` tool. It initiates GitHub device flow:

1. You'll receive a `user_code` and `verification_uri`
2. Visit the URL and enter the code
3. Authorize the Rami app
4. The tool completes and returns a session token

Session tokens are cached automatically. Re-authentication is only needed when tokens expire.

### Troubleshooting

**"Not authenticated" errors**
- Run `/rami` - it handles auth automatically in Phase 0

**"No PR found for branch"**
- Push your branch and create a PR first

**Review stuck in "queued" state**
- LLM is processing. Retry after `retry_after_seconds` from the response

**"Rate limited by GitHub"**
- Consider using a GitHub App instead of PAT for higher limits
