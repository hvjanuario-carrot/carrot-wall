# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Carrot Wall — a live Q&A/feedback wall for a 5-day Claude Code masterclass. Attendees post
from their phones, the instructor answers/pins from `/admin`, a projector shows `/tv`.
Portuguese UI, no accounts. **`spec.md` is the source of truth** (features F1–F8, acceptance
criteria, design tokens). `feature-ideas.md` is a challenge backlog for extra exercises.
`setup-docs/` holds the course guides that drove the build.

**F1–F8 are built.** Both apps have real routes, entities, REST resources, and unit tests
(JUnit under `apps/api/src/test`, Angular specs beside every component, Playwright specs under
`apps/web/e2e/`). `docs/intents/01`–`08` and `docs/specs/01`–`08` (each with a matching
`-plan.md`) map one-to-one to F1–F8 — read the intent/spec/plan for a feature before changing
it, rather than re-deriving intent from the code.

## Commands

```bash
cd apps/api && ./mvnw quarkus:dev          # API, port 8080 (first run downloads a lot)
cd apps/web && npm start                   # web, port 4200
cd apps/api && ./mvnw test                 # JUnit
cd apps/api && ./mvnw test -Dtest=ClassName#methodName      # single test
cd apps/web && npm test                    # vitest + jsdom
cd apps/web && npx ng test --filter '^App' # single test/suite (regex on test names)
cd apps/web && npx ng test --include src/app/wall/wall.spec.ts   # single file
cd apps/web && npm run e2e                 # Playwright, apps/web/e2e/
cd apps/web && npx prettier --check .      # no eslint / `npm run lint` in this repo
```

A change isn't done until `./mvnw test` and `npm test` both pass (and `npm run e2e` if the
change touches a flow one of the two specs there covers).

Only env var: `ADMIN_PIN` (defaults to `0000` in dev). Node 20+ required (Node 25 works, warns).
A `PostToolUse` hook auto-runs `prettier --write` on every `.ts`/`.html`/`.scss`/`.json` file
you edit; a `PreToolUse` hook blocks edits to existing Flyway migration files outright.

## Architecture

- `apps/web` — Angular 22 standalone components + Angular Material, TS 6. Routes: `/` wall,
  `/post` submit, `/tv` projector, `/admin`, `/materials` (static course-link page). Polls the
  API every 5s from `WallService`/`WallComponent`; no websockets. `npm start` proxies `/api` to
  port 8080 via `proxy.conf.json` (wired into `angular.json` → `serve.options.proxyConfig`), so
  there is no CORS layer in dev either.
- `apps/api` — Quarkus 3.39 / Java 21. REST resources + Panache entities (`Post`, `Prompt`,
  `PromptPreset`). H2 **file** DB in PostgreSQL mode; **Flyway owns the schema**
  (`quarkus.hibernate-orm.schema-management.strategy=none`), `%test` profile uses in-memory H2
  so `./mvnw test` never collides with a running dev server.
- Production is **one container** (root `Dockerfile`): the Angular build lands in the Quarkus
  jar's `META-INF/resources`, both served on port 8080. `SpaRoutes.java` reroutes `/post`,
  `/tv`, `/admin`, `/materials` to `index.html` so a refresh does not 404 — add any new route
  there too.

### The polling model

`WallResource` (`GET /api/wall`) is one endpoint with three modes rather than three endpoints,
specifically so the hidden-post filter can't be forgotten in one of them: no params → page 1
(pinned posts, then newest unpinned page); `before`+`beforeId` → older page ("load more");
`since` → everything changed at or after that server timestamp (the 5s poll delta), plus any
ids newly hidden since then. `WallService` on the client keeps posts in an id-keyed `Map`
signal (not an array) so a delta can upsert/remove without positional bookkeeping, and
serializes overlapping `poll()` calls so a fresh response from one can't be clobbered by an
older still-in-flight one.

This is why `Post`'s mutation methods (`pin()`, `unpin()`, `hide()`, `unhide()`, `upvote()`,
`answer(text)`) all matter: every one moves `updated_at` via `@PreUpdate`, which is the only
thing that makes it show up in a `since=` delta. **Never mutate a `Post` via a bulk
`update(...)` string** (e.g. `Post.update("upvotes = upvotes + 1 where id = ?1")`) — it
bypasses `@PreUpdate`, leaves `updated_at` stale, and the change becomes invisible to every
polling client until something else touches that row.

### Admin auth

Passcode-only, no accounts: `AdminResource.login` checks `ADMIN_PIN` and hands back an
`admin_session` cookie backed by an in-memory token set (`AdminSessionStore` — no expiry, no
logout, resets on redeploy, which is fine for a 5-day class). `AdminAuthFilter` is a JAX-RS
`ContainerRequestFilter` that enforces the cookie once, centrally, for every method annotated
`@AdminOnly` — new admin endpoints get 401-when-missing "for free" by adding that annotation,
not by re-checking the cookie themselves. `WallResource`'s own `includeHidden` query param is
deliberately *not* gated by `@AdminOnly` (attendees poll it unauthenticated every 5s); it's
downgraded to `false` server-side unless the request's session cookie is independently valid,
so appending `?includeHidden=true` as a random client can't leak hidden posts.

### Rate limiting

`RateLimiter` (used by `PostsResource`'s `POST /api/posts`) is keyed on a client-supplied
`X-Client-Token` header, not IP — the whole classroom sits behind one public IP, and IP-keying
would lock out the whole room together. A missing token gets no check at all: the header is a
friendly-room guard, not auth, and a client that can't/won't send one must never be blocked.
It's in-memory/single-process — resets on redeploy, doesn't span instances — which is
acceptable because production is one container.

## Conventions

- **Never edit an existing Flyway migration** — add a new versioned file. Migrations live in
  `apps/api/src/main/resources/db/migration/` (currently V1–V6) and are deliberately split into
  a story (posts → answers/moderation → prompts+seed → `updated_at` → its default → prompt
  presets), because Day 1's exercise is "explain the migrations"; a hook in `.claude/settings.json`
  blocks edits to existing ones at the tool-call level, not just by convention. Keep the SQL
  portable across H2-PG mode and real Postgres: no `JSONB`, no arrays.
- Endpoint input validation is **manual, not Bean Validation annotations**, where trimming has
  to happen before length is checked (a whitespace-only message must count as empty, which
  `@NotBlank`/`@Size` don't handle) — see `PostsResource.create` and `AdminResource.setAnswer`
  for the pattern. Follow it for new endpoints with the same trim-then-validate shape rather
  than reaching for annotations that would silently accept whitespace-only input.
- **Post messages render as text, never HTML.** Angular interpolation only, no `[innerHTML]`
  anywhere — acceptance criterion 12 is an XSS check.
- Hidden posts are a soft delete: excluded from every public response, never deleted from the DB.
- `docs/intents/` holds `spec.md` cut into eight buildable slices, in dependency order — start
  there, not at the spec, when picking up work on an existing feature. New feature specs go in
  `docs/specs/`, named `<NN>-<feature-name>.md` matching the intent's number and slug
  (`docs/intents/02-submit-a-post.md` → `docs/specs/02-submit-a-post.md`); a plan for that slice
  is the same name with `-plan` appended (`02-submit-a-post-plan.md`). Tests: JUnit beside the
  resource, Angular unit tests beside the component, Playwright e2e in `apps/web/e2e/`.
- Design tokens (ivory `#FAF9F5`, ink `#141413`, coral `#D97757`, hairline borders, no drop
  shadows, serif headings + Inter) are in spec §4 — follow `DESIGN.md` at the root, which has
  the fuller version.
- Slice branches stack on each other rather than always targeting `master` (e.g. a
  `slice-03-*` PR targets `slice-02-*` until that merges, then retargets `master`) — see the
  `/ship` and `/cleanup-worktree` commands for the exact flow.

A `CLAUDE.md.reference` file exists at the repo root as a course comparison artifact — it is
not an input to this file and should be left alone.
