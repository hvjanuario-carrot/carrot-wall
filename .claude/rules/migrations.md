---
paths:
  - "apps/api/src/main/resources/db/migration/**"
---

# Migration conventions

- Never edit an existing Flyway migration — add a new versioned file instead. A
  `PreToolUse` hook in `.claude/settings.json` already blocks edits to existing `V*`
  files at the tool-call level; this isn't just a style preference.
- Keep SQL portable across H2 (PostgreSQL mode) and real Postgres: no `JSONB`, no arrays.
- Each migration tells one part of the schema story (V1 posts → V2 answers/moderation →
  V3 prompts + seed → V4 `updated_at` → V5 its default → V6 prompt presets) rather than
  one file dumping the whole schema at once.
- If an already-merged migration turns out to be wrong, fix it forward with a new
  migration — never in place. V5 exists solely to add the `DEFAULT CURRENT_TIMESTAMP`
  that V4 was missing.
