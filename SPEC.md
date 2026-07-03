# BP Monitoring Log — Spec

A personal, local-first PWA for logging blood pressure readings. One patient for now (schema future-proofed for more), three readings a day, entered on a phone, no backend.

## Background & goals

- Log BP **immediately**: three readings a day — morning 8–9am, afternoon 1:30–2:30pm, night 7–8pm — one reading per slot, timestamped.
- The doctor asked for a follow-up if BP goes above 150 (systolic) and will want to see the data at the next consult, so a print-friendly report is a must (phase 2, right after logging works).
- Priorities: maintainable, easy to understand, no over-engineering.

## Decisions

| Decision | Choice |
|---|---|
| Stack | Vanilla HTML/CSS/JS + Vite — no framework, no state library |
| Schema | Flat readings (one per slot), no session grouping |
| Platform | Installable PWA on phone → static hosting + manifest + service worker in phase 1 |
| Guideline | ESC/ESH zones, plus a "doctor flag" at systolic ≥ 150 |
| Storage | localStorage behind a small storage module; local-first, no backend |
| Patients | One now (`patientId: "p1"`), multi-patient supported by the schema |
| Phasing | Phase 1 = everything needed to log today · Phase 2 = doctor report · Phase 3 = chart & add-ons |

## 1. Data model

One localStorage document under key `bp-log`:

```json
{
  "schemaVersion": 1,
  "patients": [{ "id": "p1", "name": "…" }],
  "readings": []
}
```

`Reading`:

| Field | Type | Notes |
|---|---|---|
| `id` | string | `crypto.randomUUID()` |
| `patientId` | string | `"p1"` for now; multi-patient hook |
| `takenAt` | string | ISO 8601 **with local offset** (e.g. `2026-07-03T08:15:00+05:30`) |
| `slot` | `"morning" \| "afternoon" \| "night"` | auto-suggested from time, user-overridable |
| `systolic` | number | mmHg |
| `diastolic` | number | mmHg |
| `pulse` | number \| null | bpm, optional |
| `arm` | `"left" \| "right"` | defaults to last used |
| `position` | `"sitting" \| "lying"` | defaults to last used |
| `notes` | string | optional (medication, stress, exercise) |
| `createdAt`, `updatedAt` | string | ISO, for audit/edit tracking |

- `schemaVersion` enables future migrations (e.g. to IndexedDB) without data loss.
- Slot suggestion: nominal windows are morning 8–9am / afternoon 1:30–2:30pm / night 7–8pm, but assignment uses broad boundaries (before 12:00 → morning, 12:00–17:00 → afternoon, after 17:00 → night) so an off-schedule reading still gets a sensible default. Windows live in the constants module.

## 2. Validation

Hard-block impossible values (typo protection):

- systolic 70–250
- diastolic 40–150
- pulse 30–200
- systolic > diastolic

Everything in range saves without friction.

## 3. Zones & thresholds

All thresholds live in a constants module — no magic numbers in UI code.

ESC/ESH classification; a reading's category = the **worse** of its systolic and diastolic grades:

| Category | Systolic | | Diastolic |
|---|---|---|---|
| Optimal | < 120 | and | < 80 |
| Normal | 120–129 | and/or | 80–84 |
| High-normal | 130–139 | and/or | 85–89 |
| Grade 1 HTN | 140–159 | and/or | 90–99 |
| Grade 2 HTN | 160–179 | and/or | 100–109 |
| Grade 3 HTN | ≥ 180 | and/or | ≥ 110 |

Plus the **doctor flag**: systolic ≥ 150 gets a distinct visual marker (doctor's follow-up threshold). Configurable constant.

## 4. Phase 1 — log readings today (MVP)

1. **Entry form** (the whole game — target < 15 s per entry)
   - Date/time prefilled to now; slot auto-suggested; arm/position prefilled from the last reading
   - Numeric inputs with `inputmode="numeric"`, large touch targets; notes optional and collapsed
2. **List view** — reverse-chronological, grouped by day; each reading shows SYS/DIA/pulse + slot + color-coded ESC category chip + doctor flag when systolic ≥ 150
3. **Edit / delete** — edit reuses the entry form; delete asks for confirmation
4. **JSON export / import** — full-document download and restore (import replaces after confirm). This is the backup story: browser storage is evictable and deleting an installed PWA deletes its data, so make weekly export a habit
5. **PWA shell** — `manifest.webmanifest` (name, icons, standalone display) + a minimal cache-first service worker for offline; installable from the hosted URL
6. **Storage module** — a single small wrapper owning read/write/migrate of the `bp-log` document; the only code that touches localStorage

## 5. Phase 2 — doctor report (before the next consult)

- **Print/PDF report** via a print stylesheet (no PDF library): patient name, date range, readings table grouped by day × slot, summary block with per-slot and overall averages, and a count of readings at/above the 150 doctor threshold. Browser print → Save as PDF.
- **CSV export** (secondary; column order mirrors the reading fields).

## 6. Phase 3 — later add-ons

- Line chart of SYS/DIA over time with ESC zone bands and a 7-day rolling average; morning-vs-night comparison
- Multi-patient UI (switcher; the schema already supports it)
- "Not a medical device" disclaimer if ever shared beyond personal use
- IndexedDB migration only if localStorage ever becomes a real constraint (at 3 readings/day it won't for years)

## 7. Architecture & project layout

```
index.html
styles.css
manifest.webmanifest
sw.js
src/
  main.js          # bootstrapping, routing between views (hash-based or show/hide)
  storage.js       # localStorage wrapper + schemaVersion migration
  thresholds.js    # ESC zones, doctor flag, slot windows, validation ranges
  views/…          # small render-function modules (entry, list, report)
```

No framework, no state library: render functions + event listeners. Deploy as static files to GitHub Pages or Netlify.

## 8. Design principles

Drawn from [arpit.codes/design-principles](https://v3.arpit.codes/design-principles/), the [GOV.UK design principles](https://www.gov.uk/guidance/government-design-principles), and [principles.adactio.com](https://principles.adactio.com/) — each stated with what it concretely means *for this app*.

1. **Rule of least power** *(Berners-Lee / Jeremy Keith)* — Prefer HTML/CSS over JS wherever the platform suffices: a real `<form>` with native inputs (`type="datetime-local"`, `inputmode="numeric"`, `<select>`), the doctor report as a print stylesheet rather than a PDF library. JS earns its place only for what HTML can't do (storage, list rendering).
2. **Do the hard work to make it simple** *(GOV.UK)* — The < 15 s entry form is this principle in action: auto-filled time, auto-suggested slot, last-used arm/position. Effort goes into the defaults so entry stays "three numbers and a tap".
3. **This is for everyone / good design works for everyone** *(GOV.UK / Adam Silver)* — Zone chips never communicate by color alone: always a text label alongside the color, at WCAG-AA contrast. Labeled inputs, large touch targets. Matters double if the patient or family members use it later.
4. **Understand context** *(GOV.UK)* — Used one-handed on a phone, at home, moments after cuff deflation, possibly with no connectivity. Everything works offline; nothing waits on a network.
5. **Least surprise, appearance follows behavior, consistency** *(Joshua Porter)* — Buttons look and act like buttons; edit reuses the same entry form; the same chip means the same thing in the list, the report, and the future chart. Destructive actions (delete, import-replace) always confirm.
6. **Postel's law** *(Jon Postel)* — Conservative out: exports follow the strict, versioned schema exactly. Liberal in: import tolerates unknown extra fields and formatting noise, validates, then normalizes.
7. **Progressive disclosure** *(Joshua Porter)* — One primary action per screen (log a reading). Notes collapsed by default; anything secondary stays out of the main path.
8. **Protect users' work** *(Bruce Tognazzini)* — A typed-but-unsaved reading survives an accidental back-swipe (keep form state, or persist a draft). Nothing destructive without confirmation.
9. **Own your data** *(IndieWeb)* — Local-first, no account, no lock-in: human-readable JSON export and CSV mean the data outlives the app and any browser.
10. **Iterate. Then iterate again** *(GOV.UK)* — The phases are the plan: ship logging, live with it for real readings, and let that experience shape the report and chart instead of designing them speculatively.
11. **Calm technology** *(Calm Tech Institute)* — This is a health chore, not an engagement product: no streaks, no guilt states, no notifications by default. The app is quiet, does its job, and gets out of the way.

## 9. Verification checklist

- Add → reload → reading persists; edit and delete round-trip
- Export JSON → clear site data → import → data restored
- Enter 130/85 (high-normal), 152/95 (grade 1 + doctor flag), 79/85 (blocked: sys ≤ dia) and confirm chip colors/flags/validation
- Lighthouse PWA check passes; install to home screen; toggle airplane mode and confirm the app opens and saves offline
