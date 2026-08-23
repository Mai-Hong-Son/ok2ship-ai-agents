# User Management & Permission — design reference

English summary of the working design (originally developed in Vietnamese with the product owner,
reconciled against the vendor's official SRS `SRS_OK2SHIP_User_Management.docx`, Author: Hue
Duong). This is the source of truth for any code touching this module — update it when the design
changes, don't let it drift from the code.

**v3** — BA asked for multi-role support ("reference RBAC") after using this in practice. Full
`roles`/`permissions` model is back (dropped in v2 as premature, now confirmed needed). Report
visibility moved from Line-based scoping to role-based — `lines`/`user_line_scope` removed.

## Decisions locked (do not re-litigate without a new discussion)

| # | Question | Decision |
|---|---|---|
| 1 | Is there a separate approval role, or does the uploader also review the AI result? | **No separate approver** (v2, still standing — NOT reversed by v3). The uploader is also the reviewer. Consequence: `audit_log` is a critical safety net — every module must write to it, not just User Management. Multi-role support (decision #2) makes adding a dedicated approver role possible later without a schema change, but none exists yet. |
| 2 | Can a user hold multiple roles? | **Yes** (v3, new). Full RBAC: `roles` + `permissions` + `role_permissions` + `user_roles` — replaces v2's flat 2-role column, which turned out to be too rigid once the vendor asked for real multi-role support. |
| 3 | Naming: follow the vendor's terms or industry-standard RBAC terms? | **Industry-standard**: `permissions` (atomic capability) + `roles` (named bundle of permissions). See the translation table below — the vendor's own wording maps onto this inverted, worth remembering when discussing with them. |
| 4 | How is report visibility scoped now that `user_line_scope` is gone? | **Role-based**: which reports a user can see follows from their role(s)/permissions, not a per-user Line assignment. If Line-level restriction is needed again later, model it as more roles (e.g. `Line4_QA`) — cheap to add (just new rows, no schema change) since the RBAC tables already exist. Revisit a dedicated scope table only if Line-specific roles exceed ~4–5 or Lines change often (see Open items). |
| 5 | What is `department` used for? | **Display/filter only, unchanged from v2.** Not wired into any permission logic. Stored as a plain inline enum on `users` — no separate `departments` reference table (considered, then dropped: not worth normalizing before it has an actual use beyond display). |
| 6 | Who/how does a `Locked` account get back to `Active`? | **Admin uses the same Active/Inactive toggle (UC-06)** on a `Locked` user to bring it back to `Active` — no separate "unlock" action/endpoint. |
| 7 | Refresh token storage on the client? | **httpOnly cookie** (not JS-readable storage) — safer for a long-lived, powerful credential. Requires correct CORS/CSRF setup. |
| 8 | Password complexity policy? | **Minimum 8 characters, must include letters and digits.** |

### Vendor-wording ↔ schema-naming translation (for talking to BA/Desoft, not for code)
| BA's Vietnamese term | What they described | Schema name used here |
|---|---|---|
| `role` | "chứa các tên quyền trong hệ thống" (holds the names of permissions) | `permissions` |
| `role group` | "định nghĩa nhóm quyền" (defines a permission group) | `roles` |
| field on `user` to "gán nhóm quyền" | assign a role group to a user | `user_roles` (junction — supports more than one, decision #2) |

## Schema

### `users`
| Field | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `username` | text, UNIQUE | Immutable after creation. Regex: `^[a-zA-Z][a-zA-Z0-9._-]{5,49}$` |
| `email` | text, UNIQUE | Editable by Admin; editing resets status to `Create` and re-sends activation |
| `password_hash` | text | argon2id |
| `full_name` | text | |
| `department` | enum: QA / SQE / NPI / OQC / IQC / LAB / Admin | Display-only (decision #5) — inline enum, no separate reference table (dropped — see decision #5) |
| `status` | enum: Create / Active / Locked / Inactive | See state table below |
| `is_deleted` | boolean, default false | Added later (2026-08-23) — see note below the state table |
| `failed_login_count` | int, default 0 | Drives auto-Lock |
| `last_login` | timestamp | Drives auto-Lock after 90 days idle |
| `created_at` / `updated_at` | timestamp | |

**No `role` column here anymore** (removed in v3) — role membership lives in `user_roles`.

Status transitions:
- `Create` → pending first-time password set via activation email.
- `Active` → normal.
- `Locked` → automatic: too many failed logins, OR `last_login` > 90 days ago (needs a daily job).
- `Inactive` → Admin-disabled (this is what "delete" means — soft delete only, `users.id` is
  referenced by `audit_log` and must never be hard-deleted).

**`is_deleted` (added 2026-08-23):** additive, not a replacement for the above — `Inactive` is
still the only *login-gating* state. "Xóa" (Delete) and "Vô hiệu hóa" (Deactivate) both set
`status=Inactive`; only Delete also sets `is_deleted=true`, so the two are now distinguishable in
data/audit even though the login-blocking effect is identical. `is_deleted=true` can only ever
coexist with `status=Inactive` — the backend rejects `status=Active` + `is_deleted=true` outright
and always clears `is_deleted` back to `false` on reactivation (see
`backend/app/modules/users/service.py::set_user_status`). Pre-2026-08-23 `Inactive` users
backfilled to `is_deleted=false` — there's no way to know retroactively which were originally
"deleted" in intent vs. merely deactivated.

### `permissions`
| Field | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `code` | text, UNIQUE | `resource.action` format, e.g. `user.manage`, `report.view`, `report.approve` |
| `name` | text | Display description |

### `roles`
| Field | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `code` | text, UNIQUE | Fixed, used in code, e.g. `admin`, `qa` |
| `name` | text | Display name, Admin can edit freely without touching code |

### `role_permissions` (junction, many-to-many)
`role_id` (FK `roles.id`), `permission_id` (FK `permissions.id`). Defines which permissions a role
bundles — this is the vendor's "role group defines a permission group."

### `user_roles` (junction, many-to-many)
`user_id` (FK `users.id`), `role_id` (FK `roles.id`). A user can hold more than one role
(decision #2) — e.g. both `qa` and a future `line4_qa`.

### `email_verification_tokens`
`id`, `user_id`, `token_hash` (SHA-256), `purpose` (`initial_activation` | `email_change`),
`expires_at`, `used_at`. Changing a user's email must invalidate any unused prior tokens for that
user and issue a fresh one (vendor SRS UC-04 BR-02).

### `refresh_tokens`
`id`, `user_id`, `token_hash` (SHA-256), `expires_at`, `revoked_at`, `replaced_by_token_id` (token
rotation — reuse of an already-rotated token signals theft; revoke the whole session on detection).

### `audit_log`
`id`, `user_id`, `action`, `target_type`, `target_id`, `before_value`/`after_value` (jsonb, changed
fields only — never a full record dump), `created_at`. Written to by every module, not just User
Management.

### Removed in v3 (superseded by role-based visibility, decision #4)
`lines`, `user_line_scope` — no longer part of this design.

## Non-negotiable technical notes

- **Permission checks go through one function**: `can(user, permission_code)`. It resolves a
  user's permissions via `user_roles` → `role_permissions` → `permissions` — keeping every call
  site routed through this one function means the resolution logic (today: a flat join; later:
  maybe cached) only needs to change in one place.
- **Avoid the JOIN fan-out trap** when resolving a user's full profile: don't flatten
  `users` × `user_roles` × `role_permissions` × `permissions` into a single query — a user with 2
  roles × 5 permissions each returns 10 duplicated rows. Run separate queries (or use `json_agg`
  in Postgres) and assemble the result in application code.
- **Cache resolved permissions** (embed in the JWT at login, or a fast Redis lookup) rather than
  re-joining 3 tables on every request — this hot path runs on every authenticated call.
- **Force logout must be near-instant** on Lock/Inactive/delete (vendor requirement). Long-lived
  JWTs alone cannot satisfy this — pick one explicitly: very short access-token TTL (~1–2 min), or
  a Redis-backed live status check per request.
- **Password hash: argon2id. Token hash: SHA-256.** Different algorithms on purpose — argon2id is
  deliberately slow (right for passwords), SHA-256 is fast (right for high-frequency token lookups).
- Required indexes: `users.username`, `users.email`, `users.last_login`,
  `user_roles.user_id`, `role_permissions.role_id`, `refresh_tokens.token_hash`,
  `refresh_tokens.user_id`, `audit_log.user_id`, `audit_log.created_at`,
  `audit_log(target_type, target_id)`.
- `audit_log` will grow unbounded — partition by month; apply the same Working/Historical ~1-year
  retention policy the vendor's WBS already defines for other data (don't treat it as an exception).

## Open items

### Not blocking, revisit when they surface
- Vendor's own open question (their SRS): is a company email domain enforced, or is any valid
  email accepted?
- Whether/when `department` gains an actual permission meaning (decision #5) — if it does, THAT is
  the point to normalize it into its own `departments` reference table (dropped for now) plus a
  `user_department_scope` table mirroring the old `user_line_scope` shape; don't build either
  speculatively now.
- Vendor SRS references a reference UI mockup (`user-management.html`, AG Grid Community) — not
  present anywhere in this repo yet. Source it before Phase 5 (frontend) starts.
- Role-based report visibility (decision #4) risks role-list bloat if Line-level restriction comes
  back as one role per Line. Cheap to add (rows only, no schema change) up to a handful of Lines;
  past ~4–5 Line-specific roles, or if Lines change often, revisit a dedicated scope table instead.
