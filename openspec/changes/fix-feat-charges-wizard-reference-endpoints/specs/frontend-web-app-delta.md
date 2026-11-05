# Delta Spec: `frontend-web-app`

> **Change ID:** `fix-feat-charges-wizard-reference-endpoints`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### MODIFIED: `Character Creation Wizard — Origin Feat Selection`
- In `app/characters/new/page.tsx`, the feat selector step SHALL pass `featTypes: ["ORIGIN"]` and strictly filter loaded feats by `f.featType === "ORIGIN"`, excluding general or class progression feats that lack an explicit `ORIGIN` category.

### MODIFIED: `Authentication & Session Management (RSA-OAEP + HttpOnly Cookies)`
- **Client-Side Asymmetric Password Encryption (`RSA-OAEP`):** Before submitting `POST /api/auth/register` or `POST /api/auth/login`, `lib/api/auth.ts` SHALL fetch the server's RSA public key from `GET /api/auth/public-key`, import it via the Web Crypto API (`window.crypto.subtle`), and encrypt the plaintext password using `RSA-OAEP` (`SHA-256`), sending the Base64-encoded ciphertext (`rsa:<base64>`) in the request payload so plaintext passwords never appear in the browser Network tab.
- **Elimination of `localStorage` JWT Storage:** `persistAuthSession` in `lib/api/auth.ts` SHALL no longer store raw JWT strings in `window.localStorage` or `document.cookie`. Session state relies on the `HttpOnly` cookie set by the backend and `credentials: 'include'` in `lib/api/client.ts`.
- **Edge Middleware Validation (`middleware.ts`):** Next.js middleware SHALL decode the JWT payload from the `dungeoneer_auth_token` cookie, verify structure and `exp` timestamp validity, and protect both `/characters/:path*` and `/profile/:path*`.
- **HTTP Security Headers (`next.config.ts`):** Next.js response headers SHALL include `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, and `Content-Security-Policy`.
