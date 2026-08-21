---
name: we-shadcn-conventions
description: House rules for working with shadcn/ui components — deferring to the official shadcn skill, respecting default styling and theme tokens, and using React Hook Form for forms. Use whenever shadcn components or theming are part of the discussion.
---

- Don't restyle a shadcn component from scratch — its default styling is usually sufficient. Check the component's own definition first to avoid redundant or conflicting styles.
- Shadcn's theme tokens are the primary source of color and style; don't drift from them unless necessary.
- Prefer shadcn's form setup with React Hook Form for building forms.
