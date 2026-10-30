# Delta Spec: `infrastructure-and-testing`

> **Change ID:** `gateway-routing-and-cors`
> **Target Canonical Spec:** `openspec/specs/infrastructure-and-testing.md`
> **Change Type:** Modify

## Architectural Updates

1. **Environment Variables (`[RF-02]`)**:
   - `JWT_SECRET`: Shared HMAC-SHA256 secret (≥ 32 bytes) injected into `auth-service` and `api-gateway` (never hardcoded).
   - `AUTH_SERVICE_URL`: Internal URL for `auth-service` in `api-gateway` (defaults to `http://auth-service:8080` in Docker, `http://localhost:8082` in local dev).
   - `CHARACTER_SERVICE_URL`: Internal URL for `player-character-service` in `api-gateway` (defaults to `http://player-character-service:8080` in Docker, `http://localhost:8080` in local dev).
   - `NEXT_PUBLIC_API_URL`: Set to `http://localhost:9090` so `dungeoneer-frontend` communicates exclusively through `api-gateway`.
