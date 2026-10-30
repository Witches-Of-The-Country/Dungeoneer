# Proposal: Gateway Routing And Cors

> **Change ID:** `gateway-routing-and-cors`
> **Status:** draft
> **Author:** @jaca
> **Created:** 2026-10-29
> **Last Updated:** 2026-10-29

---

## Summary

Configure `api-gateway` (`:9090`) as the unified entry point for `dungeoneer-frontend` (`:3000`). Define Spring Cloud Gateway Server WebMVC routes for `/api/auth/**` (`auth-service:8082`), `/api/reference/**` (`player-character-service:8080`), and `/api/characters/**` (`player-character-service:8080`), implement a JWT authentication & header-propagation filter that validates HMAC-SHA256 Bearer tokens (`${JWT_SECRET}`), strips any client-supplied `X-User-Id` header (preventing spoofing), and injects the verified `X-User-Id` (`sub` claim) on protected routes, configure centralized CORS at the Gateway level only, and align `docker-compose.yml` so `NEXT_PUBLIC_API_URL=http://localhost:9090`.

## Motivation

Currently, `api-gateway` has no routes, no JWT verification, and no CORS configuration in `application.properties`, and `docker-compose.yml` points `dungeoneer-frontend` directly to `http://localhost:8080`. Centralizing routing, CORS, and JWT verification in `api-gateway` satisfies `openspec/specs/system-architecture.md` (`[RF-03]`, `[RNF-02]`) and ensures `player-character-service` receives a trusted `X-User-Id` header.

## Scope

### In Scope
- Configure routes in `api-gateway`:
  - `/api/auth/**` → `${AUTH_SERVICE_URL:http://localhost:8082}` (public)
  - `/api/reference/**` → `${CHARACTER_SERVICE_URL:http://localhost:8080}` (public `GET` or authenticated)
  - `/api/characters/**` → `${CHARACTER_SERVICE_URL:http://localhost:8080}` (protected: requires valid `Authorization: Bearer <jwt>`)
- Implement JWT validation & `X-User-Id` propagation filter in `api-gateway`:
  - Always strip any incoming `X-User-Id` header from external client requests before processing.
  - On `/api/characters/**`, validate the Bearer JWT signature and expiration using `${JWT_SECRET}` (HMAC-SHA256).
  - If missing, expired, or invalid, short-circuit with `401 Unauthorized` RFC 7807 `ProblemDetail`.
  - If valid, extract `sub` (UUID) and mutate the downstream request to include `X-User-Id: <sub-uuid>`.
- Configure centralized CORS in `api-gateway` (`allowedOrigins = ${CORS_ALLOWED_ORIGINS:http://localhost:3000}`, methods `GET, POST, PUT, DELETE, OPTIONS`, headers `Authorization, Content-Type`, `allowCredentials = true`).
- Update `docker-compose.yml` and `.env.example`:
  - Add `JWT_SECRET` environment variable to `auth-service` and `api-gateway`.
  - Pass `AUTH_SERVICE_URL=http://auth-service:8080` and `CHARACTER_SERVICE_URL=http://player-character-service:8080` to `api-gateway`.
  - Set `NEXT_PUBLIC_API_URL=http://localhost:9090` on `dungeoneer-frontend`.

### Out of Scope
- Rate limiting with Redis (deferred to production hardening).
- Campaign service routing (`/api/campaigns/**` — deferred to Phase 2).

## Proposed Solution

### Approach
Using Spring Cloud Gateway Server WebMVC (`RouterFunctions` / filter chain in `api-gateway`), we will define a `GatewayRoutesConfig` and a `JwtAuthenticationFilter`. The filter inspects request paths: for `/api/characters/**` (excluding `OPTIONS` preflight requests), it verifies the HMAC-SHA256 JWT, strips any spoofed `X-User-Id` header on all routes, and injects the verified `X-User-Id` header before forwarding to `player-character-service`.

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `api-gateway/backend/pom.xml` | Modify | Add `spring-boot-starter-oauth2-resource-server` (or Nimbus JOSE JWT) and `jacoco-maven-plugin` |
| `api-gateway/backend/src/main/resources/application.properties` | Modify | Add service URLs, `jwt.secret=${JWT_SECRET}`, and `cors.allowed-origins` |
| `api-gateway/backend/src/main/java/com/dungeoneer/gateway/config/CorsConfig.java` | Create | Centralized CORS configuration for `:3000` |
| `api-gateway/backend/src/main/java/com/dungeoneer/gateway/config/GatewayRoutesConfig.java` | Create | Route definitions for `/api/auth/**`, `/api/reference/**`, and `/api/characters/**` |
| `api-gateway/backend/src/main/java/com/dungeoneer/gateway/security/JwtHeaderFilter.java` | Create | Strips incoming `X-User-Id`, validates JWT on protected routes, injects verified `X-User-Id` |
| `docker-compose.yml` | Modify | Inject `JWT_SECRET`, service URLs, and update `NEXT_PUBLIC_API_URL=http://localhost:9090` |

### Data Model Changes (if applicable)
N/A — stateless API Gateway configuration.

## Acceptance Criteria

- [ ] **Given** a request to `/api/characters` without an `Authorization: Bearer <token>` header (even if a spoofed `X-User-Id` header is sent) **When** received by `api-gateway` **Then** the request is rejected with `401 Unauthorized` ProblemDetail and never reaches `player-character-service`.
- [ ] **Given** a request to `/api/characters` with a valid JWT (`sub = <user-uuid>`) and a spoofed `X-User-Id: <other-uuid>` **When** routed through `api-gateway` **Then** the spoofed header is overwritten with `X-User-Id: <user-uuid>`.
- [ ] **Given** a browser preflight `OPTIONS` request from `http://localhost:3000` to `http://localhost:9090/api/characters` **When** received by `api-gateway` **Then** `200 OK` is returned with `Access-Control-Allow-Origin: http://localhost:3000`.
- [ ] **Given** a request to `POST /api/auth/login` or `GET /api/reference/classes` without a token **When** received by `api-gateway` **Then** it is forwarded to `auth-service` or `player-character-service` respectively.

## Defensive Security & Validation Considerations

### Input Validation
- Validates `Authorization` header format (`Bearer <token>`), JWT signature, `exp` timestamp, and `sub` UUID format.

### Secret Isolation
- `JWT_SECRET` is injected strictly via environment variables (`${JWT_SECRET}`).

### Authorization & Authentication Impact
- Eliminates header-spoofing vulnerabilities by unconditionally stripping client-provided `X-User-Id` headers at the perimeter before JWT verification.

### Other Security Risks
- Restricts CORS `Access-Control-Allow-Origin` to configured origins (`${CORS_ALLOWED_ORIGINS:http://localhost:3000}`), never `*` with credentials.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Valid JWT on `/api/characters` forwards request with `X-User-Id` equal to JWT `sub` | Unit / MockMvc | Downstream receives verified `X-User-Id` |
| TP-02 | Public routes `/api/auth/login` and `/api/reference/classes` pass through without JWT | Unit / MockMvc | Routed to downstream service |
| TP-03 | CORS preflight `OPTIONS /api/characters` from `http://localhost:3000` | API (MockMvc) | Returns `Access-Control-Allow-Origin: http://localhost:3000` |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Missing, expired, or bad-signature JWT on `/api/characters` | API (MockMvc) | Returns `401 Unauthorized` RFC 7807 `ProblemDetail` |
| TE-02 | Client sends spoofed `X-User-Id` header with or without JWT | API (MockMvc) | Spoofed `X-User-Id` is stripped/overwritten |

### Test Commands
- **Run tests:** `mvn -f api-gateway/backend/pom.xml clean test`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Docker internal hostname vs local dev localhost mismatch | Low | Medium | Parameterize downstream URLs via `${AUTH_SERVICE_URL:http://localhost:8082}` and `${CHARACTER_SERVICE_URL:http://localhost:8080}` |

## Specification Changes & Delta Stubs

### `specs/infrastructure-and-testing-delta.md`
- **Target Canonical Spec:** `openspec/specs/infrastructure-and-testing.md`
- **Summary of Updates:** Document `JWT_SECRET`, `AUTH_SERVICE_URL`, and `CHARACTER_SERVICE_URL` environment variables in `docker-compose.yml`.

## References

- `openspec/specs/system-architecture.md`
- `openspec/specs/infrastructure-and-testing.md`
