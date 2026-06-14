---
name: supabase-migration
description: Create a new Supabase migration and regenerate the project's TypeScript types after schema changes. Use when the user says "new migration", "schema change", "add a table", "add a column", "migrate the db", or "/supabase-migration".
---

# supabase-migration

Create a migration file under `supabase/migrations/` and regenerate `lib/database.types.ts`.

## Before writing

Confirm:

1. What's being changed (new table, new column, alter constraint, new index, RLS policy update)
2. The table(s) involved
3. Whether RLS needs adjustment in the same migration
4. Whether the change is destructive (dropping a column) — flag it explicitly

## Steps

1. Create the migration file with a descriptive snake_case name:
   ```bash
   pnpm supabase migration new add_billing_profiles_table
   ```

2. Write SQL. Standard new-table template:
   ```sql
   create table public.billing_profiles (
     id uuid primary key default gen_random_uuid(),
     user_id uuid not null references auth.users(id) on delete cascade,
     customer_name text not null,
     email text not null,
     created_at timestamptz not null default now(),
     updated_at timestamptz not null default now()
   );

   alter table public.billing_profiles enable row level security;

   create policy "Users read their own billing profile"
     on public.billing_profiles for select
     using (auth.uid() = user_id);

   create policy "Users update their own billing profile"
     on public.billing_profiles for update
     using (auth.uid() = user_id)
     with check (auth.uid() = user_id);

   create policy "Users insert their own billing profile"
     on public.billing_profiles for insert
     with check (auth.uid() = user_id);
   ```

3. Apply locally or push to the linked project:
   ```bash
   pnpm supabase db reset      # local
   pnpm supabase db push       # linked project
   ```

4. Regenerate types:
   ```bash
   pnpm supabase gen types typescript --linked > lib/database.types.ts
   ```

5. Run `pnpm typecheck`.

## Rules

- **Every new table must enable RLS in the same migration.** A table without RLS is a security incident.
- Every user-scoped table needs at minimum a SELECT policy keyed on `auth.uid()`.
- Use `on delete cascade` for foreign keys pointing at `auth.users(id)`.
- Never edit a migration that has already been applied — create a new one.
- Use `gen_random_uuid()` for primary keys unless there's a specific reason not to.

## Common mistakes to avoid

- Forgetting `enable row level security` on a new table
- Adding sensitive columns without revisiting RLS policies
- Editing a previously-applied migration to "fix a typo" — write a fix-up migration instead
- Forgetting to regenerate types after the schema change
