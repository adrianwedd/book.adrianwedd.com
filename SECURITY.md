# Security

This document records the security posture of `book.adrianwedd.com` and the result of the security review performed on 2026-05-29.

## Scope

The repository contains only the static frontend (`index.html`). The booking backend (`api.book.adrianwedd.com`) lives in a separate repository and is **not** covered here. Server-side validation, authentication, rate limiting, and storage of booking data are the backend's responsibility.

## Threat model

- The frontend is **untrusted client-side code**. It performs no authentication and stores no secrets. Anyone can read, copy, or tamper with it in their own browser — so the security boundary is the backend, not this page.
- All data the page sends (`name`, `email`, `slot`, `note`) must be treated as attacker-controlled by the backend.
- The page consumes one untrusted external input at runtime: the JSON response from `GET /slots`. It must remain safe even if that response is malicious (e.g. via a compromised or MITM'd API).

## Security review result (2026-05-29)

**No high-confidence vulnerabilities were found.** Data flow was traced from every input (form fields, the `/slots` API response, and the deliberate absence of any URL/hash/storage reads) to every DOM and network sink.

| Category | Result |
| --- | --- |
| XSS (DOM / stored / reflected) | No exploitable sink — see invariants below |
| Code execution / deserialization | No `eval`, `new Function`, or string-argument timers |
| Crypto & secrets | Correct SHA-256 use; no secrets in client code |
| Auth / authz | Out of scope (client-side; enforced by backend) |
| Data exposure | No PII logged or exposed beyond intended API call |

## Security invariants — preserve these when editing `index.html`

These are the properties that keep the page safe. Breaking any of them likely introduces a vulnerability.

1. **No HTML injection sinks.** Every value derived from user input or an API response is written with `textContent` and sent to the backend as JSON. The script must never use `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, or assign untrusted data to event-handler attributes. There is a comment at the top of the `<script>` block stating this — keep it true.
   - API `slots` values flow only to `fmt()` → `textContent` and into the booking payload (`index.html:279`).
   - User `email` flows only to `textContent` (`index.html:344`) and the hashed LinkedIn call.
2. **No untrusted input sources.** The page reads no URL parameters, `location.hash`/`search`, `window.name`, `postMessage`, cookies, or `localStorage`/`sessionStorage`. Do not introduce a reflected/DOM source without re-checking every sink.
3. **No dynamic code execution.** No `eval`, `new Function`, or string arguments to `setTimeout`/`setInterval`.
4. **Email is hashed before leaving for LinkedIn.** The email is SHA-256 hashed client-side (`crypto.subtle.digest`) before being passed as `li_em` (`index.html:348`). Never send the cleartext email to LinkedIn.

## Known non-issues (hardening, not vulnerabilities)

These were considered and intentionally not treated as vulnerabilities:

- **No CSP or SRI** on the third-party LinkedIn Insight Tag and Google Tag Manager scripts. This is a defense-in-depth gap; adding a `Content-Security-Policy` and SRI hashes would harden the page but is not required to be safe given the invariants above. (Note: GitHub Pages does not let you set response headers, so CSP would need a `<meta http-equiv>` tag.)
- **`novalidate` form with no client-side validation.** Backend validation is authoritative; client-side validation is UX only.

## Reporting

Found something? Email the issue to the address listed on [adrianwedd.com](https://adrianwedd.com) rather than opening a public issue.
