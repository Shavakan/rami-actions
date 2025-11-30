# Rami MCP Server

AI-powered code review for GitHub pull requests.

## Tools

| Tool | Description |
|------|-------------|
| `get_current_branch_pr` | Find PR URL from current git branch |
| `get_review_results` | Trigger or retrieve code review (blocking) |
| `get_fix_prompt` | Get detailed fix instructions for an issue |
| `rebutt` | Challenge a review finding with evidence |

## Usage

```bash
# Find PR for current branch
get_current_branch_pr()

# Get review results
get_review_results(pr_url="https://github.com/owner/repo/pull/123")

# Get fix instructions for issue 0
get_fix_prompt(pr_url="...", issue_index=0)

# Rebutt a finding
rebutt(pr_url="...", issue_index=0, author_reply="This is safe because...")
```

## Endpoints

- **Hosted**: `https://rami.shavakan.com/mcp`
- **Local**: `rami mcp --transport stdio`

## Environment Variables (Local Only)

| Variable | Required | Description |
|----------|----------|-------------|
| `GITHUB_TOKEN` | Yes | GitHub PAT with repo scope |
| `OPENROUTER_API_KEY` | Yes | LLM API key |
| `VALKEY_URL` | No | Redis-compatible cache URL |
