# ADR 002 — Split into three repositories: planning (GitHub) + backend (GitLab) + frontend (GitLab)

Date: 2026-08-23 (retroactive) | Status: accepted

## Context
`products/ok2ship-ai` was scaffolded as a single full-stack repo per the hub's standard layout
(`backend/`, `frontend/` as subdirectories of one repo — see hub `templates/fullstack/CLAUDE.md`).
By the time Phase 0–5 were built (DB/auth/RBAC, User CRUD, frontend), none of it had been
committed yet (see `docs/PROGRESS.md`, 2026-08-23 entries) — a real risk flagged in a hub-level
review that day. When the decision was made to push the work to a remote, the target wasn't the
personal GitHub used for hub/spike work: backend and frontend are deliverable code for the client
(Mektec Vietnam), whose infrastructure is GitLab; the planning/design docs (`HANDOFF.md`,
`docs/design/*.md`, `docs/PROGRESS.md`) are working notes for building the product, not something
a client-facing repo needs to carry.

## Decision
Split `products/ok2ship-ai` into three independent git repositories, each with its own remote and
its own pre-commit hook:
1. `products/ok2ship-ai/` (parent) — planning/design docs only. Remote:
   `git@github.com:Mai-Hong-Son/ok2ship-ai-agents.git`.
2. `products/ok2ship-ai/backend/` — the FastAPI backend. Remote:
   `git@gitlab.com:mektec/ok2ship-ai-backend.git`.
3. `products/ok2ship-ai/frontend/` — the React frontend. Remote:
   `git@gitlab.com:mektec/ok2ship-ai-frontend.git`.

The parent repo's `.gitignore` excludes `backend/`/`frontend/` in full (not just their build
artifacts), so it never tracks their content or creates a submodule-style gitlink for them. All
local working copies stay at the same paths — nothing is re-cloned elsewhere, and nothing
auto-syncs between the three; each is pushed independently, only when asked. Full topology detail
(remote URLs, initial commit SHAs, file/test counts, local working-copy locations):
`HANDOFF.md` §"Repo topology".

## Rationale
- Deliverable code belongs on the client's own infrastructure (GitLab, under the `mektec` group);
  planning/design working notes don't need to live there and would just be dead weight in a
  client-facing repo.
- Independent repos let backend and frontend each branch/PR/tag on their own cadence once real
  feature work starts, without the other's changes showing up in the same diff — matches the
  Constitution's "one concern per PR" more naturally than a shared monorepo would, given this
  split of ownership (personal hub-adjacent planning vs. client-owned deliverable code).
- No submodule/gitlink complexity: excluding `backend/`/`frontend/` outright in the parent's
  `.gitignore` is simpler than managing them as git submodules, at the cost of the parent repo
  never being able to pin/reference an exact backend/frontend commit from itself.

## Consequences
- This deviates from the hub's standard full-stack layout (`templates/fullstack/CLAUDE.md`
  assumes one repo, two subdirectories). Worth revisiting the template/skill if this pattern
  (client-infra deliverable code vs. personal-infra planning docs) recurs on a future product —
  not backported automatically by this ADR alone.
- `HANDOFF.md` / `docs/PROGRESS.md` / `docs/design/*.md` exist ONLY in the parent repo — a session
  working from a fresh clone of just `backend/` or `frontend/` has no access to them. Anyone
  picking up backend or frontend work independently also needs the parent repo.
- All three repos' first commits are single "initial import" squashes, not phase-by-phase
  history — a direct consequence of nothing having been committed incrementally while building
  (flagged separately in a hub-level review the same day) compounding with the split: there was no
  historical commit sequence left to preserve or divide by the time the repos were created.
- Going forward, each repo follows a normal branch-per-feature + PR + qa-reviewer flow
  independently (Constitution git rules) — no longer blocked by the single-repo assumption.

Related: `HANDOFF.md` §"Repo topology", `docs/PROGRESS.md` (2026-08-23 entries), hub
`templates/fullstack/CLAUDE.md` (the layout this deviates from), ADR 001 (the RBAC decision built
on top of this same product before the split happened).
