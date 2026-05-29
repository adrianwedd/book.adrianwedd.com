# book.adrianwedd.com

A single-page booking site where visitors schedule a 30-minute call with Adrian Wedd. The entire frontend is one self-contained file — `index.html` — with inline CSS and vanilla JavaScript. No framework, no build step, no dependencies, no `package.json`.

It talks to a separate backend at `api.book.adrianwedd.com` (in another repository) for availability and booking.

## Contents

| Path | Purpose |
| --- | --- |
| `index.html` | The whole frontend: markup, styles, and booking logic |
| `CNAME` | Custom domain for GitHub Pages (`book.adrianwedd.com`) |
| `CLAUDE.md` | Guidance for Claude Code working in this repo |
| `SECURITY.md` | Security posture, review result, and invariants |
| `.gitignore` | Ignores `.afterwords` |
| `.afterwords` | Local scratch note (git-ignored); not part of the site |

## Running locally

There is no build or tooling. Either:

```sh
# Open the file directly
open index.html

# …or serve it (recommended, so fetch() and the API behave normally)
python3 -m http.server 8000
# then visit http://localhost:8000
```

Note that the calendar and booking calls hit the live API at `https://api.book.adrianwedd.com`. To point at a different backend, change the `API` constant near the top of the `<script>` block in `index.html`.

## Deployment

The site is hosted on **GitHub Pages** and served at the domain in `CNAME`. There is no CI pipeline.

```sh
git push origin main   # publishes; Pages rebuilds automatically
```

Because Pages serves static files only, you cannot set HTTP response headers here (relevant for CSP — see `SECURITY.md`).

## How it works

### The booking flow

The page is a three-step state machine. Each step is a `<div class="step">`; `showStep(id)` toggles which one has the `active` class so exactly one is visible.

```
┌──────────────┐   pick day + time   ┌──────────────┐   submit form   ┌──────────────┐
│ step-calendar│ ──────────────────► │ step-details │ ──────────────► │ step-confirm │
│  (calendar   │  ◄── "Change" ────  │ (name/email/ │                 │ (confirmation│
│  + slots)    │                     │  note form)  │                 │  message)    │
└──────────────┘                     └──────────────┘                 └──────────────┘
```

1. **`step-calendar`** — `renderCalendar()` draws a month grid. Past dates and weekends are rendered non-clickable; only future weekdays get the `available` class and a click handler. Selecting a day calls `selectDate()`, which fetches that day's open slots and renders them as clickable pills. Picking a pill records the slot and advances to the details step.
2. **`step-details`** — shows the chosen time and a form for name, email, and an optional note. Submitting `POST`s the booking.
3. **`step-confirm`** — on a successful booking, shows a confirmation message with the formatted time and fires the analytics/conversion events.

### Backend API

The backend is **not in this repo**. The frontend uses two endpoints (base URL in the `API` constant):

| Method & path | Request | Response | Used by |
| --- | --- | --- | --- |
| `GET /slots?date=YYYY-MM-DD` | — | `{ "slots": ["HH:MM", ...] }` | `selectDate()` |
| `POST /book` | `{ name, email, slot, note }` (JSON) | `200` on success | form submit |

`slot` is sent as a full ISO timestamp with timezone offset (`selectedSlotIso`, e.g. `2026-05-30T14:00:00+11:00`); `name`, `email`, and `note` come straight from the form. All validation and persistence happen server-side.

### Timezone handling

Times are presented in Tasmania local time. Two pieces must stay consistent:

- **`OFFSET`** (constant near the top of the script) is the literal UTC offset baked into each booked slot's ISO timestamp. **It must be flipped manually for daylight saving:** `+11:00` during AEDT (~Oct–Apr) and `+10:00` during AEST (~Apr–Oct).
- **Display formatting** uses `toLocaleString(..., { timeZone: 'Australia/Tasmania' })`, which handles DST automatically, plus on-page copy labelling the zone.

If you change `OFFSET`, also update the visible "AEST/AEDT (UTC+NN)" copy so the two agree.

### Styling

All CSS is inline in the `<head>`, driven by CSS custom properties under `:root` (dark theme: near-black background, muted slate text, a desaturated-cyan accent, JetBrains Mono for times). There is no external stylesheet or font load beyond the system font stack.

## Analytics & conversion tracking

Loaded in `<head>` and fired on a successful booking:

- **Google Analytics 4** — measurement id `G-ET0FJJS7C7`. A `booking_confirmed` event (with the chosen `slot`) fires on success.
- **LinkedIn Insight Tag** — partner id `8890876`. On a successful booking it records conversion `24583532` using **enhanced matching**: the visitor's email is lowercased and SHA-256 hashed in the browser (`crypto.subtle.digest`) before being sent as `li_em`. The cleartext email is never sent to LinkedIn.

These identifiers are public client-side tracking IDs, not secrets.

## Editing safely

Before changing `index.html`, read the **security invariants** in [`SECURITY.md`](./SECURITY.md). The short version: render all dynamic data with `textContent` (never `innerHTML`), don't introduce URL/hash/storage input sources, avoid dynamic code execution, and keep the LinkedIn email hashed.
