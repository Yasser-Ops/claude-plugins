---
name: cms-retrofit
description: Use when adding a content-management / admin / CMS / automation layer to an EXISTING website so the owner can edit content and settings live instead of hardcoding them — triggers on "admin panel", "make X editable", "edit content without code", "add a dashboard / CMS", or retrofitting Supabase onto a site. Enforces a survey-first, do-not-code-until-agreed gate.
---

# CMS Retrofit (survey → interview → plan → build)

## Overview

Retrofitting an admin/CMS layer onto a live site is where agents rush to code and break the running site or build the wrong thing. This skill enforces a hard gate: **survey what exists, interview the owner one question at a time, get explicit plan sign-off — and write ZERO code until then.**

**Core principle:** *Don't change anything until we agree on a plan.* Violating the letter of this gate violates its spirit.

## The Iron Gate

```
NO CODE — not one file, command, migration, or dependency —
BEFORE the owner approves the written plan (Step 3).
```

"Code" includes: creating/editing files, running `pnpm add`/installs, writing migrations or SQL, scaffolding `/admin`, or `apply_migration`. Reading files and reporting is allowed and required. Asking questions is allowed and required.

## The Three Steps (in order, no skipping)

### Step 1 — Survey the existing project (read-only)
Investigate the actual repo (don't trust the request's description of it — it is often wrong) and report back, as a deliberate written artifact:
- Framework, language, styling, build tool (Next.js / React+Vite / other; TS or JS; Tailwind version or other).
- Whether a backend, database, or auth already exists — and what.
- Where content currently lives (hardcoded in components? `src/data/`? markdown/MDX? an existing CMS?).
- The existing page/route structure.

If the repo contradicts the request (wrong framework, content not where claimed, multiple projects in the folder), **say so and resolve which project before continuing**.

### Step 2 — Interview the owner, ONE QUESTION AT A TIME
Ask these as separate turns, waiting for each answer before the next. Do NOT collapse them into a single multi-part fork or an "A/B/C and I'll go" shortcut.
1. Which currently-static content do you want editable, and who edits it?
2. Same view for public visitors, or anything gated behind login?
3. What roles are needed (just you as admin, or others with limited access)?
4. Any feature toggles you want to flip without a deploy?
5. Do you already use Supabase, or want it added?

Never guess an answer to skip a question — not auth scheme, not roles, not fallback strategy. Defaulting under time pressure is a gate violation. You may *propose* assumptions for the unasked questions, but surface them explicitly in the Step 3 plan as assumptions awaiting confirmation — never let an inferred answer pass silently into the build.

### Step 3 — Summarize the plan and get explicit OK
State back: which tables, what becomes editable, what the admin panel covers, what roles/RLS apply. **Wait for an explicit "yes/go" before any build.** A vague "sounds good, you know best" is not sign-off — confirm the specifics.

## Build conventions (only after Step 3 sign-off)

**Backend:** Supabase (Postgres + PostgREST + RLS + Auth). If no backend exists, add it; if one exists, adapt these patterns to it.

**Retrofit strategy — don't rewrite the site:**
- For each static piece going editable: create a DB table, a read hook (`useThings` / `useThing(slug)`), and swap the component's static import for the hook — ideally one line per component.
- Keep existing static values as build-time fallbacks in `src/data/`; the hook layers DB overrides on top so the site never renders blank and SEO/meta strings never flash empty.
- Module-level constants that reference hook data must move inside the component body.

**Data fetching:**
- Public reads: raw `fetch()` to PostgREST with the anon key — never supabase-js for public reads (avoids session-hang bugs).
- Authed/admin reads & writes: helper wrappers `restGet`, `restPost`, `restPatch`, `restDelete` that attach the user's JWT.
- DB fields `snake_case`; convert at the component boundary only if needed.

**Settings / toggles:** key-value `site_settings` table (`key TEXT`, `value TEXT`) via `useSettings()` with build-time fallbacks. Boolean feature gates live in the same table. Pre-seed all keys so the admin UI only ever PATCHes, never INSERTs.

**Admin panel:** under `/admin/*`, gated in a layout component. Each editable entity gets an editor page using `restPatch`/`restPost`. Dirty-tracking: Save disabled until a field changes. JSONB columns for nested/flexible fields.

**Auth & roles:** Supabase Auth with role claims; RLS gates every table — public SELECT where appropriate, role-checked writes via an `is_admin()`-style SQL function. Scoped roles get their own panel and only the permissions they need, enforced at the DB level (RLS/RPC), not just hidden in the UI.

**Migrations & seeds:** schema changes in numbered SQL files (`supabase/migrations/001_*.sql`, …); one-time seed scripts in `scripts/seed-*.mjs`, run manually with the service-role key.

**Conventions:** no comments unless the WHY is non-obvious; validate only at system boundaries; lazy-load route-level pages; write admin mutations to an `activity_log` table via a `logAction` helper.

## Red Flags — STOP, you are about to violate the gate

- "I'll just start on step 1 (the build) now…"
- "This isn't back-and-forth, it's a 15-second confirmation."
- "Tell me only if X, otherwise I keep going." (proceeding by default)
- "The opposite of an interview — just pick A/B/C and I'll go." (collapsing the one-at-a-time interview)
- Choosing an auth scheme, role model, or fallback strategy yourself "unless you object."
- Running `pnpm add` / writing a migration before sign-off.

**All of these mean: stop, return to the survey/interview, and wait for plan approval.**

## Rationalization Table

| Excuse | Reality |
|--------|---------|
| "Owner said no questions / is in a hurry" | The owner can't approve a plan they haven't seen. A wrong build under time pressure costs more than a 2-minute survey. Survey + summarize, THEN ask the minimum. |
| "I'll pick a sensible default and note it" | Defaulting auth/roles/fallbacks IS the failure mode. Ask; don't assume. |
| "Surveying is obvious, I'll skip the report" | The written survey is what catches wrong-framework / content-not-where-claimed mismatches. Produce it. |
| "One combined fork is faster than 5 questions" | Combined forks hide the very options the owner needs to weigh. Ask one at a time. |
| "I'm following the spirit, just moving fast" | Violating the letter of the gate violates the spirit. No code before sign-off. |
| "They said 'you know best, go'" | That's not specific sign-off. Confirm tables + scope explicitly first. |

## Relationship to other workflow

Pairs with the workspace's brainstorm-first project-start workflow and the standing "build every site with a live CMS layer" philosophy. This skill is the detailed procedure for the retrofit itself.
