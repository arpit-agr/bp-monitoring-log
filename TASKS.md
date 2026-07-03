# Build plan

The spec is [SPEC.md](SPEC.md). This file is the *order of work*. Rules of the road:

1. **One milestone at a time, in order.** Don't start the next until every box in the current "Done when" list is checked.
2. **Commit at every milestone.** A checked "Done when" list = a commit. Working code in git beats perfect code in your editor.
3. **CSS discipline.** Until M8 you get *one* utility stylesheet of ~15 lines (system font stack, a max-width, `font-size: 16px+` on inputs so iOS doesn't zoom). Every other styling idea goes into [design-ideas.md](design-ideas.md) — write it down, walk away. M8 is your dedicated, guilt-free design milestone. The parking lot isn't punishment; it's how the fun survives until the app works.
4. **Stuck > 30 minutes → ask Claude.** Paste the error, say what you expected. Struggling briefly is learning; struggling long is stalling.
5. Each milestone lists **new concepts** — things you likely haven't met building static sites. Feeling confused there is expected, not a sign you're behind.
6. **Journal at every milestone.** The same sitting that checks a "Done when" list also adds an entry to [JOURNAL.md](JOURNAL.md) — five minutes, while it's fresh. This is the raw material for the portfolio write-up later; it cannot be reconstructed afterwards.

Estimates include first-timer buffer. Slipping is fine; skipping ahead isn't.

---

## M0 — Log today's readings (5 min, no code, do this first)

The app is not needed to start logging — a timestamp is. `takenAt` is editable, so everything you capture now gets backfilled in M3.

- [x] Create a note on your phone with this template and fill it in at each slot:
  ```
  2026-07-03 19:15 | 128/84 | pulse 72 | left | sitting | notes: after dinner walk
  ```
- [ ] Keep doing this every slot until M3 ships — this is an ongoing chore, not a one-off. Leave this box unchecked until M3's backfill step retires the note; checking it *is* the sign M0 is over. Pressure's off.

---

## M1 — Hello, deployed (~1–1.5 h, tonight)

**Goal:** a URL on your phone that shows a page you built, redeploying on every push. Deploy day one so every later milestone is testable on the real device.

- [ ] `npm create vite@latest . -- --template vanilla` in this repo (delete the demo counter code it generates)
- [ ] `npm run dev` works locally; edit `index.html` to say something, see it hot-reload
- [ ] Push the repo to GitHub
- [ ] Connect the repo to Netlify (build command `npm run build`, publish directory `dist`)
- [ ] Open the Netlify URL on your phone

**Done when:** you edit `index.html`, push, and the change appears at the URL on your phone within a couple of minutes.

**New concepts:** npm scripts, Vite dev server vs. `build`, a deploy pipeline. None of it is deep — it's plumbing you set up once.

---

## M2 — Data layer, no UI (~1–1.5 h)

**Goal:** the app's brain, tested entirely from the DevTools console. Building this before any UI feels backwards coming from static sites — it's the single biggest habit shift of this project.

- [ ] `src/thresholds.js`: export the ESC zone table, doctor flag (150), low-BP thresholds (< 90 / < 60), slot windows, and validation ranges from SPEC §2–3 as plain constants; add `categorize(systolic, diastolic)` returning the worse-of-the-two grade, `isLow(systolic, diastolic)`, and `suggestSlot(date)`
- [ ] `src/storage.js`: `load()` / `save(doc)` wrapping localStorage key `bp-log` (creating the empty `schemaVersion: 1` document on first run), plus `addReading()`, `updateReading()`, `deleteReading()`
- [ ] In the browser console: import and call `addReading()` with a fake reading, reload the page, `load()` shows it's still there
- [ ] Console-check: `categorize(152, 95)` → grade 1, `categorize(118, 76)` → optimal, `isLow(88, 58)` → true, `isLow(96, 62)` → false

**Done when:** a reading added from the console survives a reload, and `categorize` answers correctly for the SPEC §9 test values.

**New concepts:** ES module import/export, localStorage, JSON.stringify/parse round-trip, writing functions with no UI attached.

---

## M3 — Entry form, ugly (~2–2.5 h, the big one — target Saturday morning)

**Goal:** log a real reading on your phone. This is the moment the app enters production use. Resist styling it — browser-default ugly is the uniform of this milestone.

- [ ] A real `<form>`: `datetime-local` input prefilled to now, slot `<select>` prefilled from `suggestSlot()`, numeric inputs with `inputmode="numeric"` for sys/dia/pulse, arm & position prefilled from the last reading, notes `<textarea>`
- [ ] On submit: validate against thresholds ranges (block + show which field), build the Reading object (`crypto.randomUUID()`, `patientId: "p1"`, `createdAt`/`updatedAt`), `addReading()`, clear the form, show a plain "Saved ✓"
- [ ] Test the SPEC §9 trio: 130/85 saves, 152/95 saves, 79/85 is blocked
- [ ] Push, open on phone, log the next real reading there
- [ ] **Backfill every reading from the M0 note** (edit the datetime field to the noted time), then retire the note

**Done when:** all your real readings to date live in the app, entered on the phone, and an entry takes under 15 seconds.

**New concepts:** form events and `preventDefault()`, reading form values into an object, `datetime-local` quirks (it wants `YYYY-MM-DDTHH:MM`, no seconds/offset — format the prefill accordingly).

---

## M4 — List view (~1–2 h)

**Goal:** see what you've logged, color-coded, newest first.

- [ ] Render all readings below the form: grouped by day, reverse-chronological, each showing time, slot, SYS/DIA, pulse, arm/position
- [ ] ESC category chip using `categorize()` — **text label + color, never color alone** (SPEC §8.3)
- [ ] Doctor flag marker on any reading with systolic ≥ 150; low-BP marker on any reading where `isLow()` is true
- [ ] List re-renders after saving a new reading

**Done when:** your real readings display grouped by day with correct chips, and a fresh save appears without a reload.

**New concepts:** rendering a list from data (build HTML from the readings array — a render function you call whenever data changes; this pattern is the whole "no framework" architecture).

---

## M5 — Edit & delete (~1–2 h)

**Goal:** fix the inevitable fat-fingered 1300/80.

- [ ] Edit button per reading: loads that reading into the same entry form, submit updates instead of adds (`updateReading()`, bump `updatedAt`)
- [ ] Delete button per reading: `confirm()` first, then `deleteReading()`, re-render
- [ ] Edit → change a value → save → list shows the change; cancel path leaves data untouched

**Done when:** you can round-trip an edit and a delete on the phone without anything else breaking.

**New concepts:** one form serving two modes (track "editing id" state), event delegation for buttons inside a rendered list.

---

## M6 — JSON export / import (~1 h)

**Goal:** the backup story. Small milestone, big safety.

- [ ] Export button: download the full `bp-log` document as `bp-log-YYYY-MM-DD.json` (Blob + object URL + temporary `<a download>`)
- [ ] Import: file input → parse → sanity-check (`schemaVersion`, `readings` is an array) → `confirm()` warning it replaces everything → save → re-render
- [ ] Round-trip test: export, delete a reading, import the file, reading is back
- [ ] **Make your first real backup and put it somewhere off-device** (email/Drive)

**Done when:** the round-trip test passes and a real backup exists outside the phone.

**New concepts:** Blob/object URLs for downloads, FileReader for uploads.

---

## M7 — PWA shell (~1–1.5 h) → **Phase 1 complete 🎉**

**Goal:** installed on the home screen, works in airplane mode.

- [ ] `manifest.webmanifest`: name, short_name, icons (192 + 512 px — a plain typographic placeholder is fine; real icon is an M8 parking-lot item), `display: "standalone"`, theme/background colors
- [ ] `sw.js`: minimal cache-first service worker — precache the built assets, serve from cache, network fallback; register it from `main.js`
- [ ] Vite note: put `sw.js` in `public/` so it lands at the site root un-bundled, and cache the *built* file paths
- [ ] Install to home screen on the phone
- [ ] Airplane-mode test: app opens, saving a reading works offline

**Done when:** home-screen icon opens the app with no browser chrome, and the airplane-mode test passes. Tag it: `git tag v1`.

**New concepts:** the manifest, service worker lifecycle (install/activate/fetch — the confusing one; the SPEC only needs the simplest version), and the gotcha that a changed SW needs a cache-name bump to take over.

---

## M8 — Design milestone (open-ended — the reward)

Open [design-ideas.md](design-ideas.md) and go wild. You've earned it; the app works and is in daily use.

- [ ] Empty the parking lot in whatever order brings joy
- [ ] Only rule: after each styling session, re-check M3–M7 "done when" items still pass (entry < 15 s, chips readable, offline works)
- [ ] Contrast-check the zone chip palette against WCAG AA (SPEC §8.3)

---

## Phase 2 — before the next consult

### M9 — Doctor report (~2 h)
- [ ] Report view: patient name, date range, table grouped by day × slot, summary block (per-slot + overall averages, counts of readings ≥ 150 and of low readings)
- [ ] Print stylesheet (`@media print`): hide app chrome, black-on-white table, sensible page breaks
- [ ] Phone test: open report → system print → Save as PDF → share

### M10 — CSV export (~30 min)
- [ ] Export button producing one row per reading, columns mirroring the Reading fields

---

## Phase 3 — parked (visible, not urgent)

- Line chart of SYS/DIA with ESC zone bands + 7-day rolling average
- Multi-patient UI (schema already supports it)
- "Not a medical device" disclaimer if ever shared publicly
- Real app icon (if still in the parking lot)
- Portfolio README: what/why, screenshots, live URL, stack, key decisions, distilled from JOURNAL.md — write it when the project goes on the portfolio site

---

## Calendar sketch

| When | What |
|---|---|
| Tonight (Fri) | M0 now, then M1 (+ M2 if energy allows) |
| Sat before 4pm | M2 + M3 → **logging in the app** |
| Next week, ~1 evening each | M4 → M5 → M6 → M7 → phase 1 done ~Thu/Fri |
| Weekend after | M8 (design) and/or M9 (report) |

Buffer is built in. Slipping a day is fine; changing the order isn't.
