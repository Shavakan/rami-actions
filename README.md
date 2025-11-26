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
