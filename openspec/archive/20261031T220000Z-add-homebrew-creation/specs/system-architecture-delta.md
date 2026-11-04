# Delta Spec: `system-architecture`

> **Change ID:** `add-homebrew-creation`
> **Target Canonical Spec:** `openspec/specs/system-architecture.md`
> **Change Type:** Modify

## Architectural Updates

### 1. API Gateway Route Protection

The API Gateway (`api-gateway`) enforces role-based access control for mutating reference data:

- **Route:** `POST /api/reference/**`
- **Security Rule:** Requires valid JWT authentication and role `ROLE_DM`.
- **Enforcement Component:** `com.dungeoneer.api_gateway.security.JwtHeaderFilter`.
- **Behavior:**
  - If JWT is missing or invalid: responds with HTTP 401 Unauthorized.
  - If JWT is valid but lacks `ROLE_DM`: responds with HTTP 403 Forbidden.
  - If authorized: strips any client-provided `X-User-Id` and `X-User-Roles` headers, extracts user UUID and roles from verified JWT claims, and injects verified `X-User-Id` and `X-User-Roles` headers downstream.

### 2. Header Propagation for Read Queries

- **Route:** `GET /api/reference/**`
- **Security Rule:** Publicly accessible.
- **Behavior:**
  - If a valid JWT is present in `Authorization: Bearer <token>`, the gateway forwards the verified `X-User-Id` downstream, allowing microservices to hydrate personalized homebrew content alongside public SRD data.
  - If no JWT is present, no `X-User-Id` header is forwarded, signaling downstream services to return only public SRD records.
