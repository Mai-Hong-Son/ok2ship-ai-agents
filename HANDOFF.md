# HANDOFF — ok2ship-ai (session handoff)

> Read this first when reopening this product. Update it after each session.
> Working rules + stack: see `CLAUDE.md`. Schema + all locked decisions: see
> `docs/design/user-management.md` (User Management, done) and
> `docs/design/template-management.md` (Template Management, design in progress — not yet approved
> for implementation) — each is the source of truth for its own module; code must match it, not the
> other way round.
> Backend folder/module architecture: see `docs/design/backend-architecture.md`.
> Full phase-by-phase build history (every bug found, every decision's rationale — this file only
> keeps a short summary + what's actionable right now): see `docs/PROGRESS.md`.

## Repo topology (read this before touching git — it's not one repo)
As of 2026-08-23, this product is **three separate git repositories**, not one:
1. **`products/ok2ship-ai/`** (this repo, the one `HANDOFF.md`/`docs/` live in) — planning,
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
(`products/ok2ship-ai/{backend,frontend}` are the working copies for those two repos;
`products/ok2ship-ai/` itself is the working copy for the GitHub one) — do not clone any of the
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
Management is being tackled first). ⚠️ **NOT FINISHED — blocked on the BA, do not treat as
done** (Sơn, 2026-09-06). A large first pass is written and committed to
`feature/template-management` in both repos (against `docs/design/template-management.md`, still
the source of truth), but the module is **not complete**: several behaviours are still waiting on
the BA to come back with updated requirements, and the design doc itself will change when they do.
Code is on a branch so the work isn't sitting unbacked on one laptop — that is the ONLY reason it
was committed, not a sign it is ready. Expect to revise it, not just review it. Known gaps are in
the design doc's "Open items"; the most concrete one is editability of Mã tài liệu / Phạm vi áp
dụng / Mô tả after duplicating a Template.

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
build+deploys automatically on every push to `main`, both repos (Le Bui, 2026-08-25/26). Backend
185 tests, frontend 102 tests, both green.

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
0. ⚠️ **Template Management — UNFINISHED, waiting on the BA (Sơn, 2026-09-06). Do not merge
   branch set 1 yet.** The first pass is committed so it isn't stranded on one laptop, but the
   requirements are still moving: the BA owes updated answers, and the design doc will change with
   them. Treat the branch as work in progress to be revised, not a candidate for review-and-merge.
   Known gaps live in `docs/design/template-management.md`'s "Open items"; the most concrete is
   editability of Mã tài liệu / Phạm vi áp dụng / Mô tả after duplicating a Template. Re-targeting
   scope is the risky one — decision #9 resolves Template identity most-specific-wins at Import
   time, so allowing a re-target without new guard logic can orphan or duplicate a Template.

   **Three branch sets are open, stacked in this order** (each based on the previous, so merging
   out of order conflicts). Sets 2 and 3 are finished and independent of the BA — but they sit on
   top of set 1, so they cannot be merged before it:
   1. `feature/template-management` (backend + frontend) — ⚠️ UNFINISHED, see above.
   2. `fix/error-boundary-blank-page` (frontend only) — done. A render crash no longer blanks the
      whole page.
   3. `feature/error-logging-loki` (backend + frontend) — done. Structured JSON logs into the
      cluster's existing Loki; see ADR 004.
   If sets 2/3 are wanted in `main` before Template Management is ready, they'll need rebasing
   off `main` directly rather than waiting — that's a rebase, not a re-implementation, but it is
   not free and someone has to do it deliberately.

   Docs branches in this repo: `docs/template-management-design`, `docs/error-monitoring-adr`,
   `docs/handoff-update`.
   `wip/backup-before-split` in both code repos is a throwaway snapshot of all of it as one
   commit — delete once the above are merged.
   Object storage is **resolved (2026-09-02)** — MinIO on the existing `pre-prod` namespace,
   bucket `ok2ship-files`, credentials already in the `ok2ship-backend-env` k8s Secret; see
   `docs/design/template-management.md` decision #6 for the full detail (including the caveat that
   `pre-prod` isn't durable/backed-up storage — revisit if `ok2ship` is ever promoted to real prod).
   `audit_log` partitioning (see "Safety" below) was explicitly deferred until this module
   specifically (Sơn, 2026-08-29) — don't let it slip further now that it has landed.

   **Local test gotcha (cost an hour on 2026-09-06):** the dev `.env` points SMTP at real Gmail
   and MinIO at the in-cluster `minio.pre-prod.svc.cluster.local`, neither reachable/appropriate
   from a laptop — `pytest` then HANGS (no error, just stops). Run the suite with those overridden:
   `SMTP_HOST= MINIO_ENDPOINT=localhost:9000 MINIO_ACCESS_KEY=ok2ship MINIO_SECRET_KEY=ok2ship123 uv run pytest -q`
   Worth fixing properly by pointing the dev `.env` at Mailpit + local MinIO instead.
1. **3 open MRs, not yet merged — merge (or close) before treating User Management as fully
   settled:**
   - `backend`: `feature/async-activation-emails` — BackgroundTasks perf fix.
   - `backend`: `feature/email-drop-fallback-link` — dropped the redundant raw-link paragraph
     from the HTML email templates.
   - `frontend`: `docs/drop-git-hooks-readme-section` — README trim.
   (`frontend`'s `feature/precommit-tsc-typecheck` already merged, 2026-08-28.)
2. **Branch/PR discipline: self-adopting, not GitLab-enforced.** Decision (2026-08-28, Sơn): not
   locking `main` on GitLab (Settings → Protected branches → "Allowed to push: No one") —
   self-adopting the `feature/<slug>`-branch + Merge Request habit first (agents included), revisit
   locking it for real once the habit holds through more than one module. 4 real MRs created this
   way so far (2026-08-28/29, see item 1) — no MR-level qa-reviewer pass has actually been posted
   as a comment yet (the proposed mechanism: a subagent carrying the `qa-reviewer.md` persona,
   same technique as the 2026-08-25 retroactive audit — literal `qa-reviewer` isn't a registered
   subagent type in this environment), just ask-before-merge in conversation — worth tightening if
   this becomes routine.
3. **k8s/CI follow-ups, still open** (the deploy infra itself is done and live — see "Current
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
4. `backend/scripts/init_db.sh` — ~~commit-or-not~~ **dropped (2026-08-28, Sơn): not a priority,
   leave it as an uncommitted local convenience script, no further action.**
5. Locate the vendor's reference for the *forgot/reset/activate* screens' exact wording if one
   exists beyond `~/Downloads/app.html` (already reviewed and built against) — not blocking,
   just worth confirming nothing drifted from a newer mockup revision.
6. Get sign-off on values/decisions invented along the way, not in the original design doc:
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
7. Re-open the "blocking" open items check in `docs/design/user-management.md` before relying on
   it further — they were resolved as of the last design session, but double-check nothing new
   surfaced.

## Safety (never relaxed — this is a Serious product, not a spike)
- Secrets via env vars only, `.env` never committed, pre-commit hook stays on.
- No hard-deleting `users` rows — ever (breaks `audit_log`'s referential trail).
- Every module besides User Management must also write to `audit_log` once built (decision #1's
  consequence) — don't let this get forgotten when AI Detection / Rule Engine modules land later.
- `audit_log` isn't partitioned by month yet — the design doc calls this non-negotiable long-term
  (the table grows unbounded), deliberately deferred at planning time given low data volume so
  far, and again by Sơn (2026-08-29) specifically until the Template Management module lands —
  don't let it slip past that.
