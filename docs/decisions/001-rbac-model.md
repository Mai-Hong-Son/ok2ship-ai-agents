# ADR 001 — Full RBAC (roles + permissions), not a flat role column

Date: 2026-08-23 (retroactive — the decision itself was made 2026-08-04 → 2026-08-19 across 3
design rounds; written up now per hub Constitution #6, after the fact) | Status: accepted

## Context
WBS #5 ("User Management & Permission Assignment") needed a permission model. The vendor's first
official SRS (`SRS_OK2SHIP_User_Management.docx`, Author: Hue Duong) specified exactly two roles,
Admin/QA, with no separate approval step (the uploader also reviews the AI result) and no mention
of a user needing more than one role at once. An initial design (v1) proposed a full
roles/permissions/role_permissions/user_roles model anyway, anticipating future need; it was then
simplified to a flat `role` enum column on `users` (v2) to match the vendor's SRS exactly and
avoid speculative complexity. The vendor later asked, in practice, for real multi-role support
("tham khảo theo RBAC") — one person needing more than one role at once — which a flat
single-value column cannot represent without a schema change.

## Decision
Adopt the standard RBAC core model: `permissions` (atomic capability, e.g. `report.view`), `roles`
(a named bundle of permissions), `role_permissions` and `user_roles` as many-to-many junction
tables. A user can hold multiple roles. Naming follows industry-standard RBAC terms, not the
vendor's own inverted wording (their "role" = this schema's `permissions`, their "role group" =
this schema's `roles`) — recorded as a translation table in `docs/design/user-management.md` so
nobody talks past each other when discussing the schema with Desoft/BA.

Full schema, every field, and the other locked sub-decisions from the same design doc (report
visibility now role-based instead of Line-scoped, `department` stays display-only, a `Locked`
account is reactivated via the existing Active/Inactive toggle rather than a new endpoint, refresh
tokens live in an httpOnly cookie, minimum password policy) are recorded in
`docs/design/user-management.md` — that file is the schema source of truth; this ADR is the
architectural summary of the headline decision, not a duplicate of the detail.

## Rationale
- The flat-column version (v2) was the right call for what was known at the time — YAGNI, matched
  the vendor's spec exactly, cheapest to build. Building it first was not a mistake.
- Reverting from a flat column to full RBAC was comparatively cheap here because it happened
  before any backend code existed yet (still at the design-doc stage) — the "expensive to change
  later" risk that would have justified overbuilding RBAC speculatively from day one didn't apply,
  and didn't materialize into wasted implementation work.
- Role-based (not Line-scoped) report visibility keeps the schema at 8 tables instead of 10
  (dropping `lines`/`user_line_scope`); if Line-level restriction is needed again later, modeling
  it as more roles (e.g. `line4_qa`) costs new rows, not new tables — cheap up to roughly 4–5
  Line-specific roles, a threshold recorded as an open item in the design doc to revisit past that.

## Consequences
- No code may reference a `users.role` column — it doesn't exist. Any code touching permissions
  must resolve `user_roles` → `roles` → `role_permissions` → `permissions` through the single
  `can(user, permission_code)` choke point (`backend/app/core/permissions.py`), never a direct
  role-string comparison.
- Adding a role going forward is just inserting rows (`roles`, `role_permissions`, `user_roles`) —
  no migration required — which is the main practical payoff of this decision.
- The vendor's own terminology (their "role" / "role group") is inverted relative to this schema's
  (`permissions` / `roles`) — every conversation with BA/Desoft about this schema needs the
  translation table in `docs/design/user-management.md` to avoid talking past each other.

Related: `docs/design/user-management.md` (full schema + decision-by-decision detail),
`docs/PROGRESS.md` (chronological build log), hub `CLAUDE.md` §6 (ADR requirement).
