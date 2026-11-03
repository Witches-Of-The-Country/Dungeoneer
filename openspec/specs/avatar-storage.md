---
id: "avatar-storage"
title: "Avatar Photo Upload & Storage — Living Specification"
version: "1.0.0"
status: "active"
owners:
  - "@jaca"
scope:
  paths:
    - "auth-service/**"
    - "player-character-service/**"
    - "dungeoneer-frontend/**"
---

# Avatar Photo Upload & Storage — Living Specification

> Authoritative specification for user and character avatar image upload, compression, and persistence across services.

---

## 1. Overview

Provides self-contained avatar photo upload and rendering for both Dungeoneer Users (`auth-service`) and Player Characters (`player-character-service`).

---

## 2. Data Model & Validation

- `auth.users.avatar_url TEXT NULL`
- `player_character.characters.avatar_url TEXT NULL`
- **Format**: `data:image/(png|jpeg|jpg|webp);base64,...` or secure HTTPS URL (`https://...`).
- **Maximum Length**: 700,000 characters (~512 KB payload size).
- **Client Compression**: Frontend automatically resizes uploaded images to at most `256x256` pixels via HTML5 `<canvas>` before base64 encoding.

---

## 3. Endpoints

- `PUT /api/auth/me/avatar` — Updates the authenticated user's `avatarUrl` and returns updated `UserResponseDto`.
- `POST /api/characters` & `PUT /api/characters/{id}` — Accept optional `avatarUrl` field for character portraits.

---

## Revision History

| Version | Date | Author | Summary |
|---------|------|--------|---------|
| 1.0.0 | 2026-11-03 | @jaca | Consolidated specification for user and character avatar upload (`005-user-and-character-avatar-upload`) |
