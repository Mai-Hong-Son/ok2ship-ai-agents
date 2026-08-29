# Progress log — ok2ship-ai

The full phase-by-phase build history: every bug found, every decision's rationale, dated,
newest-first. `../HANDOFF.md` is the short version (current state + what's still open) — an agent
resuming work reads `HANDOFF.md` first, and comes here only to look up how/why something specific
was done (2026-08-28: split apart once `HANDOFF.md` grew to ~900 lines of mostly-finished history).
For the User Management schema and locked design, see `design/user-management.md`.

## Status at a glance
- **Module status: User Management & Permission Assignment (WBS #5) — ✅ complete, signed off by
  Sơn (2026-08-29).** First module of the product. Next module not chosen yet — see
  `../HANDOFF.md`'s "Next steps" #0.
- **Live in production**: `ok2ship-dev.desoft.vn`, deployed via a GitLab CI/CD pipeline that
  build+deploys automatically on every push to `main` (both repos) — see 2026-08-26's entry.
  **Branch/PR discipline is self-adopted, not yet GitLab-enforced** (2026-08-28 decision) — see
  `../HANDOFF.md`'s "Next steps" for the exact status/plan and the 3 MRs still open as of sign-off.
- **Repo state:** three separate repos (`products/ok2ship-ai/` docs/planning on GitHub;
  `backend/`/`frontend/` on GitLab), all pushed. Backend 185 tests, frontend 102 tests, both
  green. See `../HANDOFF.md`'s "Repo topology" for URLs/latest commit SHAs.

| Phase | What | Status |
|---|---|---|
| 0 | Backend scaffold (FastAPI app, config, DB session, Alembic wiring) | ✅ Done |
| 0.5 | Domain-based folder architecture + package skeleton | ✅ Done |
| 1 | DB migration (8 tables) + seed permissions/roles + bootstrap Admin | ✅ Done |
| 2 | Auth core (login, tokens, `can()` permission check, auto-lock job) | ✅ Done |
| 3 | User CRUD APIs (List/Detail/Create/Edit/Delete/Active-Inactive) | ✅ Done |
| 4 | Real email delivery | ✅ Done |
| 4.5 | Forgot/reset/change password (gaps found via mockup review) | ✅ Done |
| 5 | Frontend (login + User Management screens) | ✅ Done |
| 6 | Integration + qa-reviewer pass + merge | ✅ Module signed off (08-29) — branch/PR still self-adopted only, not GitLab-locked (carries into future modules, see HANDOFF) |
| — | k8s deploy + CI/CD (not an original phase, added once infra was ready) | ✅ Live (08-25/26) |

## Log

### 2026-08-29 — Performance fix + email/doc cleanups; User Management (WBS #5) signed off complete
**Performance fix, caught live by Sơn**: creating/editing/resending activation for a user (and
forgot-password) each waited a real 1-3s inside the API request for `send_activation_email()`/
`send_password_reset_email()` to open an SMTP connection to Gmail, TLS-handshake, authenticate,
and send — all before the response went back to the browser. Fixed with FastAPI's built-in
`BackgroundTasks` (no new infrastructure — a real task queue would be overkill at this volume):
the response now returns as soon as the DB work commits, the email send runs after. Trade-off,
confirmed with Sơn first: `UserResponse.activation_email_sent` (added 2026-08-28 specifically to
warn the Admin in the moment when delivery failed) can no longer be known synchronously — always
`None` for create/update/resend now; a genuine failure is still logged and still recoverable via
"Gửi lại email kích hoạt", just not surfaced in the moment. `request_password_reset` already
returned the same generic response either way (anti-enumeration), so backgrounding it there cost
nothing that wasn't already true. New tests confirm the actual scheduling behavior (BackgroundTasks'
own task list holds the real function + kwargs, never invoked inline), not just "the response still
works either way." Verified live: a real `POST /users` went from ~1-3s to 62ms; a Playwright
click-through (Xác nhận → success toast) timed at 0.2s. 185/185 backend tests passing.

**Two small content/doc cleanups, same day, both Sơn's direct requests:**
- Dropped the "Nếu nút trên không hoạt động, hãy sao chép liên kết sau vào trình duyệt" fallback
  paragraph (+ repeated raw link) from both HTML email templates — button only now. The
  plain-text half of each email is unaffected, still carries the raw link on its own (the real
  fallback for a client that can't render HTML at all).
- Removed the "Git hooks" section from `frontend/README.md` (Sơn: applies to him only, not asking
  Le Bui to change anything) — `scripts/git-hooks/pre-commit` and the already-installed local hook
  are unaffected, only the install instructions are gone from the README.

Both landed on their own feature branches (`feature/email-drop-fallback-link`,
`docs/drop-git-hooks-readme-section`), same branch/PR habit as the day before — see item 1 below.

**User Management & Permission Assignment (WBS #5) signed off complete by Sơn** — the first
module of the product. `HANDOFF.md` and this file's "Status at a glance" both updated to reflect
it; 3 MRs were still open at sign-off time (2 backend, 1 frontend — see `HANDOFF.md`'s "Next
steps" #1), not yet merged. A `/project-retro` was run the same day — see the entry below (or
above, depending how it landed) for whatever lessons got approved into the hub.

### 2026-08-27 — Activation / password-reset emails now carry clickable frontend links
**Done:**
- Emails previously mailed a bare token; recipients had no way to use it because the frontend
  pages (`/activate`, `/reset-password`) only read `?token=` from the URL. Both emails now send
  a full link built from a new `FRONTEND_BASE_URL` setting (deliberately separate from
  `CORS_ALLOWED_ORIGINS` — that is a list of API callers, emails need one canonical origin).
- Multipart/alternative (plain text + HTML with a CTA button). Recipient-facing copy is Vietnamese
  (matching the frontend UI; Mektec factory staff), kept in `email_templates/` so Python source
  stays English per the Constitution.
- Product display name corrected to **OK2SHIP AI** (was "OK2Ship AI Check" in email subjects /
  `smtp_from_name` / a few docs) — the frontend already used the correct name.
- `ACTIVATION_TOKEN_TTL_DAYS` consolidated into `verification.py` (was duplicated in
  `users/service.py` and `cli/bootstrap_admin.py`; email templates now also need the number).
- Tests for stub/SMTP/multipart/trailing-slash/HTML-escaping/failure-fallback-with-link.
- Deferred: async/background send (still sync with 10s timeout — fine at current user-create
  volume; revisit if bulk create lands).

**Flagging:**
- Cluster Secret `ok2ship-backend-env` still needs `FRONTEND_BASE_URL=https://ok2ship-dev.desoft.vn`
  patched in before the next deploy that sends real mail — without it, localhost links go out.
- BA should glance at the Vietnamese email copy before customer-facing use.

### 2026-08-28 — Reviewed the above; closed a compose-failure gap; surfaced send failures to the Admin
**Reviewed** the 2026-08-27 email work (backend + frontend repos, live SMTP send verified against a
real Gmail inbox — HTML + plain-text bodies, clickable links both worked end to end). Two things
fixed in the same pass:
- **`_render()` (template load + fill) used to run outside any try/except** — a broken template
  (missing file, typo'd `$placeholder`) would have raised straight out of `send_activation_email()`
  *after* the caller's DB change had already committed, surfacing as a misleading 500 on an
  otherwise-successful create/update/resend. Now caught and logged the same way an SMTP failure
  already was. New regression tests for both `send_activation_email`/`send_password_reset_email`.
- **Email-send outcome was silently swallowed** — an Admin creating a user (or resending
  activation) had no way to know the account itself succeeded but the activation email didn't.
  Both email functions now return `bool`; `create_user`/`update_user`/`resend_activation` propagate
  it as a new `activation_email_sent: bool | None` on `UserResponse` (`None` = no email attempted,
  e.g. an update that didn't touch email). `send_password_reset_email`'s return value is
  deliberately ignored by its one (unauthenticated) caller — surfacing it there would let an
  attacker enumerate which emails exist, so a failed reset-email looks identical to a successful
  one from the outside (confirmed with Sơn, 2026-08-28). Frontend shows a new amber "warning" toast
  (create/edit/resend) pointing at "Gửi lại email kích hoạt" when this comes back `false`, instead
  of the plain green success copy.
- `create_user`/`update_user`/`resend_activation` (backend service layer) now return
  `(User, bool | None)` instead of a bare `User` — updated ~20 call sites across router + tests.
- **Process finding, worth remembering for every future session on this repo**: `npx tsc --noEmit`
  is a silent no-op here — the root `tsconfig.json` is `files: [] + references`, which plain `tsc`
  never traverses (only `tsc -b`, project-reference *build* mode, does). Confirmed by deliberately
  planting a type error and getting a clean exit from `--noEmit`. `npx tsc -b` is the one that
  matches `npm run build`'s real command and actually catches errors — it immediately found 4 test
  fixtures missing the new required field, fixed here. Every `tsc --noEmit` "clean" claim earlier
  in this product's history should be treated as unverified.
- Checked the live cluster (`kubectl -n ok2ship describe secret ok2ship-backend-env`, run by Sơn):
  confirmed as of 2026-08-28 the Secret still has neither `FRONTEND_BASE_URL` nor any `SMTP_*` key
  — production is still stub-only (prints to pod logs). Sơn is patching both now, using the same
  Gmail App Password already verified for local dev as an interim production SMTP account (his
  call — a shared company mailbox is the better long-term answer, tracked as a follow-up, not
  blocking).

182/182 backend tests, 100% coverage on `email.py`; 102/102 frontend tests; `tsc -b` (the real
check) clean; both repos committed and pushed.

### 2026-08-25 — Mockup-fidelity batch (10 commits) + first retroactive qa-reviewer audit
Sơn reviewed the live app against the mockup and caught a string of drift points, fixed same-day:
header height/padding corrected to the mockup's real `.ant-header` (was h-14/px-5, mockup measures
h-16/px-6); Forgot Password's email validated client-side + echoed in the success message; "Ghi
nhớ đăng nhập" wired through `login()` for real (checked = 30-day persistent refresh cookie,
unchecked = real browser-session cookie); deleted-account login now reports "tài khoản đã bị xóa"
distinctly from "vô hiệu hóa" (`AUTH_ACCOUNT_DELETED` code added); every modal gained real open/
close animations (found a genuine Chromium bug: reusing one keyframe with only `animation-
direction` toggled doesn't reliably restart it — needed a distinct second keyframe); Add/Edit User
validated against the vendor's field-spec table for the first time (max lengths, mandatory role,
both frontend and backend); the user-detail modal and Delete/status-toggle/Logout dialogs were
being built through the wrong shared component (`Modal.tsx`'s header+X shape) — rebuilt via a new
`ConfirmModal.tsx` matching the mockup's actual icon+title+desc shape; a stale-grid-cell bug (AG
Grid's default change detection missing `is_deleted`/`status` changes on cells reading fields
outside their own column) fixed generally via `refreshCells({force:true})` after two narrower
point-fixes proved insufficient; Delete disabled for an already-deleted row; a distinct "Đã xóa"
badge + grid filter option added (removed again on 2026-08-27, see below); the name column gained
its missing avatar. Backend: `max_failed_login_attempts` 5→20, and race-safe case-insensitive
username uniqueness (functional index + `IntegrityError` translation).

**Retroactive qa-reviewer audit (Phase 6) — first time this gate actually ran.** Everything above
(and everything since the initial import) had landed straight on `main`, no branch/PR/qa-reviewer
step. Simulated via 2 parallel general-purpose subagents each carrying the real `qa-reviewer.md`
persona (the literal `qa-reviewer` subagent type isn't registered in this environment), one per
repo, read-only. Findings fixed same day:
- **[Backend, Major]** `UserUpdate` had no field constraints (only `UserCreate` did) —
  `PATCH /users/{id}` with `role_codes: []` silently stripped every role from a user on Edit.
- **[Backend, Minor]** Unbounded password length on login/change-password fields checked against
  an existing hash — CPU-DoS via forced argon2id work; capped at 128 chars.
- **[Backend, Minor]** The `IntegrityError` race-condition safety net was only tested via its
  standalone translation helper, never the real `create_user`/`update_user` code path.
- **[Backend, Minor]** Constraint-name matching used substring search on the raw driver message
  instead of psycopg's structured `diag.constraint_name`.
- **[Frontend, Major]** The single-select `RoleSelect` (Sơn's earlier "match the mockup, only 2
  roles exist" call) could silently truncate a user already holding >1 role down to 1 on Edit —
  fixed with a lock-to-read-only-chips guard rather than reverting the single-select decision.
- **[Frontend, Minor]** Two unhandled-promise-rejection sites (`RoleSelect`'s roles fetch,
  `Header`'s logout).
- **[Frontend, Major]** No component-level tests existed at all — partially addressed
  (`Header`/`RoleSelect`/`UserFormModal` tests covering the fixes above); full page-level flow
  coverage followed on 2026-08-26.

Le Bui also shipped the first k8s deploy artifacts today (Dockerfile + manifests, both repos) —
see 2026-08-26's entry for the CI/CD pipeline that followed.

### 2026-08-26 — CI/CD pipeline live; page-level test coverage backfilled
Le Bui built and shipped Kubernetes deploy infrastructure end-to-end, live at
`ok2ship-dev.desoft.vn` (Rancher cluster `rancher-lake.desoft.vn`, namespace `ok2ship`):
Dockerfiles, k8s manifests (Deployment/Service/Ingress, migration + bootstrap-admin Jobs), and a
GitLab CI/CD pipeline (Kaniko build, namespace-scoped `ci-deployer` ServiceAccount) that
build+push+migrate+deploys automatically on every push to `main`, both repos. CI's own `test`
stage is disabled — the runner has no Postgres available for `tests/test_health.py` — a
deliberate trade-off to unblock build/deploy, still tracked as open in `HANDOFF.md`.

Backfilled the frontend's missing page-level test coverage (flagged by the qa-reviewer audit the
day before): `LoginPage.test.tsx` (render/validation/submit, the `location.state.from` redirect
regression) and `UserManagementPage.test.tsx` (create/edit/delete/toggle/resend-activation against
the real AG Grid component). Also fixed `Header.test.tsx`'s mocked profile to satisfy
`UserResponse`'s full shape (caught by `tsc -b`, part of `npm run build`, after a vendor
field-spec change widened the type — an early sign of the `tsc --noEmit`-is-a-no-op issue found
and root-caused two days later, 2026-08-28).

### 2026-08-27 — Responsive layout; 4 grid usability fixes; real favicon; deleted users hidden server-side
**Responsive layout.** The app shell had zero mobile support below `lg:` (1024px) outside the
auth pages — a fixed 232px sidebar always visible ate ~60% of a phone's width, squeezing the grid
to ~106px and wrapping the breadcrumb mid-word. `Sidebar.tsx` becomes an off-canvas drawer below
`lg:` (fixed + translate-x, backdrop click to close, closes on nav); `Header.tsx` gained a
hamburger toggle + breadcrumb truncation + a capped/hidden display name (caught live: an
unconstrained name wrapped 3 lines and pushed the header taller). Verified at 390/768/1440px —
desktop pixel-identical to before. Confirmed with Sơn: the grid itself stays plain AG Grid
horizontal-scroll on mobile, not a card-list redesign (a separate, larger task if wanted later).

**4 requested grid fixes:** pagination page sizes `[6,12,24]` → `[10,20,50,100,200]`; Email/Tên
đăng nhập cell vertical-alignment (was top-aligned, missing the `flex h-full items-center`
`NameCell` already had); Role/Trạng thái filters converted from single-select dropdown to
multi-select checkboxes (`SelectColumnFilter.tsx` rewritten around a `{values: string[]}`
OR-matching model, toggle logic split into a pure, unit-tested `multiSelectFilter.ts`); deleted
users hidden from the list **entirely**, not just via a filter — decided with Sơn specifically
because it superseded the previous day's "Đã xóa" filter option (removed, since no row could ever
match it again). Backend: `list_users()` now unconditionally excludes `is_deleted=true` — still a
soft delete under the hood (`get_user(id)`/`audit_log` unaffected), only the list view hides them.

**Favicon.** Was the default Vite scaffold icon (a purple lightning bolt), unrelated to the
product — replaced with a crop of `logo.png`'s own geometric mark (the "MEKTEC" wordmark
excluded, unreadable at favicon scale), since no dedicated OK2SHIP logo file exists separately
from Mektec's.

**Email delivery rework** drafted locally today (activation/reset emails: bare token → clickable
link + HTML template) — see the entries above (2026-08-27/28) for the reviewed, finished version.

### 2026-08-19 — Backend folder architecture locked
**Done:**
- Proposed and got approval on a domain-based (feature-package) folder architecture for
  `backend/app/`, instead of a classic layered (`models/`, `schemas/`, `crud/` mixed together)
  structure — chosen because the product grows module-by-module (User Management now, AI
  Detection/SPC/Rule Engine/Alerts/Dashboards later) and each is a mostly-separate domain.
- Created the package skeleton (empty `__init__.py` only — real code is Phase 1):
  `app/core/` (security, permissions choke point, FastAPI deps, exceptions — cross-cutting,
  shared by every module), `app/modules/users/`, `app/modules/auth/`, `app/modules/audit/`
  (audit_log — shared by every module going forward), `app/jobs/` (scheduled jobs, e.g. Phase 2's
  daily auto-lock), `app/cli/` (management commands, e.g. Phase 1's bootstrap-admin), plus mirrored
  `tests/modules/users/`, `tests/modules/auth/`.
- Decided: no separate `repository.py` layer (service talks to the SQLAlchemy session directly —
  YAGNI at this scale); DB session stays sync (not async) — dashboard app, not high-concurrency.

- Stood up local Postgres via `docker-compose.yml` (`postgres:16-alpine`). Host port **5433**, not
  the default 5432 — 5432 was already taken by another project's container on this machine.
  `.env` / `.env.example` / `app/config.py` default all updated to match. Connection verified
  (`select version()` round-trip through `app.db.engine` succeeded).

- Wrote up the folder architecture decision as a proper design reference:
  `docs/design/backend-architecture.md` — directory tree, rationale (domain-based vs. layered), the
  rule for adding a future module, and the no-repository-layer / sync-SQLAlchemy / port-5433
  decisions. Linked from `HANDOFF.md`.

**Next:**
- Ready to start Phase 1 (models + migration + seed + bootstrap admin) against this DB.

### 2026-08-19 — Phase 1: models, migration, seed, bootstrap admin
**Done:**
- Wrote all 8 tables as SQLAlchemy models across `app/modules/users/`, `app/modules/auth/`,
  `app/modules/audit/` — matches `docs/design/user-management.md` exactly.
- `app/core/security.py` — password hashing (argon2id) and token hashing (SHA-256) helpers.
- Two Alembic migrations: create tables, then seed baseline `permissions`/`roles`/`role_permissions`
  (`admin` → `user.manage` + `report.view`; `qa` → `report.view` only).
- **Bug caught and fixed**: Postgres enum types are orphaned by autogenerated `downgrade()` (not
  dropped when their table is dropped) — a later `upgrade` then fails with "type already exists".
  Fixed by explicitly dropping the 3 enum types in `downgrade()`. Verified with a full
  upgrade→downgrade→upgrade cycle before moving on.
- `app/cli/bootstrap_admin.py` — idempotent, creates the first Admin account in `Create` status
  (forces the activation flow rather than handing out a real password), issues an activation
  token (printed to stdout — no email sending yet), writes an audit log entry.
- 12 tests, 96% coverage on `app/`, run against a dedicated real Postgres test DB (auto-created
  and migrated fresh each test run).
- Dev DB now has real seed data + one bootstrapped Admin (status `Create`) — ready for Phase 2's
  login/activation flow to build against.

**Next:**
- Phase 2: auth core (login, tokens, `can()` permission check, daily auto-lock job).
- Still need sign-off on the very first commit (still staged, nothing committed to `main` yet).

### 2026-08-19 — Phase 2: auth core
**Done:**
- Login (`POST /auth/login`), refresh (`POST /auth/refresh`, cookie-based rotation), logout
  (`POST /auth/logout`).
- Short-lived access token (~2 min) with roles+permissions embedded as claims — chosen explicitly
  over a Redis-backed live status check, so Lock/Inactive force-logout stays near-instant without
  a DB hit on every request.
- `can(principal, permission_code)` — the one required permission choke point — plus
  `resolve_user_permissions()`, written as two separate queries on purpose (not one flattened
  join) to avoid the duplicated-row "JOIN fan-out" trap.
- Refresh-token rotation with reuse detection: replaying an old, already-rotated token revokes
  every refresh token for that user (whole-session kill on a theft signal) — its own tested
  deliverable, not folded silently into rotation.
- Daily auto-lock job (`app/jobs/auto_lock.py`) for the 90-day-idle half of auto-Lock (the
  failed-login half is handled inline during login).
- Every auth outcome writes `audit_log`.
- **Found and fixed while testing**: default JWT secret was 16 bytes, tripping PyJWT's own
  `InsecureKeyLengthWarning` for HS256 — lengthened the placeholder to ≥32 bytes (still an
  obvious "REQUIRED-before-deploy" placeholder, not a real secret).
- 43 tests total (31 new), 96% coverage on `app/`.

**Flagging for sign-off (not in the original design doc, invented to keep moving):**
- `max_failed_login_attempts = 5` — reasonable default, not vendor-confirmed.
- Exact UX when a login attempt hits a Locked/Inactive/Create-status account — currently returns a
  clear "account not usable" error only *after* the password is verified correct (avoids leaking
  lock status to a wrong-password guesser). Worth a look before Phase 5 builds the login screen
  around it.

**Next:**
- Phase 3: User CRUD APIs (List/Detail/Create/Edit/Delete/Active-Inactive), gated by
  `require_permission("user.manage")`.
- Still need sign-off on the very first commit (still staged, nothing on `main` yet).

### 2026-08-20 — Phase 3: User CRUD APIs
**Done:**
- All 6 vendor SRS use cases: `GET/POST /users`, `GET/PATCH /users/{id}`,
  `PATCH /users/{id}/status` — every route requires `user.manage`.
- **Decision**: Delete and the Active/Inactive toggle are the same endpoint — `Inactive` is the
  soft-delete state, no separate delete route (extends decision #6's "no separate unlock action"
  logic one step further). Flagging for sign-off, not silently assumed.
- Username format validated at the schema layer (regex), not the DB — matches the design doc.
  `UserUpdate` structurally has no `username` field at all, so immutability doesn't rely on a
  runtime check catching an attempt to change it.
- Editing email: invalidates prior unused verification tokens (marked used, not deleted — keeps
  the audit trail), issues a fresh one, resets status to `Create` — vendor SRS UC-04 BR-02.
- No-op updates (nothing actually changed) write no audit entry — keeps `audit_log` free of noise.
- Stub email sender factored into its own function (`app/modules/users/email.py`) and reused by
  `bootstrap_admin.py` too, instead of duplicating the same stub print logic in two places.
- **Bug caught while testing (test-only)**: several tests used a random UUID as the "acting admin"
  for an audit-log write, which violates `audit_log.user_id`'s foreign key to a real `users` row.
  Real code never hits this (the actor always comes from a logged-in principal), but it was a
  legitimate test-setup bug — fixed with a fixture backed by a genuine committed user.
- 80 tests total (37 new), 97% coverage on `app/`.

**Flagging for sign-off:**
- Delete/Active-Inactive-toggle unification above — confirm this matches what the vendor SRS
  actually expects for its "Delete" use case before Phase 5 builds a UI around it.

**Next:**
- Phase 4: real email delivery, replacing the stub.
- Still need sign-off on the very first commit (still staged, nothing on `main` yet).

### 2026-08-21 — Phase 4: real email delivery
**Done:**
- `send_activation_email()` now sends via SMTP (stdlib `smtplib`, no new dependency) — same
  function signature as the Phase 3 stub, so no caller (`bootstrap_admin.py`,
  `users/service.py`) needed to change.
- Dev fallback kept: empty `SMTP_HOST` (default) still just logs to console — zero setup needed
  unless you actually want to send real email.
- Decision: a failed SMTP send is logged, not raised — the user create/update it's attached to has
  already committed by the time email is attempted, so a broken mail server shouldn't turn an
  already-successful API call into a 500.
- New `SMTP_*` config, all commented out in `.env.example` (empty defaults = working local-dev
  state, no change required to keep dev working).
- 83 tests total (3 new), 98% coverage.

**Flagging for sign-off:**
- No email provider chosen/confirmed yet — generic SMTP works with anything, but Sơn/vendor still
  need to pick one for the real deployment.
- Reminder (found earlier, still open): no endpoint exists yet to *redeem* an activation token —
  every phase only issues them. Needs a decision on when this gets built (see HANDOFF.md).

**Next:**
- Phase 5: frontend — blocked on locating the vendor's AG Grid mockup (`user-management.html`).
- Git setup + first commit — Sơn doing this directly.

### 2026-08-21 — Gap closed: `POST /auth/activate`
Found while Sơn was manually testing Phase 4 through Postman: created a user, saw the activation
token print to console — and then had no way to actually use it. Every phase since Phase 1 only
*issued* tokens, nothing *redeemed* one.

**Done:**
- `POST /auth/activate` — token + new password in, sets a real password, flips `Create` → `Active`,
  logs straight in (returns the same access_token + refresh cookie `Login` would).
- First place the design doc's password policy (min 8 chars, letters + digits) is actually
  enforced — every password before this point was a random unusable placeholder.
- Verified with a real HTTP round trip end-to-end (not just automated tests): create user → grab
  token from console log → activate → confirmed `status: Active` → logged in with the new
  password successfully.
- 92 tests total (9 new), 98% coverage.
- Postman collection updated with an `Activate` request (same file — no re-import needed beyond
  refreshing it once).

**Flagging for sign-off:**
- Redeeming an email-change token also requires resetting the password, same as initial
  activation — a deliberate choice (extra identity check), not something the vendor SRS asked for
  explicitly.

**Next:**
- Phase 5: frontend, still blocked on the AG Grid mockup.

### 2026-08-21 — Gap closed: `POST /users/{id}/resend-activation`
Found right after the previous gap, while Sơn was reasoning through the activation UX: opening the
email link is safe to repeat (doesn't consume the token — only actually submitting a password
does), but the token itself still expires after 7 days, and there was no way to get a new one.

**Done:**
- `POST /users/{id}/resend-activation` — invalidates the old token, issues + sends a fresh one.
  Only works while status is still `Create` (409 otherwise). Preserves whether the pending token
  was for initial signup or an email-change re-verification, instead of assuming one.
- 99 tests total (7 new), 98% coverage.
- Postman collection updated (`Users > Resend activation`).

**Next:**
- Phase 5: frontend, still blocked on the AG Grid mockup.

### 2026-08-19 — Phase 0: backend scaffold
**Done:**
- Created `backend/` with `uv`, `pyproject.toml` pinned to `requires-python >= 3.11` (dev venv on
  3.12).
- `app/main.py` — FastAPI app with `GET /health`.
- `app/config.py` — env-var-backed settings (`pydantic-settings`); no secrets hardcoded.
- `app/db.py` — SQLAlchemy engine/session/`Base`, no ORM models yet.
- `alembic/` initialized and wired to read the DB URL from `app.config` and target
  `app.db.Base.metadata` for autogenerate. No revisions yet — needs Phase 1.
- Added dependencies: FastAPI, Uvicorn, SQLAlchemy 2.0, Alembic, Pydantic v2, pydantic-settings,
  argon2-cffi, PyJWT, psycopg (Postgres driver); dev: pytest, pytest-cov, httpx.
- One test (`tests/test_health.py`) — `uv run pytest` passes (1/1).
- `.gitignore` extended to also exclude `.pytest_cache/`.

**Next:**
- Get sign-off to make the first commit (staged, not yet committed).
- Phase 1: needs a `DATABASE_URL` (Postgres instance) before migrations can actually run.

### 2026-08-22 — Vendor mockup reviewed + Phase 4.5: forgot/reset/change password
Sơn got a reference UI mockup from the vendor's BA (`~/Downloads/app.html`, Ant Design look +
real AG Grid Community). Full review: `docs/design/frontend-mockup-review.md`. It surfaced 3
missing backend flows and 3 conflicts needing a decision before Phase 5 could start for real.

**Decisions confirmed with Sơn (not silently assumed):**
- Keep multi-role RBAC — mockup's single-role dropdown is an unfinished simplification, not a
  requirements change.
- Keep design doc decision #8's password policy (letters+digit) — mockup's stricter
  upper+lower+digit copy gets fixed in the real Phase 5 form instead.
- Build the 3 missing backend flows now, before Phase 5, rather than mocking them in the frontend.

**Done (Phase 4.5):**
- `POST /auth/forgot-password` — always the same generic response either way (anti-enumeration,
  matches mockup copy exactly). Also works for `Locked` accounts, not just `Active` — redeeming
  the token self-recovers them (found in the mockup's own JS, not obvious from its screens alone).
- `POST /auth/reset-password` — 30-min single-use token (matches mockup exactly). Deliberately
  does NOT auto-login — forces a fresh login, matching the mockup's own behavior.
- `POST /auth/change-password` — self-service, current-password required, rejects
  new-equals-current. Other sessions/devices deliberately not revoked (flagged, not silent).
- `POST /auth/login` now accepts username OR email in one field, matching the mockup's label.
- Refactored token issuance into a shared `app/modules/users/verification.py` (was private to
  `users/service.py`, now also used by `auth/service.py`).
- **Bug caught and fixed**: the new `PASSWORD_RESET` enum value was added to Postgres in the
  wrong case (`'password_reset'` instead of `'PASSWORD_RESET'`) — SQLAlchemy stores the Python
  enum's member *name*, not its value, which the existing `INITIAL_ACTIVATION`/`EMAIL_CHANGE`
  rows already showed but I missed on the first pass. Dev DB had to be fully reset (Postgres can't
  drop an enum value) — `demo_admin`/`demo_qa`/bootstrap Admin recreated after.
- Two test-isolation bugs fixed (test-only) — two new tests asserted a global table count that
  broke once other tests' leftover rows were present in the shared test DB.
- Full live HTTP smoke test: forgot-password → console token → reset-password → login with new
  password → login by email → change-password (wrong current password rejected, correct one
  works).
- 120 tests total (21 new), 98% coverage.
- Postman collection updated (3 new requests; `Login` body field renamed).

**Next:**
- Phase 5: frontend (React + Vite + Tailwind, AG Grid Community) — no longer blocked, mockup
  reviewed. See `docs/design/frontend-mockup-review.md` for what to fix vs. copy from it.

### 2026-08-22 — Gap closed: `GET /roles` + bug fix: `set_user_status` allowed `Create → Active`
Both found starting Phase 5: the multi-role picker needed a roles list (didn't exist), and
reasoning through which grid icons a `Create`-status row should show surfaced that the
Active/Inactive toggle would let an Admin "activate" a pending account directly — leaving it
`Active` with no usable password (the real one only ever gets set by redeeming the activation
token).

**Done:**
- `GET /roles` — gated by `user.manage` like the rest of this module.
- `set_user_status()` now rejects `Create → Active` (409) — the only way out of `Create` is
  redeeming the activation token or an Admin resending it.
- 2 tests total.

**Next:**
- Phase 5: frontend, for real this time.

### 2026-08-22 — Phase 5: frontend
**Done:**
- `frontend/` — React 18 (pinned deliberately; Vite scaffolds 19 by default, `CLAUDE.md` locks
  18), Vite, Tailwind v4, React Router v6, AG Grid Community (the mockup's real choice, not just
  its look), axios. Screens: Login, Forgot/Reset password, app shell + User Management grid
  (multi-role checkboxes, not the mockup's single-select), header Change-password modal.
- CORS middleware added to the backend (needed from the start — a credentialed cookie forbids a
  wildcard origin).
- **Three real bugs found live-testing the built app with Playwright** (not from reading code):
  1. Vite's dev proxy routed the frontend's own `/users` page navigation to the backend on
     reload (same path name as the API) — fixed by namespacing the backend under `/api` in dev.
  2. React 18 StrictMode double-firing the on-mount silent-refresh raced the backend's
     rotate-on-use refresh token, tripping Phase 2's own reuse-detection and silently logging
     the user out on every reload — fixed by funneling every refresh trigger through one
     deduplicated call.
  3. The refresh cookie's `Path=/auth` stopped matching once the `/api` prefix was added (the
     browser matches Path against what it saw, not what the backend received after the proxy
     rewrite) — fixed by widening to `Path=/`.
- AG Grid v33+ Theming API adopted cleanly (`theme={themeQuartz}`) — had briefly mixed it with
  the legacy CSS-file themes (AG Grid's own error #239), fixed.
- Fixed the vendor mockup's copy issues rather than porting them as-is: "Xóa" no longer claims
  "không thể hoàn tác" (untrue — never a hard delete), and the toggle-button label logic
  correctly says "Kích hoạt lại" for a Locked account (the mockup's own ternary gets this
  backwards).
- 21 vitest + React Testing Library tests (`npm test`) — password policy, API-error formatting,
  the delete/toggle status-transition logic, `tokenStore`, `ProtectedRoute`.
- Verified with a full live Playwright pass: login → reload (session survives) → create → delete
  → resend-activation → logout → reload (no leftover session). Screenshots at each step.

**Flagging:**
- No checked-in tests yet for the page-level flows (Login page, the grid page itself) — verified
  live instead. Worth backfilling before Phase 6's merge gate.

**Next:**
- Phase 6: integration + qa-reviewer audit + merge.

### 2026-08-23 — Branding fix: wrong product name/logo/copy on auth pages
Sơn caught it looking at the built frontend: no logo, product name showed as "OK2Ship AI Check"
instead of the mockup's actual "OK2SHIP AI", and the login page's left-panel text was invented
rather than the vendor's real copy.

**Root cause:** the original mockup review (`docs/design/frontend-mockup-review.md`) filtered out
every line over 2000 characters to stay readable — the mockup's base64-embedded Mektec logo and
the login page's real headline/body text both lived on those filtered-out lines, so the review
concluded (by omission, never actually checked) that there was no logo to use.

**Done:**
- Extracted the real logo from the mockup's base64 data → `frontend/public/logo.png`, now
  rendered in both the sidebar and the login/forgot/reset auth pages.
- Product name corrected everywhere to "OK2SHIP AI" (browser tab title, sidebar, auth pages).
- Login page's left-panel headline/body/footer copied verbatim from the mockup instead of
  paraphrased.
- Mockup review doc corrected with a "lesson learned" note for next time: a line-length filter
  finds structure fine, but silence in a filtered view is never evidence something isn't there —
  go back and check what a long line actually contains before concluding a visual element is
  missing.

**Next:**
- Phase 6: integration + qa-reviewer audit + merge.

### 2026-08-23 — Auth pages: missing input icons, checkbox, shimmer effect, circuit background
Sơn compared a mockup screenshot side-by-side with the running app and flagged it still didn't
look right — input fields had no icons, no "Ghi nhớ đăng nhập" checkbox, and the login button had
no visual effect.

**Root cause:** same root cause as the branding fix the day before — details that lived on the
mockup's filtered-out long lines (this time CSS/markup for the icon wrappers, the password
show/hide toggle, and the checkbox, none of which are on long lines themselves, but hadn't been
cross-checked carefully against the actual rendered auth form). Re-read the mockup's CSS
("AI MOTION LAYER" section) properly this time.

**Done:**
- Prefix icons (person/lock/email) on every auth-form input; password fields get a working
  show/hide eye toggle. New `PasswordField` and `icons.tsx` components, `TextField` gained an
  optional `icon` prop.
- "Ghi nhớ đăng nhập" checkbox added (new `Checkbox` component) — not wired to different session
  behavior, matching the mockup itself: its own checkbox isn't wired to anything either.
- Shimmer sweep effect on every screen's one primary CTA button (`Button` gained a `shimmer`
  prop) — matches the mockup's own explicit design rule ("used sparingly and only with business
  meaning... ONLY on the single main call-to-action of a screen").
- **Fixed two more small mismatches found while re-reading the mockup's modal footers
  precisely**: the status-toggle confirm button is always primary/blue with the static label
  "Xác nhận" (not dynamically "Kích hoạt"/"Vô hiệu hóa" as first built); the Delete confirm
  button never gets the shimmer (a destructive-looking action shouldn't shimmer); the Logout
  confirm button is primary/blue, not red/danger (logout isn't data-destructive).
- Animated circuit-background SVG (flowing dashed lines + pulsing dots) added to the auth pages'
  dark side panel, reproducing the mockup's `.circuit-bg` exactly (path data, colors, timings).
- Re-verified with Playwright screenshots against the mockup screenshot — much closer match now.

**Next:**
- Phase 6: integration + qa-reviewer audit + merge.

### 2026-08-23 — Auth panel width: replicating a real bug in the mockup file itself
Sơn compared again and said the dark left panel didn't match — it looked like it was taking up
much more width than the mockup. This time the cause wasn't a research gap on my side; it's a
genuine bug in the mockup's own HTML.

**Found:** the mockup has two nested `<div class="auth-page">` elements (a leftover duplicate
wrapper). Because of that, the *inner* one — which actually lays out the dark panel and the form
— doesn't stretch to fill the viewport; it shrink-to-fits its content instead. Result: `.auth-side`
never actually renders at its declared `width: 46%`. Measured by opening the mockup file directly
and reading `getBoundingClientRect()` across several viewport widths (1280–2200px): it's a
**constant ~434px**, not a percentage at all — CSS percentage widths against an indeterminate
(shrink-to-fit) containing block don't resolve the way you'd expect.

**Decision (confirmed with Sơn):** match the mockup's actual buggy rendering (fixed ~434px), not
its declared-but-never-actually-applied 46%. `AuthLayout.tsx`'s dark panel changed from
`lg:w-[46%]` to a fixed `w-[434px]`. Re-verified: matches the mockup's real render pixel-for-pixel
at every width tested.

**Next:**
- Phase 6: integration + qa-reviewer audit + merge.

### 2026-08-23 — Form position + circuit-bg proportions: two more symptoms, plus one new bug
Sơn again: the login form should hug the sidebar's right edge, not float centered across the
whole remaining light area; and the animated line/dot pattern on the dark panel wasn't
proportioned right either.

**Form position — same root cause as the panel-width bug, extended.** Re-read the mockup's raw
markup and confirmed the duplicate `<div class="auth-page">` (`#authShell` and its direct child
both carry the class): the outer one is the real flex container spanning the viewport; the inner
one — which holds *both* the dark panel and the form — is a flex item of it with no `flex-grow`,
so the whole block (not just the sidebar) shrink-to-fits. Measured `.auth-card-box`'s position
across five viewport widths (1280–3008px): fixed at x≈[499, 879] every time, with only the gap to
the browser's right edge growing on wider screens. So in the real mockup the form sits flush next
to the sidebar, not centered in the full remaining space. Fixed: `AuthLayout.tsx`'s right-hand
panel changed from `flex-1` to `w-[510px] lg:flex-shrink-0` (matching the measured width), with
`bg-gray-50` moved to the outer wrapper so the leftover space reads as plain page background,
same as the mockup.

**Circuit-bg proportions — a genuine new bug, distinct from the panel-width one.** The mockup's
`.circuit-bg` SVG is `position:absolute; inset:0` with no CSS width/height set. For an
absolutely-positioned *replaced* element (svg/img/video — unlike a plain `<div>`), `inset:0` alone
doesn't stretch it: width resolves from the left/right insets, but height is derived from the
SVG's own intrinsic `viewBox` ratio (460:800) rather than the top/bottom insets. Measured: a
434px-wide panel renders the SVG at 434×755px, leaving the bottom ~145px of the dark panel without
the pattern. Reproduced (matching the actual render, per the established call) by swapping
`h-full` for `aspect-[23/40]` (=460/800) on the `<svg>` in `CircuitBackground`.

**Verification:** measured local app vs. mockup at 1600px width — aside/card/svg boxes all within
~4px of each other. Screenshots visually match. `tsc --noEmit`, `vitest run` (21/21 passing), and
`vite build` all clean.

**Next:**
- Phase 6: integration + qa-reviewer audit + merge.

### 2026-08-23 — Color check: sidebar gradient and primary button were both off
Sơn asked directly whether the sidebar background and login-button colors were correct. Measured
`getComputedStyle()` on the mockup's real render rather than trusting the CSS source — both were
wrong:
- Sidebar: `linear-gradient(160deg, #12145E 0%, #0D0E47 100%)` in the mockup vs. our
  `bg-gradient-to-br from-[#0B0C2A] to-[#0D0E47]` — wrong start color *and* wrong angle. Fixed with
  an arbitrary-value gradient matching both.
- Button: mockup's `--ant-colorPrimary` is `#2B2FA0` (a darker navy-indigo); we had Tailwind's
  stock `indigo-600` (`#4F46E5`), noticeably brighter/more purple. Fixed to `#2B2FA0` / hover
  `#4247B8` / active `#1D2088`, matching the AntD theme tokens exactly. Also fixed the disabled
  state, which the mockup renders as neutral gray, not a light indigo tint.

Verified: local app's computed colors are now byte-identical to the mockup's. `tsc --noEmit` and
`vitest run` (21/21) pass.

### 2026-08-23 — Audit: i18n not integrated, backend errors leak raw English to the UI
Sơn asked whether error messages are translated to Vietnamese. Checked and confirmed: **no.**

No i18n library is installed (`package.json` has no i18next/react-intl; no `useTranslation` usage
anywhere in `src/`). `frontend/src/lib/apiError.ts` passes the backend's `{"detail": "..."}`
string straight through whenever it's present. Each call site does define a Vietnamese `fallback`
string (e.g. LoginPage's "Tên đăng nhập/email hoặc mật khẩu không đúng."), but that fallback only
fires when `detail` is *missing* — which is rare, since the backend's exception handlers always
populate `detail` with an English message (correctly, per the Constitution's "English in the
repo" rule). Net effect: on the normal/expected error path (wrong password, locked account,
duplicate username, expired token, etc.) users see literal English strings straight from the
backend, not the Vietnamese fallback that was clearly intended to cover this case.

Fixing this properly needs a design decision first: add a stable error `code` to `AppError` and
its subclasses (backend), and a Vietnamese message catalog keyed by code (frontend) — translating
by code, not by pattern-matching free-text English. Touches both stacks, so at minimum a Standard
flow. Flagged in HANDOFF.md; not started, pending scope confirmation with Sơn.

**Next:**
- Phase 6: integration + qa-reviewer audit + merge.
- Decide scope/flow for the i18n error-message gap, then implement if approved.

### 2026-08-23 — i18n error-code mechanism implemented (Standard flow, both debates run)
Sơn approved Standard flow and the full 33-code list, then said "triển khai nhé em". Built the
error-code mechanism from the plan above:

**Backend.** `app/core/error_codes.py` — `ErrorCode` StrEnum, 33 members. `AppError` requires
`code: ErrorCode` at construction (no default — a missing code is a `TypeError` at `raise`, not a
silent gap at request time). All ~27 raise sites across `auth/service.py`, `users/service.py`,
`deps.py`, `auth/router.py` updated. `app/main.py`'s handlers add `"code"` to every error body;
new `RequestValidationError` handler gives 422s a blanket `VALIDATION_ERROR` code while keeping
FastAPI's default `detail` shape via `jsonable_encoder`. `AccountNotUsableError`'s status→code
map has an explicit unknown-status fallback, not a bare dict index.

**Sync mechanism.** `app/core/error_codes.json` (checked in, regenerated via
`uv run python -m app.core.error_codes`) — a backend test fails if it's stale.
`frontend/src/lib/errorCodes.test.ts` reads that same file via a Vite `?raw` import (needed
`server.fs.allow` added to `vitest.config.ts`, scoped to `../backend/app/core`) and asserts the
frontend's code list matches exactly.

**Frontend.** `errorMessages.ts` — `Record<ErrorCode, string>` Vietnamese catalog; the type makes
a missing translation a `tsc` build failure. `apiError.ts` rewritten to key off `code` only — the
English `detail` is never shown to a user again. Every per-screen fallback string that duplicated
what a code now covers was removed (LoginPage, ChangePasswordModal, ActivateAccountPage,
ResetPasswordPage) — keeping them would have meant they misfired on network/500 errors instead of
their original (now redundant) purpose.

**Debates.** Plan debate (reviewer sub-agent) caught: no sync mechanism, undercounted token-error
variants (10, not the 5 first estimated — activation and password-reset are separate flows),
missing codes for `Forbidden`/`NotFound`/refresh-token errors — all fixed before implementation
started. Code debate (second reviewer sub-agent, after implementation) caught three real issues,
all fixed: four call sites' fallback strings had gone stale/misleading now that `code` covers
everything real (removed); `code in ERROR_MESSAGES` was a prototype-chain check, not
own-property (`hasOwnProperty.call` now); `AUTH_TOKEN_INVALID`'s Vietnamese text said "expired"
when the same code also covers a malformed token (reworded to hedge, matching existing copy
elsewhere).

**Verification.** `curl` against the live backend confirmed the real JSON body carries `code`; a
Playwright run against the real login page confirmed the UI shows the Vietnamese message, not the
English string. 134 backend tests, 25 frontend tests, `tsc -b`, `vite build` — all green. One
branch (`activate_account`'s account-gone case) is deliberately untested: unreachable given a real
FK constraint (`email_verification_tokens_user_id_fkey` has no cascade) — confirmed by trying it
and getting an `IntegrityError`; the same code path is exercised via `change_password` instead.

**Next:**
- Phase 6: integration + qa-reviewer audit + merge.

### 2026-08-23 — Bug: failed login also fired an unnecessary /auth/refresh call
Sơn noticed (via a screenshot) that clicking "Đăng nhập" with an empty form still triggered an
`/auth/refresh` network call and asked why.

**Root cause:** `client.ts`'s response interceptor treated *any* 401 (except `/auth/refresh`
itself) as "access token expired, silently refresh and retry." But `/auth/login` (wrong
credentials), `/auth/change-password` (wrong current password), `/auth/activate` and
`/auth/reset-password` (bad/expired token) all legitimately return 401 for domain reasons that
have nothing to do with the access token — every one of those triggered a needless extra
`/auth/refresh` call.

**Fix (Standard flow, plan debated first):** gate strictly on the error `code` (from today's i18n
work) instead of raw HTTP status — only `AUTH_TOKEN_MISSING`/`AUTH_TOKEN_INVALID` (the two codes
`deps.py::get_current_user` can raise) now trigger refresh-and-retry. Extracted as a standalone
`shouldAttemptRefresh(error)` taking the raw `AxiosError` — a plan-debate reviewer caught that
`AxiosError` has its own unrelated `.code` field, easy to confuse with the backend's
`response.data.code` if they'd been separate function arguments.

**Tests:** 8 unit tests on the decision function + 2 end-to-end tests mocking `apiClient`'s
adapter and `axios.post` to exercise the real interceptor. Validated the tests actually catch the
bug by temporarily reverting to the old logic — 4 tests failed as expected, then restored.
Re-verified live via Playwright: empty-form submit now fires exactly one `/auth/login` call, zero
`/auth/refresh` calls. 134 backend / 36 frontend tests, `tsc -b`, `vite build` — all green.

**Next:**
- Phase 6: integration + qa-reviewer audit + merge.

### 2026-08-23 — Follow-up: blank login form should never call the API (Quick flow)
Sơn asked for client-side validation so submitting an empty login form doesn't call the API at
all.

Gap: `LoginPage`'s form had `noValidate`, and `TextField`/`PasswordField`'s `required` prop was
purely decorative — never set on the actual `<input>` — so a blank submit always reached
`/auth/login`.

Added `src/lib/loginValidation.ts` (`validateLoginForm`, a pure function, 5 unit tests) — checked
in `handleSubmit` before calling `login()`; returns early with field-level red errors (using the
existing `TextField`/`PasswordField` `error` prop) if either field is empty. Verified live via
Playwright: blank submit → 0 API calls; only one field filled → 0 API calls; both filled →
exactly 1 `/auth/login` call.

**Next:**
- Phase 6: integration + qa-reviewer audit + merge.

### 2026-08-23 — `is_deleted` column: distinguishing "Xóa" from "Vô hiệu hóa" (Standard flow)
Sơn asked to add an `is_deleted` boolean to `users` and use it for the Delete action, which
until now was byte-for-byte identical to Deactivate (both just set `status=Inactive`). Resolved
3 design questions with him first (all "Recommended" chosen): additive on top of `status`, not a
replacement; no change to list visibility or reactivate UX; no change to username/email
uniqueness.

**Plan-debate caught a real bug before any code was written:** the naive plan (Delete sets
`is_deleted=true` alongside `status=Inactive`) breaks on the exact two-button sequence the whole
feature exists for — Deactivate (→ Inactive, is_deleted=false) then later Delete (→ Inactive
again) both target the *same* status, and `set_user_status`'s existing no-op guard
(`if user.status == new_status: return`) only checks status, so the second call would silently
no-op and never set `is_deleted=true`. Fixed by checking both fields in the guard.

**Backend:** migration (`is_deleted BOOLEAN NOT NULL DEFAULT false`, forward-only — pre-existing
Inactive users backfill to `false`, no way to know retroactively which were "deleted" in intent);
`set_user_status()` now takes `is_deleted`, checks both status+is_deleted for its no-op guard,
rejects `status=Active` + `is_deleted=true` with a new `ConflictError`
(`USER_CANNOT_BE_ACTIVE_AND_DELETED` — same "reject, don't silently coerce" treatment as the
existing Create→Active guard) instead of quietly overriding it, and always clears `is_deleted` on
reactivation. Audit log now only includes whichever of `status`/`is_deleted` actually changed on
a given call, not both unconditionally. `login()` needed no change — `is_deleted=true` can only
ever coexist with `status=Inactive`, which already blocks login.

**Frontend:** `userStatusPlan.ts::planFor()` returns `isDeleted` per mode (`true` only for
`'delete'`); threaded through `StatusConfirmModal` → `setUserStatus()` → the PATCH body.

**Tests:** 7 new backend tests (including the exact Deactivate→Delete sequence that was broken,
plus Delete→Reactivate→Delete, the Active+is_deleted rejection, and audit-payload-only-changed-
fields), 5 new/extended frontend tests. 141 backend / 42 frontend tests, `tsc -b`, `vite build`
— all green.

**Verified live**, not just unit tests: direct API calls reproduced the Deactivate→Delete sequence
against the running backend (confirmed `is_deleted` correctly flips to `true`) and the Active+
is_deleted rejection (409, correct code). Playwright click-through on the real grid: clicking
"Xóa" sends exactly `{"status":"Inactive","is_deleted":true}`.

**Next:**
- Phase 6: integration + qa-reviewer audit + merge.

### 2026-08-23 — k8s-readiness code audit (Standard flow)
Sơn: target infra is k8s; currently there's only `backend/docker-compose.yml`, which is local-dev
dependencies only (Postgres + Mailpit) — no app Dockerfile, no k8s manifests, no CI/CD exist
anywhere yet. Asked for a plan before touching anything.

Audited the app for anything that breaks under multiple replicas / rolling restarts, scope
deliberately narrowed to code-only (Sơn deferred Dockerfile/manifests/CI/frontend-serving-strategy
to later, once cluster/registry details are known — not started). Mostly already fine: config via
env vars, no in-process state/cache/scheduler, and the auto-lock job is already a standalone
script (`python -m app.jobs.auto_lock`) whose own docstring already says wiring it to a scheduler
is a deployment concern — maps directly onto a future k8s CronJob, no code change needed.

Two real gaps, fixed:
- `/health` conflated liveness and readiness — a static "ok" with no DB check. Fine as a liveness
  probe (a DB outage failing liveness would restart every pod for nothing), but there was no
  separate readiness signal. Added `/health/ready` (`SELECT 1`, 200/503) without touching
  `/health` itself.
- DB connection pool size (`create_engine()`) was hardcoded to SQLAlchemy's own defaults with no
  way to override — under k8s, total Postgres connections = replica count × (pool_size +
  max_overflow), needs to be tunable once real replica counts are known. Added
  `db_pool_size`/`db_max_overflow` to `Settings` (defaulting to the same 5/10, no behavior change
  unless overridden) and wired them into the engine.

6 new tests. Verified live against the running dev server: both `/health` and `/health/ready`
return correct status/body. 146 backend tests, all green.

**Next:**
- Phase 6: integration + qa-reviewer audit + merge.
- k8s deploy infra (Dockerfile, manifests, CI/CD, frontend-serving decision) — deferred, pick up
  once cluster/registry details are known.

### 2026-08-23 — Sixth fidelity pass: form label font-weight/color (Quick flow)
Sơn: the "Tên đăng nhập hoặc Email"/"Mật khẩu" labels looked bolder than the mockup. Measured via
`getComputedStyle()` on both — confirmed: mockup labels are `font-weight: 400`, color
`rgba(0,0,0,0.88)` (`--ant-colorText`); local app used Tailwind's `font-medium` (500) and
`text-gray-700`, both off. Fixed in `TextField.tsx`/`PasswordField.tsx` (`font-normal
text-[rgba(0,0,0,0.88)]`). Found the same color mismatch on `Checkbox.tsx`'s "Ghi nhớ đăng nhập"
label while measuring (font-weight already matched, color didn't) — fixed the same way.
Re-measured: exact match on both files/text now. 42 frontend tests, build — all green.

### 2026-08-23 — Seventh fidelity pass: whole-app font audit — root cause was a missing webfont
Sơn: check every piece of text against the mockup, not just the labels — flagged the sidebar copy
and button text specifically as examples.

Ran a comprehensive `getComputedStyle()` sweep (fontFamily/fontSize/fontWeight/color) across every
major text role: sidebar logo/headline/body/footer, card title/subtitle, submit button, page
heading. Found the root cause first: the mockup doesn't just *declare* `'Inter', -apple-system,
...` — it actually `@import`s the real Inter webfont from Google Fonts. The app had no webfont at
all (`body { font-family: system-ui, ... }`), so **every** piece of text in the whole app was
rendering in the wrong typeface, not just the flagged examples. Fixed by loading the exact same
Google Fonts URL the mockup uses (Inter + JetBrains Mono, matching weights) via `index.html`, and
updating `index.css`'s `body` font-family + adding a `--font-mono` theme override for
`UserViewModal`'s `font-mono` cells.

With the typeface fixed, re-measured every element and found four more real mismatches on top of
it (all confirmed via computed style, not source-reading):
- Sidebar headline: mockup 26px vs app's `text-2xl` (24px) → `text-[26px]`.
- Card title ("Đăng nhập" etc.): mockup 24px/`rgba(0,0,0,0.88)` vs app's `text-xl` (20px)/
  `gray-900` → `text-2xl` + exact rgba.
- Card subtitle: mockup `rgba(0,0,0,0.65)` (`--ant-colorTextSecondary`) vs app's `gray-500`.
- Submit button: mockup's auth-page CTAs use `.ant-btn-lg` (15px/weight 400) while every modal
  button uses the smaller default `.ant-btn` (14px/weight 400) — app used one size everywhere,
  14px at `font-medium` (500, wrong on both counts for both sizes). Added a `size` prop to
  `Button.tsx` (`default`/`large`); applied `large` to all four auth-page submit buttons
  (Login/Forgot/Reset/Activate) matching the mockup's markup exactly (only 2 `.ant-btn-lg`
  instances found there, but the vendor's own comment marks this as the "single primary CTA of a
  screen" pattern — extended to all 4 analogous auth screens, only 3 of which the mockup actually
  demos).
- Also caught the User Management page's `<h1>` while spot-checking a second page: mockup's
  `.page-title` is 20px, app's was `text-lg` (18px) — fixed.

Line-height was NOT touched — a handful of elements (card h1, card subtitle) differ slightly there
(e.g. 32px vs mockup's 37.7px), but that's a spacing property, not a font one, and out of scope
for what was asked; flagged to Sơn rather than fixed silently.

Not exhaustively audited beyond the auth pages + the Users page heading — sidebar nav items, the
AG Grid table itself, and modal bodies weren't individually re-measured this pass (the font-family
fix already applies to all of them via `body`, but per-element size/weight/color on those hasn't
been spot-checked the way the auth pages and this one heading were).

42 frontend tests, `tsc -b`, `vite build` — all green.

### 2026-08-23 — Backend and frontend split into their own repos, pushed to GitLab
Sơn: push backend to GitLab. Then, separately, same for frontend. Full detail (SSH setup, the
squashed-vs-module-grouped commit discussion, the README/`.gitignore`/pre-commit-hook fixes each
needed, the `errorCodes.test.ts` cross-repo path fix) is in HANDOFF.md — this is the short
version.

`backend/` and `frontend/` are now each their own git repo (nested `.git/` inside this monorepo).
Both pushed as a single "initial import" commit — true phase-by-phase history isn't
reconstructable after the fact since nothing was committed incrementally while building.

- `backend/` → `git@gitlab.com:mektec/ok2ship-ai-backend.git`, `main`, commit `7463a70` (67 files,
  146 tests).
- `frontend/` → `git@gitlab.com:mektec/ok2ship-ai-frontend.git`, `main`, commit `2e7c6e8` (61
  files, 42 tests).

Sơn's call: keep developing at the same local paths (`products/ok2ship-ai/backend`/`frontend`),
push again only when asked — nothing auto-syncs. `HANDOFF.md`/`docs/` stay in the parent monorepo
only, not copied into either GitLab repo.

### 2026-08-23 — Parent monorepo (docs/planning) pushed to GitHub too
Sơn: push the parent repo (`products/ok2ship-ai/` itself — CLAUDE.md, HANDOFF.md, docs/) to
`git@github.com:Mai-Hong-Son/ok2ship-ai-agents.git`.

First updated `.gitignore` to exclude `backend/`/`frontend/` in full, not just their build
artifacts — now that they're separate repos with their own `.git/`, staging them here would have
created a submodule-style gitlink entry, which isn't what's wanted. Also applied the same
`.env.example`-false-positive fix to this repo's own pre-commit hook (the template the
backend/frontend hooks were copied from) for consistency, even though it now has no code/tests of
its own to run.

Single "initial import" commit, same reasoning as backend/frontend. Pushed:
`git@github.com:Mai-Hong-Son/ok2ship-ai-agents.git`, `main`, commit `4341db1` (7 files:
`CLAUDE.md`, `HANDOFF.md`, `.gitignore`, `docs/PROGRESS.md`, `docs/design/*.md`).

All three of this product's repos are now on a remote. See HANDOFF.md's "Repo topology" for the
full picture (URLs, commit SHAs, local working-copy locations).

**Next:**
- Phase 6: integration + qa-reviewer audit + merge — now a normal branch-per-feature flow in each
  repo independently, since all three have an initial commit to branch from.

### 2026-08-24 — Bug: /forgot-password (and reset/activate) silently rotated the refresh token
Sơn, manually testing: noticed `/auth/refresh` gets called just from loading `/forgot-password`.

**Root cause:** `AuthProvider` wraps every route and unconditionally fires a silent
`refreshAccessToken()` on mount to restore the session after a page reload. Useful for `/login`
(redirects away if already authenticated) and the protected `/users` route, but grep confirmed
`ForgotPasswordPage`/`ResetPasswordPage`/`ActivateAccountPage` never call `useAuth()` at all — the
refresh's result was never read on those three pages.

**Real risk, not just a wasted request:** the refresh cookie is rotate-on-use with reuse
detection. If a user is logged in in one tab and opens `/forgot-password` in another (or even
just navigates there) while still logged in, the silent refresh rotates their refresh token as a
side effect. The first tab's *next* refresh then presents the now-stale token, reads as a replay
of an already-rotated one, and revokes the whole session — same class of bug as the StrictMode
double-refresh race found and fixed in Phase 5.

**Fix:** `AuthContext.tsx` now skips the refresh entirely on a fixed list of paths
(`/forgot-password`, `/reset-password`, `/activate`) that don't read `principal`/`isLoading`.
Checked via `window.location.pathname` directly (not `useLocation()`) since this only matters at
the moment of the actual page load — the effect still only ever runs once, on mount, as before.

Verified live via Playwright: all three pages now fire zero `/auth/refresh` calls; `/login`
still fires exactly one, unaffected. 42 frontend tests, build — all green.

### 2026-08-24 — "Tổng quan" (Overview) tab + permission-less landing page
Sơn: add a "Tổng quan" sidebar tab, empty content for now, and redirect users without
`user.manage` there instead of showing a permission error — read the mockup and match it.

Checked the mockup's real nav markup and found the app's sidebar had drifted further than
expected: labels for every not-yet-built future module were paraphrased instead of copied
verbatim (e.g. "Nạp báo cáo QA" vs the mockup's actual "Data Ingestion"), one item ("Audit Log")
was missing outright, and no nav item had an icon despite the mockup giving every one its own
(sidebar bg was also `bg-gray-900` instead of the mockup's `#12145E`, width 240px not 232px).
Confirmed with Sơn before touching scope: fix all of it in the same pass, not just add "Tổng
quan" in isolation.

- New `src/components/layout/navIcons.tsx` — 9 icons, every path/viewBox copied verbatim from the
  mockup's `<svg>` markup (not approximated).
- `Sidebar.tsx` rewritten: correct bg/width (measured via `getComputedStyle()` — `rgb(18,20,94)`
  confirmed `#12145E`), all labels matched to the mockup's real English module names, "Tổng quan"
  added as the first item, "Audit Log" added (both future-module-styled except Tổng quan/Quản lý
  người dùng, which are real routes).
- New `OverviewPage.tsx` — deliberately empty placeholder; the mockup itself has no built-out
  content for this tab either (checked, only the nav item exists).
- `ProtectedRoute.tsx`: permission failure now redirects to `/overview` instead of an inline
  error message.
- `App.tsx`: added the `/overview` route (no `requirePermission` — any authenticated user);
  default/catch-all redirects (`/`, `*`) changed from `/users` to `/overview`.

**Two more pre-existing bugs surfaced and fixed while wiring this up**, both in `LoginPage.tsx`/
`ProtectedRoute.tsx`'s "return to where you came from" logic:
- `LoginPage`'s post-login `navigate()` call was hardcoded to `/users` and completely ignored
  `location.state.from` — a user bounced to `/login` from a specific protected route wouldn't
  actually be sent back there after logging in. Fixed by computing `redirectTo` once (falling
  back to `/overview`, not `/users`) and reusing it in both the already-authenticated branch and
  `handleSubmit`.
- `ProtectedRoute` redirected unauthenticated users to `/login` without ever setting
  `state.from` — meaning the above logic could never actually fire even once fixed, since nothing
  populated it. Fixed: now passes `state={{ from: location.pathname }}`.

Verified live: `demo_admin` (has `user.manage`) lands on `/overview` after login, sidebar renders
exactly as expected. `demo_qa` (role `qa`, no `user.manage`) also lands cleanly on `/overview`
with zero permission-error text anywhere. (Reset `demo_qa`'s password to a known value in the
dev DB to test this — the old one wasn't known.) 42 frontend tests, build — all green.

### 2026-08-24 — Hide "Quản lý người dùng" entirely for accounts without user.manage
Sơn: don't leave the nav item visible-but-inert for an account that can't use it (clicking it —
now correctly redirecting to `/overview` per the previous entry — still reads as "broken" if the
link itself is sitting right there). Hide it outright instead.

`NavItem` gained an optional `requirePermission` field; `Sidebar` now reads `hasPermission` from
`useAuth()` and filters both items and (for future-proofing, though none currently empty out)
whole groups down to what the current user can actually see, computed per-render rather than
baked into the static `NAV_GROUPS` list. Only "Quản lý người dùng" is gated today
(`user.manage`) — every other item is either always-visible (Tổng quan) or a disabled future
module (visible to everyone, matching the mockup's own treatment of unbuilt modules).

Verified live: `demo_admin` still sees "Quản lý người dùng" in the sidebar (1 match);
`demo_qa` (role `qa`) sees zero matches for it anywhere on the page — confirmed via both a text
search and a screenshot. 42 frontend tests, build — all green.

### 2026-08-24 — User grid: action icons were raw emoji; status badges were wrong on 3/4 counts
Sơn flagged the eye icon in the Users grid looked off vs the mockup, and asked for a full check
of that screen. Found far more than the eye icon:

**Action icons were literal emoji** (👁 ✎ ⏻ 🗑 in `UserActionsCell.tsx`), not SVG — the only place
in the whole app still doing that; everywhere else is stroke-based line icons. Read the mockup's
real `actionsCellRenderer()` JS (not just its CSS) for the exact paths. `EyeIcon` already existed
in `common/icons.tsx` and its path was already an exact match (reused as-is); added `EditIcon`,
`PowerIcon`, `TrashIcon` there too. `.gc-action-icon`'s exact box/hover styling
(28×28, `--ant-colorTextSecondary` default, `--ant-colorFillTertiary`/`--ant-colorPrimary` hover,
danger variant `--ant-colorErrorBg`/`--ant-colorError`) replicated in `UserActionsCell.tsx` — as
two complete non-overlapping class strings, not a shared base + appended "danger" modifier
(Tailwind doesn't guarantee which of two conflicting `hover:bg-*` utilities wins if both are
present in one `className`).

**StatusBadge was wrong on colors, labels, and animation** — read the mockup's actual
`STATUS_LABEL_VI`/`STATUS_TAG_CLASS`/`STATUS_PULSE` JS objects (not approximated):
- 3 of 4 labels showed the raw English status (`Active`, `Locked`, `Inactive`) instead of a
  Vietnamese translation (`Hoạt động`, `Đã khóa`, `Ngừng hoạt động`) — Create's `'Chưa kích hoạt'`
  wasn't even the mockup's actual wording (`'Chờ kích hoạt'`).
- Create was amber instead of the mockup's blue/`tag-processing`; Locked was red instead of
  amber/`tag-warning`. Exact hex values pulled from the mockup's real CSS variables, not
  approximated with Tailwind's stock palette.
- No status pulsed. Mockup pulses Active and Locked; reused the already-existing `pulse-dot`
  keyframe (confirmed byte-identical timing to the mockup's own `pulseDot`, already used for the
  auth-page circuit background) rather than adding a new one.
- Shape was `rounded-full` (pill); mockup's `.ant-tag` is `rounded` (`--ant-borderRadiusSM`, 4px)
  with a real 1px border, not a pill with no border.

Two more inline text references caught and fixed while touching this area, now stale against the
corrected badge labels: `userStatusPlan.ts`'s delete-confirmation copy said "trạng thái Inactive"
(now "Ngừng hoạt động"); `UserFormModal.tsx`'s email-change hint said "trạng thái Create" (now
"Chờ kích hoạt").

Verified live via Playwright screenshot against `demo_admin`'s real grid — badges show correct
Vietnamese labels/colors, action column shows proper line icons. 42 frontend tests, `tsc -b`,
`vite build` — all green. (Also had to reset `demo_admin`'s dev-DB password again mid-verification
— same as `demo_qa` earlier, the original wasn't known.)

### 2026-08-24 — Grid filtering rebuilt: mockup has no toolbar, filters on the column headers
Sơn: the mockup has no search box, and filters are pushed into the table's column headers —
check the whole table structure again. Confirmed against the mockup's real markup/JS, found more
than the toolbar alone: filtering is entirely client-side (AG Grid's built-in `agTextColumnFilter`
for Name/Email/Department, a custom `SelectFilter` popup class for Vai trò/Trạng thái), there's no
toolbar row at all (`.page-head` is only the title + "+ Thêm người dùng" button), the separate
Username/Email columns are one merged column in the mockup, and the grid has a full Vietnamese
`localeText` plus a branded theme the app never adopted — it had been rendering AG Grid's default
unbranded Quartz theme this whole time. Confirmed two real decisions with Sơn before rebuilding
(client-side vs server-side filtering has real scalability tradeoffs; merging the columns):

- Toolbar removed; the page now loads the full user list once (`limit=200`) instead of refetching
  on every filter change.
- New `SelectColumnFilter.tsx` — `ag-grid-react`'s `useGridFilter` hook, a React-idiomatic
  reimplementation of the mockup's `SelectFilter` class (same popup-from-the-header-funnel-icon
  mechanism, AG Grid Community, not the Enterprise Set Filter). Vai trò's dropdown options are
  fetched live from `/roles` rather than hardcoded, since real RBAC has more roles than the
  mockup's demo Admin/QA.
- New `agGridLocaleVi.ts` (mockup's `VI_LOCALE` verbatim, plus `noMatchingRows` — a newer AG Grid
  locale key the mockup's own object never defined; filled the gap rather than leaving one stray
  English string in an otherwise all-Vietnamese page) and `agGridTheme.ts` (mockup's `okTheme`
  branding — accent color, fonts, spacing).
- Pagination/row sizing now matches the mockup's own `gridOptions` exactly: 6/12/24 page sizes,
  `rowHeight: 58`, `headerHeight: 44`, `domLayout: 'autoHeight'`.

Verified live via Playwright: toolbar gone, every column's funnel-icon filter opens and actually
narrows the rows, Vietnamese empty-state text confirmed after a 0-result filter. 42 frontend
tests, `tsc -b`, `vite build` — all green.

### 2026-08-24 — Role names, breadcrumb, header/sidebar user widgets
Sơn caught 3 more drift points reviewing the rebuilt grid: Vai trò showed the raw role code
('qa') instead of its title ('QA'), like in the filter dropdown; the mockup's breadcrumb ("Quản
trị hệ thống / Quản lý người dùng") was missing entirely; "+ Thêm người dùng" (the page's one
primary CTA) never got the `shimmer` prop `Button.tsx` already supports. Fixing the breadcrumb
and re-checking the header surfaced a bigger gap: the header's user widget and the sidebar's
`.sider-foot` block both need the logged-in user's full_name/email, which the JWT deliberately
excludes (short-lived, roles/permissions claims only). Confirmed with Sơn: add a backend endpoint
rather than stuff more into the JWT.

- **Backend**: new `GET /auth/me` — bare `get_current_user`, not `user.manage`-gated, so any
  logged-in user (not just Admin) can read their own profile, unlike `GET /users/{id}`.
  `UserResponse` gained `role_names` (aligned by index with `role_codes`) since `/auth/me`'s own
  consumers have no access to the Admin-only `GET /roles`. 2 new tests + 1 existing test extended
  to assert `role_names`.
- **Frontend**: `AuthContext` fetches `/auth/me` once per session right after the token is set
  (fire-and-forget — falls back to username-only display if it fails, nothing else depends on
  it), exposed as `profile`. Header gained the breadcrumb (a per-route lookup table — only
  `/users` has real mockup ground truth, since the mockup is single-page; other routes fall back
  to their sidebar group title), the notification bell (decorative — Alert & Notification isn't
  built yet, matches the mockup which never wires an onclick on it either), and the two-line
  name+email dropdown trigger with the mockup's exact dropdown styling (icons, shadow, colors).
  Sidebar gained the `.sider-foot` block (avatar + name + role, roles joined with ", " for
  multi-role users — the mockup's demo data only ever has one).

Verified live via Playwright post-login: breadcrumb/bell/header widget/sidebar-foot all render
correctly, dropdown shows the mockup's exact icon+shadow styling, role names show titles not
codes. 148 backend / 42 frontend tests, both builds — all green.

### 2026-08-24 — Today's work committed and pushed
Frontend (`git@gitlab.com:mektec/ok2ship-ai-frontend.git`), 3 commits: `7a4677d` (icon/badge
fix), `effa97a` (grid rebuild), `e5a9acb` (breadcrumb/header/sidebar/role names). Backend
(`git@gitlab.com:mektec/ok2ship-ai-backend.git`), 1 commit: `6012b32` (`/auth/me`). All landed
directly on `main` in both repos, same as every commit since the initial import — Phase 6's
branch/PR + qa-reviewer flow still hasn't started (see HANDOFF.md's "Next steps"). One untracked
file deliberately left uncommitted: `backend/scripts/init_db.sh` (pre-existing, not authored this
session — flagged for Sơn rather than silently bundled into an unrelated commit).
