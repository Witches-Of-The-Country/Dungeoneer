# Change Proposal: 005 — User & Character Avatar Photo Upload

> **Change ID:** `005-user-and-character-avatar-upload`
> **Status:** approved
> **Created:** 2026-11-01

---

## Summary

Add Avatar Photo Upload and Display support for both **Dungeoneer Users** (`auth-service` + `/profile` + `Navbar`) and **Player Characters** (`player-character-service` + `/characters/new` + `/characters/[id]` + `/characters` list cards).

---

## Motivation & Problem Statement

Currently, neither `users` in `auth-service` nor `characters` in `player-character-service` have an `avatar_url` column or upload mechanism, forcing the UI to display generic icons. Allowing players and DMs to upload an avatar image (resized/compressed client-side to a safe Data URL `image/webp` / `image/jpeg` / `image/png` or HTTPS URL, validated server-side for MIME type and max size) works seamlessly in both local Docker Compose and cloud deployments (Vercel + Railway) without requiring external S3 buckets.

---

## Scope

### In Scope
- Adding `avatar_url TEXT` to `auth.users` via Flyway migration `V3__add_user_avatar_url.sql` in `auth-service` and exposing `PUT /api/auth/me/avatar`.
- Adding `avatar_url TEXT` to `player_character.characters` via Flyway migration `V22__add_character_avatar_url.sql` in `player-character-service` and supporting `avatarUrl` in `CreateCharacterRequest`, `UpdateCharacterRequest`, `CharacterSummaryDto`, and `CharacterSheetResponse`.
- Creating client-side image compression/validation utility `dungeoneer-frontend/lib/image-upload.ts` (`256x256` WebP/JPEG via HTML5 `<canvas>`).
- Adding avatar upload and preview UI to `/profile`, `Navbar`, `/characters/new` (Step 1), `/characters/[id]` (Sheet header), and `/characters` (Character cards).

### Out of Scope
- External cloud object storage (AWS S3 / Cloudflare R2) provisioning or animated GIF/video avatars.

---

## Proposed Solution

1. **User Avatar (`auth-service` + Frontend `/profile` & `Navbar`)**:
   - Flyway migration `V3__add_user_avatar_url.sql` in `auth-service` adding `avatar_url TEXT` to `auth.users`.
   - Update `User.java`, `UserJpaEntity.java`, `UserResponseDto.java`, and add `PUT /api/auth/me/avatar` (`UpdateAvatarRequestDto`: `String avatarUrl` validated for `^data:image/(png|jpeg|jpg|webp);base64,[A-Za-z0-9+/=]+$` or `^https://.*`, max `700,000` chars).
   - In `app/profile/page.tsx` and `components/navbar.tsx`, display the user's avatar photo (with fallback icon) and provide an interactive file picker (`Alterar Foto`) that compresses/crops the selected image via HTML5 `<canvas>` (`256x256` WebP/JPEG) and saves it via `PUT /api/auth/me/avatar`.
2. **Character Avatar (`player-character-service` + Frontend Wizard, Sheet & Cards)**:
   - Flyway migration `V22__add_character_avatar_url.sql` in `player-character-service` adding `avatar_url TEXT` to `player_character.characters`.
   - Update `PlayerCharacter.java`, `PlayerCharacterJpaEntity.java`, `CreateCharacterRequest.java`, `UpdateCharacterRequest.java`, `CharacterSheetResult.java`, `CharacterSummaryDto.java`, and `CharacterSheetResponse.java` to include `avatarUrl`.
   - In `app/characters/new/page.tsx` (Step 1), `app/characters/[id]/page.tsx` (Sheet Header with quick photo upload button), and `app/characters/page.tsx` (Character Cards), display the character's avatar photo and allow uploading/changing it at any time.

---

## Impacted Files

| File Path | Layer | Reason |
|---|---|---|
| `auth-service/backend/src/main/resources/db/migration/V3__add_user_avatar_url.sql` | Database (`auth`) | Add `avatar_url TEXT` to `users`. |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/domain/model/User.java` | Domain (`auth`) | Add `avatarUrl` field and `updateAvatarUrl()` method. |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/AuthController.java` | Web (`auth`) | Add `PUT /api/auth/me/avatar`. |
| `player-character-service/.../db/migration/V22__add_character_avatar_url.sql` | Database (`character`) | Add `avatar_url TEXT` to `characters`. |
| `player-character-service/.../domain/model/PlayerCharacter.java` | Domain (`character`) | Add `avatarUrl` field and include in `updateDetails()`. |
| `dungeoneer-frontend/lib/image-upload.ts` | Frontend Utility | Validate MIME type, resize to `256x256` on canvas, and output Data URL. |
| `dungeoneer-frontend/app/profile/page.tsx` | Frontend Page | User avatar upload and preview. |
| `dungeoneer-frontend/app/characters/[id]/page.tsx` | Frontend Page | Character avatar display and quick upload on sheet header. |
| `dungeoneer-frontend/app/characters/new/page.tsx` | Frontend Page | Character avatar upload in Step 1 of creation wizard. |

---

## Acceptance Criteria

- **Given** an authenticated user on `/profile`, **When** they upload an image file (`PNG`, `JPG`, or `WEBP`), **Then** the image is resized/encoded, persisted via `PUT /api/auth/me/avatar`, and immediately displayed on `/profile` and in the `Navbar`.
- **Given** a user creating a character in `/characters/new` or viewing `/characters/[id]`, **When** they upload a character portrait, **Then** `avatarUrl` is persisted on the character and rendered in the Character Sheet header and `/characters` dashboard cards.
- **Given** an invalid file type (such as `image/svg+xml`) or oversized payload (`> 700,000` characters / ~512 KB), **When** submitted to `PUT /api/auth/me/avatar` or `POST/PUT /api/characters`, **Then** validation rejects the request with HTTP 400 ProblemDetail.

---

## Defensive Security & Validation Considerations

- **Stored XSS Prevention (No SVG)**: Both backend regex validation and frontend file input strictly whitelist `image/png`, `image/jpeg`, `image/jpg`, and `image/webp`. `image/svg+xml` and `javascript:` / `data:text/html` URIs are unconditionally rejected with HTTP 400.
- **Payload Size Enforcement**: `@Size(max = 700000)` on `avatarUrl` prevents memory exhaustion or database bloat; client-side `<canvas>` resizing (`256x256`, quality `0.82`) produces ~15–40 KB payloads.
- **Authorization**: `PUT /api/auth/me/avatar` updates only the authenticated principal's own `User` record; `PUT /api/characters/{id}` verifies character ownership (`X-Auth-User-Id`).

---

## Testing Strategy & Shift-Left Plan

| Scenario | Type | Target Test Class / File | Expected Outcome |
|---|---|---|---|
| Update user avatar with valid `data:image/webp;base64,...` | Unit | `AuthServiceTest.java`, `AuthControllerTest.java` | Updates `user.avatarUrl` and returns 200 OK with updated `UserResponseDto`. |
| Reject `image/svg+xml` or `javascript:` URI in avatar | Unit | `UserTest.java` / `CreateCharacterServiceTest.java` | Throws `IllegalArgumentException` / HTTP 400 ProblemDetail. |
| Create and update character with `avatarUrl` | Unit | `CreateCharacterServiceTest.java` | Persists `avatarUrl` on `PlayerCharacter` and returns it in `CharacterSheetResult`. |
| Frontend profile & character sheet avatar upload UI | Unit | `__tests__/profile-page.test.tsx`, `__tests__/character-sheet.test.tsx` | Renders avatar `<img>` when `avatarUrl` is present and triggers upload handler. |

- **Verification Commands**:
  - `mvn -f auth-service/backend/pom.xml test`
  - `mvn -f player-character-service/backend/pom.xml test`
  - `npm --prefix dungeoneer-frontend run test:unit`

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Base64 Data URL size in PostgreSQL `TEXT` column | Low | Client-side 256x256 canvas downscaling keeps Data URLs under ~30 KB, which PostgreSQL TOAST compresses and stores out-of-line with zero impact on index scans. |
| JSDOM `<canvas>` unavailability during Vitest unit tests | Low | Provide a clean fallback to `FileReader.readAsDataURL` in `lib/image-upload.ts` when 2D canvas context is null in headless test environments. |
