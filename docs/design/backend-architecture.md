# Backend architecture — design reference

Folder/module architecture for `backend/`, locked 2026-08-19 (Phase 0.5, before Phase 1 code
landed). Read this before adding a new file to `backend/app/` or a new WBS module — it explains
*where new code goes and why*, not what the User Management schema is (that's
`docs/design/user-management.md`).

## Decision: domain-based, not layered

Two common ways to organize a FastAPI backend:
- **Layered** — one `models/`, one `schemas/`, one `crud/`/`routers/` folder, every module's files
  mixed together inside them.
- **Domain-based** (chosen here) — one package per module/domain, each holding its own models,
  schemas, service, router.

This product is built **module by module off the vendor's WBS**: User Management now, then AI
Detection, SPC/Cpk Validation, Rule Engine, Alert/Notification, Dashboards — each a mostly
independent domain with little shared code. Domain-based packaging keeps each module's code
together and makes it easy to see (and eventually extract) one module without wading through files
shared with unrelated ones. The tradeoff — more folders than strictly needed while only one module
exists — is accepted up front since this is a Serious product built for the long run, not a spike.

## Directory tree

```
backend/
├── app/
│   ├── main.py                  # FastAPI app instance; include_router() for every module lives here
│   ├── config.py                 # pydantic-settings — all config from env vars / .env
│   ├── db.py                     # SQLAlchemy engine, session factory, declarative Base
│   │
│   ├── core/                     # Cross-cutting infrastructure — no business data of its own
│   │   ├── security.py           # password hashing (argon2id), JWT encode/decode
│   │   ├── permissions.py        # can(user, permission_code) — THE single permission choke point
│   │   ├── deps.py               # FastAPI Depends: get_current_user, require_permission(...)
│   │   └── exceptions.py         # shared exception types + handlers (401/403/404 etc.)
│   │
│   ├── modules/                  # One package per WBS module
│   │   ├── users/                 # WBS #5 — User Management & Permission Assignment
│   │   │   ├── models.py          # User, Role, Permission, RolePermission, UserRole (ORM)
│   │   │   ├── schemas.py         # Pydantic request/response models
│   │   │   ├── service.py         # business logic (create user, lock/unlock, change email, ...)
│   │   │   └── router.py          # /users, /roles endpoints
│   │   ├── auth/                  # login, token refresh, logout
│   │   │   ├── schemas.py
│   │   │   ├── service.py
│   │   │   └── router.py
│   │   └── audit/                 # audit_log — shared sink, every module writes through this
│   │       ├── models.py
│   │       └── service.py
│   │   # future, added as each WBS module is confirmed:
│   │   # ai_detection/, spc/, rule_engine/, alerts/, dashboards/
│   │
│   ├── jobs/                      # Scheduled/background jobs
│   │   └── auto_lock.py            # Phase 2 — daily job: lock accounts on failed logins / 90d idle
│   │
│   └── cli/                       # One-off management commands, run by hand or in deploy scripts
│       └── bootstrap_admin.py      # Phase 1 — idempotent seed of the first Admin account
│
├── alembic/                       # DB migrations (see alembic/env.py — URL comes from app.config,
│   └── versions/                   # target_metadata from app.db.Base, not hardcoded in alembic.ini)
│
├── docker-compose.yml              # Local Postgres only — not a production deploy artifact
│
└── tests/
    ├── conftest.py                 # shared fixtures (test DB session, test client) — added Phase 1
    ├── test_health.py
    └── modules/                    # mirrors app/modules/ — one subfolder per module
        ├── users/
        └── auth/
```

## Rules for adding a new module later

When a new WBS module (AI Detection, SPC, Rule Engine, Alert, Dashboard) starts:
1. Create `app/modules/<name>/` with the same four files as `users/`
   (`models.py`, `schemas.py`, `service.py`, `router.py`) — skip a file if the module genuinely has
   none of that concern yet, don't pre-create empty ones.
2. Mirror it under `tests/modules/<name>/`.
3. Register its router in `app/main.py`.
4. If it writes audit-worthy changes, call `app.modules.audit.service` — do not build a second
   audit mechanism.
5. If it needs a permission check, call `app.core.permissions.can(user, permission_code)` — do not
   query `user_roles`/`role_permissions` directly from module code.

## Other decisions made alongside this architecture

| Decision | Reasoning |
|---|---|
| No separate `repository.py` layer | `service.py` talks to the SQLAlchemy session directly. Adding a repository abstraction now is premature for this app's scale (internal QA admin tool, not high-traffic) — YAGNI. Revisit only if a concrete need (e.g. swapping persistence, heavy query reuse) shows up. |
| SQLAlchemy stays **sync**, not async | This is a low-concurrency internal dashboard, not a high-throughput API. Sync is simpler to write and debug. Revisit only if a real concurrency bottleneck is measured. |
| Local Postgres via `docker-compose.yml`, host port **5433** | Port 5432 was already bound by another project's Postgres container on the dev machine. `.env`, `.env.example`, and `app/config.py`'s fallback default all use 5433 to match. Not a production deployment concern — the compose file is dev-only. |

## Postgres enum gotchas — read before touching any `sa.Enum` column or migration

Two real bugs have already been hit and fixed here (Phase 1, Phase 4.5) — both were caught in dev
because a broken migration or a failing insert is loud and immediate, not silent. **Neither
requires wiping a database to fix — that was only how dev handled it because dev data is
disposable. A production fix never wipes the database; see "if this happens in production" below.**

### Gotcha 1 — SQLAlchemy stores the Python `Enum` member's **name**, not its `.value`

```python
class VerificationPurpose(enum.Enum):
    PASSWORD_RESET = "password_reset"   # NAME = "PASSWORD_RESET", VALUE = "password_reset"
```

`sa.Enum(VerificationPurpose)` (or the autogenerated `sa.Enum('PASSWORD_RESET', ...)` you see in a
migration) makes Postgres store `'PASSWORD_RESET'` — the **name** — not `'password_reset'`. Every
enum in this codebase uses a value that differs in case from its name specifically so this is easy
to get backwards (`Department.ADMIN = "Admin"`, `UserStatus.CREATE = "Create"`,
`VerificationPurpose.PASSWORD_RESET = "password_reset"`).

**This never matters for ORM code** (`user.status = UserStatus.ACTIVE`, `.filter(User.status ==
UserStatus.ACTIVE)`) — SQLAlchemy translates consistently in both directions automatically. It
only bites when a migration hand-writes a raw literal, which is unavoidable for one specific
operation: adding a new value to an *existing* Postgres enum type (`ALTER TYPE ... ADD VALUE`) —
alembic's autogenerate does not do this for you; you always write it by hand.

**Checklist for that one line, every time:**
1. Write the literal as the Python Enum member's **name** (`'PASSWORD_RESET'`), never its `.value`
   (`'password_reset'`) — check the existing values already in the type (`\dT+ <type_name>` in
   psql) to confirm the casing convention if unsure.
2. Test it: run the migration, then actually insert a row using the new value through the ORM
   (not just run the migration and assume it worked) — the Phase 4.5 bug was a `DataError` at
   *insert* time, not at migration time. A green migration does not mean the value is right.

### Gotcha 2 — Postgres enum types are separate objects, not owned by the table (Phase 1)

`op.drop_table()` does not drop the enum type(s) its columns used — autogenerate's `downgrade()`
never adds this. Skipping it leaves an orphaned type, and a *later* `upgrade` recreating the table
fails with "type already exists". Fix: explicitly `sa.Enum(name='...').drop(bind,
checkfirst=True)` in `downgrade()`, after dropping every table that referenced it. See
`alembic/versions/2df148b1f8be_create_user_management_tables.py`'s `downgrade()` for the pattern
to copy.

### If either of these happens in production — no database wipe, ever

**Gotcha 1 in production** (wrong-case value added, inserts using it fail): ship a *new* migration
adding the correctly-cased value — `ADD VALUE` is fast (near-instant, metadata-only) and doesn't
lock the table for reads/writes of existing rows. The wrongly-cased value can simply be left in
place forever, unused and harmless (Postgres has no `DROP VALUE`, so removing it cosmetically
requires a full type rebuild — rename the old type, create a correct new one, `ALTER TABLE ...
ALTER COLUMN ... TYPE new_enum USING old_column::text::new_enum` for every column that used it,
drop the old type — worth doing only if the clutter itself is a real problem, never required for
correctness).

**Gotcha 2 in production**: don't downgrade the affected migration in the first place if it would
touch tables with real rows — fix forward with a new migration instead. Downgrading a schema
change in production is rarely the right move regardless of this specific bug.

## Status

Skeleton created (empty `__init__.py` per package) as of 2026-08-19. No module has real code yet —
Phase 1 is the first to fill in `app/modules/users/`, `app/modules/audit/`, and `app/cli/bootstrap_admin.py`.
