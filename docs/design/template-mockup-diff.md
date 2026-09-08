# Template Management — mockup vs. built app, measured

**Measured 2026-09-08.** Both sides were rendered and measured with Playwright
(`getComputedStyle` / `getBoundingClientRect`), never read from source — engineering-rules #8.
Raw output, screenshots and the scripts are listed under "Reproducing this" at the end.

## Why this exists

Sơn opened the mockup package the BA delivered (`~/Downloads/ok2ship_toolkit`, 2026-09-07) and saw
the Template Management screen differ from the running app. The first question was whether the BA
had shipped a redesign.

**They had not.** Comparing the package against `~/Downloads/2. Template Management/` (the
2026-09-02 mockup this module was built from):

| Screen | `id` attributes differing |
|---|---|
| **template-management** | **2** — and both are `id="${e}"` / `id="${t}"`, template literals inside JS that the package split out into `app-data.js` |
| data-mapping | 45 |
| spec-management | 45 |
| golden-sample | 41 |

Visible Vietnamese strings on template-management: **zero** differences. The package only re-hosts
that screen (1.5 MB inlined → 66 KB + shared CSS/JS). The screen genuinely redrawn in this delivery
is **Data Mapping**, which is separate work.

So the differences Sơn saw are between the **mockup and our build** — consistent with `HANDOFF.md`
marking this module "not complete: several behaviours are waiting on the BA".

## How the two were measured

- Mockup: the BA's Flask prototype on `:5057`, 6 seeded templates. All writes to
  `app_state.json` were intercepted so the BA's saved state was not modified (0 writes reached disk).
- App: the **real stack** — docker compose Postgres/MinIO/mailpit, FastAPI on `:8000`, Vite on
  `:5174`, real login as `admin`. 1 seeded template (`TPL-DEMO-001`, Rev A, Draft, 9 items).
- Same viewport (1440×900), same probe set, fonts and entry animations settled before measuring.

**Limitation, stated up front:** the two sides display different data (6 templates vs 1; Active vs
Draft). That makes L1/L2 findings below solid, but restricts L3 (appearance) to anchors whose
measurement does not depend on content. Differences caused only by data are excluded, not reported.

---

## L1 — Functionality

| # | Mockup | App | Prior decision? | Verdict |
|---|---|---|---|---|
| 1 | No stat cards | **4 stat cards** (Tổng số Template / Đang Active / Chưa có cấu trúc / Cập nhật trong 30 ngày) | The 2026-08-30 mockup had `statTotal`/`statActive`/`statIncomplete`/`statRecent`; the 02/09 update **removed them** | **Stale — built from the older mockup** |
| 2 | No flow banner | **5-step banner** "CẤU HÌNH NGHIỆP VỤ QA", steps 2–5 disabled | `.flow-banner` CSS exists in the mockup but is unused on this screen | **Stale — same cause as #1** |
| 3 | Breadcrumb: `Quản lý Template` | `Quản trị dữ liệu nền / Quản lý Template` | none | App addition |
| 4 | Row action **`Cấu hình`** (gear) → `data-mapping.html?tpl=<id>` | absent from the grid; equivalent lives inside the drawer's structure tab as `Data Mapping →` | none | **Not built at the grid level** (deliberate? undocumented) |
| 5 | Edit drawer: `Trạng thái` `<select>`, `Phạm vi áp dụng` checkbox tree, `Mô tả` input | edit drawer has **one** editable field — `Tên báo cáo` | `PATCH /templates/{id}` accepts `name` only. Re-targeting is called "the risky change" in HANDOFF | **Not built** — #5a scope editing, #5b description, #5c status-via-select |
| 6 | Edit drawer → inline "add a new Model" + link to full hierarchy management | absent | Hierarchy CRUD has no mockup yet (design doc, Open items) | **Intentional** |
| 7 | History tab (edit): `+ Thêm phiên bản` form (Rev / ECO# / Mô tả) | absent — new Revs only via the Import wizard | none found | **Not built** |
| 8 | History table: 5 columns | **6 columns** — extra `Hạng mục` + a per-Rev status badge | none | App addition (useful) |
| 9 | — | Overlays carry **no `role="dialog"`**, and **Escape does not close** the drawer or any modal | none | **Bug** — accessibility + keyboard dismissal |
| 10 | Sidebar: 9 items | 3 items (future modules omitted) | Sidebar.tsx omits unbuilt modules | **Intentional** |

## L2 — Content

| # | Item | Mockup | App | Verdict |
|---|---|---|---|---|
| 11 | Grid columns | `Mã tài liệu · Tên báo cáo · Project · Model · Build · Config · Trạng thái · Cập nhật lần cuối · Thao tác` (9) | `Tên báo cáo · Mã tài liệu · Phạm vi áp dụng · Cấu trúc file · Trạng thái · Cập nhật lần cuối · Thao tác` (7) | **Differs on three counts** — see below |
| 11a | first two columns | code, then name | name, then code | **Order swapped** |
| 11b | scope | 4 separate filterable columns | 1 chip column | **Collapsed** |
| 11c | — | — | extra `Cấu trúc file` column | App addition |
| 12 | Row actions | `Xem · Cấu hình · Sửa · Nhân bản · Vô hiệu hóa · Xóa` | `Xem · Sửa · Nhân bản · Kích hoạt · Xóa` | `Cấu hình` missing (#4); Kích hoạt/Vô hiệu hóa is the same button reflecting status |
| 13 | Add button | `Thêm Template` | `+ Thêm Template` | Literal `+` in the app's label |
| 14 | Structure tab CTA | `Cấu hình →` | `Data Mapping →` | Wording differs |

`TemplateManagementPage.tsx`'s own comment claims the grid "matches the mockup's 8-column
columnDefs order exactly, minus Khách hàng". **The measurement contradicts that** — the mockup has
9 columns in a different order. Dropping `Khách hàng` was Sơn's call (2026-09-04) and stands; the
rest of that claim does not hold and the comment should be corrected whichever way #11 is resolved.

## L3 — Appearance

Only anchors whose measurement is content-independent:

| # | Anchor | Mockup | App | Verdict |
|---|---|---|---|---|
| 15 | `Thêm Template` button | h 32px, border 1px `#2B2FA0`, padding 4/15 | h **36px**, no border, padding 8/14 | Differs |
| 16 | Grid row height | 62px | 64px | Differs |
| 17 | Import modal width | 640px | **600px** | Differs — and it costs layout: the 4 scope selects wrap onto 2 rows in the app, 1 row in the mockup |
| 18 | Dropzone | 592 × 165 | 552 × 154 | Follows from #17 |
| 19 | Drawer width | 760px | 760px | **Matches** |
| 20 | Page title / description | 20px 600, `rgba(0,0,0,0.88)` | same | **Matches** (line-height differs by 3px: 31.43 vs 28) |
| 21 | Grid total width | fits 1232px | **overflows** — every column has `minWidth ≥ 200`, so `Cập nhật lần cuối` is clipped at 1440px (visible in `app-list.png`) | **Bug** |

---

## L4 — Behaviour and displayed information

Added 2026-09-08 after Sơn pointed out the first pass only covered layout. Both sides were walked
end to end (`walk-mockup.mjs` / `walk-app.mjs`): every drawer tab, both wizard steps, all dialogs.

### Drawer — "Thông tin chung", view mode

Same six fields in the same order, but two carry **less information** in the app:

| Field | Mockup | App | Verdict |
|---|---|---|---|
| Phạm vi áp dụng | `Phone › V53 Centaur-5, Phone › V53 Centaur-5 › C3.1 › DOE` — the full path, so the level is unambiguous | `V53 Centaur-5` — leaf name only | **Information lost**: a Build and a Config can share a name; without the path the reader cannot tell which level a target sits at |
| Cập nhật lần cuối | `18/08/2026 · Trinh Thi Nhu Thao` | `8/9/2026` | **Information lost**: no author. Date format also differs (`DD/MM/YYYY` vs unpadded `D/M/YYYY`) |

Footer differs too: mockup `Đóng` + `Lưu thay đổi` (save hidden in view mode); app `Kích hoạt` +
`Đóng` — the app surfaces an activate action inside a read-only drawer.

### Drawer — "Cấu trúc file"

**Effectively equivalent.** Both render `N hạng mục — M đang đánh dấu Bắt buộc` plus a per-row
index, sheet name, `Sheet: <name>` sub-line, `Bắt buộc`/`Tùy chọn` badge and a green check; edit
mode adds a rename input, a required-toggle button and a delete icon on each row. Only the CTA
label differs: mockup `Cấu hình →`, app `Data Mapping →`.

### Drawer — "Lịch sử phiên bản"

App adds a 6th column `Hạng mục` (item count) and a status badge per Rev — more information than
the mockup, and useful. The mockup's edit mode has a `+ Thêm phiên bản` inline form (Rev / ECO# /
Mô tả); the app has none (already logged as #7).

### Import wizard

**Step 1 — the lock/prefill behaviour works and matches.** Selecting a node that already has its
own Template prefills and disables Mã tài liệu + Tên báo cáo and shows the explanatory hint, on
both sides. (A first measurement wrongly reported this missing in the app — the walk had selected
an arbitrary node with no Template of its own, which proves nothing. Re-measured against
`Phone › V53 Centaur-5`, which does have one.)

Wording differs deliberately and the **app's is more correct**: mockup says "Template **Active**
áp dụng" / "Giữ nguyên theo Template **đang Active**", app says "Template áp dụng" / "Template
**hiện có**" — matching decision #13, which keys off `own_template_for_node` regardless of status,
not off Active.

| | Mockup | App |
|---|---|---|
| Model dropdown option | `V53 Centaur-5` | `V53 Centaur-5 (Apple)` — customer appended |
| Scope cascade layout | 4 selects on one row | wraps to 2 rows (follows from the 600px modal, #17) |

**Step 2 — close, with one real improvement and one wording gap.** Both show the detected-sheet
list with `Nhận diện từ file — sheet: <name>` sub-lines and the same three inputs (Rev / ECO# /
Mô tả) with identical placeholders. The app adds the **ECO# helper text** the design doc's Open
items asked for ("Mã Lệnh thay đổi kỹ thuật (Engineering Change Order)…") — that item can be
closed. The app's detection is real (it reads the uploaded file: 6 sheets from a 6-sheet test
file); the mockup's is a 1400 ms `setTimeout` over a copy of the existing structure.

Primary button: mockup `Lưu Template →`, app `Lưu thành Rev A →` (names the Rev — clearer).

### Confirmation dialogs

| | Mockup | App | Verdict |
|---|---|---|---|
| Delete | "Toàn bộ cấu trúc file, cấu hình field và lịch sử phiên bản **sẽ bị xóa**" | "không thể hoàn tác **trên giao diện** — Template sẽ bị ẩn… vẫn được lưu vết (audit_log), không xoá thật" + shows the template's name and code | **App is correct**; the mockup's copy describes a hard delete this system deliberately does not do |
| Activate/Deactivate | title + description only | same, plus the template's name and code | App is clearer |

The mockup's activation-conflict warning box (`#statusConflictBox`) did not trigger in either run —
it needs a competing Active Template on the same node, which neither dataset has. **Not compared;
still open.**

### Nhân bản (duplicate)

Neither side asks for confirmation — it fires immediately. The app really does create the copy
(1 → 2 rows), with no targets, and toasts `Đã nhân bản Template «…» thành Template mới.` The
mockup toasts `… — vui lòng chọn Model áp dụng`, which tells the user what to do next; the app's
does not. Minor, but the mockup's copy is better here.

### Not measured

Toast styling/placement on the app side (the app uses an inline `Alert` above the grid, the mockup
a top-centre floating notice) — different mechanisms, not compared head to head. Activation
conflict warning, as above.

---

## What to do about it — Sơn's call

Nothing here is fixed yet. Suggested grouping:

- **Likely just delete** (built from the superseded 30/08 mockup): #1 stat cards, #2 flow banner.
- **Information the app drops and should not** (L4): the target path in `Phạm vi áp dụng`, and the
  author in `Cập nhật lần cuối`.
- **Genuine gaps to decide on**: #4 `Cấu hình` entry point, #5a–c editable fields, #7 add-version form.
- **Fix regardless of the BA round**: #9 Escape/`role="dialog"`, #21 grid overflow.
- **Cosmetic, cheap**: #13 `+` prefix, #14 wording, #15–18 sizes, the duplicate toast's missing
  "chọn Model áp dụng" follow-up.
- **Leave alone — the app is already better**: the delete dialog's copy, the Rev-named save button,
  the ECO# helper text, the history tab's extra column, #6, #8, #10.

Also worth noting: the app's Model dropdown appends the customer (`V53 Centaur-5 (Apple)`) where
the mockup shows the bare name. Decide which is wanted; it is a one-line change either way.

Do **not** copy these mockup behaviours — they are prototype bugs, several already contradicted by
locked decisions: in-place Rev bump on the parent (decision #9), cascade-delete of a targeted node
(#11), duplicated sheet names (#10), `nextRevLetter` clamping silently at `Z`, `Nhân bản` running
with no confirmation, and the entire "file analysis" during Import being a 1400 ms `setTimeout`
over a hardcoded sheet list.

---

## Resolution (2026-09-08, branches not yet merged)

Frontend `feature/template-mockup-alignment`, backend `feature/template-target-node-path`.

**Done**

| # | What changed |
|---|---|
| 1, 2 | Stat cards and flow banner removed, with `FLOW_STEPS`, `StatCard` and the `stats` memo |
| L4 | Drawer shows the full target path and the author: `Phone › V53 Centaur-5`, `8/9/2026 · admin`. Backend `TargetResponse` gained `node_path` + `project_name`/`model_name`/`build_name`/`config_name` |
| 9 | `useEscapeToClose` hook + `role="dialog"`/`aria-modal` on all three overlays; Escape is suppressed while a request is in flight. Verified on all five overlays in the running app |
| 11 | Grid rebuilt to the mockup's 9 columns in its order, scope split across four columns with multi-select filters, Model-level targets rendering `—` in Build/Config |
| 4 | `Cấu hình` gear added as the 2nd row action, disabled ("chưa xây dựng") like the drawer's own Data Mapping button |
| 21 | Grid no longer overflows — 1156px inside 1158px, no clipped column, no horizontal scrollbar |
| 13, 14, 17, 18 | Plus glyph instead of a literal "+"; import modal 640px; scope cascade back on one row; `Cấu hình →` wording; duplicate toast now says to pick a scope; Model options drop the customer suffix |

**Decided, deliberately not matching the mockup**

- **`Cấu trúc file` column stays removed** (Sơn, 2026-09-08): match the mockup exactly. Item counts
  are visible in the drawer's structure tab only.
- **Button height stays 36px** (Sơn, 2026-09-08) against the mockup's 32px. The mockup's `.ant-btn`
  is 32px on *every* screen, so this is an app-wide deviation, not a Template-screen one; changing
  `components/common/Button.tsx` would restyle User Management, which is signed off and in
  production. Revisit only as a deliberate app-wide pass.
- **Two mockup bugs deliberately not copied**, both measured rather than assumed:
  - its own grid sets minWidths summing to 1232px inside a 1158px area, so *the mockup* clips its
    last two row actions at 1440px. The flex ratios are matched; the minWidth floors are lower.
  - its `.gc-tpl-name` wraps without a cap, so a long name overflows its own 62px row. Wrapping is
    kept, capped at two lines, full name on hover.

**Drawer editability — resolved 2026-09-08**

The mockup edits everything in place because it has no revision model at all: one row holds `rev`,
`structure`, `desc` and `targets` together. Here `targets` and `change_description` belong to the
*revision*, so "edit like the mockup" would mean editing a published snapshot. Sơn's call was to
allow it **only while the revision is still Draft** — the same rule the structure tab already used
(decision #1's corollary), now applied to all three Rev-level fields:

| Field | Where it lives | Editable |
|---|---|---|
| Tên báo cáo | `templates` — Template identity | Always, in edit mode (unchanged) |
| Mã tài liệu | `templates` | Never — immutable (decision #13) |
| Phạm vi áp dụng | `template_revision_targets` — per Rev | **Draft only** — `PUT /templates/revisions/{id}/targets` |
| Mô tả | `template_revisions.change_description` — per Rev | **Draft only** — `PATCH /templates/revisions/{id}` |
| Trạng thái | `template_revisions.status` | Via the activate/deactivate buttons, not a `<select>` — the app keeps its conflict-preview dialog, and the mockup's select would also allow Active → Draft, which decision #1 forbids |

Why Draft-only, concretely — three things a published-Rev edit would bypass:

1. **The version-history table would lie.** It shows one row per Rev with that Rev's current text
   and scope. Editing them in place changes what a Rev claims to have been, with no new row to show
   it happened. Editing the description also rewrites what its ECO# authorized (decision #8).
2. **One-Active-per-node stops being enforceable.** That check lives only in `_set_revision_status`;
   adding a target to an already-Active revision never runs it, so two Active revisions could end
   up competing for one node.
3. **Decision #9 becomes bypassable** — re-pointing a parent's Template at a child node is exactly
   what that decision exists to prevent.

A Draft revision has none of these: nothing published, nothing Active, nothing to conflict with.
Both endpoints refuse otherwise with `409 TEMPLATE_REVISION_NOT_DRAFT` — verified by calling them
directly, not just by hiding the controls. The drawer explains the lock in place rather than
silently disabling the field.

Also fixed here: the Import wizard said `* Mô tả thay đổi`, but on the first Import there is no
previous Rev for it to be a change *from* — now `* Mô tả`, matching the mockup. The history table
keeps `Mô tả thay đổi`, where it is accurate.

**Cleanup done in the same pass (2026-09-08)**

- `RevisionStatusBadge.tsx` deleted — nothing imported it, and it used an explicitly unverified
  palette and different labels ("Đang hiệu lực") from the measured ones.
- `StatusTag` had been copied verbatim into both `TemplateManagementPage` and
  `TemplateDetailDrawer`; now one `RevisionStatusTag.tsx`, re-measured after the merge
  (`#0958D9 / #E6F4FF / #91CAFF`, matching `.tag-processing`).
- **`customer` dropped from the Template API** (Sơn, 2026-09-08). The column went 2026-09-02;
  the response kept deriving one by walking target → Config → Build → Model on every list, for a
  field no screen has shown since the "Khách hàng" column was removed 2026-09-04. `models.customer`
  still owns the fact and the catalog API still returns it.
- Three stale comments corrected: "8-column layout" (it is 9), "`d-customer` is editable" (dropped
  2026-09-04), and a reference to the `FLOW_STEPS` banner (deleted above).

Known and deliberately left: `oxlint` reports `set-state-in-effect` / `only-export-components` on
`RevisionScopeEditor.tsx`, the same warnings `AuthContext`, `Modal`, `StatusBadge` and `RoleSelect`
already carry. It is the house pattern; worth one sweep across the codebase, not a one-file fix.
`SelectColumnFilter` still lives under `components/users/` though both modules now use it — moving
it to `common/` touches User Management, so it is left for a deliberate pass.

**Still open — needs a decision**

- #7: whether `+ Thêm phiên bản` belongs in the history tab, or new Revs stay wizard-only.
- Whether a Template needs its own description (`templates.description`) separate from each Rev's
  `change_description`. The mockup's `tpl.desc` ("dùng chung cho cả 3 Build") is Template-level and
  survives every Rev; `change_description` is replaced by each Import. Deferred, not rejected.
- The activation-conflict warning, still unverified on both sides (needs two Active Templates
  competing for one node — no dataset here produces that).

## Reproducing this

Scripts live outside the repo (session scratchpad, `measure/`): `probe.mjs` (shared measurement),
`measure-mockup.mjs`, `measure-app.mjs`, `compare.mjs`, plus `out/mockup.json`, `out/app.json` and
`out/shots/*.png`. To re-measure: start the mockup (`config_server.py`, :5057), the backend
(`uvicorn app.main:app --port 8000`) and `npm run dev`, then run the two measure scripts and
`compare.mjs`.

Note for anyone re-running: `docker` is not on the PATH of a non-login shell on this machine —
`export PATH="$HOME/.docker/bin:$PATH"` first. The local DB needed `alembic upgrade head` +
`app.cli.bootstrap_admin` + `app.cli.seed_template_test_data` before it had any data at all.
