# Frontend mockup review — `app.html` (BA-provided, 2026-08-22)

> **Status: resolved, Phase 5 built (2026-08-22).** Every gap/conflict below was decided (with
> Sơn) and built — see `HANDOFF.md`'s Phase 4.5 and Phase 5 entries for exactly what landed. Kept
> here as the historical record of the review, not as an open task list anymore.
>
> **Correction (2026-08-22, caught by Sơn reviewing the built frontend):** this review's initial
> pass filtered out every line longer than 2000 characters to keep the file readable (base64-image
> lines were tens/hundreds of KB each) — which silently threw away the product's actual **logo**
> (a Mektec mark, embedded as base64 PNG in both the sidebar and login-page `.mark` divs) along
> with the exact product name ("OK2SHIP AI", not "OK2Ship AI Check") and the login page's real
> left-panel headline/body copy. Phase 5's first pass missed all three, inventing placeholder text
> instead. Fixed afterward — extracted the real logo to `frontend/public/logo.png`, corrected the
> product name everywhere, and copied the mockup's actual headline/paragraph/footer text verbatim.
> **Lesson for next time**: a line-length filter is fine for finding *structure* (classes, JS
> logic) but never skip going back to check what a long line actually contains before concluding
> a visual element "isn't there" — silence in a filtered view isn't evidence of absence.

Source: `~/Downloads/app.html` (not in this repo — provided by Sơn from the vendor's BA, outside
git). This review maps what it shows against what the backend (Phase 0–4 + two gap-closing
additions) actually supports, before Phase 5 (frontend build) starts.

## What the file actually is

A single static HTML file — **not** a React app, despite Ant Design/AG Grid framing:
- **Ant Design look**: hand-rolled CSS classes named `ant-btn`, `ant-modal`, `ant-form-item`, etc.
  — visually matches antd, but the actual `antd` JS library is not imported. It's a CSS-only
  imitation.
- **AG Grid**: the real thing — AG Grid Community v36.1.0 is inlined (no CDN), genuinely used for
  the user table.
- Interactivity is vanilla JS (`onclick="..."`, direct DOM manipulation) against an in-memory
  demo dataset — no real backend calls anywhere in the file.

**Conclusion**: treat this as a **visual/UX reference**, not code to port. Phase 5 still builds in
React 18 + Vite + Tailwind (locked stack, `CLAUDE.md`) and should genuinely use AG Grid Community
for the user table (matches the mockup's real choice there) — Tailwind utility classes standing in
for the hand-rolled "antd-style" CSS is a reasonable interpretation of "Ant Design look", not a
mandate to add the `antd` package.

## Screens/flows shown

1. Login (`Tên đăng nhập hoặc Email` + password, remember-me, "Quên mật khẩu?" link)
2. **Forgot password** — request a reset link by email (always-same-response copy, anti-enumeration)
3. **Reset password** — set a new password from the emailed link (30 min TTL, single-use, stated in copy)
4. Full app shell — sidebar lists the whole future product (data ingestion, AI/statistics
   processing, decision/display groups), User Management is one item under "Nền tảng & vận hành"
5. User grid (AG Grid): Họ và tên, Email/Tên đăng nhập (combined cell), Phòng ban, Vai trò, Trạng
   thái, Đăng nhập gần nhất, Thao tác (view/edit/toggle/delete icons). Search + column filters +
   client-side pagination (6/12/24 rows).
6. Create/Edit modal — username locked (disabled + "Không thể chỉnh sửa sau khi tạo.") on edit,
   matches decision immutability exactly.
7. View detail modal (read-only)
8. Delete confirm modal — copy says "Hành động này không thể hoàn tác" (cannot be undone)
9. Active/Inactive toggle confirm modal — separate from Delete
10. **Self-service Change Password** modal (header user-dropdown, while logged in — needs current
    password)
11. Logout confirm modal

## Matches confirmed (no action needed)

| Item | Mockup | Backend |
|---|---|---|
| Username immutable after creation | Field disabled + explicit copy on edit | Enforced (schema has no `username` field in `UserUpdate`) |
| Failed-login lockout threshold | Copy states "Sai mật khẩu 5 lần liên tiếp... tự động khóa" | `max_failed_login_attempts=5` default — **now vendor-confirmed**, no longer just our guess |
| Department options | QA/SQE/NPI/OQC/IQC/LAB/Admin | Matches `Department` enum exactly |
| List filters/search/pagination | Grid has search + column filters + pagination | `list_users()` already supports `status`/`department`/`search`/`limit`/`offset` |
| "Delete" and Active/Inactive as distinct UI entries | Two separate icons/modals in the grid | Reconcilable **without a backend change**: both can call the same `PATCH /users/{id}/status` — Delete = call with `Inactive` + its own (adjusted) confirmation copy; the toggle stays the reversible switch. See "Needs reconciliation" below for the copy issue. |

## Gaps — mockup expects backend that doesn't exist yet

| # | Feature | Backend status |
|---|---|---|
| 1 | **Forgot password** (request reset link by email) | Not built. Different from `/auth/activate` — this is for an already-`Active` user, not `Create`→`Active`. |
| 2 | **Reset password** (redeem the emailed link) | Not built. Mockup states a 30-minute, single-use TTL — separate lifecycle from the 7-day activation token. |
| 3 | **Self-service change password** (logged in, knows current password) | Not built. Every existing password-setting path (`/auth/activate`) is for a *pending* account; nothing lets an *Active* user rotate their own password by choice. |
| 4 | Login accepting **username OR email** | `POST /auth/login` only matches on `username` today. |

## Conflicts — need a decision before building, not silently assumed

| # | Conflict | Detail |
|---|---|---|
| 1 | **Multi-role vs single-role UI** | This is the big one. Design doc decision #2 (locked): a user can hold multiple roles, full RBAC. The mockup's role field is a single native `<select>` with exactly two hardcoded options (`QA`, `Admin`) — no multi-select anywhere (create form, edit form, grid column, view modal all show one role). **Needs a decision from Sơn/vendor BA**: does the real product actually want single-role-per-user after all (which would reopen decision #2), or is this mockup simply an unfinished simplification and Phase 5 should build proper multi-select regardless? Building multi-select against a mockup that visually shows single-select risks looking "wrong" to the vendor at review time if the answer is the latter — better to confirm first. |
| 2 | **Password policy wording** | Mockup copy (reset + change-password forms): "Tối thiểu 8 ký tự, gồm **chữ hoa, chữ thường và số**" (upper *and* lower case, and digits). Design doc decision #8 only says "minimum 8 characters, must include letters and digits" — no case requirement. Current `PASSWORD_PATTERN` in `app/modules/auth/schemas.py` matches the *design doc* (any-case letters + digit), not the *mockup's* stricter wording. Needs reconciling one way: loosen the mockup copy, or tighten the regex + update decision #8. |
| 3 | **"Delete" confirmation copy vs. actual reversibility** | Mockup: "Hành động này không thể hoàn tác" (this action cannot be undone). Product CLAUDE.md / Constitution: **no hard delete, ever** — `Inactive` is fully reversible via the same toggle. Shipping the mockup's literal copy would be misleading to real users (implies permanence that doesn't exist). Frontend copy needs to say something truthful instead (e.g. "tài khoản sẽ ngừng đăng nhập được, có thể kích hoạt lại sau" or similar) — flagging so it isn't silently copy-pasted from the reference file as-is. |

## Implementation-quality note (not a vendor question — just something to get right in Phase 5)

The mockup's own toggle-button label logic (`d.status === 'Inactive' ? 'Kích hoạt' : 'Vô hiệu hóa'`)
doesn't distinguish `Locked` or `Create` from `Active` — a `Locked` row would show "Vô hiệu hóa"
(deactivate) instead of "Kích hoạt lại" (reactivate), which is backwards for decision #6 (Locked
recovers via the same toggle set to Active). Phase 5's real implementation should branch on the
actual status, not copy this simplified ternary as-is.

## Not covered by the mockup at all

- `POST /users/{id}/resend-activation` (gap we closed 2026-08-21) has no UI entry point in the
  grid's action icons. Needs a design decision for Phase 5: add a 5th row action (visible only
  when `status == Create`), or leave it as an Admin-only/API-only tool for now.
