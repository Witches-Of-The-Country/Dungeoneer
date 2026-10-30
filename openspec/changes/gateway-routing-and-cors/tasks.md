# Tasks: Gateway Routing And Cors

> **Change ID:** `gateway-routing-and-cors`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [ ] Update `api-gateway/backend/pom.xml` with JWT verification support (`spring-boot-starter-oauth2-resource-server` / Nimbus JOSE)
- [ ] Configure `api-gateway/backend/src/main/resources/application.properties` with route URLs, `jwt.secret=${JWT_SECRET}`, and `cors.allowed-origins`
- [ ] Implement `CorsConfig` in `api-gateway` allowing `http://localhost:3000`
- [ ] Implement `JwtHeaderFilter` that strips incoming `X-User-Id`, validates Bearer JWT on `/api/characters/**`, and injects verified `X-User-Id`
- [ ] Implement `GatewayRoutesConfig` routing `/api/auth/**`, `/api/reference/**`, and `/api/characters/**`
- [ ] Update `docker-compose.yml` with `JWT_SECRET`, gateway service URLs, and `NEXT_PUBLIC_API_URL=http://localhost:9090`

## Security Validation Tasks

- [ ] Verify client-supplied `X-User-Id` headers are unconditionally stripped before routing
- [ ] Confirm `JWT_SECRET` is injected via environment variable (`${JWT_SECRET}`)
- [ ] Verify `401 Unauthorized` RFC 7807 `ProblemDetail` is returned for missing/expired/invalid tokens on `/api/characters/**`

## Shift-Left Test Suite Tasks

### Happy Path
- [ ] Write MockMvc test verifying valid JWT on `/api/characters` injects `X-User-Id` (TP-01)
- [ ] Write MockMvc test verifying public routes `/api/auth/**` and `/api/reference/**` pass through (TP-02)
- [ ] Write MockMvc test verifying CORS headers for `http://localhost:3000` (TP-03)

### Edge Cases & Failure Modes
- [ ] Write MockMvc tests for missing, expired, and tampered JWT returning `401 Unauthorized` ProblemDetail (TE-01)
- [ ] Write MockMvc test proving a spoofed `X-User-Id` header cannot bypass auth or override JWT `sub` (TE-02)

## Documentation Tasks

- [ ] Update `specs/infrastructure-and-testing-delta.md` with Gateway environment variables

## Review Gate

- [ ] All implementation tasks complete
- [ ] All security validation tasks complete
- [ ] All tests passing (`mvn -f api-gateway/backend/pom.xml clean test`)
- [ ] Proposal acceptance criteria verified (run `sdd-spec-validate`)
- [ ] Ready for `sdd-spec-archive`
