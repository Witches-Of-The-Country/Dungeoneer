# Delta Spec: `system-architecture`

> **Change ID:** `fix-homebrew-creation-cors-and-dm-role`  
> **Target Canonical Spec:** `openspec/specs/system-architecture.md`  
> **Change Type:** Modify  

## Architectural Updates

### API Gateway CORS on Filter Rejections
- When `JwtHeaderFilter` rejects an incoming request directly (e.g. 401 Unauthorized or 403 Forbidden), it must guarantee that CORS headers are appended to the `HttpServletResponse` prior to flushing the response.
- Allowed origins must match configured origins (`cors.allowed-origins`, supporting both `http://localhost:3000` and `http://127.0.0.1:3000`).
- Credentials flag `Access-Control-Allow-Credentials: true` must be preserved when an allowed origin is matched.

### Role Assignment & Reference Data Mutability
- Content under `/api/reference/**` with ruleset `HOMEBREW` can only be authored or mutated by users holding the `DUNGEON_MASTER` role.
- The `auth-service` must support role assignment to grant users the `DUNGEON_MASTER` capability.
