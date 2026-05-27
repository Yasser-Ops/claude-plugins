---
name: server-action
description: Create a new Server Action with zod validation, Supabase auth check, and clean error handling. Use when the user says "server action", "form action", "create an action", "/server-action", or describes wanting to write a mutation triggered by a form.
---

# server-action

Scaffold a Server Action with the project's standard pattern: zod input schema, server-side auth via Supabase, typed return value, no client-side trust.

## Before writing

Confirm:

1. What the action does (insert, update, delete, mutate external service)
2. Which table or external API it touches
3. What inputs it takes and their types
4. Who can call it (any signed-in user, owner only, admin)

## Standard pattern

Actions live next to the route that uses them in `_actions.ts` (the underscore prefix prevents Next.js from treating it as a route):

```ts
// src/app/dashboard/billing/_actions.ts
'use server';

import { z } from 'zod';
import { revalidatePath } from 'next/cache';
import { createClient } from '@/lib/supabase/server';

const UpdateBillingSchema = z.object({
  customerName: z.string().min(1).max(120),
  email: z.string().email(),
});

type ActionResult<T = void> =
  | { ok: true; data: T }
  | { ok: false; error: string };

export async function updateBilling(
  input: z.infer<typeof UpdateBillingSchema>,
): Promise<ActionResult> {
  const parsed = UpdateBillingSchema.safeParse(input);
  if (!parsed.success) return { ok: false, error: 'Invalid input' };

  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return { ok: false, error: 'Not authenticated' };

  const { error } = await supabase
    .from('billing_profiles')
    .update(parsed.data)
    .eq('user_id', user.id);

  if (error) return { ok: false, error: error.message };

  revalidatePath('/dashboard/billing');
  return { ok: true, data: undefined };
}
```

## Rules

- Every action begins with `'use server'`
- Inputs ALWAYS through zod — never trust the client-supplied shape
- Auth via `supabase.auth.getUser()` (validates JWT), not `getSession()`
- Return a discriminated union (`{ ok: true } | { ok: false }`) — never throw raw for expected failures
- Call `revalidatePath` or `revalidateTag` after mutations
- Never expose internal IDs or raw DB error details in the returned `error` string

## Called from a form

Use `useFormState` or `react-hook-form` with the action passed to `<form action={...}>`. The client component owns the form; the action stays server-only.

## Common mistakes to avoid

- Forgetting `'use server'` at the top of the file
- Calling the service-role client from an action without a documented reason
- Wrapping `redirect()` in `try/catch` — Next.js redirects throw internally; let them propagate
- Trusting `formData` shape without parsing through zod
