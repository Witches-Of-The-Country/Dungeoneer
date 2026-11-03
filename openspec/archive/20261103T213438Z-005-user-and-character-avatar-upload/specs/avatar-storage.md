# Specification: Avatar Photo Upload & Storage (`avatar-storage`)

> **Change ID:** `005-user-and-character-avatar-upload`
> **Change Type:** New Subsystem

## 1. Overview
Provides self-contained avatar photo upload and rendering for both Dungeoneer Users (`auth-service`) and Player Characters (`player-character-service`).

## 2. Data Model & Validation
- `auth.users.avatar_url TEXT NULL`
- `player_character.characters.avatar_url TEXT NULL`
- Accepted values: `data:image/(png|jpeg|jpg|webp);base64,...` or `https://...` with max length `700,000` characters (~512 KB).
- Frontend automatically resizes uploaded images to at most `256x256` pixels via HTML5 `<canvas>` before encoding.

## 3. Endpoints
- `PUT /api/auth/me/avatar` — Updates the authenticated user's `avatarUrl` and returns updated `UserResponseDto`.
- `POST /api/characters` & `PUT /api/characters/{id}` — Accept optional `avatarUrl` field.
