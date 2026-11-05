---
id: "auth-domain"
title: "Authentication & Identity Domain — Living Specification"
version: "1.0.0"
status: "draft"
owners:
  - "@jaca"
scope:
  paths:
    - "auth-service/**"
    - "api-gateway/**"
resources: []
---

# Authentication & Identity Domain — Living Specification

> This is the authoritative source of truth for the `auth-service` module's functional and security state.
> Update this file only after merging a completed change from `openspec/changes/`.

---

## Context & Objectives

The `auth-service` governs user registration, credential authentication, public-key payload encryption (`RSA-OAEP`), session token issuance (`HttpOnly` cookies + JWT), profile management, and role assignment (`PLAYER`, `DUNGEON_MASTER`) across the Dungeoneer platform.

## Functional Requirements

- **[RF-AUTH-01] Public Key Discovery (`GET /api/auth/public-key`):** The service SHALL expose its RSA-2048 public key in SPKI Base64 format (`algorithm: "RSA-OAEP"`, `hash: "SHA-256"`) so clients can encrypt password fields prior to transmission.
- **[RF-AUTH-02] Encrypted Registration & Login (`POST /api/auth/register`, `POST /api/auth/login`):** The service SHALL decrypt incoming `RSA-OAEP` encrypted password ciphertexts via `PasswordCipherPort` prior to validating password length constraints and computing/verifying the BCrypt (10 rounds) password hash.
- **[RF-AUTH-03] HttpOnly Session Cookie & Logout (`POST /api/auth/logout`):** Upon successful authentication or registration, the service SHALL emit a signed JWT (`HS256`, 24h TTL) inside an `HttpOnly; SameSite=Lax` (and `Secure` in production) cookie named `dungeoneer_auth_token`, and provide `POST /api/auth/logout` to expire the cookie immediately (`Max-Age=0`).
- **[RF-AUTH-04] Authenticated Profile & Role Management (`GET /api/auth/me`, `PUT /api/auth/profile`, `POST /api/auth/users/{userId}/roles`):** All user profile and role mutation endpoints SHALL require a valid authenticated caller (`X-User-Id` verified by the API Gateway) and enforce that a user can only modify their own roles/profile (`callerId.equals(userId)`), returning `401 Unauthorized` or `403 Forbidden` otherwise.

## Non-Functional Requirements

- **[RNF-AUTH-01] Perimeter Defense (`X-Internal-Secret`):** All requests reaching `auth-service` MUST carry a valid `X-Internal-Secret` header injected by `api-gateway`; direct unauthorized requests MUST be rejected with `403 Forbidden`.
- **[RNF-AUTH-02] Brute-Force Rate Limiting:** `/api/auth/login` and `/api/auth/register` SHALL be rate-limited at the API Gateway perimeter, returning `429 Too Many Requests` when exceeded.
- **[RNF-AUTH-03] Safe Error Handling:** Unhandled internal exceptions (`500 Internal Server Error`) MUST return a generic RFC 7807 `ProblemDetail` message without leaking SQL or stack details.

## Out of Scope

- External OAuth2 / OpenID Connect identity providers.
- Multi-Factor Authentication (MFA/TOTP).

## Revision History

| Version | Date | Author | Summary |
|---------|------|--------|---------|
| 1.0.0 | 2026-11-04 | @jaca | Initial specification with RSA-OAEP password encryption, HttpOnly cookies, and RBAC hardening |
