---
name: scaffold-route
description: Scaffold a new Next.js App Router route segment with page.tsx, loading.tsx, error.tsx, and optionally a layout. Use when the user says "scaffold a route", "new route", "new page", "add a page at /foo", "/scaffold-route", or describes wanting a new URL in the app.
---

# scaffold-route

Create a new App Router route under `src/app/`. Default to Server Components. Never add `'use client'` to `page.tsx` unless interactivity is explicitly required.

## Before writing files

Confirm:

1. The URL path (e.g. `/dashboard/billing`, `/settings/[id]`)
2. Who can access it: public, signed-in user, admin, or org-scoped
3. Whether the route is dynamic (`[id]`, `[...slug]`) and what shape the param must be
4. What data the page renders and where it comes from (Supabase table, search params, static)

If the user already supplied the path and intent, state the inferred answers in one sentence and proceed.

## Files to create

- `page.tsx` — Server Component
- `loading.tsx` — Suspense fallback using shadcn `Skeleton`
- `error.tsx` — error boundary, must be `'use client'`
- `layout.tsx` — only if the segment introduces shared chrome

## page.tsx (signed-in user, Supabase-backed)

```tsx
import { redirect } from 'next/navigation';
import { createClient } from '@/lib/supabase/server';

export default async function Page() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');

  const { data, error } = await supabase
    .from('table_name')
    .select('*')
    .eq('user_id', user.id);
  if (error) throw error;

  return <main className="container mx-auto p-6">{/* render data */}</main>;
}
```

## loading.tsx

```tsx
import { Skeleton } from '@/components/ui/skeleton';

export default function Loading() {
  return (
    <div className="container mx-auto p-6 space-y-4">
      <Skeleton className="h-8 w-1/3" />
      <Skeleton className="h-64 w-full" />
    </div>
  );
}
```

## error.tsx

```tsx
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="container mx-auto p-6">
      <h2 className="text-lg font-semibold">Something went wrong</h2>
      <p className="text-sm text-muted-foreground">{error.message}</p>
      <button onClick={reset} className="mt-4 underline">Try again</button>
    </div>
  );
}
```

## Dynamic params

Validate the param shape with zod before using it:

```tsx
import { z } from 'zod';

const ParamsSchema = z.object({ id: z.string().uuid() });

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = ParamsSchema.parse(await params);
  // ...
}
```

## After creating files

Run `pnpm typecheck` and report any errors. If the user mentioned tests, add a Playwright spec at `e2e/<route-name>.spec.ts` covering the auth gate and a happy-path render.

## Common mistakes to avoid

- Adding `'use client'` to `page.tsx` when it could stay a Server Component
- Using `supabase.auth.getSession()` for authorization — use `getUser()` instead, which validates the JWT
- Trusting `searchParams` or `params` without zod validation
- Putting auth checks only in middleware and skipping them in the page
