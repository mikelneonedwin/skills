---
name: gh-report
description: Generates casual, human-readable daily reports from GitHub activity — commits, PRs, reviews, and collaboration. Use when the user asks for a daily standup, daily report, daily summary, "what did I do today/yesterday", a status update, or wants to turn their Git history into a narrative.
---

# Daily Report

Turns your GitHub activity into a casual daily check-in that actually sounds like you wrote it — not like someone dumped a commit log into a document.

## First, round up everything you touched

Figure out the date first (today or yesterday). Then grab all your stuff.

**Commits:**

```bash
gh search commits \
  --author="$(gh api user --jq .login)" \
  --author-date=">=<date>" \
  --author-date="<=<date>" \
  --sort=author-date --order=desc \
  --json repository,sha,commit,url
```

Dig into each one to see what actually changed:

```bash
gh api repos/{owner}/{repo}/commits/{sha} --jq '.commit.message, .files[] | {path, status, additions, deletions}'
```

**PRs you opened:**

```bash
gh search prs --author=@me --updated=">=<date>" --sort=updated --order=desc \
  --json title,number,state,url,repository,updatedAt,createdAt
```

**PRs you reviewed** (easy to forget this one):

```bash
gh search prs --review-requested=@me --updated=">=<date>" --sort=updated --order=desc \
  --json title,number,state,url,repository,updatedAt,createdAt
```

**Spy on each PR:**

```bash
gh pr view <number> --repo <owner/repo> \
  --json title,body,files,comments,reviews,commits,state,mergedAt,createdAt
```

## Group the chaos into real tasks

Never list commits one by one. Bunch them into actual things you worked on.

Bad: "Commit A, Commit B, Commit C, Commit D"
Good: "Added retry logic for webhooks"

Those four commits were probably: initial code + tests + lint + review fixes. That's one real task.

## Figure out the "why" and "how"

For each chunk of work, ask:
- **Why?** — the PR body or linked issues tell the real story ("Users were getting failed uploads...")
- **How?** — skim the changed files. Feature? Bug fix? Refactor?
- **Scope?** — which folders? api/, frontend/, docs/?

This turns "changed upload_service.go" into "fixed race conditions that broke uploads."

## Check the collab stuff

Read reviews and comments. If someone said "move this to middleware" and you did it, mention that. If you reviewed someone else's code and gave feedback, include it. That's real work.

## Turn tech into accomplishments

Skip line counts. Say what actually got better:
- "Made webhook deliveries way more reliable with retries"
- "So search doesn't hammer the API as much anymore"

## Catch the invisible work

Commits miss a lot. Look for debugging sessions, abandoned experiments, CI fixes, code reviews you gave. Clues: messages like "WIP", "debug", "fix CI", "attempt 2".

## Write it

Keep it casual. Like you're telling someone what you did today.

```
• **Got assessment image uploads working** — was storing data URLs in the DB
  instead of uploading to R2 like question images already did. Quick fix once
  I spotted the gap. ([PR #7](url))

• **Reviewed a CORS PR** — a collaborator tried disabling CORS by commenting
  code out. Asked them to do it properly, they fixed it, merged.
  ([PR #13](url))

• **Shipped some API improvements** — students can now see how many attempts
  they have left on assessments, and the frontend has a polling endpoint to
  track AI course generation progress. ([PR #6](url))

Still open: nothing major, everything from yesterday got merged.
```

Link every item. If there's nothing still open, either say so or skip it.

This is roughly what you're aiming for:

```markdown
• Finished the webhook retry stuff and addressed all the review comments
• Kept chipping away at the upload race condition bug
• Fixed that annoying CI failure
• Opened a PR for the new caching layer
```

## A few things to watch out for

- `gh search commits` can leak outside your date range — use both `>=` and `<=` with the same date to lock it down
- Fork PRs won't show up under the upstream repo, check your fork too
- Skip merge commits, they're noise. Focus on the actual work
- `--author=@me` finds PRs you opened; `--review-requested=@me` finds ones you reviewed (easy to miss this)
- Private repos are fine, `gh` shows what you have access to
- Dates are YYYY-MM-DD: `date -d "yesterday" +%F` or `date -d "today" +%F`

## A useful mental checklist

For each piece of work, quickly run through:

- What problem was I solving?
- Why did it matter?
- What did I actually change?
- What's better now?
- What's still left to do?
- How sure am I about this? (high if there's a PR and linked issue, medium if I'm inferring from code, low if I'm guessing from a commit message)

Then write it casually. If it sounds like a standup update from a real person, you're done.
