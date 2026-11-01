# Proposal: Fix Homebrew Creation CORS and Dungeon Master Role Assignment

> **Change ID:** `fix-homebrew-creation-cors-and-dm-role`  
> **Status:** Draft  
> **Created:** 2026-10-31  

---

## 1. Summary

When users attempt to create Homebrew reference records (such as feats or lineages) from the frontend (`http://localhost:3000`), the browser blocks the request with a CORS error:
`Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at http://localhost:9090/api/reference/feats. (Reason: CORS header ‘Access-Control-Allow-Origin’ missing). Status code: 403.`

The root cause is twofold:
1. `JwtHeaderFilter` in `api-gateway` enforces that mutation routes on `/api/reference/**` require the `DUNGEON_MASTER` role. However, `auth-service` assigns only `Role.PLAYER` upon registration, leaving no supported way for a user to acquire or be assigned the `DUNGEON_MASTER` role.
2. When `JwtHeaderFilter` rejects a request (401 Unauthorized or 403 Forbidden), it writes the `ProblemDetail` directly to `HttpServletResponse` without injecting CORS headers (`Access-Control-Allow-Origin`, `Access-Control-Allow-Credentials`, etc.), which triggers browser CORS blocking and masks the underlying RBAC error.

---

## 2. Motivation

Users cannot create custom feats, lineages, or homebrew content, which is a core feature of the Dungeoneer application. Furthermore, because CORS headers are omitted on 401/403 rejections, the frontend cannot inspect the `ProblemDetail` error payload and presents a confusing generic network failure instead of explaining that the user lacks the `DUNGEON_MASTER` role. Resolving this restores homebrew creation and provides proper error feedback and robust role management.

---

## 3. Scope

### In Scope
- Fixing `JwtHeaderFilter` in `api-gateway` to write verified CORS headers on short-circuited 401/403 responses.
- Ensuring `CorsConfig` in `api-gateway` supports both `http://localhost:3000` and `http://127.0.0.1:3000`.
- Providing role management/promotion in `auth-service` allowing users to be granted `Role.DUNGEON_MASTER` (e.g., via administrative/development assignment or seed profile).
- Verifying frontend client error handling for HTTP 403 responses so that the server-provided `detail` is surfaced to the user.

### Out of Scope
- Full multi-tenant RBAC administration UI / dashboard.
- Redesign of the reference data persistence schema in `player-character-service`.
- Changing authorization rules for standard campaign or character creation endpoints.

---

## 4. Proposed Solution

1. **API Gateway CORS in Error Responses**:
   - In `JwtHeaderFilter`, introduce an error-writing helper that inspects the incoming `Origin` header. If it matches the configured allowed origins (`http://localhost:3000` or `http://127.0.0.1:3000`), it dynamically writes `Access-Control-Allow-Origin`, `Access-Control-Allow-Credentials: true`, and `Vary: Origin` before flushing the `ProblemDetail` JSON.
   - Configure `CorsFilter` with explicit priority via `FilterRegistrationBean` to ensure consistent preflight and normal request handling.

2. **Dungeon Master Role Assignment**:
   - In `auth-service`, introduce domain and application logic to grant `Role.DUNGEON_MASTER` to a user.
   - Expose a secure controller endpoint (with Jakarta Validation on request DTO) to update user roles.
   - Ensure newly minted or refreshed JWT tokens reflect the updated roles array (`"roles": ["PLAYER", "DUNGEON_MASTER"]`).

3. **Frontend Feedback**:
   - Ensure the dialog in `dungeoneer-frontend` displays the specific `ProblemDetail.detail` message if a 403 status is returned.

---

## 5. Defensive Security & Validation Considerations

- **CORS Origin Validation**: Origins must be strictly validated against configured allow-lists (`http://localhost:3000`, `http://127.0.0.1:3000`). Never reflect arbitrary wildcard `*` origins when credentials (`Access-Control-Allow-Credentials: true`) are enabled.
- **Input Validation**: Role update requests in `auth-service` must use Jakarta Validation (`@NotNull`, `@NotEmpty`, valid enum values) to prevent malformed or invalid role assignments.
- **Privilege Escalation Protection**: Endpoints modifying user roles must be protected by authentication and authorization checks (or scoped strictly to local/development configurations).
- **Secret Isolation**: JWT signing keys remain isolated within configuration properties and injected via environment variables.

---

## 6. Testing Strategy & Shift-Left Plan

- **API Gateway (Unit & Integration)**:
  - `JwtHeaderFilterTest`: Verify that requests to `/api/reference/feats` without auth return 401 with `Access-Control-Allow-Origin: http://localhost:3000`.
  - `JwtHeaderFilterTest`: Verify that authenticated requests with role `PLAYER` only return 403 with `Access-Control-Allow-Origin: http://localhost:3000` and `ProblemDetail` body.
  - `JwtHeaderFilterTest`: Verify that an unapproved origin (e.g., `http://malicious-site.com`) does not receive `Access-Control-Allow-Origin`.
- **Auth Service (Unit & Integration)**:
  - Unit test `User` entity adding role `Role.DUNGEON_MASTER`.
  - MockMvc test on role update endpoint verifying 200 OK for valid inputs and 400 Bad Request for invalid role payloads.
- **End-to-End Flow**:
  - Authenticated user with `DUNGEON_MASTER` role posts to `/api/reference/feats` and succeeds with 201 Created.

---

## 7. Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| Insecure CORS wildcard with credentials | High | Low | Validate `Origin` strictly against known whitelist before writing headers. |
| Privilege escalation via public role endpoint | High | Medium | Guard role update endpoint with authentication and restrict to administrative/dev contexts. |
| Stale token on client after role promotion | Medium | High | Require re-authentication / token refresh after role update so new JWT contains `DUNGEON_MASTER`. |

---

## 8. Impacted Files

| File | Reason |
|---|---|
| [`api-gateway/.../security/JwtHeaderFilter.java`](file:///D:/Projects/Dungeoneer/api-gateway/backend/src/main/java/com/dungeoneer/api_gateway/security/JwtHeaderFilter.java) | Add CORS response headers on 401/403 errors before response is flushed. |
| [`api-gateway/.../config/CorsConfig.java`](file:///D:/Projects/Dungeoneer/api-gateway/backend/src/main/java/com/dungeoneer/api_gateway/config/CorsConfig.java) | Ensure explicit filter registration order and allowed origins coverage. |
| [`auth-service/.../domain/model/User.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/domain/model/User.java) | Add role addition/management capabilities for `Role.DUNGEON_MASTER`. |
| [`auth-service/.../adapter/in/web/controller/UserController.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/controller/UserController.java) | Endpoint to manage/promote user roles with Jakarta Validation. |
| [`dungeoneer-frontend/lib/api/homebrew.ts`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/lib/api/homebrew.ts) | Verify error handling display for 403 Forbidden with proper problem details. |

---

## 9. Acceptance Criteria (Given-When-Then)

### AC 1: Gateway Emits CORS Headers on Error
- **Given** an unauthenticated or unauthorized request originating from `http://localhost:3000`
- **When** `POST /api/reference/feats` is received by `api-gateway`
- **Then** the response must return HTTP 401 or 403 AND contain `Access-Control-Allow-Origin: http://localhost:3000` so that the browser allows reading the `ProblemDetail` payload.

### AC 2: Dungeon Master Role Allows Homebrew Creation
- **Given** an authenticated user who has the `DUNGEON_MASTER` role
- **When** the user sends `POST /api/reference/feats` or `POST /api/reference/lineages` with a valid payload
- **Then** `api-gateway` must forward the request with `X-User-Id` and `X-User-Roles` to `player-character-service`, returning HTTP 201 Created.

### AC 3: Player Role Is Blocked with Clear 403 Feedback
- **Given** an authenticated user with only `PLAYER` role
- **When** the user sends `POST /api/reference/feats`
- **Then** `api-gateway` returns HTTP 403 with a readable JSON `ProblemDetail` explaining that the `DUNGEON_MASTER` role is required, without CORS blocking.
