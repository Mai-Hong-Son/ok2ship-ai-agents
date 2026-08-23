# HANDOFF — ok2ship-ai (session handoff)

> Read this first when reopening this product. Update it after each session.
> Working rules + stack: see `CLAUDE.md`. Schema + all locked decisions: see
> `docs/design/user-management.md` (source of truth — code must match it, not the other way round).
> Backend folder/module architecture: see `docs/design/backend-architecture.md`.
> Daily-glance status/log (quick check of what's done and where things stand): see
> `docs/PROGRESS.md`.

## Repo topology (read this before touching git — it's not one repo)
As of 2026-08-23, this product is **three separate git repositories**, not one:
1. **`products/ok2ship-ai/`** (this repo, the one `HANDOFF.md`/`docs/` live in) — planning,
   design docs, session handoff. No remote configured; nothing pushed anywhere. `backend/` and
   `frontend/` are present on disk here but are each their own **nested** git repo (their own
   `.git/`) — this parent repo does not track their contents at all.
2. **`backend/`** — its own repo, pushed to `git@gitlab.com:mektec/ok2ship-ai-backend.git`,
   branch `main`. Initial commit `7463a70` ("chore: initial import — User Management &
   Permission module (WBS #5)"), 67 files, 146 tests.
3. **`frontend/`** — its own repo, pushed to `git@gitlab.com:mektec/ok2ship-ai-frontend.git`,
   branch `main`. Initial commit `2e7c6e8` ("chore: initial import — User Management &
   Permission frontend"), 61 files, 42 tests.

Both were single "initial import" commits (not a phase-by-phase history — nothing was committed
incrementally while building, so there's no real snapshot history to replay; see their HANDOFF
entries below for why). Each has its own pre-commit hook (`.git/hooks/pre-commit` inside
`backend/`/`frontend/` respectively — copied from this parent repo's hook, then fixed: the
original used bare `pytest`/checked test files at the wrong root, neither of which actually
worked once each subfolder became its own repo root).

**Sơn's chosen workflow going forward**: keep developing at these same local paths
(`products/ok2ship-ai/backend`, `products/ok2ship-ai/frontend`) — do not clone the GitLab repos
to a new location. Push to GitLab again only when asked; nothing auto-syncs.

`HANDOFF.md`/`docs/PROGRESS.md`/`docs/design/*.md` live ONLY in the parent repo — they are not
copied into either GitLab repo. A session working from a fresh clone of just `backend/` or
`frontend/` won't have them.

## What this product is
Backend + web dashboard for **OK2Ship AI Check** (Mektec Vietnam, delivered alongside vendor
Desoft). 🚀 Serious product (not a spike) — tests mandatory, branch-per-feature, qa-reviewer before
merge. Built module by module following the vendor's WBS. **Currently building the first module:
User Management & Permission Assignment** (WBS #5).

Sibling spikes already proved feasibility for later modules — reuse, don't re-derive:
- `../_spikes/ok2ship-anomaly` — golden/one-class anomaly detection, future "image vs golden
  sample" module.
- `../_spikes/ok2ship-report-parser` — reading structured data out of real factory Excel reports,
  future data/spec-check modules.

## Current state (as of 2026-08-19)
- [x] Repo scaffolded: `CLAUDE.md`, `.gitignore`, git init, pre-commit hook installed.
- [x] `docs/design/user-management.md` — full schema (9 tables) + every locked decision,
  reconciled against the vendor's official SRS (`SRS_OK2SHIP_User_Management.docx`) after 3 design
  rounds (flat 2-role → full RBAC per vendor's later multi-role request). This file is the source
  of truth — always read it before touching User Management code.
- [x] A separate clean Vietnamese schema-only doc was sent to the vendor's BA:
  `~/Downloads/User_Management_Permission_Design.docx` (outside this repo, not tracked in git).
- [x] Build plan for the module reviewed adversarially (see "Approved plan" below) — 8 real gaps
  found and fixed into the design doc (a `department` enum-vs-FK inconsistency — resolved by
  DROPPING the separate `departments` table, see decision #5 below, not by keeping it as a FK;
  missing daily auto-lock job; missing Locked→Active recovery path; bootstrap-admin mechanics;
  email-interface sequencing; JWT permission-embedding; refresh-token-reuse detection; audit_log
  partitioning deferral note).
- [x] **Phase 0 done (2026-08-19): backend scaffold.** `backend/` created with `uv` (package
  manager choice — not a stack deviation, tooling only; `pyproject.toml` still declares
  `requires-python = ">=3.11"` per the locked stack, dev venv pinned to 3.12 since it's already
  installed locally). Structure:
  - `app/main.py` — FastAPI app + `GET /health` (only route so far; auth/User Management routers
    are Phase 2/3).
  - `app/config.py` — `pydantic-settings` `Settings`, all values from env vars / `.env`
    (never committed — `.env.example` is the checked-in template). Includes the access/refresh
    token TTL fields the design doc calls for (`access_token_ttl_seconds` default 120s).
  - `app/db.py` — SQLAlchemy engine/session/`Base`; `get_db()` FastAPI dependency. No models yet
    (Phase 1).
  - `alembic/` — initialized and wired to `app.config`/`app.db`: `env.py` pulls the DB URL from
    `get_settings().database_url` (not from `alembic.ini`, which keeps only an unused placeholder)
    and sets `target_metadata = Base.metadata` for autogenerate. Verified with `uv run alembic
    history` (loads clean, no DB connection needed for that command). No revisions yet.
  - `tests/test_health.py` — one passing test (`uv run pytest` → 1 passed).
  - Runtime deps added: fastapi, uvicorn[standard], sqlalchemy>=2.0, alembic, pydantic>=2,
    pydantic-settings, argon2-cffi, pyjwt, psycopg[binary] (Postgres driver, v3). Dev deps: pytest,
    pytest-cov, httpx (for FastAPI `TestClient`).
  - `.gitignore` extended with `backend/.pytest_cache/` / `.pytest_cache/` (missed in the original
    scaffold).
  - **Update (2026-08-23): committed.** `.venv`/`__pycache__`/`.pytest_cache`/`.env` correctly
    excluded, confirmed. This landed as part of `backend/`'s single "initial import" commit
    (`7463a70`) once `backend/` became its own repo — see "Repo topology" at the top of this file.
- [x] **Backend folder architecture locked (2026-08-19):** domain-based, one package per WBS
  module under `app/modules/`, plus `app/core/` (cross-cutting), `app/jobs/`, `app/cli/`. Full
  writeup: `docs/design/backend-architecture.md`. Package skeleton created.
- [x] **Local Postgres via Docker (2026-08-19):** `backend/docker-compose.yml`, host port **5433**
  (5432 already taken by another project's container on this machine). `.env`/`.env.example`/
  `app/config.py` default all match.
- [x] **Phase 1 done (2026-08-19): models + migration + seed + bootstrap admin.**
  - `app/modules/users/models.py` — `User`, `Role`, `Permission`, `role_permissions`/`user_roles`
    (association tables), `EmailVerificationToken`, plus `Department`/`UserStatus`/
    `VerificationPurpose` enums. `app/modules/auth/models.py` — `RefreshToken`.
    `app/modules/audit/models.py` — `AuditLog`. All match `docs/design/user-management.md` exactly
    (8 tables, no `departments` table).
  - `app/core/security.py` — argon2id password hash/verify, SHA-256 token hash, token generation.
  - `app/modules/audit/service.py::log_change()` — the one place any module should write an
    `AuditLog` row (fields-changed-only, per product CLAUDE.md rule #2).
  - Two migrations: `2df148b1f8be` (create all 8 tables) and `0f482c9106be` (seed 2 baseline
    permissions — `user.manage`, `report.view` — and 2 roles — `admin`, `qa` — with
    `role_permissions`; deliberately no `report.approve`, per decision #1 there's no separate
    approval role). **Gotcha hit and fixed**: Postgres enum types aren't dropped by
    Alembic-autogenerated `downgrade()` when their table is dropped — left orphaned, breaking a
    later `upgrade` with "type already exists". Fixed by explicitly dropping the 3 enum types at
    the end of `downgrade()` in `2df148b1f8be_...py` — copy this pattern for any future enum
    column. Verified with a full upgrade→downgrade→upgrade roundtrip.
  - `app/cli/bootstrap_admin.py` — idempotent (checks for an existing `admin`-role user first),
    creates the account in `Create` status (forces the activation flow instead of a real initial
    password), issues an `EmailVerificationToken`, writes an `AuditLog` row with `user_id=None`
    (system-initiated — no actor exists yet). Prints the activation token to stdout since email
    delivery isn't wired up until Phase 4. Run: `uv run python -m app.cli.bootstrap_admin`.
  - **Tests: 12 passing, 96% coverage on `app/`** — run against a real dedicated Postgres DB
    (`ok2ship_test`, same Docker container), recreated + migrated fresh each `pytest` session
    (`tests/conftest.py`). Covers: security helpers, RBAC model relationships (multi-role, seed
    data), `RefreshToken` rotation shape, bootstrap-admin creation + idempotency (incl. its
    audit-log and verification-token side effects).
  - Dev DB (`ok2ship`, not the test DB) now has the real seed data + one bootstrapped Admin
    account (status `Create`) — left in place for Phase 2 to build login/activation against.
- [x] **Phase 2 done (2026-08-19): auth core.**
  - `app/core/security.py` — added `create_access_token`/`decode_access_token` (PyJWT, HS256).
    JWT secret default lengthened to ≥32 bytes so a forgotten override doesn't also trip PyJWT's
    `InsecureKeyLengthWarning` — still an obviously-fake placeholder, still MUST be overridden
    outside local dev.
  - `app/core/permissions.py` — **the** `can(principal, permission_code)` choke point +
    `resolve_user_permissions(db, user)` (two separate queries — user's roles, then permissions
    for those role ids — deliberately not one flattened join, avoiding the JOIN fan-out trap the
    design doc calls out). `AuthenticatedPrincipal` dataclass carries the resolved role/permission
    codes for the life of one request.
  - `app/core/deps.py` — `get_current_user` (decodes the bearer token into a principal, **no DB
    hit** — that's the point of embedding permissions in the token) and `require_permission(code)`
    dependency factory (403 via `ForbiddenError` if missing).
  - `app/core/exceptions.py` — `AppError`/`UnauthorizedError`/`ForbiddenError`/`NotFoundError`;
    `app/main.py` registers handlers translating them to 401/403/404. Domain code never
    constructs an `HTTPException` directly.
  - `app/modules/auth/service.py` — `login()` (verifies password, blocks non-Active statuses,
    resets `failed_login_count`/`last_login` on success), `refresh()` (rotates the refresh token;
    **reuse of an already-rotated token is detected and revokes every refresh token for that
    user** — the explicit theft-detection deliverable from the approved plan, not an implicit
    detail), `logout()`. Every outcome writes `audit_log` (`user.login_success`,
    `user.login_failed`, `user.login_rejected_status`, `user.auto_locked`,
    `auth.refresh_token_reuse_detected`, `user.logout`).
  - **Chosen explicitly** (design doc flagged this as a decision to make, not defer): short-lived
    access token (~2 min, `access_token_ttl_seconds`) for near-instant force-logout on
    Lock/Inactive, **not** a Redis-backed live status check.
  - `app/modules/auth/router.py` — `POST /auth/login`, `/auth/refresh`, `/auth/logout`. Refresh
    token in an httpOnly cookie scoped to `/auth` (decision #7); `Secure` flag conditional on
    `environment != "development"` (a strict `Secure` cookie is silently dropped by browsers over
    plain HTTP, which local dev is).
  - `app/jobs/auto_lock.py::run_auto_lock()` — the other auto-Lock driver (90 days idle,
    `inactivity_lock_days`) that login-time code can't catch. Run via
    `python -m app.jobs.auto_lock`; wiring an actual scheduler (cron/systemd timer/etc.) is a
    deployment concern, not built here.
  - **New config, not previously specified in the design doc** — flagging for Sơn/vendor BA to
    confirm, not treated as silently locked: `max_failed_login_attempts` (default 5) and
    `inactivity_lock_days` (default 90, matches the design doc's stated number).
  - **Tests: 43 passing, 96% coverage on `app/`** (up from 12/96% after Phase 1). New coverage:
    JWT round-trip + expiry, `can()`/`resolve_user_permissions` (incl. multi-role permission
    union), `get_current_user`/`require_permission`, full login/lockout/refresh-rotation/
    reuse-detection/logout state machine, router-level integration tests (cookie set/rotated/
    cleared through a real `TestClient`), `auto_lock` (idle-past-threshold, recently-active,
    never-logged-in-but-old, never-logged-in-and-new, already-Locked/Inactive untouched).
- [x] **Phase 3 done (2026-08-20): User CRUD APIs.** All 6 vendor SRS use cases
  (List/Detail/Create/Edit/Delete/Active-Inactive), gated behind `require_permission("user.manage")`.
  - **Decision**: "Delete" and the Active/Inactive toggle collapse into **one** endpoint/service
    function (`PATCH /users/{id}/status`, `service.set_user_status`) — extends decision #6 (a
    Locked account recovers via the same toggle, no separate unlock action) one step further:
    `Inactive` *is* the soft-delete state, so there's no separate delete endpoint either. Flagged
    in "Next steps" for sign-off — not previously spelled out this explicitly in the design doc.
  - `app/modules/users/schemas.py` — `UserCreate`/`UserUpdate`/`UserStatusUpdate`/`UserResponse`.
    Username format (`^[a-zA-Z][a-zA-Z0-9._-]{5,49}$`) validated here, not in the DB, per the
    design doc. `UserUpdate` has no `username` field at all — immutability enforced structurally,
    not by a runtime check. `UserStatusUpdate` only accepts Active/Inactive (Create/Locked are
    reached automatically, never Admin-set directly).
  - `app/modules/users/service.py` — `list_users` (status/department/search filters + pagination),
    `get_user`, `create_user`, `update_user`, `set_user_status`. Editing email: invalidates all
    prior unused verification tokens (marks `used_at`, doesn't delete — keeps the audit trail),
    issues a fresh one, resets status to `Create` (vendor SRS UC-04 BR-02). No-op updates (nothing
    actually changed) write **no** audit entry — avoids audit noise.
  - `app/modules/users/email.py::send_activation_email()` — the Phase 3 stub (console log).
    `app/cli/bootstrap_admin.py` refactored to call this too, instead of duplicating the same stub
    print logic in two places.
  - `app/core/exceptions.py` — added `ConflictError` (409, e.g. duplicate username/email);
    `app/main.py` registers its handler alongside the Phase 2 ones.
  - `app/modules/users/router.py` — `GET/POST /users`, `GET/PATCH /users/{id}`,
    `PATCH /users/{id}/status`. Every route requires `require_permission("user.manage")`.
  - **Bug caught while testing (test-only, not product code)**: several service-layer tests
    passed a random `uuid.uuid4()` as `actor_id`, violating `audit_log.user_id`'s FK to `users.id`
    — real callers never hit this since `actor_id` always comes from a logged-in principal's real
    `user_id`. Fixed by adding an `actor_id` pytest fixture backed by a genuine committed user.
  - **Tests: 80 passing, 97% coverage on `app/`** (up from 43/96% after Phase 2). New coverage:
    username/email schema validation, full CRUD service logic (incl. duplicate/unknown-role/
    no-op/email-change-reissue-token paths), status toggle (deactivate/reactivate/Locked-recovery/
    resets `failed_login_count`/same-status no-op), router-level integration tests through a real
    `require_permission` gate (401 unauthenticated, 403 wrong role, 200/201/409/404 as expected).
- [x] **Phase 4 done (2026-08-21): real email delivery.** `app/modules/users/email.py` now sends
  via SMTP (stdlib `smtplib`, no new dependency) instead of only printing to console.
  - **Zero call-site changes** — `send_activation_email()` kept the exact same signature, so
    `bootstrap_admin.py` and `users/service.py` didn't need to change at all (the whole point of
    factoring it out as its own function in Phase 3).
  - **Dev fallback preserved**: empty `SMTP_HOST` (the default) keeps the Phase 3 console-log stub
    — local dev needs zero setup, nothing broke for anyone not configuring SMTP.
  - **Decision**: a failed SMTP send is logged, not raised. By the time it's called, the user
    create/update has already committed — a broken mail server shouldn't turn an already-successful
    API response into a 500. Flagging this as a real product decision, not an oversight: the
    operator finds out from server logs, not from the API response.
  - New config: `smtp_host`/`smtp_port`/`smtp_username`/`smtp_password`/`smtp_use_tls`/
    `smtp_from_email`/`smtp_from_name`, all in `.env.example` (commented out, since the empty
    default is the working local-dev state).
  - Generic SMTP, not a specific provider SDK — works with a company mail server, or any
    provider's SMTP interface (SES/SendGrid/Mailgun/etc.), without vendor lock-in or an extra
    dependency. No provider has been chosen/confirmed by Sơn or the vendor yet.
  - **Tests: 83 passing, 98% coverage on `app/`** (up from 80/97% after Phase 3). New: stub
    fallback when unconfigured, real SMTP path (mocked `smtplib.SMTP`, no real network calls),
    failure-is-logged-not-raised.
- [x] **Gap closed (2026-08-21): `POST /auth/activate`.** Found while manually testing Phase 4 —
  every phase since Phase 1 only *issued* activation tokens (bootstrap-admin, user create, email
  change), nothing *redeemed* one. Not in any phase's originally-approved scope; built once the
  gap became concrete/blocking during hands-on testing.
  - `app/modules/auth/service.py::activate_account()` — validates the token (unknown/used/expired/
    account-not-pending all rejected as `InvalidActivationTokenError`, 401 — same treatment as a
    bad login credential, since the token *is* the credential here), sets a real password
    (`UserStatus.CREATE` → `ACTIVE`), marks the token used, and **logs the user straight in**
    (same tokens `login()` issues) — no redundant extra login step.
  - **Decision**: redemption always requires setting/resetting a password — including for an
    email-change re-verification token, even though that user already has a working password.
    Deliberate simplification (defense in depth: re-proving identity on email change), not an
    oversight — flagged for sign-off, not silently assumed.
  - `app/modules/auth/schemas.py::ActivateAccountRequest` — first place `PASSWORD_PATTERN`
    (decision #8: min 8 chars, letters + digits) is actually enforced; every other password in the
    system so far has been a random unusable placeholder, never user-chosen.
  - Verified with a full live HTTP round trip (not just tests): create user → console-logged
    token → `POST /auth/activate` → status flips to `Active` → login with the new password
    succeeds.
  - **Tests: 92 passing, 98% coverage on `app/`** (up from 83/98% after Phase 4). New: token
    unknown/used/expired/account-not-pending rejection, successful redemption sets password +
    logs in + marks token used, weak-password schema rejection, router integration.
  - Postman collection updated (`Auth > Activate` added) — same file, no re-export needed by Sơn.
- [x] **Gap closed (2026-08-21): `POST /users/{id}/resend-activation`.** Found immediately after
  the previous gap, while Sơn reasoned through the activation UX: the original activation token
  is only good for `ACTIVATION_TOKEN_TTL_DAYS` (7 days) — until this, there was no way to get a
  new one if it expired before the user acted (opening the link itself doesn't consume the token —
  only submitting `/auth/activate` does — but the 7-day window is still real).
  - `app/modules/users/service.py::resend_activation()` — only valid while the user is still
    `Create` status (`ConflictError`/409 otherwise); reuses whichever `purpose` the *last* issued
    token had (`INITIAL_ACTIVATION` vs `EMAIL_CHANGE`) rather than hardcoding one, so a resend
    doesn't silently relabel which flow it's for. Reuses the existing `_issue_activation_token`
    (invalidate-then-issue) and `send_activation_email` — no new plumbing needed.
  - `POST /users/{id}/resend-activation`, gated by `user.manage` like every other route in this
    module.
  - **Tests: 99 passing, 98% coverage on `app/`** (up from 92/98%). New: old token invalidated +
    new one issued with the right purpose, the new token actually redeems successfully end-to-end
    (calls `auth_service.activate_account` with it), rejected on an already-Active user, 404 on
    unknown user, 403 without `user.manage`.
  - Postman collection updated (`Users > Resend activation` added).
- [x] **Phase 4.5 done (2026-08-22): self-service password flows, closing 3 gaps found reviewing
  the vendor's reference UI mockup** (`~/Downloads/app.html`, BA-provided — full review in
  `docs/design/frontend-mockup-review.md`). Decisions taken before building (confirmed with Sơn):
  keep multi-role RBAC (mockup's single-role dropdown is just an unfinished simplification, not a
  requirements change); keep design doc decision #8's password policy as-is (mockup's stricter
  upper+lower+digit copy needs fixing in the real Phase 5 form, not the backend regex).
  - `VerificationPurpose.PASSWORD_RESET` added to the enum (migration `b339be482f7f`). **Bug hit
    and fixed**: SQLAlchemy's `Enum` type stores the Python member *name* ('PASSWORD_RESET'), not
    its `.value` ('password_reset') — the first migration attempt used the value, causing a
    `DataError` at insert time. Fixed to match the existing `INITIAL_ACTIVATION`/`EMAIL_CHANGE`
    convention; dev DB had to be fully reset (`docker compose down -v`) since Postgres can't drop
    an enum value once added.
  - Refactored token issuance: `_issue_activation_token` (private, `users/service.py`) promoted to
    a shared public `app/modules/users/verification.py::issue_verification_token()` — both
    `users/service.py` and `auth/service.py` need it now, and reaching into another module's
    private function would have been worse than a small shared file.
  - `app/modules/users/email.py` refactored: `_send_email()` extracted as the one place SMTP
    actually happens; `send_activation_email()`/`send_password_reset_email()` are thin wrappers.
  - `POST /auth/forgot-password` — always the same generic response regardless of whether the
    email exists (anti-enumeration, matches the mockup's own stated copy exactly). Only issues a
    token for `Active` or **`Locked`** accounts — a `Locked` account redeeming the resulting token
    self-recovers to `Active`, matching a behavior found in the mockup's own JS that wasn't
    initially obvious from its screens alone (self-service unlock via "forgot password", not just
    password recovery).
  - `POST /auth/reset-password` — redeems the token (30 min TTL, single use, matches mockup copy
    exactly). Deliberately does **not** auto-login (unlike `/auth/activate`) — forces a fresh
    login, matching the mockup's own post-reset behavior (a considered difference: recovery flow
    vs. first-time onboarding).
  - `POST /auth/change-password` — self-service, requires current password, rejects
    new-equals-current (both this and reset-password check via `verify_password` against the
    hash, not a plaintext compare — the mockup's demo fakes this with plaintext since it's not a
    real backend). Session stays active afterward — **other devices/sessions are deliberately not
    revoked**, flagged as a considered omission (reasonable hardening for later), not a silent gap.
  - `POST /auth/login` now accepts **username or email** in one field (renamed
    `LoginRequest.username` → `username_or_email`) — matches the mockup's own label exactly.
    Touched existing tests/Postman collection (same-session cleanup, not a breaking change to
    anything committed).
  - Renamed `InvalidActivationTokenError` → `InvalidVerificationTokenError` (now shared by both
    `activate_account()` and `reset_password()` — both validate the same token shape, just
    different `purpose`).
  - **Two test-isolation bugs caught and fixed (test-only)**: two new tests asserted a global
    `EmailVerificationToken` table count, which broke once other tests' leftover rows (the shared
    test DB isn't wiped between individual tests, only between full `pytest` sessions) were
    present — fixed by scoping to the specific user or a before/after delta instead of an absolute
    count.
  - **Known pre-existing fragility, not fixed this session**: `tests/cli/test_bootstrap_admin.py`
    assumes it's the only test that ever assigns the `admin` role — running a narrow subset of
    test files in an order where other admin-role users get created first can make its "exactly
    one admin" assertion fail. Full-suite runs are unaffected (confirmed: 3 consecutive full runs,
    120/120 passing every time) — flagging as a known sharp edge for future test authors, not an
    active bug.
  - Full live HTTP smoke test performed (not just automated tests): forgot-password → console
    token → reset-password → login with new password → login **by email** instead of username →
    change-password (wrong current password rejected, correct one works, new password logs in).
  - **Tests: 120 passing, 98% coverage on `app/`** (up from 99/98% after the two earlier
    gap-closes).
  - Postman collection updated (`Auth > Forgot password`/`Reset password`/`Change password`
    added; `Login` body field renamed).
- [x] **Gap closed (2026-08-22): `GET /roles`.** Found immediately at the start of Phase 5 — the
  frontend's multi-role picker needs to know what roles exist, and nothing exposed the seeded
  `admin`/`qa` roles. `app/modules/users/service.py::list_roles()` + `GET /roles` (gated by
  `user.manage`, same as everything else in this module). Small enough to return unpaginated.
  Tests: +2 (list succeeds for admin, 403 for non-admin). Postman collection updated (`Roles >
  List roles`).
- [x] **Bug caught and fixed while building Phase 5's UI (2026-08-22):
  `set_user_status()` allowed `Create` → `Active` directly.** Reasoning through which grid action
  icons to show for a pending (`Create`-status) row surfaced it: nothing stopped an Admin from
  flipping a pending account straight to `Active` via the toggle endpoint — but that account still
  has the random *unusable* `password_hash` from `create_user()` (real password is only ever set
  by redeeming the activation token). The result: status says `Active`, but the user can never log
  in, and their real activation token — if still unused — stops working too (`activate_account()`
  requires `status == Create`). `set_user_status()` now rejects `Create → Active` with
  `ConflictError` (409); the only legitimate way out of `Create` is redeeming the activation token
  (`/auth/activate`) or `resend_activation()`. Test added. Frontend's action-icon logic reflects
  this: a `Create`-status row gets Resend-activation, never the Active/Inactive toggle.
- [x] **Phase 5 done (2026-08-22): frontend.** `frontend/` — React 18 (deliberately pinned; Vite's
  scaffold defaults to 19, `CLAUDE.md` locks 18 — pinning avoids an ADR rather than deviating),
  Vite, Tailwind v4, React Router v6, AG Grid Community (real, not just visually — matches the
  vendor mockup's own actual choice), axios. Full architecture + auth model in
  `frontend/README.md`. Screens: Login (username-or-email) → Forgot/Reset password → app shell
  (sidebar previews the full future product, User Management is the one real entry) → User
  Management grid (List/Detail/Create/Edit/Delete-as-Inactive/toggle/resend-activation) with
  **multi-role** checkboxes (not the mockup's single-select — confirmed decision, Phase 4.5 entry
  above) → header Change-password modal.
  - **Backend addition needed and built first**: CORS middleware (`app/main.py`,
    `cors_allowed_origins` config) — credentialed requests (the refresh cookie) forbid a wildcard
    origin, so this had to be an explicit allow-list from the start, not bolted on later.
  - **Three real bugs found and fixed live-testing the built app** (Playwright, not just
    inspection — see "Verification" below), none of them things a code read would have caught:
    1. **Vite dev proxy path collision.** Proxying bare paths (`/users` → backend) collided with
       the frontend's own `/users` SPA route — a hard reload on `/users` made Vite's proxy forward
       the *page navigation itself* to the backend, which returned a raw `{"detail":"Missing
       bearer token"}` JSON body instead of the app shell. Every future module will add both a
       frontend page and a backend route with the same natural name, so this wasn't a one-off.
       **Fixed**: backend now lives under `/api` in dev (`vite.config.ts` proxies `/api` and
       strips the prefix before forwarding) — structurally can't collide with an SPA route again.
    2. **Refresh-token race → self-inflicted theft detection.** React 18 StrictMode
       double-invokes effects in dev; `AuthContext`'s on-mount silent-refresh calling
       `authApi.refresh()` directly meant two real, concurrent `POST /auth/refresh` calls raced
       the backend's rotate-on-use refresh token — the loser looked exactly like a replay of an
       already-rotated token and tripped Phase 2's reuse-detection, revoking the whole session.
       Reloading the page was silently logging the user out. **Fixed**: every refresh trigger
       (the interceptor's 401 retry, and `AuthContext`'s on-mount attempt) now goes through one
       shared, deduplicated `refreshAccessToken()` (`api/client.ts`) — at most one real HTTP call
       in flight at a time, no matter how many places ask for it.
    3. **Refresh cookie `Path=/auth` broke once fronted by `/api`.** The browser matches a
       cookie's `Path` against the path *it* used, not whatever the backend received internally
       after a proxy rewrite — `/api/auth/refresh` doesn't start with `/auth`, so the cookie
       silently stopped being sent. **Fixed**: cookie `Path` changed to `/` (`app/modules/auth/
       router.py`) — decouples the backend from any specific reverse-proxy prefixing scheme,
       correct regardless of what sits in front of it in dev or prod.
  - **AG Grid v33+ Theming API**: importing the legacy CSS-file themes *and* not setting a
    `theme` grid option trips AG Grid's own error #239 (both styling systems active at once).
    Fixed by adopting the new Theming API cleanly (`theme={themeQuartz}` prop, no CSS imports) —
    the modern default for this AG Grid version, not the deprecated path.
  - **Frontend tests**: vitest + React Testing Library, 21 tests (password policy, API-error
    formatting, the delete/toggle status-transition logic — including a regression test that it
    doesn't repeat the mockup's own buggy Locked-label ternary — `tokenStore` pub/sub,
    `ProtectedRoute`'s loading/redirect/permission-gate states). `npm test`. Known gap, flagged in
    `frontend/README.md`: no component tests yet for the page-level flows (Login, the User
    Management grid itself) — verified live instead (below), not by a checked-in suite.
  - **Verification**: a real Playwright smoke test (headless Chromium, not just reasoning about
    the code) — login, page reload preserves the session, create a user, delete
    (→ `Inactive`, confirmed via screenshot that the copy no longer claims irreversibility),
    resend-activation, logout, and confirms no leftover session survives a post-logout reload.
    Screenshots taken at each step. This is how bugs 1–3 above were actually caught — none were
    visible from reading the code.
- [ ] `frontend/` has no automated tests for page-level flows yet (see "Frontend tests" above) —
  the live Playwright pass substituted for Phase 5 itself, but a checked-in suite is still owed.
- [x] **Branding fix (2026-08-22, caught by Sơn): wrong product name/logo/copy on the auth
  pages.** Root cause: `docs/design/frontend-mockup-review.md`'s original pass filtered out every
  line over 2000 chars (base64 image data) to stay readable — which also silently dropped the
  real Mektec logo and the login page's actual left-panel headline/copy, both embedded on those
  long lines. Phase 5 built against the filtered (incomplete) view and invented placeholder
  text/name instead ("OK2Ship AI Check" instead of the real "OK2SHIP AI", no logo, paraphrased
  copy). Fixed: extracted the real logo to `frontend/public/logo.png`
  (`AuthLayout.tsx`/`Sidebar.tsx` now render it), corrected the product name everywhere
  (`index.html` title, sidebar, auth pages), copied the mockup's actual headline/body/footer text
  verbatim instead of paraphrasing. Mockup review doc corrected with the lesson learned — see its
  "Correction" note.
- [x] **Second fidelity pass (2026-08-23, caught by Sơn comparing screenshots side by side):
  auth-form inputs had no icons, no "Ghi nhớ đăng nhập" checkbox, buttons had no shimmer effect,
  no animated circuit background.** Same underlying lesson as the branding fix — re-read the
  mockup's actual CSS ("AI MOTION LAYER" section) instead of approximating. New shared components:
  `PasswordField` (lock icon + show/hide eye toggle), `Checkbox`, `icons.tsx` (person/lock/mail/eye
  SVGs matching the mockup exactly); `TextField` gained an optional `icon` prop; `Button` gained a
  `shimmer` prop — used deliberately sparingly (one primary CTA per screen, matching the mockup's
  own stated design rule, not slapped on every button). Also fixed while re-reading the mockup's
  modal footers precisely: the status-toggle confirm button is always primary/blue with the
  static label "Xác nhận" (was dynamically "Kích hoạt"/"Vô hiệu hóa"); Delete's confirm button
  never shimmers (destructive action, no motion); Logout's confirm button is primary/blue, not
  danger/red (logout isn't data-destructive). Animated `circuit-bg` SVG (dashed flowing lines +
  pulsing dots) added to the auth pages' dark panel, reproducing the mockup's path data/colors/
  timings exactly. Re-verified with Playwright screenshots.
- [x] **Third fidelity pass (2026-08-23): auth panel width — a real bug in the mockup file
  itself, not a research gap this time.** The mockup has two nested `<div class="auth-page">`
  elements (a leftover duplicate wrapper) — the inner one, which actually lays out the dark panel
  + form, shrink-to-fits its content instead of stretching to the viewport, so `.auth-side` never
  actually renders at its declared `width: 46%`. Confirmed by opening the mockup file directly and
  reading `getBoundingClientRect()` across viewport widths 1280–2200px: constant **~434px**, not a
  percentage at all. Decision (confirmed with Sơn): match the mockup's actual buggy rendering, not
  its declared-but-inapplicable 46% — `AuthLayout.tsx`'s dark panel is a fixed `w-[434px]`, not
  `lg:w-[46%]`. Re-verified pixel-for-pixel against the mockup's real render at multiple widths.
- [x] **Fourth fidelity pass (2026-08-23): form position + circuit-bg proportions — two more
  symptoms of the same root causes as the third pass, plus one new bug.** Sơn: "content ui đăng
  nhập sẽ bám cạnh phải, chứ không nằm giữa ô bên phải như hiện tại. các vân chuyển động của
  sidebar bên trái cũng chưa chuẩn tỉ lệ hiển thị."
  - **Form position.** Read the mockup's raw markup directly and confirmed the duplicate
    `<div class="auth-page">` from the third pass (`#authShell` and its immediate child both carry
    the class) — the OUTER one is the real flex container spanning the viewport; the INNER one
    (which actually holds `.auth-side` + `.auth-form-wrap`) becomes a flex ITEM of it with no
    `flex-grow`, so the *entire* auth block shrink-to-fits instead of stretching. This caps not
    just the sidebar (third pass) but also `.auth-form-wrap` at a measured constant **~510px**,
    sitting flush against the sidebar's right edge — never centered across the full remaining
    viewport the way `flex-1` (previous `AuthLayout.tsx`) rendered it. Confirmed by measuring
    `.auth-card-box`'s position across 5 viewport widths (1280–3008px): fixed at x≈[499, 879] in
    every case, with the gap to the browser's right edge only growing. Fixed: right-hand panel is
    now `w-[510px] lg:flex-shrink-0` (not `flex-1`), with `bg-gray-50` moved to the outer wrapper
    so the leftover space past ~944px reads as page background, exactly as in the mockup.
  - **Circuit-bg proportions — a genuine third mockup bug, not a rediscovery of the second.** The
    mockup's `.circuit-bg` SVG has `position:absolute; inset:0` but no CSS width/height. For an
    absolutely-positioned *replaced* element (svg/img/video), `inset:0` alone does not stretch it
    like a `<div>` — width resolves from the left/right insets (100% of `.auth-side`), then height
    is derived from the SVG's own intrinsic `viewBox` ratio (460:800), ignoring the top/bottom
    insets. Measured: a 434px-wide panel renders the svg at 434×755px, leaving the bottom ~145px
    of the dark panel bare. Reproduced (not "fixed") by dropping `h-full` in favor of
    `aspect-[23/40]` (=460/800) on the `<svg>` in `CircuitBackground` — same visual result, matched
    to 0.4px against the mockup's real render.
  - Verified: local app vs. mockup at 1600px width — aside/card/svg boxes all within ~4px.
    `tsc --noEmit`, `vitest run` (21/21), and `vite build` all pass.
- [x] **Fifth fidelity pass (2026-08-23): sidebar gradient + primary button color, wrong on both.**
  Sơn asked to double-check, not just eyeball — measured `getComputedStyle()` on the mockup's real
  render instead of trusting the CSS source read. Both were off:
  - Sidebar gradient was `linear-gradient(to bottom right, #0B0C2A, #0D0E47)` (Tailwind's
    `bg-gradient-to-br`); the mockup's real computed value is `linear-gradient(160deg, #12145E
    0%, #0D0E47 100%)` — wrong start color AND wrong angle. Fixed with an arbitrary-value
    `bg-[linear-gradient(160deg,#12145E_0%,#0D0E47_100%)]` in `AuthLayout.tsx`.
  - Primary button was Tailwind's stock `indigo-600` (`#4F46E5`); the mockup's AntD theme token
    `--ant-colorPrimary` is `#2B2FA0` (visibly darker, less purple) — confirmed via
    `getComputedStyle()` on the real "Đăng nhập" button, not just the CSS variable declaration.
    `Button.tsx`'s `primary` variant now uses `#2B2FA0` / hover `#4247B8` / active `#1D2088`,
    matching `--ant-colorPrimary` / `*Hover` / `*Active` exactly. Also corrected the disabled
    state, which the mockup renders as a neutral gray fill (`--ant-colorFillQuaternary`,
    `#FAFAFA`), not a lighter tint of the primary color.
  - Verified: `getComputedStyle()` on the local app now returns byte-identical
    `rgb()`/`linear-gradient()` values to the mockup. `tsc --noEmit` and `vitest run` (21/21) pass.
- [x] **i18n gap fixed (2026-08-23, Standard flow — plan debated, code debated, both closed
  before merge): backend error messages no longer leak to the UI in raw English.** Found while
  auditing: `apiError.ts` passed the backend's English `detail` straight through whenever
  present; per-caller Vietnamese `fallback` strings only fired on the rare case `detail` was
  *absent*. Fixed with a stable error-code mechanism, not text translation:
  - **Backend**: `app/core/error_codes.py` — a new `ErrorCode` StrEnum (33 members). Every
    `AppError` (and its ~10 subclasses across `auth/service.py`, `users/service.py`, `deps.py`,
    `auth/router.py`) now requires `code: ErrorCode` at construction — no default, so a raise site
    that forgets one is a `TypeError` at the `raise` itself, not a silent gap. `app/main.py`'s
    exception handlers add `"code"` to the JSON body alongside the existing English `"detail"`; a
    new `RequestValidationError` handler gives every Pydantic 422 a blanket
    `"code": "VALIDATION_ERROR"` (preserves FastAPI's default `detail` shape via
    `jsonable_encoder`, only adds the one field). `AccountNotUsableError`'s status→code mapping
    has an explicit `AUTH_ACCOUNT_STATUS_UNKNOWN` fallback for schema drift, not a bare dict
    index.
  - **Sync, not hope**: `app/core/error_codes.json` (checked in) is regenerated from the enum by
    `uv run python -m app.core.error_codes`; a backend test fails CI if it's stale.
    `frontend/src/lib/errorCodes.test.ts` `?raw`-imports that same file (needed adding
    `server.fs.allow` in `vitest.config.ts`, scoped to `../backend/app/core`) and asserts the
    frontend's `ERROR_CODES` array matches exactly — a code added on one side without the other
    fails a test, not silently at runtime.
  - **Frontend**: `src/lib/errorMessages.ts` — `Record<ErrorCode, string>` Vietnamese catalog;
    the type itself makes a missing translation a `tsc` build failure, not a runtime gap.
    `apiError.ts` rewritten to key off `code` only — raw English `detail` is never shown to a
    user again, in any path. Every per-call-site domain-specific `fallback` string (LoginPage,
    ChangePasswordModal, ActivateAccountPage, ResetPasswordPage) was removed once `code` covered
    every real error each flow can raise — a hardcoded "wrong password" fallback would otherwise
    keep firing (and misleading) on the one case left uncovered: a genuine network/500 error.
  - **Both a plan debate and a code debate ran** (independent reviewer sub-agents) before this
    landed. Plan debate caught: missing sync mechanism (fixed above), undercounted token-error
    variants (10 not 5 — activation *and* password-reset flows both covered), missing codes for
    `ForbiddenError`/`NotFoundError`/refresh-token errors (all added). Code debate caught three
    real issues, all fixed: 4 call sites' fallback strings were stale/misleading post-`code`
    (removed, see above); `code in ERROR_MESSAGES` in `apiError.ts` was a prototype-chain check,
    not an own-property one (`Object.prototype.hasOwnProperty.call` now); `AUTH_TOKEN_INVALID`'s
    Vietnamese text overclaimed "expired" when the same code also covers a malformed/garbage
    token (reworded to "không hợp lệ hoặc đã hết hạn", matching the hedge already used
    elsewhere).
  - **Verified end-to-end**, not just unit tests: `curl` against the live backend confirmed the
    real JSON body carries `code`; a Playwright run against the real login page confirmed the UI
    now shows the Vietnamese message and not the English string. 134 backend tests / 25 frontend
    tests / `tsc -b` / `vite build` all green.
  - One branch (`activate_account`'s `AUTH_ACCOUNT_GONE`) is intentionally untested — a
    verification-token row surviving while its user row is deleted is impossible given
    `email_verification_tokens_user_id_fkey` has no `ON DELETE CASCADE` (confirmed by trying it:
    real `IntegrityError`). The same code path IS tested via `change_password`'s equivalent
    branch, which doesn't hit that FK.
- [x] **Bug found by Sơn (2026-08-23): clicking "Đăng nhập" with an empty/wrong form also fired
  an unnecessary `/auth/refresh` call. Standard flow, plan debated, fixed.** Root cause:
  `client.ts`'s axios response interceptor treated *any* 401 (except from `/auth/refresh` itself)
  as "the access token expired — silently refresh and retry" — but `/auth/login` (wrong
  credentials), `/auth/change-password` (wrong current password), `/auth/activate` and
  `/auth/reset-password` (bad/expired/used token) all legitimately return 401 for domain reasons
  that have nothing to do with the access token. Every one of those got misread as "token
  expired," firing a needless extra `/auth/refresh` call. Fixed by gating strictly on the error
  `code` (from the i18n mechanism above) instead of the bare HTTP status: only
  `AUTH_TOKEN_MISSING`/`AUTH_TOKEN_INVALID` — the two codes `deps.py::get_current_user` can raise,
  on protected routes only — now trigger refresh-and-retry; every other 401 code just surfaces its
  (translated) error. The decision logic is now a standalone exported `shouldAttemptRefresh(error)`
  in `client.ts`, taking the raw `AxiosError` rather than pre-destructured `status`/`code`/`config`
  — deliberately, to avoid a real trap a plan-debate reviewer caught: `AxiosError` has its own
  unrelated `.code` field (e.g. `"ERR_BAD_REQUEST"`), easy to grab by mistake instead of the
  backend's `error.response.data.code`. New `client.test.ts`: 8 unit tests on the pure decision
  function plus 2 end-to-end tests that mock `apiClient`'s adapter and the global `axios.post` to
  exercise the *actual* interceptor wiring, not just the extracted function — confirmed to
  actually catch the regression by temporarily reverting to the old logic and watching 4 tests
  fail, including the end-to-end one, before restoring the fix. Re-verified live via Playwright:
  submitting the empty login form now fires exactly one `/auth/login` call and zero
  `/auth/refresh` calls.
- [x] **Follow-up requested by Sơn (2026-08-23, Quick flow — single file, client-side only, low
  risk): the login form should never call the API at all if it's blank.** Root gap: `LoginPage`'s
  `<form noValidate>` had no client-side check of its own, and `TextField`/`PasswordField`'s
  `required` prop was purely decorative (only rendered the red asterisk label, never actually set
  the HTML `required` attribute on the `<input>`) — so a blank submit always reached the API.
  Fixed with a new pure `validateLoginForm(usernameOrEmail, password)` in
  `src/lib/loginValidation.ts` (mirrors the repo's existing pattern of extracting testable logic
  out of components — see `userStatusPlan.ts`/`password.ts`) — `LoginPage.handleSubmit` calls it
  first and returns immediately if either field is empty, showing a field-level red error message
  under the specific empty input (using `TextField`/`PasswordField`'s existing `error` prop) rather
  than only a generic top banner. 5 new unit tests. Verified live via Playwright: blank submit →
  zero API calls (both field errors shown); only-username-filled → zero API calls; both filled →
  exactly one `/auth/login` call, as expected.
- [x] **`is_deleted` column added, Standard flow (plan debated, code verified live) — distinguishes
  "Xóa" (Delete) from "Vô hiệu hóa" (Deactivate) in data/audit for the first time.** Sơn's request;
  3 design questions resolved with him first (all "Recommended"): additive on top of `status`, not
  a replacement (Delete still targets `status=Inactive`, just also sets `is_deleted=true`); no
  change to list visibility or the reactivate UX; no change to username/email uniqueness.
  - **Plan-debate caught a real bug before any code was written**: the two-button sequence the
    feature exists for — Deactivate (→ Inactive, is_deleted=false) then, later, Delete (→
    Inactive again) — both target the *same* status. `set_user_status`'s existing no-op guard
    (`if user.status == new_status: return`) only checked `status`, so the second call would
    silently no-op and never actually set `is_deleted=true`. Fixed: the guard now checks both
    fields.
  - `set_user_status()` signature gained `is_deleted: bool = False`. Rejects `status=Active` +
    `is_deleted=True` with a new `ConflictError` (`USER_CANNOT_BE_ACTIVE_AND_DELETED`) — same
    "reject, don't silently coerce" treatment this function already gives Create→Active — rather
    than quietly overriding the request. Always clears `is_deleted` back to `False` whenever
    `status` becomes `Active`. Audit log's `before`/`after` now includes only whichever of
    `status`/`is_deleted` actually changed on a given call, not both unconditionally.
  - `login()` needed **no change** — `is_deleted=True` can only ever coexist with
    `status=Inactive` (enforced both directions above), which already blocks login via the
    existing `status != ACTIVE` check. `is_deleted` is purely an additive audit/reporting
    dimension, never a second security gate.
  - Migration is forward-only: pre-existing `Inactive` users (including any that were
    functionally deletions in intent, from before this column existed) backfill to
    `is_deleted=false` — there's no way to know retroactively which were which.
  - Frontend: `userStatusPlan.ts::planFor()` now returns `isDeleted` per mode (`true` only for
    `'delete'`), threaded through `StatusConfirmModal` → `api/users.ts::setUserStatus()` → the
    PATCH body.
  - 7 new backend tests (the exact Deactivate→Delete sequence that was broken, Delete→Reactivate→
    Delete-again, the Active+is_deleted rejection, audit-only-changed-fields) + 5 new/extended
    frontend tests. Verified live (not just unit tests): direct API calls against the running
    backend reproduced the Deactivate→Delete sequence and confirmed `is_deleted` correctly flips
    to `true`; a Playwright click-through on the real grid confirmed clicking "Xóa" sends exactly
    `{"status":"Inactive","is_deleted":true}`. 141 backend / 42 frontend tests, `tsc -b`,
    `vite build` — all green.
- [x] **k8s-readiness code audit (2026-08-23, Standard flow) — target infra is k8s, currently only
  docker-compose (local dev deps only, no app Dockerfile/manifests/CI anywhere yet).** Audited the
  codebase for anything that breaks under multiple replicas / rolling restarts before writing any
  infra artifacts (out of scope for this pass — Sơn deliberately deferred Dockerfile/k8s
  manifests/CI/frontend-serving-strategy to later, once cluster/registry details are known). Found
  mostly already fine (config via env vars, no in-process state/cache/scheduler, the auto-lock job
  already a standalone script matching a future CronJob) — two real gaps, both fixed:
  - **`/health` conflated liveness and readiness.** It returned a static `{"status": "ok"}` with no
    DB check — fine as a k8s *liveness* probe (a DB outage failing liveness would restart every pod
    for no reason, possibly all at once), but there was no separate signal for k8s to pull a pod out
    of traffic rotation while its DB dependency is down/starting. Added `/health/ready` — runs
    `SELECT 1`, 200 if reachable, 503 otherwise. `/health` itself is unchanged.
  - **DB connection pool size was hardcoded** (SQLAlchemy's own defaults, 5/10, baked into
    `create_engine()` with no way to override). Under k8s, total Postgres connections = replica
    count × (pool_size + max_overflow) — needed to be tunable once real replica counts are known,
    without a code change. Added `db_pool_size`/`db_max_overflow` to `Settings` (defaulting to the
    exact same 5/10, so behavior is unchanged unless explicitly overridden) and wired them into
    `create_engine()`.
  - 6 new tests (`/health/ready` reachable/unreachable via a swapped-in broken DB session; pool
    settings default/override/actually-wired-to-the-engine). Verified live against the running
    dev server: both endpoints return correct status/body. 146 backend tests, all green.
- [x] **Seventh/eighth fidelity passes (2026-08-23): app-wide font audit, then a page-title
  spot-check.** Sơn: check every text element against the mockup, flagged sidebar copy and button
  text as examples. Root cause found first: the mockup doesn't just declare `'Inter', ...` in CSS
  — it actually `@import`s the real Inter webfont from Google Fonts. The app had no webfont at
  all, so every piece of text app-wide was rendering in the wrong typeface. Fixed by loading the
  same Google Fonts URL (Inter + JetBrains Mono) in `index.html` + updating `index.css`. With the
  typeface fixed, re-measured and found 5 more real mismatches (sidebar headline 26px not 24px;
  card title 24px/`rgba(0,0,0,.88)` not 20px/gray-900; card subtitle
  `rgba(0,0,0,.65)` not gray-500; auth submit buttons need the mockup's `.ant-btn-lg` treatment —
  15px/weight 400 vs the smaller default `.ant-btn` used everywhere else, 14px/weight 400 — added
  a `size` prop to `Button.tsx`; Users page `<h1>` 20px not 18px). Not exhaustively audited beyond
  auth pages + one heading — sidebar nav, the AG Grid table, and modal bodies inherit the
  font-family fix but weren't individually re-measured. 42 frontend tests, build — all green.
- [x] **Backend repo split off + pushed to GitLab (2026-08-23).** Sơn asked what's needed to push
  backend to a dedicated GitLab repo — audited first: `backend/docker-compose.yml` was always
  local-dev-only, nothing had ever been containerized/deployed, and the whole product repo had
  zero commits (nothing committed all session). Resolved with Sơn: SSH auth (key already on
  GitLab), backend-only (not the whole monorepo), target repo empty, single squashed "initial
  import" commit (a true phase-by-phase replay isn't achievable — nothing was committed
  incrementally while building, so there are no real historical snapshots to split apart without
  fabricating them — offered a module-grouped 8-commit alternative, Sơn chose the single commit).
  Found and fixed along the way: `backend/README.md`'s `cd backend` line would mislead once
  `backend/` is a repo root itself (removed); needed a `backend/.gitignore` scoped to the new root
  (the parent repo's `.gitignore` used `backend/`-prefixed paths, wrong once `backend/` is its own
  root); the pre-commit hook copied from the parent repo silently skipped its own test-running
  step (`command -v pytest` fails — this project only has `pytest` via `uv run pytest`) and its
  `.env`-blocking regex was too broad (blocked `.env.example`, a safe checked-in template with no
  real secrets — verified its contents before concluding that) — both fixed before the first
  commit. Result: `git@gitlab.com:mektec/ok2ship-ai-backend.git`, branch `main`, commit `7463a70`.
- [x] **Frontend repo split off + pushed to GitLab (2026-08-23), same pattern.** Also trimmed
  `frontend/README.md` down to just Setup/Run (Sơn: everything else already lives in
  HANDOFF.md/docs/PROGRESS.md) and dropped its `cd frontend` line once frontend became its own
  repo root. Found a real problem this time that backend's push didn't have: `errorCodes.test.ts`
  (the i18n sync check — see its own HANDOFF entry above) reads
  `../../../backend/app/core/error_codes.json`, a path that only resolves when `backend/` is
  checked out as a sibling — true in this monorepo checkout, false for anyone who clones just the
  new frontend repo. First fix attempt (dynamic `import()` wrapped in try/catch) didn't actually
  work — verified by simulating the missing-file case, watched it crash the whole test file
  anyway, because Vite's module-resolution failure happens at the transform stage, not as a
  catchable JS rejection. Fixed for real with `import.meta.glob(...)` instead (Vite's own
  zero-or-more-matches mechanism — an absent file resolves to an empty object, not a throw); the
  sync test now degrades to `it.skipIf`-skipped rather than failing when `backend/` isn't present.
  Verified both branches for real (temporarily moved the backend file aside, confirmed skip, then
  confirmed the real check still runs with it restored) before committing. Result:
  `git@gitlab.com:mektec/ok2ship-ai-frontend.git`, branch `main`, commit `2e7c6e8`.

## Decisions locked — do not re-litigate (full detail + rationale in docs/design/user-management.md)
1. No separate approval role — uploader also reviews. `audit_log` is the critical safety net.
2. Full RBAC — a user can hold multiple roles (`roles`/`permissions`/`role_permissions`/`user_roles`).
3. Schema uses industry-standard RBAC naming, NOT the vendor's inverted terms (translation table in
   the design doc — use it when talking to BA/Desoft so nobody talks past each other).
4. Report visibility is role-based, not Line-scoped (`lines`/`user_line_scope` removed). If Line
   restriction returns, model it as more roles first (cheap) — only build a scope table again past
   ~4–5 Line-specific roles.
5. `department` is informational only, stored as a plain inline enum on `users` — NO separate
   `departments` reference table (considered, then dropped: normalizing it isn't worth it before
   it has an actual permission use beyond display).
6. A `Locked` account is reactivated via the same Active/Inactive toggle (UC-06) — no separate
   "unlock" endpoint/use-case.
7. Refresh tokens live in an **httpOnly cookie** on the client (not JS-readable storage).
8. Password policy: minimum 8 characters, must include letters and digits.

## Approved plan — build order (reviewed adversarially before approval)
1. **Phase 0** — Backend scaffold: `backend/pyproject.toml` (FastAPI, SQLAlchemy, Alembic,
   Pydantic v2, argon2-cffi, pyjwt, pytest), DB config via env vars, Alembic init.
2. **Phase 1** — Migration creating all **8** tables (`users, permissions, roles, role_permissions,
   user_roles, email_verification_tokens, refresh_tokens, audit_log` — no `departments` table, see
   decision #5). Seed baseline permissions + roles (`admin`, `qa`) + `role_permissions`. Bootstrap
   Admin account via a **separate idempotent CLI/seed command** (NOT inside the Alembic migration —
   migrations must stay safely re-runnable), force password change on first login, write an
   `audit_log` row for its own creation.
3. **Phase 2** — Auth core: argon2id hashing (policy above), login issuing a short-lived
   (~2 min) access token with **roles+permissions embedded as claims** (avoid re-joining 3 tables
   per request) + a rotated refresh token in an httpOnly cookie. `can(user, permission_code)` is
   the single permission-check choke point. Increment/reset `failed_login_count` on
   login attempts. A **daily scheduled job** auto-locks accounts (too many failed logins, or
   `last_login` > 90 days). Refresh-token-reuse detection (theft signal → revoke whole session) is
   an explicit deliverable with its own test, not an implicit detail.
4. **Phase 3** — User CRUD APIs matching the vendor SRS's 6 use cases (List/Detail/Create/Edit/
   Delete/Active-Inactive). Create uses a **stub email sender (logs to console)** so it isn't
   blocked on Phase 4. Active/Inactive toggle also handles unlocking `Locked` (decision #6). Every
   action writes `audit_log`. Editing email invalidates old verification tokens, issues a new one,
   resets status to `Create`.
5. **Phase 4** — Real email delivery (SMTP/transactional provider), secrets via env vars, replacing
   the Phase 3 stub.
6. **Phase 5** — Frontend: login page with a silent-refresh/401 interceptor + User Management
   screens. **Blocked on locating the vendor's reference UI mockup** (`user-management.html`, AG
   Grid Community) — not present anywhere in this repo; ask Sơn to source it before starting this
   phase.
7. **Phase 6** — Integration + qa-reviewer pass + merge. Each phase above should land as its own
   small PR (branch `feature/<slug>`), not one giant end-of-build diff.

## Next steps (pick up here)
1. **Done (2026-08-23): git setup + first commits.** `backend/` and `frontend/` are now separate
   repos, each pushed as one "initial import" commit to their own GitLab project — see "Repo
   topology" at the top of this file for URLs/commit SHAs/hook details. Not a phase-by-phase
   commit history (impossible to reconstruct honestly after the fact — nothing was committed
   incrementally while building; see the git-workflow HANDOFF entries below for the full
   reasoning). Future changes should land as normal small commits/branches per the Constitution's
   git rules, in each repo independently.
2. **Phase 6** — Integration + qa-reviewer audit + merge (see "Approved plan" above). Now that
   initial commits exist in both repos, this can proceed as a normal branch-per-feature flow from
   here on, in whichever repo(s) a given change touches.
3. Checked-in frontend tests for page-level flows (Login, the User Management grid) are still
   owed — Phase 5 was verified live via Playwright instead (see its HANDOFF entry above). Decide
   whether to backfill before Phase 6's merge gate or treat it as a fast-follow.
4. Locate the vendor's reference for the *forgot/reset/activate* screens' exact wording if one
   exists beyond `~/Downloads/app.html` (already reviewed and built against) — not blocking,
   just worth confirming nothing drifted from a newer mockup revision.
5. Dev DB was fully reset during Phase 4.5 (enum fix) — `demo_admin`/`demo_qa` and the bootstrap
   Admin were recreated, but note for next session: any manual test data from before Phase 4.5 is
   gone.
6. Get sign-off on values/decisions invented along the way, not in the original design doc:
   - `max_failed_login_attempts` (default 5, now vendor-confirmed via the mockup) and the exact
     lockout UX — Phase 2.
   - "Delete" and the Active/Inactive toggle collapsing into one endpoint (no separate delete
     route) — Phase 3.
   - Failed SMTP send is logged, not raised, and no email provider has been chosen yet — Phase 4.
   - `/auth/activate` requiring a password reset even for an email-change token, not just an
     initial signup — gap-closing addition.
   - `/auth/change-password` not revoking other sessions — Phase 4.5.
7. Re-open the "blocking" open items check in `docs/design/user-management.md` before relying on
   it further — they were resolved as of the last design session, but double-check nothing new
   surfaced.
8. **k8s deploy infra — deliberately deferred, not started.** Only the code-level readiness audit
   is done (see its HANDOFF entry above). Still needed once cluster/registry details are known:
   Dockerfile (backend + frontend), k8s manifests (Deployment/Service/Ingress/CronJob for
   auto_lock/a migration Job for `alembic upgrade head`), CI/CD to build+push images, and a
   decision on how the frontend is served (its own container behind Ingress vs. CDN/object
   storage outside the cluster — Sơn hasn't decided yet). `backend/docker-compose.yml` is local
   dev only and was never meant to be a production deploy artifact.

## Safety (never relaxed — this is a Serious product, not a spike)
- Secrets via env vars only, `.env` never committed, pre-commit hook stays on.
- No hard-deleting `users` rows — ever (breaks `audit_log`'s referential trail).
- Every module besides User Management must also write to `audit_log` once built (decision #1's
  consequence) — don't let this get forgotten when AI Detection / Rule Engine modules land later.
