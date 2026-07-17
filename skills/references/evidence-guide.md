# Evidence Guide

Use this reference when evaluating how trustworthy each piece of evidence is and when checking your work for each item.

## Evidence hierarchy (trust in this order)

1. **PR discussion / reviews** — most trustworthy. Shows what actually happened, what was debated, what changed as a result.
2. **Linked issues** — tells you *why* the work exists (bug report, feature request).
3. **PR description** — the author's summary of what and why.
4. **Commit diffs** — ground truth for what code changed.
5. **Commit messages** — the author's explanation (variable quality).
6. **Commit titles** — weakest signal. Often generic or truncated.

## Per-item checklist

For every cluster of work in the report, quickly answer:

- **What problem was I solving?** — not the code change, but the human problem
- **Why did it matter?** — who was affected, what was at stake
- **What did I actually change?** — the technical scope
- **What's better now for users/devs?** — the outcome
- **What's still left to do?** — follow-ups, blockers, unfinished work
- **How sure am I about this summary?** — high (PR + linked issue), medium (inferred from code), low (guessing from commit message alone)

## Example

> Worked on improving webhook reliability by implementing retry logic with exponential backoff, adding test coverage, and incorporating reviewer feedback before updating the open PR. The changes reduce the likelihood of dropped webhook deliveries while keeping retry behavior configurable. Follow-up work remains to monitor production behavior after merge.

This answers all six questions concisely, in casual language.
