# Delta Spec: `auth-service` (`system-architecture`)

> **Change ID:** `mechanical-properties-and-class-progression`
> **Target Canonical Spec:** `openspec/specs/system-architecture.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Toggleable `DUNGEON_MASTER` Role in `auth-service`
- **Domain Invariant (`com.dungeoneer.auth.domain.model.User`)**:
  - `Role.PLAYER` is the mandatory base role for every user account and cannot be removed (`removeRole(Role.PLAYER)` throws `IllegalArgumentException` / domain validation exception).
  - `Role.DUNGEON_MASTER` can be toggled on (`addRole(Role.DUNGEON_MASTER)`) or off (`removeRole(Role.DUNGEON_MASTER)`).
- **Endpoint (`POST /api/auth/me/roles/toggle-dm`)**:
  - Authenticated via `Authorization: Bearer <jwt-token>`.
  - If the user currently holds `Role.DUNGEON_MASTER`, removes `Role.DUNGEON_MASTER` (leaving `Role.PLAYER` active); otherwise adds `Role.DUNGEON_MASTER`.
  - Persists the updated `User` entity and returns `200 OK` with `AuthResponseDto` containing a freshly signed HMAC-SHA256 JWT (`accessToken`) whose `roles` claim reflects the updated role set, plus the updated `UserProfileDto`.
