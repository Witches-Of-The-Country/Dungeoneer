# Delta Spec: `system-architecture`

> **Change ID:** `fix-feat-charges-wizard-reference-endpoints`
> **Target Canonical Spec:** `openspec/specs/system-architecture.md`
> **Change Type:** Modify

## Architectural Updates

### MODIFIED: `[RNF-02] Security Posture & API Gateway Perimeter`
- **Network Segmentation:** Downstream microservices (`player-character-service`, `auth-service`) MUST NOT publish host ports (`8080`, `8082`) in `docker-compose.yml`; they SHALL use internal container network exposure (`expose: ["8080"]`) accessible only via `api-gateway` (`9090`).
- **Service-to-Service Perimeter Authentication (`X-Internal-Secret`):** `api-gateway` SHALL inject a shared secret header `X-Internal-Secret` (configured via `${INTERNAL_GATEWAY_SECRET}`) on all forwarded requests. Downstream microservices SHALL validate `X-Internal-Secret` via a security filter and reject requests missing or mismatching the secret with `403 Forbidden`.
- **HttpOnly Cookie Session Extraction:** `JwtHeaderFilter` in `api-gateway` SHALL extract the JWT from the `dungeoneer_auth_token` `HttpOnly` cookie (falling back to `Authorization: Bearer` for API clients), validate its signature and expiration, and propagate `X-User-Id` downstream.
- **Protected Auth Routes:** `JwtHeaderFilter` SHALL require authentication for `/api/auth/me`, `/api/auth/profile`, and `/api/auth/users/**`, keeping only `/api/auth/login`, `/api/auth/register`, `/api/auth/public-key`, and `/api/auth/logout` public.
- **Centralized CORS & Security Headers:** CORS SHALL be configured exclusively at `api-gateway` (downstream CORS beans removed). Gateway responses SHALL include `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, and `Content-Security-Policy`.
- **Swagger UI Restriction:** OpenAPI/Swagger UI endpoints SHALL be disabled when `SWAGGER_ENABLED=false` (production profile).
