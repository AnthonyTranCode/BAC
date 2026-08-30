# Implementation brief — mobile support for Monolith

Paste everything below the line into Claude Code from the repo root.

---

## Goal

`index.html` is a year-planner web app (single file, ~5,520 lines). It works on desktop and is
unusable on a phone. Add a mobile experience below 768px. Desktop behaviour must not change at all.

Reference prototype for the target design (Concept B is the one we're building):
https://claude.ai/code/artifact/5d09c4ab-8cf7-48d0-939c-42727059f46c

## Why it's currently broken

The calendar is a 12-row × 31-column grid: `--label-w: 76px` + `31 × --cell-w: 57px` ≈ **1,900px wide**.
`fitToScreen()` reshapes rather than zooms, so on a 390px viewport it solves for roughly **8px per
column** against 10px type. There is also no responsive treatment of `.page-header` (the only rule
below 1100px hides `.acct-email`), and the `.zoom-bar` / `#size-panel` controls are meaningless on touch.

## Architecture — read this before writing code

**Build the mobile views as a separate, additive render path. Do not refactor the desktop grid.**

The single most important thing this codebase already gives us: `openModal(dateStr, ev)` (line ~2996)
is the one and only entry point to the event editor, and it is wired to a plain `click` on day cells
(line 2638). The mobile views can call it directly.

Because mobile renders its own DOM, it never produces `.chip`, `.bar`, or `.day-cell` elements — so
**none of the desktop `mousedown` drag handlers need converting in this pass.** They simply never fire.
That includes:

- bar drag / resize — lines 2586, 2597, 2603 → `startDrag`, doc listeners 3360–3361
- chip drag — line 2689 → `onChipDragMove` / `onChipDragEnd`, 3501–3502
- chip resize — line 2703 → `onChipResizeMove` / `onChipResizeEnd`, 3603–3604
- size-panel grip — line 4129 → 4133–4134

Leave all of the above exactly as they are.

## Work items

### 1. Mode detection

Add a single `matchMedia('(max-width: 767px)')` source of truth. Set `document.body.classList` to
carry `mobile` / `desktop`, re-evaluate on change, and re-render.

### 2. Branch `renderCalendar()`

`renderCalendar()` (line ~2374) is the shared entry point and is already called from everywhere,
including the debounced resize handler near line 4684. Branch at the top of it:

```js
function renderCalendar() {
  if (isMobile()) { renderMobile(); return; }
  buildGrid();
  if (fitMode && !_fitting) fitToScreen();
}
```

`fitToScreen()` must never run in mobile mode — it measures `#page-outer` and would fight the new layout.

### 3. Month view (the default mobile view)

Render into `#cal-grid`, replacing the desktop grid entirely.

- Sticky month header: `‹  September 2026  ›`, arrows step the month, wrapping at the year boundary.
  Also support horizontal swipe between months.
- One row per day of the month, **including days with no events** — a free week should read as a gap.
- Row gutter: two-letter weekday + date number. Today's number gets a filled circle.
- Weekend rows take the `--secondary` tint, consistent with the desktop `--bg-weekend` treatment.
- Event chips are full width with readable titles at ~12.5px — no truncation to a colour block.
- Multi-day events repeat on every day they cover; on days after the first, append a muted `cont.`
  so it's clear it's a continuation rather than a second event.
- Dashed categories (`style: 'dashed'` — `misogi`, `bucketlist`) render as a dashed outline with
  `--foreground` text, matching the desktop chip treatment.
- Minimum row height 38px; with padding every tap target clears 44px.
- Tapping a row calls the existing `openModal(dateStr, null)`. Tapping a chip calls `openModal(ev.startDate, ev)`.
- Floating `+` button, bottom right, above the tab bar.

### 4. Year view (secondary, behind a tab)

The "monolith": **days run down as 31 rows, months run across as 12 columns.**

- ~28px columns, 20px rows, sticky month-initial header row (J F M A M J J A S O N D).
- Colour blocks only, no text — up to 3 per cell, side by side. This view is deliberately illegible
  at the event level; it exists to show the shape of the year.
- Short months get inactive cells for days 29–31, using `--muted`.
- Category legend underneath.
- Tapping a day opens a **read-only bottom sheet** listing that day's events, with an "Add" button
  that hands off to `openModal`. Tapping a month header switches to Month view on that month.

### 5. Tab bar

Fixed to the bottom in mobile mode, two tabs: `Year` / `Month`. **Month is the default landing view.**
Persist the choice in `localStorage` alongside the existing keys (the app already uses `bac-categories`).

### 6. Make `openModal` a bottom sheet on mobile

`.modal` is currently a centered card at a fixed 460–520px with `max-width: 96vw`. In mobile mode it
should slide up from the bottom, full width, rounded top corners, `max-height: 85vh`, internally
scrollable, with a drag-grabber. Same for the date picker (`dpOpen`, line ~2789), the category manager
(`openCatManager`, ~3875) and the holiday manager (`openHolManager`, ~3707).

Do not change any of their logic — only presentation.

### 7. Header, and the two real touch bugs

- Collapse `.page-header` on mobile: keep the title and year nav, move `#today-btn`, `#undo-btn`,
  `#cat-mgr-btn`, `#hol-mgr-btn`, `#export-btn`, `#signout-btn` into an overflow menu. Keep `#sync-dot` visible.
- Hide `.zoom-bar` and `#size-panel` in mobile mode (there is already a `body.chrome-off` precedent for this).
- The legend scrolls horizontally already — keep that, just reduce padding.
- **Change `mousedown` → `pointerdown` on exactly two lines: 4248 (emoji picker click-away) and
  4676 (size panel click-away).** These are the only two dismiss handlers that will otherwise not
  fire on touch. Two one-word changes; do not touch any other mouse handler.
- Audit `:hover`-only affordances that mobile markup inherits — notably `.chip:hover .chip-resize-handle`
  and `.day-cell:hover`. Scope them behind `@media (hover: hover)`.

## Reuse, don't reimplement

Both mobile views must read through the existing data layer so Supabase sync, undo, and repeats keep working:

`eventsForYear(curYear)` · `loadHolidayList(curYear, curCountry)` · `categories` / `loadCategories()` ·
`todayStr()` · `openModal()` / `closeModal()` / `saveEvent()` / `deleteEvent()` · `performUndo()` ·
`curYear` and the `#year-prev-btn` / `#year-next-btn` handlers

## Must not break

- Desktop layout, drag, resize, zoom, Fit, and the size panel — byte-for-byte identical behaviour.
- `html2canvas` export. It relies on `body.exporting` and measures the real grid; **export should
  force the desktop grid regardless of viewport**, or be hidden in mobile mode. Your call — say which you chose.
- The landing page (`#landing`) and auth gate (`#auth-gate`), which are already responsive at 900/560.
- Supabase sync and the undo stack.

## Acceptance criteria

1. At 390×844: no horizontal page scroll anywhere, every event title readable, all tap targets ≥44px.
2. Month view: create, edit, and delete an event by tapping; changes persist and sync.
3. Year view: all 365 days present, short months correctly stubbed, tapping any day opens its sheet.
4. Resizing a desktop browser across 768px switches cleanly in both directions with no reload.
5. At 1440px the app is indistinguishable from `git stash` — diff the desktop rendering to confirm.
6. No console errors in either mode.

## Non-goals for this pass

Drag-to-move and drag-to-resize on touch. Tap-to-edit only. Do not add Pointer Events plumbing for
dragging — we'll scope that separately once this lands.

## Finally

Work incrementally and keep the app runnable at each step. The repo is git-tracked — commit before
you start. When you're done, summarise what changed and flag anything in the brief that turned out
to be wrong about the code.
