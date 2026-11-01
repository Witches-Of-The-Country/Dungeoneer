# Tasks: Fix Homebrew Creation CORS and Dungeon Master Role Assignment

> **Change ID:** `fix-homebrew-creation-cors-and-dm-role`  
> **Linked Proposal:** [proposal.md](./proposal.md)  
> **Status:** completed  
>  
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] <!-- task: cors-headers-gateway --> In `api-gateway`, ensure `JwtHeaderFilter` writes CORS headers (`Access-Control-Allow-Origin`, `Access-Control-Allow-Credentials`) on unauthorized and forbidden responses.
- [x] <!-- task: cors-config-precedence --> In `api-gateway`, ensure `CorsFilter` or `FilterRegistrationBean` precedence handles preflight and origin verification cleanly.
- [x] <!-- task: dm-role-support --> In `auth-service`, provide support to grant/manage `DUNGEON_MASTER` role for users (e.g. promoting a user or assigning during registration/update).
- [x] <!-- task: frontend-error-handling --> In `dungeoneer-frontend`, verify that error toasts properly extract and display the 403 `detail` message instead of failing on network/CORS error.

## Test Tasks

- [x] <!-- task: test-gateway-cors-errors --> Unit/Integration tests in `api-gateway` verifying that 401 and 403 responses contain `Access-Control-Allow-Origin` and `Vary: Origin`.
- [x] <!-- task: test-dm-role-assignment --> Unit/Integration tests in `auth-service` for role assignment/promotion logic.
- [x] <!-- task: test-homebrew-creation-flow --> Integration test verifying `POST /api/reference/feats` with `DUNGEON_MASTER` succeeds (201) and with `PLAYER` fails with 403 + readable CORS.

## Documentation Tasks

- [x] Update delta spec `specs/system-architecture-delta.md` with Gateway CORS error response requirements.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
