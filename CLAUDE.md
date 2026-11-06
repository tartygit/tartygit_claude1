# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file IT PMO Kanban board (`index.html`) — an internal demo/training tool for a **fictitious** bank. Do not add any real bank logo, trademarks, or styling that imitates an official system; the UI uses a neutral "IT PMO" text wordmark and a corporate blue palette.

## Hard constraints (from the original spec — keep them)

- Vanilla HTML/CSS/JS only, all in `index.html` (one `<style>`, one `<script>`). No frameworks, build step, npm, CDNs, web fonts, or image files — icons are inline SVG/Unicode.
- Must work by double-clicking the file (`file://`), no server.
- **No persistence:** no `localStorage`, `sessionStorage`, IndexedDB, or cookies. A refresh re-seeds the board on purpose, and the toolbar note says so.
- The only network call is FormSubmit's AJAX endpoint. Never send data anywhere else.
- No `!important`; palette and spacing come from CSS custom properties on `:root`.
- Every user-supplied string goes through `escapeHtml()` before being put into HTML; toast text uses `textContent`.
- No native `alert()`/`confirm()` — validation errors appear inline under fields; delete uses the inline "Delete? Yes / No" toggle.

## Running / checking

There is no build, linter, or test suite. Open `index.html` in a browser. Quick sanity check that the script parses and the constraints still hold:

```sh
node -e 'const h=require("fs").readFileSync("index.html","utf8");const js=h.match(/<script>([\s\S]*?)<\/script>/)[1];new Function(js);console.log("ok",/localStorage|sessionStorage|indexedDB|document\.cookie/.test(js),h.includes("!important"))'
```

## Architecture (inside the `<script>`)

The script is split into numbered, commented sections (CONFIG → constants → state → icons → utilities → seed → state operations → rendering → toasts → FormSubmit → form → dialog → board interactions → filters → init).

- **Single source of truth:** `state = { tasks, filters, ui }`. `state.ui` holds transient UI state (ID counter `nextSeq`, open move menu, pending delete confirmation, drag id, submitting flag).
- **Render-from-state:** change `state`, then call `renderBoard()`, which rebuilds every column's card list, count badges and the header summary. Card HTML comes only from `renderCard()` and its helpers. Don't edit card DOM directly; the only DOM tweaks outside rendering are the drag/drop CSS classes.
- **State operations:** `addTask()`, `moveTask()`, `deleteTask()`, `applyFilters()` — all changes to tasks go through these.
- **Domain lists** (`STATUSES`, `PROJECTS`, `CATEGORIES`, `PRIORITIES`) drive the column shells (`buildColumns()`) and every `<select>` (`fillSelect()`). To add a column or option, edit the constant, not the markup. Priority/status colours are also keyed in CSS by `card--<priority>` and `[data-status="…"]`, so a new value needs matching CSS.
- **Events are delegated:** one click/keydown/drag handler set on `#board`, using `data-action` + `data-id` attributes. Because a re-render replaces the elements, handlers call `focusAfterRender(selector)` to put keyboard focus back on the right control.
- **Drag & drop:** native HTML5 DnD; `state.ui.draggingId` gates `dragover` so files dragged in from outside are ignored. `handleDrop` must call `endDrag()` itself, because the re-render detaches the card before `dragend` can bubble.
- **Add Task:** a native `<dialog>` styled as a right-side drawer. Submitting is optimistic: `addTask()` → reset form → success toast, then `notifyNewTask()` runs. If it fails, a warning toast appears and the card stays.
- **Toasts while the dialog is open:** the `aria-live` toast region is moved inside the `<dialog>` while it is open (and back on close). Otherwise toasts would sit under the modal top layer and be made inert.
- **Dates:** stored as local `YYYY-MM-DD` strings (`toISODate`, not `toISOString`, to avoid UTC off-by-one), so string comparison works for overdue/past checks. Seed due dates are offsets from today, so "Overdue" always shows.

## FormSubmit

- Config constants live at the top of the script: `FORMSUBMIT_ENDPOINT` (swap the email here only), `TASK_ID_PREFIX` (`UOB-ITPM-`), `EMAIL_SUBJECT_PREFIX`, `NOTIFY_TIMEOUT_MS`.
- While the endpoint contains the `YOUR_EMAIL@example.com` placeholder, `notifyNewTask()` throws without making a network call.
- FormSubmit needs a one-time activation (the first submission emails a confirmation link). Before activation it can answer HTTP 200 with `success: "false"`, which the code treats as a failure.
