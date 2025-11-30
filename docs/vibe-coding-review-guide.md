# Automated Code Review for Vibe Coding

A practical guide to integrating AI code review into fast-paced, AI-assisted development workflows.

## The Problem

Vibe coding—rapid development with AI assistance—trades velocity for oversight. You ship faster but blind spots multiply:
- Security vulnerabilities slip through
- Edge cases get missed
- Technical debt accumulates silently
- LLM-generated code often looks correct but isn't

Automated code review closes this gap without slowing you down.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Your AI Agent                           │
│              (Claude Code, Cursor, Windsurf)                │
└─────────────────────┬───────────────────────────────────────┘
                      │ MCP Protocol
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                     Rami MCP Server                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │get_review_   │  │get_fix_      │  │rebutt        │      │
│  │results       │  │prompt        │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────┬───────────────────────────────────────┘
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│   GitHub API    │     │   LLM Analysis  │
│   (PR diffs)    │     │   (OpenRouter)  │
└─────────────────┘     └─────────────────┘
```

## Setup

### 1. Install Rami

```bash
# From source
git clone https://github.com/Shavakan/rami-code-review
cd rami-code-review
make build
sudo mv dist/rami /usr/local/bin/

# Or use the hosted endpoint (no install needed)
```

### 2. Configure Environment

```bash
# Required
export GITHUB_TOKEN="ghp_..."          # GitHub PAT with repo scope
export OPENROUTER_API_KEY="sk-or-..." # LLM API key

# Optional - for caching (recommended for teams)
export VALKEY_URL="redis://localhost:6379"
export CACHE_TTL_DAYS="7"

# Optional - stronger model for rebuttals
export LLM_REBUTT_MODEL="anthropic/claude-opus-4.5"
```

### 3. Configure Your AI Agent

Add to your agent's MCP configuration:

**Claude Code** (`~/.claude.json`):
```json
{
  "mcpServers": {
    "rami": {
      "command": "rami",
      "args": ["mcp", "--transport", "stdio"]
    }
  }
}
```

**Or use hosted endpoint** (`.mcp.json` in project root):
```json
{
  "mcpServers": {
    "rami": {
      "type": "http",
      "url": "https://rami.shavakan.com/mcp"
    }
  }
}
```

---

## Agent Self-Setup

For LLM agents bootstrapping this pipeline:

**1. Test:** Call `get_current_branch_pr()`. If it returns data or `not_found`, skip to step 3.

**2. Request human config** (tools not available):
```
Create <project>/.mcp.json:
{"mcpServers":{"rami":{"type":"http","url":"https://rami.shavakan.com/mcp"}}}

Then restart agent session.
```

**3. Write to `<project>/.claude/CLAUDE.md`** (or `.cursorrules` for Cursor):
```markdown
## Code Review [MANDATORY]

After `git push` to PR branch:
1. `get_current_branch_pr()` → get PR URL (stop if `not_found`)
2. `get_review_results(pr_url)` → get issues
3. For each issue: `get_fix_prompt(pr_url, issue_index)` → fix → test → commit
4. Push → repeat from step 2 until `issue_count == 0` or 5 iterations

Rebutt false positives: `rebutt(pr_url, issue_index, author_reply="<evidence>")`
```

**4. Verify:** `get_current_branch_pr()` succeeds. Done.

---

## Workflows

### For Human Users

#### Quick Check (After Push)
```
You: "check my PR for issues"

Agent runs:
1. get_current_branch_pr → finds PR URL
2. get_review_results → triggers/retrieves review
3. Reports findings
```

#### Fix Loop
```
You: "fix the blocking issues"

Agent runs:
1. get_fix_prompt(issue_index=0) → gets detailed instructions
2. Implements fix
3. Commits and pushes
4. get_review_results → re-checks
5. Repeats until clean
```

#### Disagree with Finding
```
You: "that SQL injection finding is wrong, we're using parameterized queries"

Agent runs:
1. rebutt(issue_index=X, author_reply="We use parameterized queries via...")
2. Returns verdict (valid/invalid/partial)
3. Suggests professional response if posting to GitHub
```

### Trigger Phrases for Humans

| Say this | Agent does |
|----------|------------|
| "review my PR" | Full code review |
| "any issues?" | Check review status |
| "fix review issues" | Fix loop until clean |
| "that's a false positive because..." | Rebuttal flow |
| "get fix for issue 3" | Detailed fix prompt |

---

## Agent Instructions

Add these to your project's agent configuration (e.g., `CLAUDE.md`, `.cursorrules`).

### Minimal Configuration

```markdown
## Code Review

After every `git push` to a PR branch:
1. `get_current_branch_pr` → find PR URL
2. `get_review_results` → get issues (blocking call)
3. If issues exist: use `get_fix_prompt` and fix them
4. Repeat until `issue_count == 0`
```

### Full Configuration (Recommended)

```markdown
## Post-Push Code Review [MANDATORY]

After EVERY `git push` to a PR branch, this workflow runs automatically.

### Process

1. **Find PR**
   ```
   get_current_branch_pr(working_dir)
   ```
   - `not_found` → No PR exists. Inform user to create PR. Skip workflow.
   - `error` → Log error, skip workflow.

2. **Get Review**
   ```
   get_review_results(pr_url)
   ```
   - Blocking call. Waits for review to complete.
   - Returns issue list with severity levels.

3. **Handle Issues**

   If `issue_count > 0`:
   - Process in priority order: blocking → high → medium → low
   - For each issue:
     - **Fix**: Use `get_fix_prompt(pr_url, issue_index)` for instructions
     - **Or Rebutt**: If finding is incorrect, use `rebutt(pr_url, issue_index, author_reply)`
   - Run tests and lint after fixes
   - Commit: `fix: address review feedback`
   - Push and return to step 2

4. **Exit Conditions**
   - `issue_count == 0` (clean)
   - All issues resolved via fix or valid rebuttal
   - Max 5 iterations (prevent infinite loops)
   - User interrupts

### Rebuttal Guidelines

Use `rebutt` when:
- Finding is a false positive (code is actually safe)
- Context makes the concern inapplicable
- Trade-off was intentional and documented

Provide specific evidence in `author_reply`:
- Reference existing safeguards
- Cite framework guarantees
- Point to related code that handles the concern

Do NOT rebutt:
- Style preferences (just fix them)
- Things you're unsure about
- To avoid work
```

### Pre-Commit Gate (Optional)

```markdown
## Pre-Commit Review [BLOCKING]

Before committing, this gate must pass:

1. **Tests exist?** If changes lack tests → write them first
2. **Run validation**: `make test && make lint`
3. **All pass?** → Proceed to commit
4. **Failures?** → Fix and retry

Only after this gate passes can you commit.
```

---

## MCP Tool Reference

### get_current_branch_pr

Find the PR for current git branch.

```
Input:  { working_dir?: string }
Output: { status, pr_url, number, title, branch, author, draft, owner, repo }
```

### get_review_results

Trigger or retrieve code review. Blocking—waits for completion.

```
Input:  { pr_url: string }
Output: {
  status: "completed" | "in_progress" | "queued" | "failed",
  issue_count: number,
  issues: [{ severity, file, line, problem, risk, fix }],
  retry_after_seconds?: number
}
```

### get_fix_prompt

Get detailed fix instructions for a specific issue.

```
Input:  { pr_url: string, issue_index: number }
Output: { prompt: string }  // Contains problem, risk, steps, suggested code
```

### rebutt

Challenge a review finding with a counter-argument.

```
Input:  {
  pr_url: string,
  author_reply: string,
  issue_index?: number,   // For internal findings
  comment_id?: number     // For GitHub comments
}
Output: {
  verdict: "valid" | "invalid" | "partial",
  reasoning: string,
  suggestion: string,
  response: string  // Professional reply to post
}
```

---

## Severity Levels

| Level | Meaning | Action |
|-------|---------|--------|
| **Blocking** | Security vulnerability, data loss risk, crash | Must fix before merge |
| **High** | Correctness bug, significant performance issue | Should fix |
| **Medium** | Edge case handling, minor performance | Fix if time permits |
| **Low** | Code hygiene, minor improvements | Optional |

---

## Best Practices

### 1. Don't Skip the Loop

The temptation: "It's just a small change, I'll push without review."

The reality: Small changes cause big outages. Run the review.

### 2. Rebutt Thoughtfully

Bad rebuttal:
> "This is fine, I know what I'm doing"

Good rebuttal:
> "This SQL query uses parameterized binding via `db.Query($1, userInput)`. The driver escapes all parameters. See Go database/sql documentation."

### 3. Fix Blocking Issues First

Don't get distracted by medium/low issues when blocking issues exist. Priority order matters.

### 4. Cache Your Reviews

With Valkey/Redis caching:
- Repeated reviews of unchanged code return instantly
- 60-80% token savings
- Team members share cached results

### 5. Use Pre-Commit Gates

Catch issues before they hit the PR. Faster feedback, less context switching.

---

## Troubleshooting

### "Review stuck in queued state"

The LLM is processing. Wait for `retry_after_seconds` and retry.

### "No PR found for branch"

Push your branch and create a PR first. The tool needs a PR to review.

### "Rate limited by GitHub"

Use a GitHub App instead of PAT for higher rate limits.

### "Review finds too many issues"

1. Focus on blocking/high first
2. Consider if your codebase needs architectural fixes
3. Add `.rami-ignore` patterns for generated code

---

## Integration Examples

### GitHub Actions

```yaml
name: Rami Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: Shavakan/rami-actions@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          openrouter-api-key: ${{ secrets.OPENROUTER_API_KEY }}
```

### Git Hook (pre-push)

```bash
#!/bin/bash
# .git/hooks/pre-push

PR_URL=$(rami pr-url 2>/dev/null)
if [ -n "$PR_URL" ]; then
  echo "Running code review..."
  rami review "$PR_URL" --exit-on-blocking
fi
```

---

## Summary

1. **Install**: Rami CLI or use hosted endpoint
2. **Configure**: Add MCP server to your agent
3. **Add instructions**: Copy agent instructions to your project
4. **Push code**: Review runs automatically
5. **Fix or rebutt**: Handle findings systematically
6. **Repeat**: Until clean

The goal isn't perfection—it's catching the issues that matter before they reach production.
