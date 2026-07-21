# Interactive Comments Workflow (GitLab & GitHub)

## Overview

Post review comments to GitLab MRs or GitHub PRs with user approval at each step. Never post without explicit confirmation.

---

## Workflow Steps

### 1. Review Code and Identify Issues

Perform standard code review (see SKILL.md). Categorize findings:
- 🔴 Critical (blocking)
- 🟡 Important (should fix)
- 💡 Suggestions (nice-to-have)

### 2. Present Review Summary and Wait for Confirmation

```markdown
## Review Summary for MR/PR !<number> / #<number>

Found X total issues:
- 🔴 Y critical issues
- 🟡 Z important issues
- 💡 W suggestions

### 🔴 Critical Issues
1. [Brief description] (file.py:123)

### 🟡 Important Issues
1. [Brief description] (file.py:456)

Would you like me to prepare comments for these issues?
```

Wait for user confirmation before proceeding.

### 3. Prepare Comment Drafts

Write draft files to `.review-comments/mr-<number>/` (use `mr-` prefix for both platforms):

**File naming**: `.review-comments/mr-<number>/01-critical.md`, `02-important.md`, etc.

**Comment file format**:
```markdown
<!-- META: {"file": "path/to/file.py", "line": 123, "severity": "critical", "status": "pending"} -->

[Detailed explanation of the issue]

**Why it matters**: [Impact explanation]

**Suggested fix**:
```python
# Code example
```
```

**metadata.json** (fetch SHA hashes before drafting line comments):
```json
{
  "platform": "gitlab|github",
  "mr_number": 123,
  "base_sha": "abc123...",
  "head_sha": "def456...",
  "start_sha": "abc123...",
  "created_at": "2026-01-13T14:30:00Z"
}
```

Fetch SHAs:
```bash
# GitLab
glab api projects/<project-path-encoded>/merge_requests/<mr-number> \
  | jq '{base_sha: .diff_refs.base_sha, head_sha: .diff_refs.head_sha, start_sha: .diff_refs.start_sha}'

# GitHub
gh api repos/<owner>/<repo>/pulls/<pr-number> \
  | jq '{base_sha: .base.sha, head_sha: .head.sha}'
```

### 4. Interactive Comment Approval

For each comment, show:
```
Comment X of Y — [SEVERITY]
File: path/to/file.py:123

--- Preview ---
[Full comment text]
---

[A]dd / [E]dit / [S]kip / [Q]uit?
```

Handle responses:
- **A**: Mark for posting, move to next
- **E**: Let user modify text, then mark for posting
- **S**: Discard, move to next
- **Q**: Stop, save progress

### 5. Final Confirmation

```markdown
## Comments Ready to Post

Approved X comments:
1. ✓ file.py:123 — [Critical] Brief description
2. ✓ file.py:456 — [Important] Brief description

Skipped: Y comments

[P]ost all / [R]eview again / [C]ancel
```

### 6. Post Comments

Only post after the user types "P".

---

## Posting Commands

### GitLab

**Line-level comment** (requires SHA refs):
```bash
glab api -X POST projects/<project-path-encoded>/merge_requests/<mr-number>/discussions \
  -f "body=<comment-text>" \
  -f "position[base_sha]=<base-sha>" \
  -f "position[head_sha]=<head-sha>" \
  -f "position[start_sha]=<start-sha>" \
  -f "position[position_type]=text" \
  -f "position[new_path]=<file-path>" \
  -f "position[new_line]=<line-number>"
```

**General MR comment**:
```bash
glab api -X POST projects/<project-path-encoded>/merge_requests/<mr-number>/notes \
  -f "body=<comment-text>"
```

### GitHub

**Line-level review comment** (requires commit SHA and diff position):
```bash
gh api -X POST repos/<owner>/<repo>/pulls/<pr-number>/comments \
  -f "body=<comment-text>" \
  -f "commit_id=<head-sha>" \
  -f "path=<file-path>" \
  -f "line=<line-number>" \
  -f "side=RIGHT"
```

**General PR comment**:
```bash
gh api -X POST repos/<owner>/<repo>/issues/<pr-number>/comments \
  -f "body=<comment-text>"
```

**Submit a review** (batch line comments + overall verdict):
```bash
# Create review in PENDING state, then submit
gh api -X POST repos/<owner>/<repo>/pulls/<pr-number>/reviews \
  -f "commit_id=<head-sha>" \
  -f "event=COMMENT|APPROVE|REQUEST_CHANGES" \
  -f "body=<overall-comment>" \
  --input <(echo '{"comments": [...]}')
```

---

### 7. Confirm Success

```markdown
## Comments Posted

Posted X comments to MR/PR !<number>:
✓ file.py:123 — [Critical] Brief description
✓ file.py:456 — [Important] Brief description

View at: <MR/PR URL>
```

---

## Error Handling

| Situation | Action |
|---|---|
| Comment fails to post | Save to file, notify user, continue with rest |
| Network failure | Save all pending to `.review-comments/`, provide recovery path |
| Permission denied | Save all to disk, notify user |
| Line number mismatch (GitHub) | Fall back to general PR comment, note the file:line in the body |

---

## Best Practices

1. **Group related issues** — combine closely related findings into one comment when it reads better
2. **Always include file + line** — make it easy to navigate to the issue
3. **Explain why** — impact matters more than just naming the problem
4. **Suggest fixes** — include a concrete code example
5. **Acknowledge good code** — not every comment has to be a problem
6. **Prioritize** — critical issues first
7. **Markdown works everywhere** — GitLab and GitHub both render full Markdown in comments