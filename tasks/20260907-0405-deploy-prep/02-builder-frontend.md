# Task 02 — Builder: frontend live-API transport layer, mock fallback intact (v2)

## CRITICAL — state discipline
A previous builder died before writing ANY state. After EVERY numbered step
below, IMMEDIATELY append one status line to the `## Result` section at the
bottom of this file (replace PENDING with "in progress: <step> done"). Never
batch writes to the end.

## Repo
`/workspace/platform` (branch main, clean at 4476c37). Frontend: `frontend/`,
React + Vite + TS, bun. API client: `src/lib/api/client.ts` (467 lines, every
method `await delay(n); return mockX`). Types: `src/lib/api/types.ts`.
`src/vite-env.d.ts` exists. package.json scripts: dev/build/test(vitest)/lint —
NO typecheck script; typecheck = `bunx tsc --noEmit`.

## Context (already gathered — do not re-discover)
- Backend endpoints (all under `/v1`): `POST /auth/login` body `{email,password}` → `{token, expiresAt, user}`; 401 on bad creds. `POST /auth/logout` → 204. `GET /jobs` (Bearer; query params: portal, minScore, market, fitVerdict) → plain array of job dicts. `GET /portals` → array. `GET /markets` → array.
- Backend list endpoints return raw dicts (keys as stored in the backend MockDB seed — inspect `backend/app/db.py` job/portal/market seed entries to learn exact key casing before writing any mapping).
- `client.ts` current methods: `async getPortals(): Promise<Portal[]>` at ~line 43; `getJobs()` similar (find it, keep its exact signature).
- Auth dependency backend-side: `Authorization: Bearer <token>` header.

## Work (in order)
1. **`frontend/src/lib/api/transport.ts`** (~80 lines):
   ```ts
   const BASE = (import.meta.env.VITE_API_BASE ?? "").trim().replace(/\/$/, "");
   export const apiEnabled = BASE.length > 0;
   export class ApiError extends Error { constructor(public status: number, message: string) { super(message); } }
   ```
   `apiFetch<T>(path, opts?)`: 15s AbortController timeout, JSON in/out,
   `Authorization: Bearer` from the auth store (step 2), non-2xx →
   `ApiError(status, body.detail ?? statusText)`. Export helpers
   `getAuth(): {token,user}|null`, `setAuth(s)`, `clearAuth()` using
   `localStorage` key `jp.auth`.
2. **`vite-env.d.ts`**: ensure `interface ImportMetaEnv { readonly VITE_API_BASE?: string }` and `ImportMeta` uses it.
3. **client.ts — auth methods** (add near the top of the apiClient object):
   `login(email, password)`: if `apiEnabled` → POST `/auth/login`, setAuth, return user; else return the current mock user (find how the rest of the client identifies the mock user — line ~22 comment mentions it). `logout()`: clearAuth + POST `/auth/logout` when enabled (ignore failures). `401` anywhere → clearAuth + rethrow.
4. **Three live reads** — same signature, flag-gated:
   - `getJobs()`: enabled → `apiFetch('/jobs')` mapped to `Job[]` (write an explicit mapping function based on the backend key casing you inspected; defensive: skip entries missing required fields); disabled → current mock return, byte-identical.
   - `getPortals()`: enabled → `apiFetch<Portal[]>('/portals')` (+ map if keys differ); disabled → unchanged.
   - `getMarkets()`: same pattern with `/markets`.
   NOTHING else migrates. Every other method keeps `await delay(...); return mock...`.
5. **Verify** (append each as you go):
   - `cd /workspace/platform/frontend && bunx tsc --noEmit`
   - `bun run test` (vitest, expect previously-green count)
   - `bun run build`
   - Do NOT boot a dev server; do NOT test against a live backend (none deployed). Do NOT commit — orchestrator commits after critique; leave tree dirty.

## Output — final Result block format
```
## Result
Files: <path — one-line what> (one per line)
Endpoints wired: <exact paths>
Verification: <command — outcome> (one per line)
Unverified: live-backend round-trip (no deployment yet), <anything else>
```

## Result
(Frontend half — orchestrator-executed inline, full detail in 01-builder-backend.md Result block; commits 61e2134 + 809471f cover both units. Endpoints wired: POST /v1/auth/login, POST /v1/auth/logout, GET /v1/jobs, GET /v1/portals, GET /v1/markets. Verification: tsc clean, vitest 1/1, build ✓ 1m18s. Unverified: live-backend round-trip.)