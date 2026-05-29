# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page booking site served at `book.adrianwedd.com` where visitors book a 30-minute call. The entire frontend is one file: `index.html` (HTML + inline CSS + vanilla JS, no framework, no build step, no dependencies, no `package.json`).

For the full picture see [`README.md`](./README.md) (architecture, API, deploy, analytics) and [`SECURITY.md`](./SECURITY.md) (threat model and the invariants below). This file is the quick reference.

## Develop / preview / deploy

- **Preview locally:** open `index.html` in a browser, or `python3 -m http.server` then visit `localhost:8000`. There is no build, lint, or test tooling.
- **Deploy:** push to `main`. The site is GitHub Pages, mapped to the domain via `CNAME`. There is no CI pipeline.

## Architecture

The page is a 3-step state machine (`.step` divs toggled by `showStep()`): **calendar → details form → confirmation**. Only one `.step.active` is visible at a time.

The frontend is decoupled from a separate backend at `API = https://api.book.adrianwedd.com` (**not in this repo**). Two endpoints:
- `GET /slots?date=YYYY-MM-DD` → `{ slots: ["HH:MM", ...] }` — availability for a day.
- `POST /book` with `{ name, email, slot, note }` → 200 on success. `slot` is a full ISO timestamp with offset (`selectedSlotIso`), the others are the only data the API receives.

## Conventions that matter

- **XSS safety is deliberate.** All user/API-supplied values are rendered via `textContent` or sent as JSON — never interpolated into HTML markup or `innerHTML`. Preserve this when touching rendering code (see the comment at the top of the `<script>`).
- **Timezone is hardcoded.** `OFFSET` (top of script) is the Tasmania UTC offset and must be flipped manually for daylight saving: `+11:00` (AEDT) vs `+10:00` (AEST). Display formatting uses `Australia/Tasmania` via `toLocaleString`. The "AEST/AEDT" copy and the `OFFSET` value must stay consistent.
- **Weekends and past dates are non-bookable** in calendar rendering; only weekday cells get the `available` class and a click handler.

## Analytics & tracking (in `<head>` and on booking success)

- **GA4** id `G-ET0FJJS7C7`; fires a `booking_confirmed` event on successful booking.
- **LinkedIn Insight Tag** partner id `8890876`; on booking it fires conversion `24583532` with enhanced matching — the email is SHA-256 hashed client-side (`crypto.subtle.digest`) before being passed as `li_em`. Keep the email hashed; never send it in clear text to LinkedIn.
