# ADR 004 — Error monitoring: existing cluster Loki, not Sentry SaaS (supersedes ADR 003)

Date: 2026-09-04 | Status: accepted, supersedes ADR 003

## Context
ADR 003 (same day, hours earlier) picked Sentry SaaS for error monitoring, on the condition that
scrubbing kept raw customer data from ever crossing the boundary to a third party. That was
implemented and verified (233 backend + 124 frontend tests, live `X-Request-ID` check).

Before wiring up Sentry projects, Sơn confirmed the Rancher cluster (`rancher-lake.desoft.vn`,
namespace `ok2ship`) already runs its own centralized logging service: **Loki + Grafana**. That
changes the calculus this ADR is about:
- **No new external dependency.** Sentry SaaS needed two new accounts/projects at a third party;
  Loki is already running, already operated by whoever runs the cluster, zero new signup.
- **The PII boundary moves.** ADR 003's whole scrubbing design existed because Sentry is outside
  the "approved environment" the Constitution's principle #3 draws around customer data. Loki
  runs on the *same* cluster as this app's own Postgres/MinIO — already inside that boundary. The
  strict "replace the message with just its `code`" scrub Sentry needed is no longer required.
- **New problem Sentry didn't have:** Loki only ingests container stdout (via the cluster's
  Fluent Bit/Promtail). It has no visibility into a browser tab. Sentry solved browser-side crash
  capture with a dedicated JS SDK reporting straight from the client; Loki needs this app to build
  its own bridge — a backend endpoint the frontend calls, which then logs server-side.

## Decision
1. **Sentry SaaS is fully removed** (`app/core/sentry.py`, `sentry-sdk`, `@sentry/react`,
   `src/lib/sentry.ts`, both `SENTRY_*`/`VITE_SENTRY_*` env vars) — replaced entirely, not kept
   dormant alongside the new approach (Sơn's explicit call, 2026-09-04).
2. **Backend: structured JSON logs to stdout** (`app/core/logging.py`). The cluster's existing
   Loki log pipeline already scrapes stdout — this app has no Loki-specific code at all, no push
   endpoint, no API key. `init_logging()` replaces the root logger's handler; every
   `logging.getLogger(__name__)` call in the app inherits it by propagation. Uvicorn's own
   `uvicorn.access` logger is a known, accepted exception — see `logging.py`'s docstring.
3. **No scrubbing of the message text.** Unlike ADR 003's `before_send`, an `AppError`'s real
   message (which may interpolate a username/email) is logged as-is — see Context above for why
   that's fine now. Still don't log raw passwords/tokens/cookies (basic hygiene, always true,
   independent of this decision).
4. **New: `POST /client-errors`** (`app/modules/observability/`), no auth, rate-limited
   (`app/core/rate_limit.py`, in-process sliding window, 20/min/IP) — the frontend's one bridge
   from a browser crash to a backend log line. No auth because a crash can happen before login
   (including inside the login flow itself); rate limiting instead of auth is the actual guard
   against abuse.
5. **Frontend**: `src/lib/errorReporting.ts` replaces `src/lib/sentry.ts` — `reportClientError()`
   POSTs to `/client-errors` directly via `fetch` (not through `apiClient`, to avoid a
   report-about-a-report loop if that endpoint itself ever 500s), fire-and-forget, truncates every
   field before sending. Wired into the same two places Sentry was: `ErrorBoundary.componentDidCatch`
   and `api/client.ts`'s 5xx response interceptor.
6. **Request-ID correlation kept**, mechanism changed: `app/core/logging.py`'s contextvar
   attaches `request_id` to every backend log line for a request instead of a Sentry tag; the
   frontend still echoes the `X-Request-ID` response header back on a report about a failed call.
   **`user_id` is attached the same way** (added 2026-09-06, Sơn — backend error logs answered
   "which request" but not "which person", so a reported error couldn't be traced to an account).
   Both are set in app/main.py's middleware from a best-effort *signature-verified* token decode,
   NOT in the `get_current_user` dependency that already decodes the same token. That placement
   looks redundant and isn't: FastAPI runs sync dependencies and sync routes via
   `run_in_threadpool`, which gives each a *copy* of the context — a `.set()` there is invisible
   everywhere else, including the catch-all 500 handler this field exists for. Measured, not
   assumed: the dependency version produced no `user_id` on any log line at all. Locked in by
   `tests/test_request_context_logging.py`, which crashes a real route through the real middleware
   rather than calling the setter directly (a direct-setter test passes just as happily with the
   broken wiring). The signature check matters even on this log-only path: without it, a forged
   token would let anyone stamp someone else's `user_id` onto their own errors.
7. **Source maps**: `vite.config.ts`'s `sourcemap: 'hidden'` (added for Sentry's symbolication)
   is removed — nothing consumes it now. Stack traces reported to `/client-errors` stay minified;
   revisit only if that turns out to actually matter in practice.
8. **`ErrorBoundary` itself is unaffected** — it exists independently of where crashes get
   reported (the blank-page fix, same day, earlier). Only its `componentDidCatch` reporting call
   changed.
9. **Every 4xx is logged** (added 2026-09-06). Before this, no 4xx was logged at all — a failed
   login, a locked account or a permission denial left nothing in Loki, only in the `audit_log` DB
   table (which covers business/security events, needs a database query to read, and doesn't cover
   permission denials at all). 404/409/422 were initially excluded as "ordinary outcomes nobody
   would ask Loki about"; Sơn overruled that on a concrete counterexample — a BA hit a 409 on the
   real deployment and there was no way to find out what it had been — with the explicit
   understanding that volume gets tightened later if it becomes a problem.
   **The level split is the load-bearing part**: access tokens live ~2 minutes by design, so
   routine "token expired → silent refresh" 401s (`AUTH_TOKEN_MISSING`, `AUTH_TOKEN_INVALID`,
   `AUTH_REFRESH_COOKIE_MISSING`, `AUTH_REFRESH_TOKEN_EXPIRED`) outnumber genuine failures by
   orders of magnitude and log at **INFO**, as do all 404/409/422; failed login, locked/inactive
   account, refresh-token reuse detection and permission denial log at **WARNING**. Sharing one
   level would make a "today's problems" query return thousands of normal refreshes and hide the
   handful of real events, the exact failure mode that makes monitoring useless in practice.
   A URL with no route at all never reaches these handlers (Starlette answers its own 404), so
   bots probing random paths can't inflate the volume — verified, and pinned by a test.
9b. **Pydantic 422s log field names and error types only, never `exc.errors()` wholesale.**
   Pydantic v2's error dicts embed the submitted value under `input` — verified against this app's
   own `/auth/reset-password`, where a rejected weak password appears there in the clear. Logging
   those dicts as-is would write real user passwords into Loki permanently. ADR 004 relaxed
   *message* scrubbing (Loki is in-cluster, so ordinary PII is acceptable there), but passwords and
   tokens were always outside that relaxation — `app/main.py`'s `_validation_failure_summary`
   builds `"body.new_password:value_error"` instead, keeping the debugging value without the
   value itself. Pinned by a test that asserts the real password string is absent from the raw
   log output.
10. **Alerting**: deferred, same as ADR 003 — Grafana Alerting (rules on LogQL queries) to
    Slack/Telegram, configured in the Grafana UI once real traffic through this exists. No code
    change needed for that step either.

## Rationale
- Reusing infra the team already runs and pays for beats paying for and operating a second,
  redundant thing (Sentry) — especially once the harder condition (PII must never leave the
  environment) turns out to not even apply to the in-cluster option.
- Removing Sentry outright rather than keeping it dormant: half-wired, unused SaaS integration
  code is dead weight that looks live (imports, config keys, docs) and invites someone reaching
  for it later without re-deriving why it was dropped. If Sentry (or another SaaS APM) is ever
  wanted again, that's a fresh decision, not a flag flip.
- `/client-errors` with no auth, not behind `get_current_user`: the alternative (requiring a
  valid access token) would silently swallow reports for exactly the crashes most worth seeing —
  a crash before login, or a crash inside the login flow itself.

## Consequences
- **This app now depends on the cluster's Loki actually working** — if the cluster's own log
  pipeline breaks or isn't scraping this app's pods, there's no fallback (Sentry's SaaS reliability
  wasn't dependent on our own infra). Not a new risk exactly — `kubectl logs` always depended on
  the same pipeline being healthy — but this is the first case where losing it also breaks the
  *intentional* monitoring path, not just ad-hoc debugging.
- **No dedicated error-tracking UI.** Grafana/LogQL is a general log browser, not Sentry's
  purpose-built issue view (automatic grouping/deduplication, per-issue history, one-click
  resolve). Building any of that on top of Loki, if it's ever wanted, is a separate follow-up.
- **Frontend stack traces stay minified** in the logged report (no source-map upload pipeline) —
  a real cost when actually debugging a reported frontend crash; matches what DevTools already
  showed before any of this work, so not a regression, just not improved either.
- **Rate limiter is per-pod, not cluster-wide** (`app/core/rate_limit.py`'s own docstring) — the
  effective `/client-errors` ceiling scales with replica count. Acceptable for now; a Redis-backed
  limiter is the fix if that ever matters.
- ADR 003 is now historical — its Sentry-specific decisions are void, but its record of *why*
  the PII question needed asking at all (Constitution principle #3) still stands and is what this
  ADR reasons from.

Related: `app/core/logging.py`, `app/core/rate_limit.py`, `app/modules/observability/`,
`app/main.py`, `frontend/src/lib/errorReporting.ts`,
`frontend/src/components/common/ErrorBoundary.tsx`, ADR 003 (superseded), Constitution engineering
principle #3.
