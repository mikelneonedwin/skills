---
name: we-drizzle
description: Conventions for Drizzle ORM schemas, queries, and migrations — relational queries, transactions, enum style, jsonb typing, and relations exports. Use whenever writing or editing a Drizzle schema, query, or migration.
---

- Prefer relational queries over manual join syntax.
- Use `update`/`delete` (with a `returning` clause) to read affected rows in the same operation instead of a separate read.
- Minimize total query count without overcomplicating the query or its syntax.
- Wrap independent parallel queries in `Promise.all`.
- Use transactions for multi-step creates. When steps must cross a transaction boundary, retain enough data to clean up partial records on failure.
- Never run Drizzle CLI commands or hand-write migrations — only edit the schema file. If asked to push/PR and migrations haven't been generated, flag it and wait before proceeding.
- `updatedAt` gets `.$onUpdate(() => new Date()).notNull()`; `createdAt` gets `.defaultNow().notNull()` — except for schemas managed by an external service (e.g. better-auth).
- Prefer text-based enums over native pg enums:

  ```ts
  status: text("status", {
    enum: ["pending", "processing", "completed", "failed"],
  });
  ```

- `jsonb` columns must have a fully-typed corresponding zod schema, not `z.record`/`z.any()`:

  ```ts
  // schema.ts
  export interface VideoManifest { videoId: string; renditions: { resolution: string; url: string }[] }
  manifest: jsonb("manifest").$type<VideoManifest | null>(),

  // schema.zod.ts
  const manifest = z.object({
    videoId: z.string(),
    renditions: z.array(z.object({ resolution: z.string(), url: z.string() })),
  });
  export const selectVideoSchema = createSelectSchema(videos, { manifest: manifest.nullish() });
  ```

- Always define and export relations for every table:

  ```ts
  export const courseRelations = relations(courses, ({ one, many }) => ({
    institution: one(institutions, {
      fields: [courses.institutionId],
      references: [institutions.id],
    }),
    lessons: many(lessons),
  }));
  ```
