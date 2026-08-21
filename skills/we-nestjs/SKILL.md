---
name: we-nestjs
description: Conventions for NestJS controllers, services, DTOs, and modules — OpenAPI specs, the nestjs-zod DTO pattern, layering rules, and route param naming. Use whenever writing or editing NestJS code, DTOs, controllers, services, or modules.
---

- Every endpoint gets an OpenAPI spec and, where relevant, request/response DTOs, with `@ApiOperation({ summary, description })`.
- **Default DTO pattern**: controller methods return the response DTO as their return type, so TypeScript flags any mismatch between the Swagger spec and the actual returned data.
- **New-project pattern (preferred when starting fresh)**: use `nestjs-zod` + `zod`. Build DTOs with `createZodDto(zodSchema)`; the schema is always reachable via `Dto.schema`. In controllers, call `.decode()` (not `.parse()`) to verify the return value against the schema — this makes an explicit return type unnecessary. Use `.array()` on the schema for list endpoints.

  ```ts
  export class UserDto extends createZodDto(
    z.object({ name: z.string().min(1), age: z.number(), email: z.email() }),
  ) {}

  // controller
  return UserDto.schema.decode(data); // single
  return UserDto.schema.array().decode(data); // list, pair with isArray: true on @ApiOkResponse
  ```

- Layering: controllers handle request/response concerns only and call services; services hold business logic and call other services/repositories; repositories handle DB access and call other repositories.
- Module folder convention: `.module.ts`, `.controller.ts`, `.service.ts`, `.repository.ts`, `.dto.ts`, `.constants.ts`, co-located at the same level.
- Never use `z.date()` in a DTO — OpenAPI can't represent it. Use a shared `dateSchema` that preprocesses to/from ISO strings instead.
- With `nestjs-zod` + Drizzle, use `drizzle-zod` too: every table not managed by an external service gets a `.zod.ts` with `createInsertSchema`, `createUpdateSchema`, `createSelectSchema`, and corresponding types. `.dto.ts` files should build on these exports so the DB schema and zod schema stay the single source of truth. Use `.pick()`/`.omit()`/`.extend()`/`.partial()` deliberately — not to pick or omit nearly every field.
- Route params: never `:id` — use `:<entity>Id` (e.g. `:userId`, `:merchantId`), same for slugs.
