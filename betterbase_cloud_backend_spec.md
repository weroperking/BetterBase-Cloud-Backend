# BetterBase Cloud Backend — Cursor Build Spec
> Feed this entire document to Cursor. It contains everything needed to build the Supabase-backed backend that powers `bb login` and project management.

---

## What You Are Building

A lightweight backend using **Supabase** (PostgreSQL + Auth + Edge Functions or a simple Express/Hono API deployed on Railway/Fly/Vercel) that serves two purposes:

1. **CLI device auth flow** — lets `bb login` authenticate a developer via browser and store credentials locally
2. **Project registry** — lets developers register and list their BetterBase projects

This is NOT a dashboard. It is a JSON API. The CLI is the only client right now.

---

## Tech Stack Decision

- **Database + Auth**: Supabase (PostgreSQL + Supabase Auth)
- **API layer**: Hono on Bun, deployed to Railway or Fly.io (or Supabase Edge Functions if you prefer serverless)
- **No frontend needed** — all endpoints are consumed by the CLI

> If you want zero infra to manage right now, use Supabase Edge Functions for all endpoints. If you want a proper server you can extend later, use Hono on Bun on Railway.

---

## Supabase Schema

Run this SQL in the Supabase SQL editor.

```sql
-- ─────────────────────────────────────────────────────────────
-- device_codes
-- Stores pending CLI auth requests (device flow)
-- ─────────────────────────────────────────────────────────────
create table public.device_codes (
  code          text        primary key,         -- format: XXXX-XXXX (CLI-generated)
  user_id       uuid        references auth.users(id) on delete cascade,
  token         text,                             -- JWT issued after auth, returned to CLI
  email         text,                             -- user email, returned to CLI
  expires_at    timestamptz not null,             -- code expires 5 minutes after creation
  authenticated_at timestamptz,                  -- set when user completes auth in browser
  created_at    timestamptz default now()
);

-- Only the service role can write; anon can only poll their own code
alter table public.device_codes enable row level security;

create policy "anon can read own device code by code"
  on public.device_codes for select
  using (true); -- code is a secret; possession = access

create policy "service role full access"
  on public.device_codes for all
  using (auth.role() = 'service_role');

-- ─────────────────────────────────────────────────────────────
-- projects
-- BetterBase projects registered by developers
-- ─────────────────────────────────────────────────────────────
create table public.projects (
  id            uuid        primary key default gen_random_uuid(),
  user_id       uuid        not null references auth.users(id) on delete cascade,
  name          text        not null,
  local_id      text        unique,               -- nanoid used in local dev (optional)
  created_at    timestamptz default now()
);

alter table public.projects enable row level security;

create policy "users can manage their own projects"
  on public.projects for all
  using (auth.uid() = user_id);
```

---

## Environment Variables

```env
# Supabase
SUPABASE_URL=https://xxxxxxxxxxxx.supabase.co
SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key   # used server-side only, never exposed

# App
PORT=3000
```

---

## API Endpoints

### Base URL
- Local dev: `http://localhost:3000`
- Production: whatever your deployment URL is (Railway/Fly/Supabase Edge)

The CLI has this hardcoded as:
```
BETTERBASE_API=https://app.betterbase.com   (or your actual URL)
```

---

### 1. POST `/api/cli/auth/init`

Called by the web browser after the user authenticates. The web app hits this to mark a device code as authenticated and attach credentials.

**Request body:**
```json
{
  "code": "ABCD-1234",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "email": "user@example.com",
  "userId": "uuid-of-user"
}
```

**Logic:**
1. Look up the device code in `device_codes`
2. If not found or expired → return 404
3. If already authenticated → return 409
4. Update the row: set `token`, `email`, `user_id`, `authenticated_at = now()`
5. Return 200

**Response (200):**
```json
{ "ok": true }
```

**Use service role key** for this endpoint (it writes to device_codes).

---

### 2. GET `/api/cli/auth/poll?code=XXXX-XXXX`

Polled every 2 seconds by `bb login` until authenticated or timed out.

**Logic:**
1. Look up the device code
2. If not found → 404
3. If `expires_at < now()` → 410 (Gone / expired)
4. If `authenticated_at` is null → 202 (still pending)
5. If authenticated → 200 + credentials

**Response (200):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "email": "user@example.com",
  "userId": "uuid-of-user",
  "expiresAt": "2026-04-03T12:00:00Z"
}
```

**Response (202):**
```json
{ "status": "pending" }
```

**Use anon key** — the code itself is the secret.

---

### 3. POST `/api/cli/auth/device`

Called by the CLI to register a new device code before opening the browser. Optional but cleaner than creating the code only on the web side.

**Request body:**
```json
{ "code": "ABCD-1234" }
```

**Logic:**
1. Insert a new row into `device_codes` with `expires_at = now() + interval '5 minutes'`
2. Return 201

**Response (201):**
```json
{ "ok": true }
```

**Use service role key.**

---

### 4. GET `/api/projects`

Returns the authenticated user's projects.

**Headers:**
```
Authorization: Bearer <token>   ← the token stored in ~/.betterbase/credentials.json
```

**Logic:**
1. Verify the JWT using Supabase (pass it to `supabase.auth.getUser(token)`)
2. If invalid → 401
3. Query `projects` where `user_id = user.id`
4. Return array

**Response (200):**
```json
{
  "projects": [
    {
      "id": "uuid",
      "name": "my-app",
      "localId": "abc123",
      "createdAt": "2026-03-01T00:00:00Z"
    }
  ]
}
```

---

### 5. POST `/api/projects`

Registers a new BetterBase project.

**Headers:**
```
Authorization: Bearer <token>
```

**Request body:**
```json
{
  "name": "my-app",
  "localId": "abc123def456"   ← optional, the nanoid from local betterbase.config.ts
}
```

**Logic:**
1. Verify JWT → get user
2. Insert into `projects`
3. Return 201 + project

**Response (201):**
```json
{
  "id": "uuid",
  "name": "my-app",
  "localId": "abc123def456",
  "createdAt": "2026-03-01T00:00:00Z"
}
```

---

## Web Auth Page (minimal HTML, no framework needed)

The CLI opens:
```
https://your-api-url/cli/auth?code=XXXX-XXXX
```

This page needs to:
1. Show a Supabase Auth UI (email/password or magic link — whatever you have configured)
2. After user signs in, call `POST /api/cli/auth/init` with the code + user's token
3. Show a "You can close this window" message

This can be a single static HTML file with inline JS. No React needed. Example:

```html
<!DOCTYPE html>
<html>
<head><title>BetterBase Login</title></head>
<body>
  <h2>Authenticating CLI...</h2>
  <div id="status">Waiting for login...</div>

  <script>
    const code = new URLSearchParams(window.location.search).get('code');

    // After Supabase auth completes and you have the session token:
    async function completeAuth(token, email, userId) {
      const res = await fetch('/api/cli/auth/init', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ code, token, email, userId })
      });
      if (res.ok) {
        document.getElementById('status').textContent =
          '✓ Authenticated. You can close this window.';
      }
    }

    // Wire this to your Supabase auth callback
    // supabase.auth.onAuthStateChange((_event, session) => {
    //   if (session) completeAuth(session.access_token, session.user.email, session.user.id)
    // })
  </script>
</body>
</html>
```

---

## CLI Integration Points

The CLI code in `packages/cli/src/commands/login.ts` is already complete and just needs to be uncommented. The constants it uses:

```typescript
const BETTERBASE_API = process.env.BETTERBASE_API_URL ?? "https://app.betterbase.com"
```

Set `BETTERBASE_API_URL` to your actual deployment URL in `.env` for local testing.

The device code format generated by the CLI: `XXXX-XXXX` (8 alphanumeric chars, uppercase, no ambiguous characters).

The credentials file stored locally: `~/.betterbase/credentials.json`
```json
{
  "token": "...",
  "email": "user@example.com",
  "userId": "uuid",
  "expiresAt": "2026-04-03T12:00:00Z"
}
```

---

## Hono Server Structure (if not using Edge Functions)

```
betterbase-cloud/
├── package.json
├── .env
├── src/
│   ├── index.ts          ← Hono app entry, registers all routes
│   ├── lib/
│   │   └── supabase.ts   ← createClient (anon) + createServiceClient (service role)
│   └── routes/
│       ├── auth.ts        ← /api/cli/auth/* endpoints
│       └── projects.ts    ← /api/projects endpoints
└── public/
    └── cli-auth.html      ← the browser auth page served at /cli/auth
```

**`src/lib/supabase.ts`:**
```typescript
import { createClient } from '@supabase/supabase-js'

export const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_ANON_KEY!
)

export const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
)
```

**`src/index.ts`:**
```typescript
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { serveStatic } from 'hono/bun'
import authRoutes from './routes/auth'
import projectRoutes from './routes/projects'

const app = new Hono()

app.use('*', cors())
app.route('/api/cli/auth', authRoutes)
app.route('/api/projects', projectRoutes)
app.use('/cli/auth', serveStatic({ path: './public/cli-auth.html' }))

export default { port: process.env.PORT ?? 3000, fetch: app.fetch }
```

---

## Validation Checklist (test these manually after deploy)

- [ ] `POST /api/cli/auth/device` with a code → 201
- [ ] `GET /api/cli/auth/poll?code=XXXX-XXXX` → 202 (pending)
- [ ] `POST /api/cli/auth/init` with code + valid Supabase token → 200
- [ ] `GET /api/cli/auth/poll?code=XXXX-XXXX` again → 200 + credentials
- [ ] Expired code (manually set `expires_at` in past) → 410
- [ ] `GET /api/projects` with valid token → 200 + array
- [ ] `POST /api/projects` → 201 + project
- [ ] Open browser to `/cli/auth?code=XXXX-XXXX` → auth page loads
- [ ] Run `bb login` end to end → credentials saved to `~/.betterbase/credentials.json`

---

## Notes for Cursor

- Use `@supabase/supabase-js` v2
- Use `hono` v4
- Runtime is Bun — use `bun:sqlite` is NOT needed here, all data is in Supabase
- All endpoints return `application/json`
- No session cookies — the CLI uses bearer tokens
- The service role key is NEVER sent to the browser or the CLI — server-side only
- The `expiresAt` returned to the CLI should be `now() + 30 days` (long-lived token for CLI UX)
- Do NOT implement refresh tokens for the CLI — just re-login after 30 days
