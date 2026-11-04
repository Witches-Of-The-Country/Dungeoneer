# Tasks: 005 — User & Character Avatar Photo Upload

> **Change ID:** `005-user-and-character-avatar-upload`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** complete
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Create Flyway migration `V2__add_user_avatar_url.sql` in `auth-service` and update `User.java`, `UserJpaEntity.java`, `UserProfileDto.java`, `AuthService.java`, and `AuthController.java` (`PUT /api/auth/me/avatar`).
- [x] Create Flyway migration `V22__add_character_avatar_url.sql` in `player-character-service` and update `PlayerCharacter.java`, `PlayerCharacterJpaEntity.java`, `CreateCharacterRequest.java`, `UpdateCharacterRequest.java`, `CharacterSummaryResult.java`, and `CharacterSheetResponse.java` with `avatarUrl`.
- [x] Create a reusable client-side image compression/upload helper (`lib/image-upload.ts`) and avatar upload UI in `app/profile/page.tsx`, `components/navbar.tsx`, `app/characters/new/page.tsx`, `app/characters/[id]/page.tsx`, and `app/characters/page.tsx`.

## Test Tasks

- [x] Add unit tests in `AuthServiceTest.java` and `CreateCharacterServiceTest.java` / `UpdateCharacterServiceTest` covering `avatarUrl` persistence and validation.
- [x] Add frontend Vitest tests in `__tests__/image-upload.test.ts` and character pages verifying avatar display, WebP compression, and SVG rejection.

## Documentation Tasks

- [x] Document the avatar upload endpoints and storage constraints in `specs/avatar-storage.md`.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
