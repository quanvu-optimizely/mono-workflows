# Example Review — canonical output

This is a reference example of a reviewer report. It demonstrates the exact
format defined in `SKILL.md § Output Template`. Findings are illustrative.

---

# Review Summary

**Scope:** branch `feat/user-profile-avatar` vs `main` (STORY-042)
**Files reviewed:** 6
**Diff size:** ~210 changed

Adds avatar upload + display to the user profile. Implementation is mostly
sound but the upload handler skips MIME validation and the avatar URL is
rendered without escaping, which combine into an XSS vector. Tests cover the
happy path only.

## Critical Issues
- `src/server/upload.ts:58` — Uploaded file's `mimetype` is trusted from the multipart header and never re-validated against the actual bytes; combined with the unescaped render below, an attacker can upload an `image/svg+xml` containing a `<script>`. Validate via magic bytes and reject SVG, or set `Content-Disposition: attachment` on serve.
- `src/components/Avatar.tsx:24` — `<img src={user.avatarUrl}>` is fine, but the adjacent `dangerouslySetInnerHTML={{ __html: user.avatarCaption }}` renders user input without sanitization. Drop `dangerouslySetInnerHTML` and render as text, or sanitize via the existing `sanitizeHtml` helper.

## Major Issues
- `src/server/upload.ts:91` — `await fs.writeFile(...)` is missing — promise is created but not awaited, so the HTTP 200 returns before the file is on disk. Subsequent GETs can 404 intermittently. Add `await`.
- `src/server/profile.ts:33` — Exported `getProfile` signature changed from `(id)` to `(id, opts)` but the two existing call sites in `src/routes/me.ts` and `src/jobs/digest.ts` were not updated; they will pass `undefined` for `opts` and break the new "include avatar" branch.
- `tests/upload.test.ts` — No test covers the rejection path (non-image upload, oversized upload). Required by AC-3.

## Minor Issues
- `src/components/Avatar.tsx:12` — Falls back to `defaultAvatar.png` when `avatarUrl` is empty string but not when it is `null`. Unify the check.
- `src/server/upload.ts:14` — `MAX_SIZE` is `5_000_000` but the comment says "5 MB" — should be `5 * 1024 * 1024` or update the comment.

## Suggestions
- `src/components/Avatar.tsx:8` — Component re-creates the `Image` object on every render. `useMemo` would help if this becomes a hot path; not urgent.

## Acceptance Criteria Check
- [x] AC-1: User can upload an image from the profile page — Passed.
- [ ] AC-2: Uploaded image is shown in the navbar within the same session — Failed. Missing `await` (see Major) means the URL is sometimes empty on first GET.
- [ ] AC-3: Non-image uploads are rejected with 415 — Failed. No MIME validation; tests missing.
- [x] AC-4: Avatar persists across sessions — Passed.

## Final Status
NEEDS_CHANGES — 2 critical (XSS / MIME), 3 major (missing await, broken call sites, missing tests), 2 failed acceptance criteria.
