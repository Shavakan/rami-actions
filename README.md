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
/plugin install mcp-rami@shavakan
/plugin install rami-commands@shavakan
```

### Components

| Plugin | Type | Description |
|--------|------|-------------|
| `mcp-rami` | MCP Server | AI code review tools |
| `rami-commands` | Commands | `/rami` slash command |

### /rami Command

Run a full code review cycle on your current PR branch:

```
/rami
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
| `get_current_branch_pr` | Find PR URL from current git branch |
| `get_review_results` | Trigger or retrieve code review |
| `get_fix_prompt` | Get detailed fix instructions |
| `rebutt` | Challenge a finding with evidence |
