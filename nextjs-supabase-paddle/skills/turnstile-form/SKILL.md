---
name: turnstile-form
description: Build a form protected by Cloudflare Turnstile with server-side token verification. Use when the user says "turnstile", "captcha", "bot protection", "protected form", or "/turnstile-form".
---

# turnstile-form

Add Cloudflare Turnstile to a public form (signup, login, contact, password reset) and verify the token server-side. Client-side success is not authoritative.

## Before writing

Confirm:

1. Which form is being protected
2. Whether a Turnstile site/secret key pair already exists for this environment
3. Whether `lib/turnstile/` already exists (re-use if yes)

## Client side

Use `@marsidev/react-turnstile`:

```tsx
// src/app/(auth)/login/login-form.tsx
'use client';

import { Turnstile } from '@marsidev/react-turnstile';
import { useState } from 'react';
import { login } from './_actions';

export function LoginForm() {
  const [token, setToken] = useState<string | null>(null);

  async function handleSubmit(formData: FormData) {
    if (!token) return;
    formData.set('turnstileToken', token);
    await login(formData);
  }

  return (
    <form action={handleSubmit} className="space-y-4">
      {/* email / password inputs */}
      <Turnstile
        siteKey={process.env.NEXT_PUBLIC_TURNSTILE_SITE_KEY!}
        onSuccess={setToken}
      />
      <button type="submit" disabled={!token}>Sign in</button>
    </form>
  );
}
```

## Server side

`lib/turnstile/verify.ts`:

```ts
import { env } from '@/lib/env';

const VERIFY_URL = 'https://challenges.cloudflare.com/turnstile/v0/siteverify';

export async function verifyTurnstile(token: string, ip?: string): Promise<boolean> {
  if (!token) return false;
  const body = new URLSearchParams({
    secret: env.TURNSTILE_SECRET_KEY,
    response: token,
  });
  if (ip) body.set('remoteip', ip);

  const res = await fetch(VERIFY_URL, { method: 'POST', body });
  if (!res.ok) return false;
  const data = (await res.json()) as { success: boolean };
  return data.success === true;
}
```

In the Server Action:

```ts
'use server';
import { headers } from 'next/headers';
import { verifyTurnstile } from '@/lib/turnstile/verify';

export async function login(formData: FormData) {
  const token = formData.get('turnstileToken')?.toString() ?? '';
  const ip = (await headers()).get('x-forwarded-for') ?? undefined;
  const ok = await verifyTurnstile(token, ip);
  if (!ok) return { ok: false, error: 'Bot check failed' };
  // ... actual login logic
}
```

## Env vars

- `NEXT_PUBLIC_TURNSTILE_SITE_KEY` (public)
- `TURNSTILE_SECRET_KEY` (server-only)

Add both to `lib/env.ts`. If `env-validator` skill exists, run it after.

## Rules

- Always verify server-side. Anyone can POST a forged token.
- Pass the user's IP to `siteverify` when available.
- Turnstile tokens are single-use — treat reuse as failure.
- Turnstile says "probably not a bot," not "this is user X." Don't conflate with auth.

## Common mistakes to avoid

- Skipping server-side verification because the widget shows green
- Putting the secret under `NEXT_PUBLIC_*` (leaks to browser)
- Reusing the same token across two server actions
