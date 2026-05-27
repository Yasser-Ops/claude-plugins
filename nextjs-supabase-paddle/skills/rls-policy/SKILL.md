---
name: rls-policy
description: Write or audit Supabase Row Level Security policies for a table. Use when the user says "RLS", "row level security", "policy", "secure this table", "review my policies", or "/rls-policy".
---

# rls-policy

Help write tight, correct RLS policies or audit existing ones for security holes.

## Before writing

Confirm:

1. Which table
2. Who should be allowed to read each row (owner only, org members, public, signed-in users, admins)
3. Who should be allowed to insert / update / delete
4. Whether the table has `user_id`, `org_id`, or another ownership column

## Owner-scoped table

```sql
alter table public.<table> enable row level security;

create policy "<table>_select_own"
  on public.<table> for select
  using (auth.uid() = user_id);

create policy "<table>_insert_own"
  on public.<table> for insert
  with check (auth.uid() = user_id);

create policy "<table>_update_own"
  on public.<table> for update
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

create policy "<table>_delete_own"
  on public.<table> for delete
  using (auth.uid() = user_id);
```

## Org-scoped table (membership check)

```sql
create policy "<table>_select_org_member"
  on public.<table> for select
  using (
    exists (
      select 1 from public.org_members
      where org_members.org_id = <table>.org_id
        and org_members.user_id = auth.uid()
    )
  );
```

## Audit checklist for an existing table

1. **RLS enabled?** `select relrowsecurity from pg_class where relname = '<table>';` — false is a critical bug.
2. **At least one policy per allowed operation.** Without an explicit policy, the operation is denied for non-service-role roles.
3. **`using` vs `with check`** — `using` filters which rows the operation sees; `with check` validates new/updated row values. UPDATE needs both; INSERT needs `with check`; DELETE and SELECT need `using`.
4. **No `using (true)` or `with check (true)`** unless the table is genuinely public.
5. **`auth.uid()` is referenced correctly** — it returns null for anon users.
6. **Views inherit RLS only when defined `with (security_invoker)`.** Without it, views run with definer privileges and may leak data.
7. **Service-role usage** — any admin client call that bypasses RLS should have a code comment explaining why.

## Common mistakes to avoid

- Forgetting `with check` on UPDATE — lets a user change `user_id` to someone else's
- Writing only a SELECT policy and assuming it covers INSERT/UPDATE/DELETE
- Using `auth.role() = 'authenticated'` as the whole condition (any signed-in user sees every row)
- Disabling RLS "temporarily" during dev and forgetting to re-enable
