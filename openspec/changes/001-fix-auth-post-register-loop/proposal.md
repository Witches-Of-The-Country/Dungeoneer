# Bugfix Proposal: Fix Auth Post-Register & Login Redirect Loop

> **Change ID:** `001-fix-auth-post-register-loop`
> **Execution Order:** `001`
> **Type:** Bugfix & Legacy Adaptation
> **Priority:** Critical (Prioridade Máxima)
> **Status:** draft
> **Created:** 2026-11-04
> **Last Updated:** 2026-11-04

---

## Summary

Fixes the critical infinite redirect loop that occurs immediately after registering a new user (or logging in when cookies are dropped across ports), preventing navigation to `/characters` and trapping the user on `/login`. Synchronizes client-side in-memory JWT storage, cross-origin credential cookies, and Next.js Edge Middleware session verification.

## Motivation

Beta testers reported that newly registered users cannot reach the characters screen (blocker, top priority). The fix must also honor the security hardening in `fix-feat-charges-wizard-reference-endpoints` (SEC-HIGH-02: HttpOnly session, no JWT in JS-accessible storage).

## Grouped Bugs & Symptoms

- **Bug 1 (Prioridade Máxima):** Após criar novo usuário, não está indo para a tela de personagens; fica travado na tela de login em loop infinito (`/characters` ↔ `/login?redirect=/characters`).

## Root Cause Analysis (Causa Raiz)

1. **Active Cookie Deletion in `persistAuthSession`**:
   In `dungeoneer-frontend/lib/api/auth.ts` (lines 143–161), `persistAuthSession` actively deletes `dungeoneer_auth_token` and `dungeoneer_token` cookies (`deleteCookie(AUTH_TOKEN_KEY)`) and stores `data.accessToken` exclusively in a module-scoped variable `inMemoryToken`.
2. **Missing `credentials: 'include'` in `apiFetch`**:
   In `dungeoneer-frontend/lib/api/client.ts` (lines 168–174), `fetch` is invoked without `credentials: 'include'`. Because the frontend (`localhost:3000`) and API Gateway (`localhost:9090`) run on different origins/ports in local/Docker setups, the browser rejects the `Set-Cookie` header sent by `AuthController.java`.
3. **Next.js Edge Middleware Interception**:
   When `register-form.tsx` executes `router.push("/characters")`, `dungeoneer-frontend/middleware.ts` checks `request.cookies.get('dungeoneer_auth_token')`. Finding no cookie, it returns a `307 Temporary Redirect` to `/login?redirect=/characters`.
4. **Infinite SPA Bounce Loop**:
   When `/login` mounts, `login-form.tsx` evaluates `useAuth().isAuthenticated`. Because `inMemoryToken` in `auth.ts` is still populated in SPA memory, `isAuthenticated` is `true`, triggering `router.replace("/characters")`, which Middleware immediately redirects back to `/login` in an endless loop.
5. **Disconnected `getStoredToken()` in `client.ts`**:
   `client.ts` defines its own `getStoredToken()` that reads only from `localStorage` and `document.cookie`, ignoring `inMemoryToken` in `auth.ts`. Subsequent REST calls fail to attach `Authorization: Bearer <token>`, resulting in `401 Unauthorized`.

## Systemic Ripple Effects & Similar Latent Bugs (Repercussão no Código)

- **Page Refresh (F5) Session Loss**: Refreshing any protected route resets `inMemoryToken` in JS memory, and since the cookie was deleted by `persistAuthSession`, the user is immediately logged out.
- **Role Promotion (`toggleDmRole` / `selfPromoteToDm`)**: Invokes `persistAuthSession`, wiping cookies and breaking subsequent protected route transitions.
- **Login Flow (`login-form.tsx`)**: Suffers from the exact same redirect loop whenever `Set-Cookie` from `:9090` is not persisted for `:3000`.

## Legacy Code & Data Adaptation Strategy (Adequação de Legado)

### Legacy Impact Matrix
| Legacy Artifact / Component | Discrepancy Identified | Adaptation / Migration Action |
|-----------------------------|------------------------|-------------------------------|
| Browser Storage (`localStorage` / `document.cookie`) | Missing `HttpOnly` cookie because `fetch` lacked `credentials: 'include'`; `inMemoryToken` out of sync | Add `credentials: 'include'`, share `inMemoryToken` via `getAccessToken()`, never write the JWT from JS. Stale cached user without a valid cookie is cleared on `/login?redirect=`. |
| `auth.users` Database Table | Existing registered users | No schema or data migration needed; existing user accounts in PostgreSQL are valid. |

## Scope

### In Scope
- Unify token retrieval between `lib/api/auth.ts` and `lib/api/client.ts` so `Authorization: Bearer` is always attached when a token is present.
- Add `credentials: 'include'` to `apiFetch` in `lib/api/client.ts` and auth calls in `lib/api/auth.ts`.
- Rely on the backend `HttpOnly` `dungeoneer_auth_token` cookie (accepted via `credentials: 'include'`) so Next.js `middleware.ts` and page refreshes (`F5`) recognize the session; rehydrate SPA state via `GET /api/auth/me`. No JWT in `localStorage`/`document.cookie` (SEC-HIGH-02).
- Guard `login-form.tsx` and `register-form.tsx` redirect effects against unverified cookie state to prevent infinite bounce loops if a token is expired or rejected by middleware.

### Out of Scope
- OAuth2 social login providers.
- Backend JWT signing algorithm changes.

## Proposed Solution

### Technical Approach
> **Security constraint (SEC-HIGH-02):** the JWT MUST NOT be written to `localStorage` or `document.cookie` by JavaScript. `dungeoneer_auth_token` is set exclusively by `auth-service` as an `HttpOnly; SameSite=Lax` cookie.

1. **`credentials: 'include'` as primary mechanism (`lib/api/client.ts`)**:
   - Add `credentials: 'include'` to `apiFetch` (and auth calls in `auth.ts`) so the browser accepts and stores the backend's `HttpOnly` cookie across `:9090` → `:3000`, and sends it on later requests (the `api-gateway` `JwtHeaderFilter` already reads this cookie).
2. **In-memory token unification (`lib/api/auth.ts` & `lib/api/client.ts`)**:
   - Export `getAccessToken()` from `auth.ts` returning `inMemoryToken` only.
   - `client.ts` `getStoredToken()` delegates to `getAccessToken()` and attaches `Authorization: Bearer` when present. `persistAuthSession` keeps NOT touching `localStorage`/`document.cookie` with the JWT.
   - `clearAuthSession()` clears `inMemoryToken` and the cached user, and session termination calls `POST /api/auth/logout` (the `HttpOnly` cookie cannot be deleted from JS).
3. **Session rehydration on refresh (`lib/hooks/use-auth.ts`)**:
   - On mount with null `inMemoryToken` and a cached user, call `GET /api/auth/me` with `credentials: 'include'`; the gateway validates the `HttpOnly` cookie and returns the profile.
4. **Loop prevention (`login-form.tsx`, `register-form.tsx`, `middleware.ts`)**:
   - `login-form.tsx`: when mounted with `?redirect=` (middleware bounced an unauthenticated request), do NOT auto `router.replace` from stale SPA state; clear stale `inMemoryToken`/user and wait for explicit login.
   - `middleware.ts`: when redirecting to `/login`, clear malformed/expired `dungeoneer_auth_token` cookie on the response.
   - `register-form.tsx`: await successful `register()` and confirmed session before `router.push("/characters")`.

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `dungeoneer-frontend/lib/api/auth.ts` | Modify | Export `getAccessToken()`; add `credentials: 'include'`; no JS-written cookie |
| `dungeoneer-frontend/lib/hooks/use-auth.ts` | Modify | Rehydrate session via `GET /api/auth/me` on refresh |
| `dungeoneer-frontend/lib/api/client.ts` | Modify | Use unified token getter from `auth.ts` and add `credentials: 'include'` |
| `dungeoneer-frontend/components/register-form.tsx` | Modify | Ensure auth state & cookie are committed before `router.push("/characters")` |
| `dungeoneer-frontend/components/login-form.tsx` | Modify | Respect `redirect` param and avoid blind loop if cookie is missing |
| `dungeoneer-frontend/middleware.ts` | Modify | Clear invalid cookies on redirect to `/login` to prevent client/edge state mismatch |

### Data Model & API Changes (if applicable)
- None (Frontend session & Edge middleware synchronization only).

## Acceptance Criteria

- [ ] **Given** a visitor on `/register` **When** they submit valid registration details **Then** the auth session and `dungeoneer_auth_token` cookie are persisted, and the browser navigates directly to `/characters` without looping back to `/login`.
- [ ] **Given** an authenticated user on `/characters` **When** they press F5 (page refresh) **Then** the session remains active and `/api/characters` requests include the `Authorization: Bearer` header.
- [ ] **Given** a user whose session cookie has expired or been cleared **When** Next.js middleware redirects them to `/login` **Then** `login-form.tsx` does not auto-redirect back to `/characters` until a new login succeeds.

## Defensive Security & Validation Considerations

### Input Validation
- Registration form continues validating email, username, and password constraints via Zod and backend Jakarta Validation.

### Secret Isolation
- JWT secrets remain strictly in backend environment variables (`JWT_SECRET`).

### Authorization & Authentication Impact
- Fixes broken authentication state persistence between SPA memory, browser cookies, and Next.js Edge Middleware. Ensures `clearAuthSession()` wipes all storage layers on logout or 401 expiration.

### Other Security Risks
- Cookie remains `HttpOnly; SameSite=Lax` set by `auth-service` only (mitigates XSS exfiltration and CSRF); JS never reads or writes it.

## Testing Strategy & Shift-Left Plan

### Regression & Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TR-01 | Register a new user via `RegisterForm` | Unit / Vitest | `getAccessToken()` returns the JWT, `document.cookie`/`localStorage` never contain it, router navigates to `/characters` once |
| TR-02 | Subsequent `apiFetch('/api/characters')` after register/login | Unit / Vitest | Request headers contain `Authorization: Bearer <token>` and `credentials: 'include'` |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Expired token in cookie when hitting `/characters` | Unit / Middleware | Middleware redirects to `/login` and clears stale cookie; `LoginForm` stays on `/login` without looping |

### Test Commands
- **Frontend tests:** `npm test`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Stale `inMemoryToken` surviving after cookie deletion | Low | High | Synchronize `getAccessToken()` with cookie verification and clear `inMemoryToken` whenever `/login` is reached via unauthenticated redirect |

## Specification Changes & Delta Stubs

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Document unified JWT session storage between client memory, `dungeoneer_auth_token` cookie, `apiFetch` credentials policy, and Next.js Edge Middleware route protection.
