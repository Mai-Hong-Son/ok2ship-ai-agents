# ADR 005 — Data Mapping storage: fixed columns for the frame, JSONB for per-type settings

Date: 2026-09-11 | Status: accepted

## Context
Data Mapping (WBS #5.3) configures, per sheet of a Template revision, which cells/images the
checker reads and what each is compared against. One configured check is a **field**; a field
has one or two **zones** (left = the data being checked, right = what it is compared to).

The shape of a zone depends entirely on its type. Captured from the mockup's own save payload
(2026-09-11, not read from its source), every zone carries 19 attributes, of which a given type
uses between one and seven:

| Zone type | Attributes it uses |
|---|---|
| `excel` | `entries[]` |
| `image` | `entries[]`, `ocrRows`, `backend` |
| `spec` | `specSrc`, `specOperator`, `specValue1`, `specValue2`, `specUnit`, `specSetId`, `specParamKey` |
| `golden` | `goldenSetId` |
| `formula` | `formulaSources[]`, `formulaType`, `formulaTolerance` |
| `table` | `tableKeyAnchor`, `tableValueCol`, `tableKeyFilter`, `tableMode` |

Fields have the same property one level up: the `unique` check type carries `mode`/`scope`, and
the BA checklist already asks for `trend` to carry a comparison scope (model/config/build/lot).

Three facts decided this:

1. **The shape is still moving.** Between the 2026-09-07 and 2026-09-09 mockup builds the BA
   added a zone type (`table`), added two image attributes (`ocrRows`, `backend`) and moved
   `dataType` from the zone up to the field — three structural changes in two days, with seven
   of eight checklist groups not yet surveyed.
2. **The attributes that would justify real columns have nothing to point at yet.** The strongest
   reason to normalize is a real foreign key for `specSetId`/`goldenSetId`. Neither the Spec
   Management nor the Golden Sample module exists, so there is no table to reference.
3. **The mockup persists stale values.** Setting a zone to Spec `> 3.7`, then switching it to an
   Excel cell, saves an Excel zone that still carries `specOperator: "gt"`, `specValue1: "3.7"`
   (measured 2026-09-11). Whatever we store must not inherit that.

JSONB is not new to this schema — `audit_log.before_value`/`after_value` already use it.

## Decision
1. **Fields hang off the revision's structure item**, not off the Template:
   `field_mappings.structure_item_id → template_revision_structure_items.id`, `ON DELETE CASCADE`.
   Same reasoning as every other part of a revision: a report checked under Rev D must stay
   explainable by Rev D's configuration after Rev E exists.
2. **Fields are copied, matched by sheet name, in two places:** importing a new Rev onto a
   Template that already has one (copies from its previous latest Rev), and "Nhân bản" —
   duplicating into a new Template. A sheet that survives keeps its configuration; a sheet that is
   gone takes its configuration with it (it stays on the source, never orphaned); a new sheet
   starts empty. Formula sources are re-pointed at the copies. Copying on duplicate departs from
   the mockup, whose duplicate leaves field mappings behind because it keys them by the source
   Template's id — Sơn's call (2026-09-11): re-entering dozens of fields by hand defeats the point
   of duplicating.
3. **Two tables, one rule applied at both levels** — the stable frame is real columns, the
   per-type settings are one JSONB column:

   ```
   field_mappings        id · structure_item_id · name · check_type · data_type · required
                         sort_order · settings JSONB · created_* · updated_*
   field_mapping_zones   id · field_id · side ('left'|'right') · zone_type · config JSONB
                         UNIQUE(field_id, side)
   ```

   `check_type` and `zone_type` are plain strings, not Postgres enums — adding a value to a
   Postgres enum is a migration, which is exactly the cost this ADR avoids. The valid set lives in
   Python.
4. **Validation moves up to Pydantic, one schema per type, registered in one place**
   (`app/modules/datamapping/zones.py`: `ZONE_SCHEMAS`, `CHECK_DEFS`). The API picks the schema
   by type, rejects a bad payload with 422 at save time, and **keeps only the attributes that type
   uses** — the stale `specValue1` on an Excel zone never reaches the database. The number of
   zones a field sends must match its check type (one-sided vs two-sided). Every rule has a test:
   this is what stands in for the database constraints JSONB gives up.
5. **Edits are allowed on any revision status and every change is audited** (before/after of the
   changed field and zones only, product CLAUDE.md rule #2) — confirmed by Sơn, 2026-09-11. Unlike
   a revision's scope or description, a field is *how* reports are checked, not *what the revision
   is*; locking it to Draft would force a new Rev — and the deactivation that comes with activating
   it — to fix a typo in a cell address.
6. **A duplicate cannot go Active until its Model-specific fields are confirmed** (Sơn,
   2026-09-11). A duplicate is created with no scope and is meant for a different Model, so a
   copied field whose value only held for the source's Model — any `spec` zone (a typed-in
   threshold, or a library Spec, which the mockup scopes to one Model) or `golden` zone — comes out
   with `field_mappings.needs_review = true`. Saving the field again clears it, and that save is
   audited even when the configuration is unchanged: "confirmed for the new Model" is the event.
   `_set_revision_status`, the only path to Active, refuses while any flag is set
   (`TEMPLATE_REVISION_FIELDS_NEED_REVIEW`). A new Rev of the same Template keeps its Model, so it
   flags nothing new but carries an unconfirmed flag over. Thresholds read from a cell of the
   report (an `excel` zone such as `A21`) are never flagged — each report brings its own.
   `needs_review` is a real column, not part of `settings`: it is review state, and the activation
   guard queries it.

## Consequences
- Adding a zone type or a check-type setting is a Pydantic class plus one registry line — no
  migration, no table change.
- The database no longer guards the contents of `config`/`settings`. A wrong value written
  outside the API (a hand-run SQL fix, a future import script) is not caught. Anything writing
  these columns must go through the schemas in `zones.py`.
- "Which fields use Spec X?" needs a JSONB lookup, not an indexed column.
- **Planned promotion:** when Spec Management and Golden Sample get real tables, `spec_set_id`
  and `golden_sample_id` become real FK columns on `field_mapping_zones`, backfilled from
  `config`. That is the point where normalizing buys something; until then it buys nothing.
- **The review flag is coarse on purpose.** It cannot tell a threshold that happens to be the
  same for both Models from one that is not — the new Model is not known at duplicate time, and
  Spec Management does not exist to ask. It flags every `spec`/`golden` zone and relies on a person.
  Once Spec Management and Golden Sample exist, activation can check precisely whether each
  referenced Spec/Golden Sample covers the revision's own targets, instead of relying on the flag.
- **Constraint on a future module:** because fields stay editable after activation, a stored
  check *result* must carry a snapshot of the field configuration it ran with. Without it, a
  verdict from January cannot be explained after the mapping is corrected in June.

## Rejected alternatives
- **Fully normalized** — one column per attribute (19, mostly NULL on any row) plus a child table
  for `entries[]`. Every BA change to a zone becomes a migration against production; its main
  benefit (real FKs to Spec/Golden Sample) has no target tables today.
- **Whole field as one JSONB document.** Loses the columns that are genuinely stable and
  queried — which sheet a field belongs to, its type, its order — and with them ordinary
  indexing, FK cascade from the structure item, and per-zone audit.
- **Attach fields to the Template instead of the revision.** Configuration outlives the sheet it
  was written for (orphans when a Rev drops a sheet), and a past report can no longer be traced
  to the configuration it was checked against.
- **Store the mockup's payload verbatim.** Persists every default and every stale value from a
  changed-mind edit, where they read as real configuration.
