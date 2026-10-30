# Delta Spec: `system-architecture`

> **Change ID:** `auth-jwt-backend`
> **Target Canonical Spec:** `openspec/specs/system-architecture.md`
> **Change Type:** Modify

## Architectural Updates

1. **API Gateway Routing (`[RF-03]`)**:
   - Add `/api/auth/**` → `auth-service:8082` (public endpoints: `POST /api/auth/register`, `POST /api/auth/login`).
2. **Security Posture (`[RNF-02]`)**:
   - Replace MVP unauthenticated mode with JWT authentication:
   - `auth-service` authenticates users with BCrypt password hashing and issues HMAC-SHA256 signed JWTs (`sub` = user UUID) using `${JWT_SECRET}`.
   - `api-gateway` validates Bearer JWTs on protected routes (`/api/characters/**`) and forwards the verified `X-User-Id` header to downstream microservices.
   - `player-character-service` requires `X-User-Id` on all character endpoints and enforces character ownership (`user_id NOT NULL`).
