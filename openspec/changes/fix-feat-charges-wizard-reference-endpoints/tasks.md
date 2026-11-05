# Tasks: Fix Feat Charges, Wizard Origin Filter, Reference Endpoints & Full Security Hardening

> **Change ID:** `fix-feat-charges-wizard-reference-endpoints`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

### Phase 1: Feat Charges on Rest & Wizard Origin Filter
- [x] **[P1-T1]** Update `FeatChargeCalculator`, `CharacterFeat`, and `RestService` in `player-character-service` to recalculate missing `maxCharges` on legacy feats and apply ruleset-aware short/long rest recovery (e.g., Rage recovering 1 charge on short rest in `SRD_2024` and all charges on long rest).
- [x] **[P1-T2]** Create Flyway migration `V28__adapt_legacy_feat_charges_and_recovery.sql` in `player-character-service` to backfill legacy feat charge formulas and recovery metadata.
- [x] **[P1-T3]** Update `dungeoneer-frontend/app/characters/new/page.tsx` to request `featTypes: ["ORIGIN"]` and strictly filter `f.featType === "ORIGIN"` (removing the `!f.featType` fallback).

### Phase 2: RSA-OAEP Client-Side Password Encryption & HttpOnly Cookie Session (`auth-service`, `api-gateway`, `dungeoneer-frontend`)
- [ ] **[P2-T1]** Create `PasswordCipherPort` and `RsaOaepPasswordCipherAdapter` in `auth-service` (`RSA/ECB/OAEPWithSHA-256AndMGF1Padding`, 2048-bit key pair) and decrypt encrypted password payloads in `AuthService` before validation/BCrypt hashing.
- [ ] **[P2-T2]** Update `AuthController` in `auth-service` to expose `GET /api/auth/public-key`, set `HttpOnly; SameSite=Lax` (and configurable `Secure`) cookie `dungeoneer_auth_token` on `/register` and `/login`, add `POST /api/auth/logout` to clear the cookie, and enforce authentication/ownership on `POST /api/auth/users/{userId}/roles` (`[SEC-CRIT-01]`).
- [ ] **[P2-T3]** Update `JwtHeaderFilter` in `api-gateway` to extract JWT from the `dungeoneer_auth_token` cookie (in addition to `Authorization: Bearer`), protect `/api/auth/users/**`, inject `X-Internal-Secret` on downstream requests, apply IP rate limiting on `/api/auth/login` and `/api/auth/register` (`[SEC-LOW-03]`), and attach HTTP security headers (`[SEC-MED-03]`).
- [ ] **[P2-T4]** Update `dungeoneer-frontend/lib/api/auth.ts` and `lib/api/client.ts` to fetch `/api/auth/public-key`, encrypt passwords using Web Crypto API (`RSA-OAEP` with `SHA-256`) on `login()` and `register()`, remove JWT persistence from `localStorage` and `document.cookie` (`[SEC-HIGH-02]`), use `credentials: 'include'`, and call `POST /api/auth/logout` on logout.
- [ ] **[P2-T5]** Update `dungeoneer-frontend/middleware.ts` to validate JWT structure and `exp` claim from the `dungeoneer_auth_token` cookie and extend route protection matcher to `/characters/:path*` and `/profile/:path*` (`[SEC-HIGH-03]`), and add HTTP security headers in `dungeoneer-frontend/next.config.ts` (`[SEC-MED-03]`).

### Phase 3: Downstream Perimeter Protection, Homebrew IDOR Fix & Infrastructure Hardening (`player-character-service`, `auth-service`, `docker-compose.yml`)
- [ ] **[P3-T1]** Add `InternalGatewaySecretFilter` and update `SecurityConfig` in both `player-character-service` and `auth-service` to verify `X-Internal-Secret` (`[SEC-CRIT-02]`), restrict `/api/auth/**` endpoints properly, and remove redundant downstream CORS configurations (`[SEC-LOW-02]`).
- [ ] **[P3-T2]** Update `ReferenceDataController` (`required = true` for `X-User-Id` on all `PUT` endpoints) and `HomebrewService` in `player-character-service` to verify `existing.getOwnerId().equals(userId)` on all homebrew update operations, throwing `UnauthorizedActionException` mapped to `403 Forbidden` (`[SEC-HIGH-01]`).
- [ ] **[P3-T3]** Sanitize `GlobalExceptionHandler.handleGenericException` in `auth-service` and `player-character-service` so HTTP 500 responses return a generic safe message instead of raw `ex.getMessage()` (`[SEC-MED-02]`).
- [ ] **[P3-T4]** Update `docker-compose.yml` and service configuration files (`application.properties` / `application.yml`) to remove public port bindings `8080:8080` and `8082:8080` (`[SEC-CRIT-02]`), inject `INTERNAL_GATEWAY_SECRET`, and make Swagger UI configurable (`SWAGGER_ENABLED: false` in prod) (`[SEC-MED-01]`, `[SEC-LOW-01]`).

## Test Tasks

- [x] **[T-01]** Write/update unit tests in `RestServiceTest` and `FeatChargeCalculatorTest` for short/long rest charge recovery and legacy feats with null `maxCharges`.
- [ ] **[T-02]** Write/update unit and MockMvc tests in `auth-service` (`RsaOaepPasswordCipherAdapterTest`, `AuthServiceTest`, `AuthControllerTest`, `GlobalExceptionHandlerTest`) covering `RSA-OAEP` encryption/decryption, `HttpOnly` cookie emission, `/api/auth/logout`, `/api/auth/users/{userId}/roles` access control (`401`/`403`), `X-Internal-Secret` validation, and sanitized 500 errors.
- [ ] **[T-03]** Write/update unit tests in `api-gateway` (`JwtHeaderFilterTest`) covering `HttpOnly` cookie authentication, `/api/auth/users/**` protection, `X-Internal-Secret` header propagation, security headers, and rate limiting (`429 Too Many Requests`).
- [ ] **[T-04]** Write/update unit and MockMvc tests in `player-character-service` (`HomebrewServiceTest`, `ReferenceDataControllerTest`, `GlobalExceptionHandlerTest`) covering Homebrew ownership verification (`403 Forbidden` on IDOR attempt), `X-Internal-Secret` enforcement, and sanitized 500 errors.
- [ ] **[T-05]** Write/update frontend unit tests in `dungeoneer-frontend` (`character-wizard.test.tsx`, `auth.test.ts`, `middleware.test.ts`) verifying strict `ORIGIN` feat filtering, `RSA-OAEP` password encryption, absence of JWT in `localStorage`, and middleware JWT expiration checking on `/characters` and `/profile`.
- [ ] **[T-06]** Run full test suites and coverage checks across all modules (`mvn clean verify` on backend services and `npx vitest run` on frontend).

## Documentation Tasks

- [x] **[D-01]** Create and update delta specifications under `openspec/changes/fix-feat-charges-wizard-reference-endpoints/specs/` (`auth-domain.md`, `system-architecture-delta.md`, `player-character-domain-delta.md`, `frontend-web-app-delta.md`).

## Review Gate

- [ ] All implementation tasks complete
- [ ] All tests passing
- [ ] Proposal acceptance criteria verified
- [ ] Ready for `sdd-spec-archive`
