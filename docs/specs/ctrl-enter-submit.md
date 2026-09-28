# Ctrl+Enter submits the post form

## Problem

On `/post`, the only way to submit is clicking "Enviar". Someone typing a longer message
has to reach for the mouse/trackpad every time, which is friction on a form the spec wants
usable in under 20 seconds.

## In scope

- While focused in the message `<textarea>` on `/post`, pressing **Ctrl+Enter** (Windows/Linux)
  or **Cmd+Enter** (Mac) submits the form — same effect as clicking "Enviar".
- A small hint near the textarea telling people the shortcut exists (e.g. "Ctrl+Enter para
  enviar"), platform-appropriate if easy, "Ctrl+Enter" as a fallback otherwise.
- Empty message + shortcut → same inline PT validation error as clicking "Enviar" with an
  empty message.
- A submit already in flight + shortcut pressed again → no-op (mirrors the disabled "Enviar"
  button during `submitting()`).

## Out of scope

- The shortcut while focused elsewhere in the form (name field, type chips) — default browser
  behavior there, unchanged.
- Any shortcut on `/admin`'s answer editor or anywhere else in the app.
- A visible key-hint badge design beyond plain text (no icon/kbd styling pass).
- Mobile: no on-screen keyboard sends Ctrl/Cmd+Enter, so this is a desktop-only affordance by
  nature — no separate mobile behavior to build.

## Acceptance criteria

1. Focus the message textarea, type a valid message, press Ctrl+Enter (or Cmd+Enter on Mac) →
   the post submits and the redirect-to-`/`-with-highlight behavior fires, identical to
   clicking "Enviar".
2. Focus the textarea, leave the message empty, press Ctrl+Enter → the existing inline error
   ("Escreve qualquer coisa antes de enviar.") appears; no request is sent.
3. Press Ctrl+Enter a second time while a submit is already in flight → nothing happens (no
   duplicate request).
4. Ctrl+Enter while focused in the name field or on a type chip does not submit the form.
5. A short hint mentioning the shortcut is visible near the textarea.
6. Clicking "Enviar" continues to work exactly as before (no regression).

## Constraints

- Web-only change (`apps/web/src/app/post/post.ts` / `post.html`); no API changes.
- Must go through the existing `submit()` method, not a duplicate code path, so validation and
  rate-limit/error handling stay in one place.
- No `[innerHTML]` — the hint text is static, interpolated normally.
