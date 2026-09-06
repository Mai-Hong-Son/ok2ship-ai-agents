# Template Management — design reference

English summary of the working design (developed in Vietnamese with the product owner, reconciled
against `SRS-OK2SHIP-AI.docx` **v2.4 (01/09/2026)**, the delivered mockup `template-management.html`
(**updated 02/09/2026**), and the internal SOP `VT-QQA-059A` — "Quy định nghiệp vụ kiểm tra báo cáo
Ok2ship SMT", Mektec Manufacturing Corporation (Vietnam) Ltd, rev A, 3/10/2025 — now historical
background only, no longer an active schema source, see decision #5). This is the first module of a
larger, not-yet-built configuration layer (Template / Data Mapping / Spec / Golden Sample / Save &
Publish / Report Upload) — see "Open items" for what's deliberately out of scope here. Source of
truth for any code touching this module; update it when the design changes, don't let it drift from
the code.

This module reuses the RBAC (`roles`/`permissions`/`user_roles`) and `audit_log` tables already
established in [`docs/design/user-management.md`](./user-management.md) — no parallel permission or
audit system.

> ⚠️ **Status (2026-09-06): NOT FINAL — a further BA round is pending.** Sơn is waiting on the BA
> to come back with updated requirements, and parts of this document are expected to change when
> they do. The "all blocking questions resolved" note below refers to the 2026-09-02 round only and
> is **no longer a statement that the design is settled**. A first implementation pass exists on
> `feature/template-management` (both repos) — committed so the work isn't stranded, NOT because it
> is ready. Do not treat this doc, or that branch, as done.

**Status of the 2026-09-02 round: all blocking questions from it resolved** — see "Open items —
Blocking" (empty as of that round; superseded by the pending BA round above).
Schema-level implementation (ORM models, migration, mock test-data seed script) is starting; the
API/business-logic layer (routers, Import-wizard flow) still waits on this doc being read alongside
the actual code as it's written, per the usual "don't let it drift" rule in the intro.

**2026-09-02 — major revision.** SRS v2.4 + an updated mockup landed the same day and reversed a core
decision from the previous revision of this doc: there is **no fixed "22 hạng mục" catalog**. See
decision #5 — this replaces the seeded-reference-table approach entirely, not an addition to it. If
you read an older copy of this doc (or code written against it), the `structure_items` table it
described **no longer exists** in this design.

## Decisions locked

| # | Question | Decision |
|---|---|---|
| 1 | How does a Template's revision history stay inspectable (FR-2.5: "every Rev's structure must remain viewable")? | **Each revision is an immutable snapshot row**, not an in-place mutation. `templates` holds the durable identity (code/name/customer); `template_revisions` holds one row per Rev, and its child rows (`template_revision_targets`, `template_revision_structure_items`) are scoped to that specific revision and never rewritten after creation. The reference mockup mutates a single row's `rev`/`structure` fields in place — that loses old Rev structure on overwrite and does not satisfy FR-2.5; not carried over into this schema. **Corollary, new 2026-09-02**: a revision's structure rows may only be edited (add/remove/rename item, toggle required — see decision #5) while that revision is still `Draft`; once `Active` or `Inactive` it is historical and locked, matching this decision's whole premise. |
| 2 | How is Rev numbered? | **Manually entered by the user** (`rev_label`, free text), not auto-incremented. Confirmed by BA (chat, 2026-08-31). The updated wizard (mockup 02/09) also requires two more fields at the same moment, now added to the schema — see decision #8. |
| 3 | How does a Template attach to a Model/Build/Config node? | **Three separate nullable FK columns** (`model_id`, `build_id`, `config_id`) on `template_revision_targets`, gated by a CHECK constraint that exactly one is set, rather than a polymorphic `(level, id)` pair. Gives the database real referential integrity (can't target a deleted node) at the cost of one extra column per row — worth it for a table on the hot resolution path. |
| 4 | How does the system pick the effective Template for a node? | **Most-specific-wins**: Config > Build > Model. Activating a new Active revision only auto-deactivates another Active revision targeting the *exact same node* — a parent and child revision may both be Active simultaneously, surfaced only as a non-blocking warning (matches SRS FR-2.2 and the mockup's `applyTemplateActivation`/`openStatusModal`, verified against source). |
| 5 | Where do "hạng mục" (structure items) come from? | **REVISED 2026-09-02 — reverses the previous answer.** There is **no shared/seeded catalog**. SRS v2.4 rewrote its own definition of "hạng mục": *"Số lượng và tên gọi hạng mục khác nhau giữa các Template — không có danh mục chuẩn chung cho toàn hệ thống."* Each Template's structure is entirely its own: on "Thêm Template" (the single Import-wizard entry point, decision #… see Q2 below), the system reads every sheet in the uploaded file and records **1 sheet = 1 structure item**, no comparison against any fixed list. After creation, QA may freely add, rename, or remove structure items on the current Draft revision, and toggle each one's `required` flag independently (mockup: `addStructureItem`/`removeStructureItem`/`renameStructureItem`/`toggleItemRequired`) — `required` is a per-item, per-revision property, never a global default. The previous revision of this doc chose a seeded `structure_items` reference table (sourced from VT-QQA-059A) specifically to resolve a contradiction between BA's chat answer and the 47-real-sheets-vs-22-SOP-items evidence; **SRS v2.4 resolves that same contradiction the other way** — BA's original answer was correct, drop the fixed catalog. VT-QQA-059A remains useful background reading, not a schema source. |
| 6 | Report/image files in the DB? | **Never** — object storage only, DB stores `storage_key`/URL only. **Provider confirmed (2026-09-02, Sơn)**: MinIO already running on the same Rancher cluster (`local`), namespace **`pre-prod`** — matches `ok2ship`'s own current environment (domain `ok2ship-dev.desoft.vn`). Bucket **`ok2ship-files`** created, with a dedicated Access Key (IAM policy restricted to that bucket only — cannot see the cluster's other buckets `lake-files`/`wrm`/`wrm-dev`/`wrm-uat`). Credentials stored in the existing `ok2ship-backend-env` k8s Secret (namespace `ok2ship`): `MINIO_ENDPOINT=minio.pre-prod.svc.cluster.local:9000`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`, `MINIO_BUCKET=ok2ship-files`, `MINIO_USE_SSL=false` (internal cluster traffic only, no TLS cert on that path — different from the public `https://` Ingress domains). **If `ok2ship` is ever promoted to a real production namespace, this MinIO instance must be revisited — `pre-prod` is not durable/backed-up storage.** |
| 7 | Permission/audit model for this module? | **Reuse existing tables**, no new system — see intro. New `permissions.code` values and `audit_log.action` values only (listed under "Non-negotiable technical notes"). |
| 8 | What identifies *why* a revision changed? | **New, 2026-09-02.** The updated wizard requires two more fields at Rev-save time, alongside `rev_label`: **ECO#** (Engineering Change Order number — Mektec's internal change-management reference that authorized this revision) and a **change description** (free text, "what/why changed"). Both mandatory in the mockup (`iw-eco`, `iw-desc`) — added to `template_revisions` as `eco_number`/`change_description`. UX note: the BA-side reviewer didn't recognize the ECO# field unprompted — flag for a tooltip/helper text when this screen is actually built (see "Open items"). |
| 9 | Q3 — parent/child Template creation. | **Resolved (2026-09-02, Sơn): always create a separate, brand-new Template** (own `templates` row + its own first `template_revisions` row, Rev A) scoped only to the child node — never extend/bump the parent's existing Template. The parent's Template is left completely untouched. This corrects the mockup's actual behavior (`confirmImportSave` bumps the parent's Rev in place, silently dropping the child's own scope) — that behavior is confirmed a bug, not a spec to copy. |
| 10 | Can two structure items in the same revision share a sheet name? | **No — blocked (2026-09-02, Sơn).** `UNIQUE(revision_id, sheet_name)` on `template_revision_structure_items`; saving a duplicate is rejected, not silently allowed. |
| 11 | What happens deleting a Model/Build/Config that a Template still targets? | **Blocked, not cascaded (2026-09-02, Sơn).** FKs from `template_revision_targets.model_id`/`build_id`/`config_id` to `models`/`builds`/`configs` are `ON DELETE RESTRICT` — deleting a node that any revision (Active or not) still targets fails outright; the target row must be removed/reassigned first. Chosen over `ON DELETE CASCADE` (the mockup's own behavior) specifically because silently losing a Template's scope on an unrelated node deletion is the more dangerous failure mode. |
| 12 | Concrete implementation of decision #6 (object storage). | **Wired 2026-09-02.** `template_revisions.storage_key` (nullable `String`) holds the object key; `app/core/object_storage.py` wraps a boto3 S3-compatible client (path-style addressing + `s3v4` signing, required for MinIO) with `upload_file(key, data, content_type)`. `import_template()` uploads the wizard's sample file right after the revision row is flushed (so the revision's own id is available for the key), **before** creating its `template_revision_targets`/`structure_items` rows — key shape is `templates/{template_id}/{revision_id}/{original_file_name}`. `RevisionResponse` exposes only a derived `has_sample_file: bool` (`storage_key is not None`) — the raw key is an internal path shape, never sent to the frontend; no download endpoint exists yet (deliberately out of scope, see "Open items"). `duplicate_revision()` does **not** copy `storage_key` — a duplicate has no sample file of its own until its own Import (consistent with it also not copying `targets`, see the Q&A this decision followed from). Local dev: `docker-compose.yml` runs a `minio` service (mirrors the existing `mailpit` local-SMTP pattern) so local testing never touches the real cluster bucket; `Settings` defaults point at it, real cluster credentials live only in the (gitignored) `.env`/k8s Secret from decision #6. |
| 13 | Template `code`/`name`/`customer` naming rule (Sơn asked; see the now-resolved Open item this replaces). | **Resolved 2026-09-02 — mockup update.** QA-entered at Import time via 3 new wizard fields (Mã tài liệu/Khách hàng/Tên báo cáo), not system-derived — the earlier `f"{node_name} OK2SHIP TEMPLATE"`/auto-incrementing code/`model.customer` placeholders are gone. **Locked+prefilled** when the exact selected node already has its own Template (any status — decision #9's same "own" check, exposed as a new `GET /templates/own`), **blank+required** when creating a brand-new one. The backend never trusts the lock: `import_template()` only reads the submitted `code`/`name`/`customer` in the brand-new-Template branch; bumping a Rev on an existing Template ignores them outright, even if a tampered request sends different values. Deliberately built against `own_template_for_node`, **not** the inherited-aware `resolve_effective_revision` — reusing that one here would wrongly lock+prefill (and then collide on) a *parent's* code when the selected node only inherits, the same class of mockup bug decision #9 already corrected once. `code` is immutable after creation (mirrors `users.username`); `name`/`customer` are editable later via `PATCH /templates/{id}` (drawer's "Sửa" general-info tab) — **not** gated on the current revision being Draft, since they're Template-level identity (the `templates` table, shared by every Rev), not Rev-level structure (decision #1's corollary only gates the structure tab). `customer` is constrained to a fixed frontend list (`Apple`/`Samsung`/`Google`/`Khác`, matching the mockup's `<select>`) — the DB column itself stays a plain string, no enum. |

### Terminology translation (Vietnamese business term → schema name)
| Vietnamese | Schema name |
|---|---|
| hạng mục (1 structure item, own to a single Template revision — **not** a shared catalog entry, see decision #5) | `template_revision_structure_items` row |
| Rev (phiên bản của 1 Template) | `template_revisions` |
| ECO# (Lệnh thay đổi kỹ thuật) | `template_revisions.eco_number` |
| cấp áp dụng (Model/Build/Config) | `level` + one of `model_id`/`build_id`/`config_id` on `template_revision_targets` |
| Bật/Tắt (Active/Inactive toggle) | `template_revisions.status` |

## Schema

### `projects` / `models` / `builds` / `configs`
Shared hierarchy, defined here first since Template Management is the first module that needs it —
Spec, Golden Sample and Report Upload (future modules) will reuse these same four tables, not
redefine their own. **Creation/edit UI for these 4 tables is a separate module (SRS §3.1, FR-1.1–
FR-1.6, "Quản lý Project/Model/Build/Config") — no mockup delivered for it yet, see "Open items."**

| Table | Field | Type | Notes |
|---|---|---|---|
| `projects` | `id` | uuid | PK |
| | `code` | text, UNIQUE | |
| | `name` | text | |
| `models` | `id` | uuid | PK |
| | `project_id` | uuid | FK → `projects.id` |
| | `customer` | text | one Model = exactly one customer (SRS) |
| | `name` | text | |
| `builds` | `id` | uuid | PK |
| | `model_id` | uuid | FK → `models.id` |
| | `name` | text | |
| | `sort_order` | int | stored, not parsed from `name` at query time |
| `configs` | `id` | uuid | PK |
| | `build_id` | uuid | FK → `builds.id` |
| | `name` | text | |

Only Model/Build/Config are valid "node" levels for targeting — Project is a grouping concept only,
never a target (confirmed against the SRS's own reference data model, `targets[].level` is
`'model'|'build'|'config'`).

### `templates`
| Field | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `code` | text, UNIQUE | |
| `name` | text | |
| `customer` | text | |
| `created_at` | timestamptz | |

### `template_revisions`
| Field | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `template_id` | uuid | FK → `templates.id` |
| `rev_label` | text | manually entered (decision #2) |
| `eco_number` | text | **new 2026-09-02** — mandatory at save time (decision #8) |
| `change_description` | text | **new 2026-09-02** — mandatory at save time, free text (decision #8) |
| `status` | enum: Draft / Active / Inactive | Active only via Save & Publish (future module), never on creation/import |
| `published_at` | timestamptz, nullable | |
| `created_by` | uuid | FK → `users.id` |
| `created_at` | timestamptz | |

### `template_revision_targets`
| Field | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `revision_id` | uuid | FK → `template_revisions.id` |
| `level` | enum: model / build / config | |
| `model_id` / `build_id` / `config_id` | uuid, nullable, `ON DELETE RESTRICT` | exactly one set, matching `level` — CHECK constraint (decision #3); deleting a targeted node is blocked, never cascaded (decision #11) |

### `template_revision_structure_items`
**Revised 2026-09-02** — no more link to a shared catalog (decision #5 no longer has one). Every row
is entirely local to its revision.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `revision_id` | uuid | FK → `template_revisions.id` |
| `sheet_name` | text, NOT NULL | actual sheet name as found on Import, or entered by QA when manually added afterward |
| `required` | boolean, NOT NULL, default true | per-item, per-revision — toggled independently by QA (decision #5); no global default anywhere |

Dropped from the previous revision of this doc: `structure_item_id` (no catalog left to point to) and
`mapped` (the new mockup has no such concept — every row that exists *is* the structure, there's
nothing to "map" against). `UNIQUE(revision_id, sheet_name)` — confirmed (decision #10): two rows
naming the same sheet within one revision are rejected, not allowed.

## Non-negotiable technical notes

- **Resolution query is the hottest path in this module** — it runs on every report upload (future
  module) and every screen that shows "which Template applies here." Required indexes:
  `template_revision_targets(model_id)`, `(build_id)`, `(config_id)`, each combined with
  `template_revisions.status = 'Active'`. Not an optimization to add later — build it in from the
  first migration.
- **New `permissions.code` values**: `template.create`, `template.publish`, `template.activate`,
  `template.deactivate` (`template.create` covers the single Import-wizard entry point — there is no
  separate manual-create permission, since that flow doesn't exist, see Q2 below). **New
  `audit_log.action` values**: `template.activate`, `template.deactivate`, `template.publish`,
  `template_revision.create`. Every status change and every new Rev must write to `audit_log`
  (NFR-4), same discipline as User Management. `audit_log.before_value`/`after_value` capture only
  the fields that changed, never a full record dump (matches the existing rule in
  `docs/design/user-management.md`).
- **Report/image files never touch the DB** — object storage only, `storage_key` stores the
  reference. MinIO endpoint/bucket/credentials confirmed and already provisioned — see decision #6.
- **`reports`/`check_results` (future Report Upload module) will grow unbounded** — plan monthly/
  quarterly partitioning from the first migration, same lesson already noted for `audit_log` in the
  User Management design.

## Open items

### Blocking — none left as of 2026-09-02

All 4 blocking items are now resolved by Sơn/BA — none left pending implementation of the schema
itself:
~~**Q1 — 22 hạng mục source**~~ — **resolved 2026-09-02**, see decision #5 (SRS v2.4 + updated
mockup independently reverse the previous answer — treat this as BA's answer, no further re-ask
needed).
~~**Q2 — "Thêm Template" (manual create) button**~~ — **resolved 2026-09-02.** SRS v2.4, FR-2.4:
*"Thêm Template — duy nhất 1 điểm vào, luôn qua wizard 2 bước đọc file mẫu (không có luồng nhập tay
riêng biệt)."* The updated mockup deletes the manual-create modal (`openCreateModal`/`submitCreate`)
entirely — confirmed leftover code, correctly dropped on both sides.
~~**Q3 — parent/child Template creation**~~ — **resolved 2026-09-02**, see decision #9 (Sơn: always
a separate new Template, never extend the parent's — corrects a mockup bug, don't copy it).
~~**Storage infra**~~ — **resolved 2026-09-02**, see decision #6.

### Not blocking, revisit when they surface
- **Project/Model/Build/Config creation UI missing** (new 2026-09-02): the updated mockup's scope
  picker for "Thêm Template" is now 4 cascading read-only dropdowns (Project → Model → Build →
  Config) — the old mockup's inline "create a new Model right here" affordance is gone, and no
  mockup exists yet for SRS §3.1's own management screen (FR-1.1–FR-1.6). Not blocking Template
  Management's own schema (these 4 tables don't change either way), but worth asking BA whether that
  screen is coming separately or whether inline-create needs to return to this wizard.
- **ECO# needs a tooltip**: BA-side reviewer didn't recognize the field unprompted when looking at
  the mockup — add a short helper text next to the input when this screen is actually built (cosmetic
  UX note, no schema impact).
~~**Template `name`/`code` on Import — no confirmed naming rule**~~ — **resolved 2026-09-02**, see
decision #13 (mockup update landed with actual QA-entry fields, superseding both the SRS's silence
and the mockup's earlier filename/counter placeholder logic).
- **No download endpoint for the uploaded sample file yet** (new 2026-09-02, see decision #12): the
  file lands in MinIO and `storage_key` is recorded, but nothing in the API lets QA fetch it back —
  the drawer only shows a `has_sample_file` boolean. Add a `GET .../revisions/{id}/sample-file`
  endpoint (resolving `storage_key` server-side, e.g. a pre-signed URL) once there's an actual UI
  need to view/re-download the original.
- Data Mapping, Spec, Golden Sample, Save & Publish, Report Upload schemas are deliberately **not**
  covered by this document yet — each gets its own design doc (or an addition to this one) once its
  module is taken up; a rough sketch already exists from the SRS walkthrough but isn't locked here.
- Mockup files for Data Mapping, Spec, Golden Sample, and Save & Publish are missing from the
  delivered mockup folder (only `template-management.html` and `upload-report.html` were provided) —
  requested from BA (2026-08-31), not yet received.
- NFR-2 in the SRS specifies Ant Design v5 + AG Grid Community for the mockup's stack — mismatched
  against the already-built `ok2ship-ai` frontend (React + Vite + Tailwind, no AntD). Unresolved
  whether this is the same frontend app or a separate one.
