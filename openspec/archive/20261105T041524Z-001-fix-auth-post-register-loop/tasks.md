# Tasks: Fix Auth Post-Register & Login Redirect Loop

> **Change ID:** `001-fix-auth-post-register-loop`
> **Execution Order:** `001`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] `lib/api/auth.ts`: export `getAccessToken()` (returns `inMemoryToken`); add `credentials: 'include'` to auth calls; keep NOT writing the JWT to `localStorage`/`document.cookie` (SEC-HIGH-02).
- [x] `lib/api/client.ts`: `getStoredToken()` delegates to `getAccessToken()`; add `credentials: 'include'` to `apiFetch`.
- [x] `lib/hooks/use-auth.ts`: rehydrate session via `GET /api/auth/me` (with credentials) when `inMemoryToken` is null on mount.
- [x] `components/login-form.tsx`: when `?redirect=` is present, do not auto-redirect from stale state; clear stale in-memory session.
- [x] `components/register-form.tsx`: await confirmed session before `router.push("/characters")`.
- [x] `components/navbar.tsx`: dynamically point brand logo link to `/characters` (Dashboard) when authenticated, and `/` when unauthenticated.
- [x] `middleware.ts`: clear malformed/expired `dungeoneer_auth_token` on redirect to `/login`.
- [x] Logout uses `POST /api/auth/logout` (HttpOnly cookie cannot be deleted from JS).

## Test Tasks

- [x] Vitest: after register/login, `getAccessToken()` set, JWT absent from `document.cookie`/`localStorage`, `apiFetch` sends `Authorization: Bearer` and `credentials: 'include'`.
- [x] Vitest: `LoginForm` with `?redirect=` and stale in-memory auth does not call `router.replace("/characters")`.
- [x] Vitest: `Navbar` brand logo links to `/characters` when authenticated and `/` when not authenticated.

## Documentation Tasks

- [x] Update `specs/frontend-web-app-delta.md` (already aligned with HttpOnly model).

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing (`npm test`)
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
