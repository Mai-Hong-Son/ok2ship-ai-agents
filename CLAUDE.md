@~/Documents/ai-company/CLAUDE.md

> 👉 READ `HANDOFF.md` FIRST — current state, locked decisions, next steps (full phase-by-phase
> history lives in `docs/PROGRESS.md`). Schema source of truth: `docs/design/user-management.md`.

# ok2ship-ai — full-stack (OK2SHIP AI, Mektec Vietnam)

> This product inherits the whole Constitution (the @ line above). Everything below is
> product-SPECIFIC, and may only be STRICTER than the Constitution, never looser.

## What this product is
Backend + web dashboard for **OK2SHIP AI** (Mektec Vietnam, delivered with vendor Desoft —
see `docs/design/` for the reconciled requirements). The full system checks QA report data/images
against spec, golden samples, and cross-report history before a shipment is approved. Built
incrementally, module by module, following the vendor's WBS. **First module: User Management &
Permission Assignment** (WBS #5) — everything else (AI Detection, SPC/Cpk Validation, Rule Engine,
Alert/Notification, Dashboards) is future work, added to this same repo as it's confirmed.

Two upstream sibling spikes already proved feasibility for later modules — reuse their findings,
don't re-derive:
- `../_spikes/ok2ship-anomaly` — golden/one-class anomaly detection (Anomalib/PatchCore), for the
  future "check image vs golden sample" module.
- `../_spikes/ok2ship-report-parser` — reading structured data out of real factory Excel reports
  (label-keyed parsing, template drift findings), for the future data/spec-check modules.

## Stack (locked at project-init — matches Constitution default, no ADR needed)
- Backend: **Python 3.11+ / FastAPI / Pydantic v2**, **PostgreSQL**.
- Web: **React 18 + Vite + Tailwind** — no router/state library until genuinely needed.
- Tests: **pytest** (backend) / **vitest** (frontend) — mandatory for every feature (Serious product).
- Any deviation from this stack → an ADR in `docs/decisions/` BEFORE any code.

## Layout (full-stack → one subdir per tier)
```
backend/                    # FastAPI app (pyproject.toml at backend/ root)
frontend/                   # React + Vite app (package.json at frontend/ root)
docs/
  decisions/                 # product-local ADRs
  design/                    # design references carried in from planning (English — see below)
```
Code skeleton (`backend/pyproject.toml`, `frontend/package.json`, etc.) is built in the next
session, after this scaffold — not part of project-init itself.

## User Management & Permission — locked design (WBS #5, first module)
Full reasoning + comparison against the vendor's SRS lives in `docs/design/user-management.md`
(English summary of a longer Vietnamese working doc kept outside this repo). Key points any code
touching this module must respect:

- **Full RBAC: a user can hold multiple roles** (`roles` + `permissions` + `role_permissions` +
  `user_roles`, standard many-to-many). Supersedes an earlier, simpler "flat 2-role column"
  version — that one matched the vendor's first SRS draft but the vendor later asked for real
  multi-role support ("tham khảo theo RBAC"). No flat `role` column on `users` anymore.
- **Naming note**: the vendor's own Vietnamese wording is inverted vs. industry-standard RBAC terms
  (their "role" = our `permissions`, their "role group" = our `roles`) — see the translation table
  in `docs/design/user-management.md` before discussing schema with Desoft/BA, to avoid talking
  past each other.
- **Report visibility is role-based, not Line-scoped** — the earlier `lines`/`user_line_scope`
  tables are removed. If per-Line restriction is needed again, model it as more roles (cheap: new
  rows, no schema change) rather than reviving a scope table, unless Line-specific roles start
  exceeding ~4–5 (see `docs/design/user-management.md`, Open items).
- **All permission checks must go through a single `can(user, permission_code)` choke point** in
  backend code — it resolves `user_roles` → `role_permissions` → `permissions` in ONE place, so
  swapping the resolution strategy (e.g. adding a cache) never means hunting down scattered checks
  across the codebase. Watch for the JOIN fan-out trap when resolving a user's full role/permission
  set — see `docs/design/user-management.md` for the concrete pitfall and fix.
- **`users.username` is immutable** once created; `email` is editable by Admin (editing it forces
  the account back to `Create` status and re-sends an activation link — old link is invalidated).
- **4 account statuses**: `Create` (pending activation) → `Active` → `Locked` (auto, after failed
  logins or 90 days without login) → `Inactive` (Admin-disabled). No hard delete, ever — `users.id`
  is referenced by `audit_log`; hard-deleting would sever that trail.
- **Force logout on Lock/Inactive/delete must be near-instant** (vendor SRS requirement) — this
  conflicts with embedding permissions in a long-lived JWT for speed. Pick ONE deliberately when
  building auth: short-lived access tokens (~1–2 min), or a Redis-backed live status check per
  request. Don't silently default to plain long-lived JWTs — that fails this requirement.
- **`department` is informational only for now** (QA/SQE/NPI/OQC/IQC/LAB/Admin) — a plain inline
  enum on `users`, no separate reference table. If it ever gains real permission meaning, that's
  the point to normalize it into its own `departments` table (plus a `user_department_scope`
  table) — not before.
- **Password hashing: argon2id. Token hashing (refresh + email-verification tokens): SHA-256** —
  deliberately different algorithms (argon2id is intentionally slow for passwords; tokens need
  fast, frequent lookups).

## Product-specific rules (stricter than Constitution)
1. Real factory report data/images (future modules) are customer data — never leave the approved
   environment, never touch a free-tier AI service (Constitution #3). This applies even though the
   User Management module itself doesn't touch customer QA data.
2. Every `audit_log` write captures only the fields that actually changed (`before`/`after`), never
   a full record dump — avoids accidentally logging sensitive fields.
3. Float/threshold comparisons (once spec-check modules land) use a tolerance, never `==`
   (Constitution #4), with the epsilon and its reason stated at each comparison site.

## Notes
- This repo starts Serious (not a spike) — tests are mandatory from the first commit, branch per
  feature, qa-reviewer before merge, per the Constitution's Standard/Full flow.
- Add further product-specific rules below as new WBS modules land in this repo.
