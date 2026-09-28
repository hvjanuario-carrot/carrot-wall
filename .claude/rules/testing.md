---
paths:
  - "apps/web/**/*.spec.ts"
  - "apps/api/src/test/**/*.java"
---

# Testing conventions

- Web tests run with `npm test` (vitest + jsdom). API tests run with `./mvnw test`.
- A bug fix is not done until a test reproduces the bug first.
- Never weaken or delete an existing assertion to make a suite pass —
  say what you found instead.