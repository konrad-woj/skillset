# VCS Integration Guide (GitLab & GitHub)

## Detecting the Platform

Before fetching MR/PR data, determine which platform the repo uses:

```bash
# Check remote URL
git remote get-url origin
```

- `gitlab.com` or self-hosted GitLab → use `glab`
- `github.com` → use `gh`

Check CLI is installed and authenticated:
```bash
glab auth status   # GitLab
gh auth status     # GitHub
```

---

## Fetching MR/PR Information

### GitLab (glab)

```bash
# View MR details
glab mr view <mr-number>

# Get diff
glab mr diff <mr-number>

# Get comments/notes (URL-encode the project path, e.g. org%2Frepo)
glab api projects/<project-path-encoded>/merge_requests/<mr-number>/notes

# Human-readable comments
glab api projects/<project-path-encoded>/merge_requests/<mr-number>/notes \
  | jq '.[] | {author: .author.name, body: .body, created: .created_at}'

# Get SHA refs for line-level comments
glab api projects/<project-path-encoded>/merge_requests/<mr-number> \
  | jq '{base_sha: .diff_refs.base_sha, head_sha: .diff_refs.head_sha, start_sha: .diff_refs.start_sha}'

# List your assigned MRs
glab mr list --assignee=@me
```

### GitHub (gh)

```bash
# View PR details
gh pr view <pr-number>

# Get diff
gh pr diff <pr-number>

# Get review comments (inline, on code)
gh api repos/<owner>/<repo>/pulls/<pr-number>/comments

# Get general PR comments (issue-style)
gh api repos/<owner>/<repo>/issues/<pr-number>/comments

# Human-readable comments
gh api repos/<owner>/<repo>/pulls/<pr-number>/comments \
  | jq '.[] | {author: .user.login, body: .body, path: .path, line: .line}'

# Get commit SHAs
gh api repos/<owner>/<repo>/pulls/<pr-number> \
  | jq '{base_sha: .base.sha, head_sha: .head.sha}'

# List your assigned PRs
gh pr list --assignee=@me
```

---

## Review Workflow

### Step 1: Identify Target

```bash
# GitLab
glab mr view <mr-number>

# GitHub
gh pr view <pr-number>
```

### Step 2: Fetch Full Context

Gather MR/PR metadata, diff, and existing comments in sequence:

```bash
# GitLab
glab mr view <mr-number>
glab mr diff <mr-number>
glab api projects/<project-path-encoded>/merge_requests/<mr-number>/notes

# GitHub
gh pr view <pr-number>
gh pr diff <pr-number>
gh api repos/<owner>/<repo>/pulls/<pr-number>/comments
gh api repos/<owner>/<repo>/issues/<pr-number>/comments
```

### Step 3: Analyze Comments

Parse existing comments to understand:
- What has already been flagged by other reviewers (don't duplicate)
- Responses to your previous comments — has the author addressed them?
- Unresolved threads that still need attention

### Step 4: Review Code

Apply the standard code review process from the main SKILL.md. Focus on:
- Changes not already covered by other reviewers
- New issues introduced in recent pushes
- Follow-ups to your previous comments

### Step 5: Summarize Findings

```markdown
## MR/PR Review: <title> (!<number> / #<number>)

### Summary
- **Platform**: GitLab / GitHub
- **Status**: opened / changes_requested / etc.
- **Your previous comments**: <count>
- **Unresolved discussions**: <count>

### Review Findings
#### 🔴 Critical Issues
#### 🟡 Important Issues
#### 💡 Suggestions
#### ✅ Responses to Your Comments

### Recommendation
Approve / Request Changes / Comment only
```

---

## Understanding Discussion Threads

### GitLab
- Notes can be top-level (general) or on specific diff lines
- Check `resolved` field — unresolved threads block merge
- System notes (assignments, pushes) appear alongside user comments — filter them out

### GitHub
- Pull request review comments are attached to specific diff lines (`/pulls/<n>/comments`)
- Issue comments are general notes (`/issues/<n>/comments`)
- Reviews have a state: `APPROVED`, `CHANGES_REQUESTED`, `COMMENTED`
- Check `state` on review objects to understand current approval status

---

## Error Handling

| Situation | Fix |
|---|---|
| `glab` not installed | `brew install glab` or see gitlab.com/gitlab-org/cli |
| `gh` not installed | `brew install gh` or see cli.github.com |
| Auth failure | `glab auth login` / `gh auth login` |
| MR/PR not found | Verify number and that you have repo access |
| Rate limited | Wait and retry; use PAT with higher limits |

---

## Tips

1. **Context first**: Read MR/PR description and existing comments before the diff
2. **Avoid duplication**: Don't flag issues already raised by others
3. **Focus on gaps**: Review areas not covered by other reviewers
4. **Track your history**: Reference your previous comments when following up
5. **Be constructive**: Acknowledge good changes, provide specific fixes