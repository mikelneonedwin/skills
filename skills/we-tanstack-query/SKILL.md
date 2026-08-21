---
name: we-tanstack-query
description: Conventions for TanStack Query hooks — file structure, queryOptions usage, loading states, and mutation feedback. Use whenever working with data-fetching or mutation in TanStack Query.
---

- Define custom query/mutation hooks in `hooks.ts` / `mutations.ts`.
- For codebases with many hooks that lean on the query client, use `queryOptions` to centralize query configuration.
- Prefer shadcn skeleton components for loading states where shadcn is already set up.
- Use toasts for mutation loading/success/error states when no other indicator exists.
