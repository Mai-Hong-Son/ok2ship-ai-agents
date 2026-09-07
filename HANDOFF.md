# HANDOFF — ok2ship-ai (session handoff)

> Read this first when reopening this product. Update it after each session.
> Working rules + stack: see `CLAUDE.md`. Schema + all locked decisions: see
> `docs/design/user-management.md` (User Management, done) and
> `docs/design/template-management.md` (Template Management, built but NOT final — a BA round is
> still pending, see below) — each is the source of truth for its own module; code must match it,
> not the other way round.
> Backend folder/module architecture: see `docs/design/backend-architecture.md`.
> Full phase-by-phase build history (every bug found, every decision's rationale — this file only
> keeps a short summary + what's actionable right now): see `docs/PROGRESS.md`.

## Repo topology (read this before touching git — it's not one repo)
**Moved 2026-09-07: this product now lives at `~/Documents/ok2ship-ai/`**, no longer under
`ai-company/products/`. The `ai-company` hub was dissolved the same day — its constitution was
absorbed into the `simon-brain` wiki (`~/Documents/simon-brain/`), which is also where the shared
skills now live. Any older path in these docs that starts `products/ok2ship-ai/` means
`~/Documents/ok2ship-ai/`.

⚠️ **`CLAUDE.md` references simon-brain in prose, but does NOT `@import` it** — so a session gets
the product's own CLAUDE.md and nothing from the wiki unless it goes and reads the files itself
(`simon-brain/wiki/concepts/engineering-rules.md`, `wiki/projects/ok2ship-ai.md`,
`simon-brain/AGENTS.md`). Only an `@path` line auto-loads; a sentence naming the file does not.
This bit a session on 2026-09-07, which worked the whole day from the *old* constitution without
realising the hub had been restructured. Either add the `@import` lines or expect every session to
read them by hand.

As of 2026-08-23, this product is **three separate git repositories**, not one:
1. **`~/Documents/ok2ship-ai/`** (this repo, the one `HANDOFF.md`/`docs/` live in) — planning,
   design docs, session handoff only; `.gitignore` here explicitly excludes `backend/` and
   `frontend/` in full (not just their build artifacts), so this repo never tracks their content
   or creates a submodule-style gitlink for them. Pushed to
   `git@github.com:Mai-Hong-Son/ok2ship-ai-agents.git`, branch `main`.
2. **`backend/`** — its own repo, pushed to `git@gitlab.com:mektec/ok2ship-ai-backend.git`,
   branch `main`.
3. **`frontend/`** — its own repo, pushed to `git@gitlab.com:mektec/ok2ship-ai-frontend.git`,
   branch `main`.

`backend/`/`frontend/` each have their own pre-commit hook. Frontend's is tracked in-repo
(`frontend/scripts/git-hooks/pre-commit` — see `frontend/README.md`'s "Git hooks" section to
install it into a fresh clone); backend's currently only exists locally (not yet tracked the same
way — worth doing, same pattern, not done yet).

**Sơn's chosen workflow**: keep developing at these same local paths
(`~/Documents/ok2ship-ai/{backend,frontend}` are the working copies for those two repos;
`~/Documents/ok2ship-ai/` itself is the working copy for the GitHub one) — do not clone any of the
three to a new location. Push again only when asked; nothing auto-syncs between them.

`HANDOFF.md`/`docs/PROGRESS.md`/`docs/design/*.md` live ONLY in the parent repo — they are not
copied into either GitLab repo. A session working from a fresh clone of just `backend/` or
`frontend/` won't have them.

## What this product is
Backend + web dashboard for **OK2SHIP AI** (Mektec Vietnam, delivered alongside vendor
Desoft). 🚀 Serious product (not a spike) — tests mandatory, branch-per-feature (being adopted,
see "Next steps"), qa-reviewer before merge. Built module by module following the vendor's WBS.
**First module, User Management & Permission Assignment (WBS #5), is signed off complete by Sơn
(2026-08-29)** — see "Current state" below. **Second module chosen: Template Management** (part of
a larger configuration-layer scope from a separate SRS, `SRS-OK2SHIP-AI.docx` v2.3, 28/08/2026,
covering Template / Data Mapping / Spec / Golden Sample / Save & Publish / Report Upload — Template
Management is being tackled first). ⚠️ **NOT FINISHED — blocked on the BA** (Sơn, 2026-09-06),
**but now merged to `main` and live on the dev site** (2026-09-06/07). A first pass of backend +
frontend is written against `docs/design/template-management.md` (still the source of truth), but
the module is **not complete**: several behaviours are waiting on the BA to come back with updated
requirements, and the design doc will change when they do. Expect to revise it, not just review it.
Known gaps are in the design doc's "Open items"; the most concrete is editability of Mã tài liệu /
Phạm vi áp dụng / Mô tả after duplicating a Template.

⚠️ **Consequence of it being on `main`**: CI auto-deploys `main`, so the unfinished Template
Management UI is **visible to anyone using `ok2ship-dev.desoft.vn`**, including the BA. If it
shouldn't be seen yet, hide the sidebar entry (a permission condition in `Sidebar.tsx`) — nobody
has decided this either way yet.

Sibling spikes already proved feasibility for later modules — reuse, don't re-derive:
- `../_spikes/ok2ship-anomaly` — golden/one-class anomaly detection, future "image vs golden
  sample" module.
- `../_spikes/ok2ship-report-parser` — reading structured data out of real factory Excel reports,
  future data/spec-check modules.

## Current state
**User Management & Permission Assignment (WBS #5) — complete, signed off by Sơn (2026-08-29).**
Phases 0 through 5 (backend scaffold through frontend), many rounds of mockup-fidelity/UX fixes, a
retroactive qa-reviewer audit, and a performance fix (activation/reset emails moved off the
request path via BackgroundTasks — was adding a real 1-3s delay to every create/edit/resend/
forgot-password click) since. **Live in production** at `ok2ship-dev.desoft.vn` (Rancher cluster
`rancher-lake.desoft.vn`, namespace `ok2ship`), deployed via a GitLab CI/CD pipeline that
build+deploys automatically on every push to `main`, both repos (Le Bui, 2026-08-25/26).

**As of 2026-09-07 `main` also carries** (all merged, deployed, verified live): Template
Management (unfinished, see above), the ErrorBoundary fix, and structured JSON logging into the
cluster's Loki (ADR 004). Backend **256 tests**, frontend **126 tests**, both green; migrations
verified to apply cleanly onto a database that already holds data, not just an empty one.

A `/project-retro` ran the same day this module was signed off — see `docs/PROGRESS.md`'s Log for
the dated entry and whatever lessons were approved into the hub.

**Full history — every phase, every bug found, every design decision's rationale, dated — lives
in `docs/PROGRESS.md`'s Log, newest first.** This file only tracks what's still open; don't
duplicate finished work back into it.

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
9. "Delete" and the Active/Inactive toggle are the same endpoint (`PATCH /users/{id}/status`) — no
   separate delete route. `is_deleted` (added later) is an additive audit/reporting flag on top of
   `status=Inactive`, not a second security gate — see `docs/PROGRESS.md`'s 2026-08-23 entry.
10. Deleted users (`is_deleted=true`) are excluded from `GET /users` entirely, unconditionally, no
    query param to override it — decided 2026-08-27 specifically because it superseded an earlier
    "Đã xóa" filter option that this made permanently unmatchable. Still a soft delete under the
    hood (`get_user(id)`/`audit_log` unaffected) — only the list view hides them.

## Next steps (pick up here)
0. 🔴 **UNRESOLVED — login and logout feel slow on the dev site since the 2026-09-07 deploy**
   (Sơn). Not reproduced, and **not** traced to any of the new code. Measured and ruled out:
   backend API latency (`/health` 27–34 ms, failed login 38 ms, 12 samples against the live site),
   JS bundle (1.44 MB but gzipped, 0.26 s), memory (app uses 93 MB of the pod's 512 MB limit), pod
   count (1 replica, timings consistent), post-login route (unchanged, still `/overview`), local
   timings on the same code (login 47 ms, logout 8–11 ms), log volume (one login = one log line),
   and JS errors on the login page (none; page loads in 0.95 s).
   Two suspects remain, **both pre-existing rather than caused by the new code**:
   - `deploy/05-deployment.yaml` limits the pod to `cpu: 500m`, while argon2id is configured
     `parallelism=4` — the password hash is the only genuinely CPU-heavy step in the system, and
     only runs on a *successful* login (which is exactly what couldn't be measured from outside,
     for lack of a test account). Estimated ~273 ms per login from that alone. Fix if confirmed:
     raise `limits.cpu`, or lower argon2's `parallelism` — either is a one-line change.
   - Cold start right after a deploy (pod restart ≈ 0.9 s to first response, plus image pull).
   **To settle it**: open DevTools → Network, log in and out, and read the times for
   `auth/login` and `auth/logout`. Slow request → backend (almost certainly the CPU limit above).
   Fast request but a frozen UI → frontend, look at `AuthContext`. A throwaway test account on the
   dev site would let a session measure this directly instead of asking.
1. **Verify the Loki pipeline end to end — the last unverified step of ADR 004.** The app side is
   confirmed working (JSON on stdout, `X-Request-ID` present on live responses, 401/403/404/409/422
   all logged at the intended levels, passwords confirmed absent from log output). What nobody has
   checked yet is whether Grafana Alloy actually scrapes the `ok2ship` namespace: open Grafana →
   Explore → Loki and run `{namespace="ok2ship"}`. Nothing there while the pod clearly prints JSON
   means the gap is in Alloy's config, not in this codebase — that needs whoever runs the `logging`
   namespace, no code change.
2. ⚠️ **Template Management — UNFINISHED, waiting on the BA.** It is on `main` and deployed (see
   "What this product is"), so the remaining work is *revision*, not a merge decision. Known gaps
   live in `docs/design/template-management.md`'s "Open items"; the most concrete is editability of
   Mã tài liệu / Phạm vi áp dụng / Mô tả after duplicating a Template. Re-targeting scope is the
   risky one — decision #9 resolves Template identity most-specific-wins at Import time, so
   allowing a re-target without new guard logic can orphan or duplicate a Template.
   `audit_log` partitioning (see "Safety" below) was deferred by Sơn (2026-08-29) specifically
   until this module landed — it now has, so this is due.
   Object storage is **resolved (2026-09-02)** — MinIO on the existing `pre-prod` namespace,
   bucket `ok2ship-files`, credentials already in the `ok2ship-backend-env` k8s Secret; see
   `docs/design/template-management.md` decision #6 (including the caveat that `pre-prod` isn't
   durable/backed-up storage — revisit if `ok2ship` is ever promoted to real prod).
3. **Housekeeping left over from the 2026-09-06 branch split:**
   - `wip/backup-before-split` on **both** GitLab repos — a throwaway one-commit snapshot taken
     before splitting nine days of uncommitted work into branches. Everything is merged now, so
     these can be deleted.
   - Merged feature branches on both remotes (`feature/template-management`,
     `feature/error-logging-loki`, `fix/error-boundary-blank-page`, plus the older
     `feature/async-activation-emails`, `feature/email-drop-fallback-link`,
     `docs/drop-git-hooks-readme-section`) — all in `main`, safe to delete.
   - **GitHub PR trap worth remembering**: PRs #2 and #3 in this repo were opened against their
     *stacked parent* branches rather than `main`. Both show "MERGED", but their content never
     reached `main` — GitHub did not retarget them when #1 merged. Caught on 2026-09-07; the
     content is carried by the PR this handoff update belongs to. When stacking PRs, re-check the
     base branch after each merge instead of trusting auto-retarget.
5. **Branch/PR discipline: self-adopting, not GitLab-enforced.** Decision (2026-08-28, Sơn): not
   locking `main` on GitLab (Settings → Protected branches → "Allowed to push: No one") —
   self-adopting the `feature/<slug>`-branch + Merge Request habit first (agents included), revisit
   locking it for real once the habit holds through more than one module. 4 real MRs created this
   way so far (2026-08-28/29, see item 1) — no MR-level qa-reviewer pass has actually been posted
   as a comment yet (the proposed mechanism: a subagent carrying the `qa-reviewer.md` persona,
   same technique as the 2026-08-25 retroactive audit — literal `qa-reviewer` isn't a registered
   subagent type in this environment), just ask-before-merge in conversation — worth tightening if
   this becomes routine.
6. **k8s/CI follow-ups, still open** (the deploy infra itself is done and live — see "Current
   state"):
   - CI's `test` stage is disabled in both repos ("per explicit request to unblock build/deploy" —
     the runner has no Postgres available for `tests/test_health.py`). Fix properly: add a
     `services:` Postgres container to `.gitlab-ci.yml` (GitLab supports this natively) rather than
     leaving it off indefinitely.
   - Cluster Secret `ok2ship-backend-env` — confirm `FRONTEND_BASE_URL` and `SMTP_*` ended up
     patched in correctly (Sơn was doing this as of 2026-08-28/29, using the Gmail App Password
     already verified for local dev as an interim production SMTP account — revisit once a proper
     shared company mailbox is set up, not blocking). Verify with `kubectl -n ok2ship describe
     secret ok2ship-backend-env` (lists key names only, safe to run/share).
   - `backend`'s pre-commit hook still isn't tracked in-repo the way `frontend`'s now is (see
     `frontend/scripts/git-hooks/pre-commit`, merged 2026-08-28) — same fix, not yet done, low
     priority (Python doesn't have an equivalent "wrong flag silently no-ops" trap `tsc --noEmit`
     had, so the urgency that drove the frontend fix doesn't apply the same way here).
7. `backend/scripts/init_db.sh` — ~~commit-or-not~~ **dropped (2026-08-28, Sơn): not a priority,
   leave it as an uncommitted local convenience script, no further action.**
8. Locate the vendor's reference for the *forgot/reset/activate* screens' exact wording if one
   exists beyond `~/Downloads/app.html` (already reviewed and built against) — not blocking,
   just worth confirming nothing drifted from a newer mockup revision.
9. Get sign-off on values/decisions invented along the way, not in the original design doc:
   - `max_failed_login_attempts` (now 20, per Sơn — failed logins during testing shouldn't lock
     the account) and the exact lockout UX.
   - `/auth/activate` requiring a password reset even for an email-change token, not just an
     initial signup — gap-closing addition.
   - `/auth/change-password` not revoking other sessions.
   - `UserResponse.activation_email_sent` always being `None` now for create/update/resend
     (2026-08-29 BackgroundTasks change, see "Current state") — a real delivery failure is only
     discoverable via server logs or the user reporting "never got the email," not surfaced to the
     Admin in the moment anymore. Confirmed as an acceptable trade-off with Sơn at the time; worth
     re-confirming if this module gets busier.
10. Re-open the "blocking" open items check in `docs/design/user-management.md` before relying on
   it further — they were resolved as of the last design session, but double-check nothing new
   surfaced.

## Known traps (each one cost real time to find — read before debugging something similar)
- **`npx tsc --noEmit` type-checks NOTHING in `frontend/`.** The root `tsconfig.json` is
  `files: []` plus `references`, which plain `tsc` never traverses — only build mode does. Use
  **`npx tsc -b`**. Found 2026-09-06 after a whole session of reporting "tsc clean"; the
  pre-commit hook (which correctly runs `tsc -b`) then caught a real type error the no-op command
  had been silently passing over. The hook's own comment documents this — trust the hook, not the
  bare command.
- **`pytest` HANGS with no error message** if run without overriding the dev `.env`, which points
  SMTP at real Gmail and MinIO at the in-cluster `minio.pre-prod.svc.cluster.local` — neither
  reachable from a laptop. It doesn't fail, it just stops. Run:
  `SMTP_HOST= MINIO_ENDPOINT=localhost:9000 MINIO_ACCESS_KEY=ok2ship MINIO_SECRET_KEY=ok2ship123 uv run pytest -q`
  Worth fixing properly by pointing the dev `.env` at Mailpit + local MinIO instead. (Also note the
  suite currently reaches out to real Gmail on every forgot-password test unless overridden.)
- **Two concurrent `pytest` runs deadlock each other.** `conftest.py` drops and recreates the test
  database per session, so a second run blocks on the first. Symptom is the same silent hang —
  check for stray `pytest` processes before assuming a code problem.
- **Migration `fdc25656d371` (drop templates customer column) has a broken `downgrade()`** — it
  restores `customer` as `NOT NULL` without handling existing rows, so downgrading fails with
  `NotNullViolation`. Harmless today (CI only ever runs `upgrade`) but it blocks a rollback if one
  is ever needed. Not fixed.
- **Pydantic 422 errors embed the submitted value** under `input` — for `/auth/reset-password`
  that is the user's real password. Never log `exc.errors()` wholesale; `app/main.py`'s
  `_validation_failure_summary` keeps field name + error type only, and a test asserts the
  password never reaches the log output. Keep it that way.

## Safety (never relaxed — this is a Serious product, not a spike)
- Secrets via env vars only, `.env` never committed, pre-commit hook stays on.
- No hard-deleting `users` rows — ever (breaks `audit_log`'s referential trail).
- Every module besides User Management must also write to `audit_log` once built (decision #1's
  consequence) — don't let this get forgotten when AI Detection / Rule Engine modules land later.
- `audit_log` isn't partitioned by month yet — the design doc calls this non-negotiable long-term
  (the table grows unbounded), deliberately deferred at planning time given low data volume so
  far, and again by Sơn (2026-08-29) specifically until the Template Management module lands —
  don't let it slip past that.
