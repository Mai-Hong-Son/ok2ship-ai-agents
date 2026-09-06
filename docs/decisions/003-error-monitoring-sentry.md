# ADR 003 — Error monitoring: Sentry SaaS, with local PII scrubbing before send

Date: 2026-09-04 | Status: **superseded by ADR 004** (same day — the cluster turned out to already
run its own Loki+Grafana logging service, changing the tradeoff this ADR was built around). Kept
for the record of *why* the PII question needed asking in the first place; none of the Sentry-
specific decisions below are still in effect.

## Context
Neither stack had any error visibility before this. Confirmed by reading the code, not assumed:
backend had structured 4xx handlers (`app/main.py`) but **no handler at all for an unhandled
exception** — it fell through to Starlette's bare 500, logged nowhere except a pod's stdout at
the moment it happened. Frontend had zero error boundaries anywhere (fixed separately the same
day — see `ErrorBoundary.tsx`), so a render crash blanked the whole page with nothing recorded.
This surfaced concretely chasing a stray 409 on `/users` (Sơn, 2026-09-04): the real bug (a
double-submit race) was never the risk — the blank page it exposed, with zero trace anywhere,
was.

The product handles real factory QA data (Mektec/Desoft), and the hub Constitution's engineering
principle #3 says customer data never leaves an approved environment. Any error-monitoring tool
choice has to be evaluated against that before anything else — a request body, an exception
message built from user input, or a stack trace can all carry real data.

## Decision
1. **Sentry SaaS** (sentry.io), not self-hosted (e.g. GlitchTip on the existing Rancher cluster).
   Sơn's explicit call after the self-host/SaaS tradeoffs were explained, on one hard condition:
   **no raw customer data leaves the process, ever** — enforced in code, not by policy alone.
2. **Errors only**, not full request/access logging — no performance/session tracing, no
   `BrowserTracingIntegration`, `traces_sample_rate=0.0` on both sides. A separate decision if
   full observability (every request, not just failures) is ever needed later.
3. **Scrub locally, before anything leaves**, on both sides:
   - Backend (`app/core/sentry.py`, `before_send`): strips `request.data`/`.cookies`/most
     `.headers` from every event unconditionally. Separately, for the app's own `AppError`
     hierarchy specifically, replaces the exception's message text with just its `code`
     (`error_codes.py`) — several existing raise sites interpolate real request data into the
     message for readability (e.g. `f"Username '{data.username}' is already taken"`), and `code`
     is the stable, PII-free identifier already built for machine consumption. A genuinely
     unexpected exception (not an `AppError` — a real bug) keeps its real message: those don't
     interpolate user input by convention, and the debug value outweighs the narrower residual
     risk for that class of event.
   - Frontend (`src/lib/sentry.ts`, `beforeSend`/`beforeBreadcrumb`): same request-body/cookie/
     header strip; XHR/fetch breadcrumbs (one per failed `apiClient` call) are reduced to an
     allowlist of `method`/`url`/`status_code`/`from`/`to`.
   - `send_default_pii` / `sendDefaultPii` explicitly `False` on both sides (also the SDK
     default — stated explicitly here so it reads as a decision, not an accident).
4. **User context = `user_id` only**, never username/email — set from the JWT's `sub` claim
   (frontend: `AuthContext.tsx`'s single token-change subscription; backend: not set at all
   today, nothing currently threads the authenticated principal into the Sentry scope).
5. **Request-ID correlation**: `app/main.py` middleware generates a UUID per request, echoes it
   as `X-Request-ID`, tags it onto the request's Sentry scope. The frontend reads that header
   back on any 5xx and tags its own captured event with it — one ID finds both sides of the same
   request.
6. **Alerting**: Slack/Telegram (Sơn's choice) — not wired up in this pass; needs a webhook URL
   created in whichever channel, configured in the Sentry project UI directly (no code change).
7. Added alongside this: a catch-all `@app.exception_handler(Exception)` (backend) so an
   unhandled exception gets a structured `{"detail", "code": "INTERNAL_SERVER_ERROR"}` response
   instead of Starlette's bare 500 — independent of whether Sentry is even configured.

## Rationale
- SaaS was chosen over self-host for setup speed and zero ongoing ops burden (no team to run a
  GlitchTip deployment) — viable specifically *because* the scrubbing above makes the
  environment-boundary concern moot at the data level, not because the concern doesn't apply.
- Scrubbing at `before_send`/`beforeBreadcrumb` (client-side, pre-transport) rather than relying
  on Sentry's own server-side "Data Scrubber" (regex-based, runs after the event has already
  reached their servers) — the whole point is that unscrubbed data never crosses the boundary in
  the first place, not that it gets redacted a moment after arriving.
- Replacing an `AppError`'s message with its `code` (rather than trying to regex-detect which
  words in a free-text message are someone's real data) was chosen because it's exact, not
  heuristic — every such message's PII risk comes from deliberate string interpolation at a known
  set of raise sites, and `code` already exists as the safe alternative for exactly this reason
  (frontend already never shows the raw message to a user either — see `apiError.ts`).
- Scope kept to errors only (not full logs) because that's the actual current pain point (zero
  visibility into failures) — full request-level observability is a materially bigger tool
  (Loki/Promtail-class) with its own scrubbing surface, better decided separately if the need
  actually shows up.

## Consequences
- **Source maps**: `vite.config.ts` now builds with `sourcemap: 'hidden'` (maps written to disk,
  not referenced by the shipped JS) — but nothing uploads them to Sentry yet. Without that,
  frontend stack traces in Sentry stay as minified positions (`index-CkLdJ8vy.js:32`) — exactly
  what made the original 409 hard to read in DevTools. Wiring `@sentry/vite-plugin` to
  auto-upload needs a Sentry auth token (a value only Sơn can create, at sentry.io) — follow-up,
  not done in this pass.
- **Two Sentry projects needed**: one for the FastAPI backend (Python DSN), one for the React
  frontend (JS DSN) — both currently read from empty-string-default settings
  (`SENTRY_DSN`/`VITE_SENTRY_DSN`, see each repo's `.env.example`), so nothing is monitored in
  any environment until Sơn creates the projects and sets both.
- **Scrubbing is a discipline, not a one-time guarantee**: any *new* `AppError` raise site that
  interpolates request data into its message is automatically safe (still just reports `code`).
  A *new* kind of event context (a manually-attached `extra`/`tags` value somewhere) is NOT
  automatically covered by the current allowlists — adding one that carries PII needs the same
  scrubbing discipline applied deliberately, not assumed.
- Alerting (Slack/Telegram) still needs to actually be configured in the Sentry project UI once
  the projects exist — no code change required for that step, but it doesn't happen by itself.
- Full request/access logging (every request, not just errors) is explicitly out of scope for
  now — revisit as a separate decision if that need surfaces.

Related: `app/core/sentry.py`, `app/main.py` (middleware + catch-all handler),
`frontend/src/lib/sentry.ts`, `frontend/src/components/common/ErrorBoundary.tsx`,
`docs/design/template-management.md` (unrelated module, but the same "customer data never leaves
the approved environment" principle this ADR is built around), Constitution engineering
principle #3.
