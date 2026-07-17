# gh-report

Distribution repo for the gh-report skill. The skill lives in `skills/gh-report/`.

## Repo structure

```
.claude-plugin/          # Plugin manifest for marketplace install
skills/gh-report/
  SKILL.md               # Main skill instructions
  references/
    evidence-guide.md    # Reference material
.managed-skills          # Tracked skill directories
install.md               # Install guide for users
```

## Key constraints

- Do not edit `skills/` directly via automation that bypasses review. The skill is manually maintained.
- `.managed-skills` tracks which skill directories are owned. Don't edit it manually.
- Non-skill files (README, AGENTS.md, install.md, .claude-plugin/) are safe to edit directly.
