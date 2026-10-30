# Repository Instructions

This repository manages personal custom skills used by my agent to perform tasks. Because these skills document instructions, prompts, and example reference patterns rather than a deployable runtime application, application-level conventions such as using an environment variable manager over direct `process.env` access are irrelevant across these files.

Every new skill added to this codebase must be registered across all three inventory surfaces before committing and shipping to GitHub:

1. `README.md` under the Included Skills list.
2. `.managed-skills` with the bare skill folder name.
3. `.claude-plugin/plugin.json` in the `skills` array as `./skills/<skill-name>`.
