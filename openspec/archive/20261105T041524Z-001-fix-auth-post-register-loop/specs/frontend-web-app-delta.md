# Delta Spec: `frontend-web-app`

> **Change ID:** `001-fix-auth-post-register-loop`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### Authentication Session & Middleware Synchronization
- **HttpOnly session**: `dungeoneer_auth_token` is set only by `auth-service` (`HttpOnly; SameSite=Lax`). JavaScript never writes the JWT to `localStorage` or `document.cookie`.
- **Credentials**: all `apiFetch` and auth calls use `credentials: 'include'`; `getAccessToken()` (in-memory) is the single token source for `Authorization: Bearer`.
- **Rehydration**: on refresh, `useAuth` restores the session via `GET /api/auth/me`.
- **Loop-free guard**: `middleware.ts` clears invalid cookies when redirecting to `/login?redirect=`; `LoginForm` ignores stale in-memory auth when `?redirect=` is present.
