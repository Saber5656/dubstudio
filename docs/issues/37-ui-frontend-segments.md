# Issue 37: UI frontend — segments review table

## Title

Implement the segments review table view (edit, status, audio preview)

## Summary

Implement the primary frontend view in vanilla ES modules (no build step) per
DESIGN.md §10.4: a keyboard-navigable segments table with inline translation editing,
status chips, per-row original/dub audio playback, fit warning badges, and filters.

## Context

This is the review loop's face. It must stay dependency-free (CSP `self`-only, §10.2)
and safe (all dynamic content via `textContent` — B3).

## Scope

In: `static/index.html` shell, `static/app.css`, `static/segments.js` (+ shared
`api.js`, `audio.js`). Out: pipeline panel (38), server routes (35/36).

## Detailed Requirements

1. `api.js`: fetch wrapper adding `X-Dubstudio-Csrf` (token parsed once from the
   handshake-set cookie? No — cookie is HttpOnly: the server injects the CSRF token
   into `index.html` as `<meta name="csrf">` at serve time; issue 34 exposes the
   hook), JSON error normalization ({error:{code,message}} → thrown typed error),
   ETag-aware GET for `/api/segments`.
2. Table (semantic `<table>`, virtualization NOT required in v1 — up to ~2000 rows
   acceptable; guard: >2000 rows renders a notice + windowed rendering fallback of
   500 rows/page):
   columns: `#` (id), time (`mm:ss.d` start–end + slot), source text (read-only),
   target text (editable), status chip (draft/edited/approved), fit badge
   (ok/shortened/overflow with overrun ms tooltip), audio (▶ original / ▶ dub), chars
   (len/budget, red when over).
3. Editing: click/Enter → textarea inline; Esc cancels; Cmd/Ctrl+Enter or blur saves
   via PATCH; optimistic UI with rollback+toast on error (409 lock → toast "run in
   progress"); status cycle button (draft→approved, edited→approved, approved→edited)
   PATCHes status; per-row "re-synth" button POSTs resynthesize (disabled while a job
   is active — job state from `jobs.js` shared store fed by SSE (38); before 38 lands,
   poll `/api/jobs` — keep a tiny poller behind the same interface).
4. Audio: single shared `<audio>` element; play stops previous; source =
   `/api/audio/{kind}/{id}?lang=`; buttons show loading/na states (404 → disabled
   with tooltip "not synthesized yet").
5. Filters/nav: status filter chips (all/draft/edited/approved/warnings), text search
   (client-side, both languages), j/k row navigation, `o`/`d` play original/dub, `e`
   edit — documented in a `?` help overlay. `aria-live=polite` for toasts; focus
   management on edit open/close.
6. All user data rendered via `textContent`/`value` (regression test greps sources
   for `innerHTML` — allowed only for the static help overlay template literal with
   no interpolation).
7. Language selector (manifest targets, from `/api/project`) drives `?lang=` for all
   calls; persisted in `localStorage`.
8. Dark/light via `prefers-color-scheme`; no external fonts/assets (CSP).

## Acceptance Criteria

- [ ] Serves and renders on the fixture project with 50 segments incl. one of each
      fit result; manual checklist in PR (screenshots light+dark).
- [ ] `rg "innerHTML" static/` shows only the allowed help-overlay case.
- [ ] Editing flow: save → row shows `edited` + stale re-synth indicator; 409 during
      job → rollback toast (simulated via TestClient-driven job).
- [ ] Keyboard nav + audio shortcuts work (manual checklist); help overlay lists all
      bindings.
- [ ] ETag: unchanged segments fetch → no re-render (console counter in debug mode).
- [ ] Over-budget chars render red with count.

## Validation

Manual checklist in PR against a mock-provider project + `pytest tests/ui/test_static.py`
(files served, CSP intact, csrf meta present, innerHTML grep). Frontend logic is
plain-DOM; no JS unit framework in v1 (documented trade-off).

## Dependencies

35, 36 (34 for csrf meta hook).

## Non-goals

Waveforms, drag-retiming, transcript editing, i18n of the UI chrome (English-only v1).

## Design References

DESIGN.md §10.4, §10.2 (CSP), §11.2 B3.
