# Proposal: Fix Feat Charges, Wizard Origin Filter, Reference Endpoints & Full Security Hardening

> **Change ID:** `fix-feat-charges-wizard-reference-endpoints`
> **Status:** draft
> **Author:** @jaca
> **Created:** 2026-11-04
> **Last Updated:** 2026-11-04

---

## Summary

Unifica a correção de três problemas funcionais e de entrega contínua com o **pacote completo de blindagem de segurança (Security Hardening)** identificado na auditoria do repositório:

1. **Recuperação de Cargas de Talentos em Descansos (`player-character-service`):** Descansos curto e longo não recuperam cargas de talentos com carga (ex.: Fúria) quando `maxCharges` está nulo em talentos legados ou quando a regra de recuperação por ruleset (`SRD_2014` vs `SRD_2024`) não é resolvida.
2. **Filtro de Talentos de Origem no Wizard (`dungeoneer-frontend`):** O wizard de criação de personagem exibe talentos sem filtro estrito de categoria (`ORIGIN`), aceitando registros com `!f.featType`.
3. **Endpoints de Referência 404 em Produção (`subclasses`, `progression-feats`, `subclass-progression-feats`):** Os endpoints existem em `ReferenceDataController`, mas o ambiente de produção usa imagem/ponteiro de submódulo anterior ao merge da branch de criação.
4. **Criptografia Assimétrica de Senha no Cliente (`RSA-OAEP`) + Sessão via Cookie `HttpOnly`:** Evita que a senha saia em texto literal na aba Network do navegador durante `/register` e `/login` (usando Web Crypto API `RSA-OAEP` com chave pública fornecida por `GET /api/auth/public-key`) e elimina o armazenamento do JWT em `localStorage`/`document.cookie`, movendo a sessão para cookie `HttpOnly; Secure; SameSite=Lax` gerenciado pelo backend/gateway.
5. **Correção de Vulnerabilidades Críticas, Altas, Médias e Baixas (`SEC-CRIT-01` a `SEC-LOW-03`):**
   - **[SEC-CRIT-01]** Bloqueio de escalonamento de privilégio não autenticado em `POST /api/auth/users/{userId}/roles`.
   - **[SEC-CRIT-02]** Remoção da exposição pública das portas `8080` e `8082` no `docker-compose.yml` e validação de segredo de perímetro interno (`X-Internal-Secret`) entre `api-gateway` e microsserviços downstream para impedir falsificação de `X-User-Id` (IDOR total).
   - **[SEC-HIGH-01]** Verificação obrigatória de propriedade (`existing.getOwnerId().equals(userId)`) em todas as atualizações `PUT` de conteúdo `HOMEBREW`, retornando `403 Forbidden` em caso de divergência.
   - **[SEC-HIGH-03]** Validação de estrutura e expiração (`exp`) do JWT no `middleware.ts` do Next.js e ampliação das rotas protegidas para incluir `/profile`.
   - **[SEC-MED-01]** Remoção de chaves fallback hardcoded (`JWT_SECRET` e credenciais de banco) em perfis produtivos.
   - **[SEC-MED-02]** Sanitização do handler genérico `@ExceptionHandler(Exception.class)` nos `GlobalExceptionHandler` para não vazar `ex.getMessage()` bruto em erros HTTP 500.
   - **[SEC-MED-03]** Adição de HTTP Security Headers (`X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy`, `Content-Security-Policy`, `Strict-Transport-Security`) no `api-gateway` e `next.config.ts`.
   - **[SEC-LOW-01/02/03]** Desativação do Swagger UI em perfil de produção, remoção de configurações CORS duplicadas nos serviços downstream (mantendo apenas no `api-gateway`) e adição de Rate Limiting para `/api/auth/login` e `/api/auth/register`.

---

## Motivation

- **Segurança Defensiva:** A auditoria de código (`sdd-code-explorer`) revelou que a senha trafegava em JSON puro visível no inspetor local, o JWT ficava exposto em `localStorage` (vulnerável a XSS), o endpoint `POST /api/auth/users/{userId}/roles` permitia promoção arbitrária de roles sem autenticação, os serviços downstream confiavam cegamente no header `X-User-Id` com portas expostas no Docker Compose, e conteúdos Homebrew podiam ser sobrescritos por terceiros (IDOR).
- **Integridade de Regras D&D 5e/2024:** Personagens com talentos de carga (como Fúria) precisam recuperar cargas corretamente em descansos curtos/longos conforme o ruleset, e a criação de personagem no nível 1 deve restringir a seleção exclusivamente a Talentos de Origem (`ORIGIN`).

---

## Scope

### In Scope
- **Cargas de Talentos & Descanso:** Atualização de `RestService`, `FeatChargeCalculator`, `CharacterFeat` e migração Flyway `V28__adapt_legacy_feat_charges_and_recovery.sql` para recalcular/hidratar `maxCharges` e regras de recuperação em talentos legados.
- **Wizard Frontend:** Filtro estrito `featType === "ORIGIN"` no passo de talentos do wizard de criação (`app/characters/new/page.tsx`).
- **Entrega de Endpoints de Referência:** Verificação de contrato e atualização de ponteiros de submódulo/imagens para `/api/reference/subclasses`, `/api/reference/progression-feats` e `/api/reference/subclass-progression-feats`.
- **Criptografia de Senha (`RSA-OAEP`) no Registro e Login:**
  - `auth-service` gera/carrega par de chaves RSA-2048 (`KeyPair`), expõe `GET /api/auth/public-key` (PEM/SPKI Base64) e descriptografa senhas cifradas com `RSA-OAEP` (`SHA-256`) antes do hash BCrypt.
  - `dungeoneer-frontend` cifra o campo `password` via Web Crypto API (`crypto.subtle.encrypt`) antes de enviar `POST /api/auth/register` e `POST /api/auth/login`.
- **Sessão Segura com Cookie `HttpOnly`:**
  - `auth-service` emite cookie `dungeoneer_auth_token` com `HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=86400` no login/registro e limpa no novo endpoint `POST /api/auth/logout`.
  - `api-gateway` (`JwtHeaderFilter`) extrai e valida o JWT a partir do cookie `HttpOnly` (mantendo suporte a `Authorization: Bearer` apenas para clientes de API/testes).
  - `dungeoneer-frontend` remove a persistência do token JWT em `localStorage` e `document.cookie`, passando a enviar `credentials: 'include'` em todas as chamadas `apiFetch`.
- **Hardening de Controle de Acesso, Infraestrutura e Headers (`SEC-CRIT-01` a `SEC-LOW-03`):**
  - Proteção de `/api/auth/users/{userId}/roles` exigindo autenticação e verificação de permissão/próprio usuário.
  - Filtro de segredo interno (`X-Internal-Secret`) validado pelos serviços downstream (`player-character-service` e `auth-service`) e injetado pelo `api-gateway`, além de remover `ports: 8080:8080` e `8082:8080` do `docker-compose.yml`.
  - Validação de `ownerId` (`existing.getOwnerId().equals(userId)`) em `HomebrewService` e obrigatoriedade de `X-User-Id` nos endpoints `PUT /api/reference/**`.
  - Validação de payload/expiração do JWT no `middleware.ts` do Next.js (cobrindo `/characters/:path*` e `/profile/:path*`).
  - Sanitização de mensagens de erro 500, headers de segurança HTTP, rate limiting em `/api/auth/login` e `/api/auth/register`, remoção de CORS downstream e restrição do Swagger UI.

### Out of Scope
- Substituição do provedor de autenticação próprio por OAuth2/OIDC externo (Keycloak/Auth0).
- Novas mecânicas de talentos além das cargas existentes.
- Novo conteúdo SRD além do já especificado.

---

## Proposed Solution

### Approach
1. **Fase 1 — Cargas de Talentos, Wizard e Endpoints de Referência:**
   - Ajustar `FeatChargeCalculator`, `CharacterFeat` e `RestService` para recalcular `maxCharges` ausentes e aplicar recuperação por ruleset (`SRD_2014` vs `SRD_2024`). Criar migração Flyway `V28__adapt_legacy_feat_charges_and_recovery.sql`.
   - Restringir o filtro de talentos no wizard (`app/characters/new/page.tsx`) exclusivamente para `featType === "ORIGIN"`.
2. **Fase 2 — Criptografia `RSA-OAEP` de Senha e Sessão `HttpOnly` (`auth-service`, `api-gateway`, `dungeoneer-frontend`):**
   - Criar porta `PasswordCipherPort` e adaptador `RsaOaepPasswordCipherAdapter` em `auth-service`, expondo `GET /api/auth/public-key` e decriptando senhas no `AuthService` (mantendo compatibilidade controlada ou rejeitando texto plano conforme configuração).
   - Emitir cookie `HttpOnly` no login/register e implementar `POST /api/auth/logout`.
   - Atualizar `JwtHeaderFilter` no `api-gateway` para ler o cookie `dungeoneer_auth_token`, aplicar Rate Limiting em `/api/auth/login` e `/api/auth/register`, injetar `X-Internal-Secret` para os serviços downstream e anexar headers de segurança HTTP.
   - Atualizar `dungeoneer-frontend` (`lib/api/auth.ts`, `lib/api/client.ts`, `middleware.ts`, `next.config.ts`) para cifrar senhas com Web Crypto API (`RSA-OAEP`), remover `localStorage` do JWT e validar expiração do cookie no middleware.
3. **Fase 3 — Controle de Acesso Downstream, IDOR Homebrew e Hardening (`player-character-service`, `auth-service`, `docker-compose.yml`):**
   - Proteger `/api/auth/**` exceto rotas públicas (`/login`, `/register`, `/public-key`, `/logout`) em `auth-service` e `api-gateway`.
   - Adicionar filtro `InternalGatewaySecretFilter` em `player-character-service` e `auth-service` exigindo `X-Internal-Secret` válido.
   - Tornar `@RequestHeader("X-User-Id")` obrigatório em todos os `PUT` de `ReferenceDataController` e validar `existing.getOwnerId().equals(userId)` em `HomebrewService` (lançando `UnauthorizedActionException` → `403 Forbidden`).
   - Sanitizar `GlobalExceptionHandler` (HTTP 500), remover CORS redundante dos microsserviços downstream, restringir Swagger UI (`!prod`) e remover portas públicas `8080` e `8082` do `docker-compose.yml`.

### Impacted Files
<!-- Populated by sdd-code-explorer subagent. -->
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/.../application/service/RestService.java` | Modify | `recoverFeats`/`useFeat` recalculam/tratam feats legados com `maxCharges == null` |
| `player-character-service/.../domain/model/FeatChargeCalculator.java` | Modify | `resolveRecoveryType`/`calculateRecoveryAmount` por ruleset |
| `player-character-service/.../domain/model/CharacterFeat.java` | Modify | `recoverCharges` preserva/clamp do máximo recalculado |
| `player-character-service/backend/src/main/resources/db/migration/V28__adapt_legacy_feat_charges_and_recovery.sql` | Create | Backfill de metadados de recuperação e cargas nulas em talentos legados |
| `player-character-service/.../adapter/in/web/controller/ReferenceDataController.java` | Modify | Tornar `X-User-Id` obrigatório (`required = true`) em todos os endpoints `PUT` de Homebrew |
| `player-character-service/.../application/service/HomebrewService.java` | Modify | Validar `existing.getOwnerId().equals(userId)` em `updateSpell`, `updateFeat`, `updateItem`, `updateClass`, `updateLineage`, `updateSubclass` (`403 Forbidden`) |
| `player-character-service/.../core/config/SecurityConfig.java` | Modify | Validar `X-Internal-Secret` e remover configuração CORS redundante |
| `player-character-service/.../core/web/GlobalExceptionHandler.java` | Modify | Mapear violação de propriedade para `403 Forbidden` e sanitizar `handleGenericException` (500) |
| `player-character-service/backend/src/main/resources/application.yml` | Modify | Restringir Swagger/OpenAPI por variável de ambiente e exigir `INTERNAL_GATEWAY_SECRET` |
| `auth-service/.../application/port/out/PasswordCipherPort.java` | Create | Porta de saída para descriptografia `RSA-OAEP` e exportação da chave pública |
| `auth-service/.../adapter/out/security/RsaOaepPasswordCipherAdapter.java` | Create | Adaptador RSA-2048 (`RSA/ECB/OAEPWithSHA-256AndMGF1Padding`) |
| `auth-service/.../application/service/AuthService.java` | Modify | Descriptografar senha antes do BCrypt/verificação e validar autorização em `addRole` |
| `auth-service/.../adapter/in/web/controller/AuthController.java` | Modify | Adicionar `GET /api/auth/public-key`, `POST /api/auth/logout`, emitir cookie `HttpOnly` e proteger `POST /api/auth/users/{userId}/roles` |
| `auth-service/.../core/config/SecurityConfig.java` | Modify | Restringir `/api/auth/users/**` e `/api/auth/me`, validar `X-Internal-Secret` e remover CORS redundante |
| `auth-service/.../core/web/GlobalExceptionHandler.java` | Modify | Sanitizar `handleGenericException` (500) para não vazar `ex.getMessage()` |
| `auth-service/backend/src/main/resources/application.properties` | Modify | Configurar `INTERNAL_GATEWAY_SECRET` e desabilitar Swagger em produção |
| `api-gateway/.../security/JwtHeaderFilter.java` | Modify | Extrair JWT de cookie `HttpOnly`, proteger `/api/auth/users/**`, injetar `X-Internal-Secret`, aplicar Rate Limiting em `/api/auth/login` e `/api/auth/register` e adicionar Security Headers |
| `api-gateway/backend/src/main/resources/application.properties` | Modify | Configurar `INTERNAL_GATEWAY_SECRET` e restringir Swagger UI |
| `dungeoneer-frontend/app/characters/new/page.tsx` | Modify | Passar `featTypes: ["ORIGIN"]` e filtrar estritamente `featType === "ORIGIN"` |
| `dungeoneer-frontend/lib/api/auth.ts` | Modify | Cifrar senha com Web Crypto API (`RSA-OAEP`) usando `/api/auth/public-key`, remover JWT de `localStorage`/`document.cookie` e usar `/api/auth/logout` |
| `dungeoneer-frontend/lib/api/client.ts` | Modify | Enviar `credentials: 'include'` em `apiFetch` sem depender de token em `localStorage` |
| `dungeoneer-frontend/middleware.ts` | Modify | Validar estrutura e expiração (`exp`) do JWT no cookie e proteger `/characters/:path*` e `/profile/:path*` |
| `dungeoneer-frontend/next.config.ts` | Modify | Configurar HTTP Security Headers (`CSP`, `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`) |
| `docker-compose.yml` | Modify | Remover mapeamento público de portas `8080:8080` e `8082:8080` (usar `expose`) e injetar `INTERNAL_GATEWAY_SECRET` |

### Data Model Changes (if applicable)
- Migração Flyway `V28__adapt_legacy_feat_charges_and_recovery.sql` em `player-character-service` para garantir metadados de recuperação e `max_charges` em talentos legados (ex.: Fúria).

---

## Acceptance Criteria

- [ ] **Given** um personagem com cargas de Fúria gastas (incluindo ficha legada com `maxCharges` nulo) **When** realiza descanso longo ou curto **Then** as cargas são recalculadas e recuperadas conforme a regra do ruleset (`SRD_2014` ou `SRD_2024`).
- [ ] **Given** o wizard de criação de personagem no nível 1 **When** o usuário abre a seleção de talentos **Then** apenas talentos com `featType === "ORIGIN"` são exibidos.
- [ ] **Given** o formulário de `/register` ou `/login` no navegador **When** o usuário submete suas credenciais **Then** o frontend obtém a chave pública de `GET /api/auth/public-key`, cifra o campo `password` com `RSA-OAEP` (`SHA-256`) antes do envio (ocultando a senha literal na aba Network) e o `auth-service` descriptografa antes de aplicar/verificar o BCrypt.
- [ ] **Given** um login ou registro bem-sucedido **When** a resposta retorna ao navegador **Then** o JWT é definido em um cookie `HttpOnly; Secure; SameSite=Lax` e **não** é gravado em `localStorage` nem acessível via `document.cookie`.
- [ ] **Given** uma requisição para `POST /api/auth/users/{userId}/roles` sem JWT válido ou para outro `userId` sem permissão **When** processada pelo gateway/auth-service **Then** retorna `401 Unauthorized` ou `403 Forbidden`.
- [ ] **Given** uma requisição enviada diretamente a `player-character-service` ou `auth-service` sem o header `X-Internal-Secret` válido **When** interceptada pelo filtro de segurança **Then** retorna `403 Forbidden`.
- [ ] **Given** um item/talento/magia/classe/linhagem/subclasse `HOMEBREW` pertencente ao usuário A **When** o usuário B tenta atualizá-lo via `PUT /api/reference/**` **Then** o serviço retorna `403 Forbidden` (RFC 7807 `ProblemDetail`).
- [ ] **Given** um cookie de autenticação inválido ou expirado no navegador **When** o usuário acessa `/characters` ou `/profile` **Then** o `middleware.ts` redireciona imediatamente para `/login`.
- [ ] **Given** uma exceção inesperada não tratada (HTTP 500) em qualquer microsserviço **When** o `GlobalExceptionHandler` formata o `ProblemDetail` **Then** a resposta contém uma mensagem genérica segura sem expor detalhes internos de SQL/stack.

---

## Defensive Security & Validation Considerations

### Input Validation
- `RegisterRequest` e `LoginRequest` continuam validados com Jakarta Validation (`@NotBlank`, `@Email`, `@Size`), aplicando validação de tamanho da senha em texto claro após a descriptografia `RSA-OAEP` no `AuthService`.
- Todos os endpoints `PUT /api/reference/**` exigem `@RequestHeader(value = "X-User-Id", required = true)` validado como UUID.

### Secret Isolation
- `JWT_SECRET` e `INTERNAL_GATEWAY_SECRET` são injetados via variáveis de ambiente em todos os serviços (`api-gateway`, `auth-service`, `player-character-service`).
- Par de chaves RSA-2048 pode ser injetado via variável de ambiente (`AUTH_RSA_PRIVATE_KEY` / `AUTH_RSA_PUBLIC_KEY`) ou gerado na inicialização em memória para proteção de transporte de payload.

### Authorization & Authentication Impact
- Defesa em profundidade: perímetro autenticado no `api-gateway` (cookie `HttpOnly` ou `Bearer` → cabeçalhos `X-User-Id` e `X-Internal-Secret`) + validação de `X-Internal-Secret` e propriedade (`ownerId`) nos serviços downstream.
- O segredo JWT exposto anteriormente em captura deve ser rotacionado em produção.

### Other Security Risks
- **Mitigação de XSS:** Remoção completa do token JWT do `localStorage` e adição de `Content-Security-Policy` e `X-Content-Type-Options: nosniff`.
- **Mitigação de Brute-Force:** Rate limiting por IP no `api-gateway` para `/api/auth/login` e `/api/auth/register` (retornando `429 Too Many Requests`).

---

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Descanso longo/curto restaura cargas de Fúria (incluindo legado) | Unit (`RestServiceTest`, `FeatChargeCalculatorTest`) | Cargas restauradas conforme ruleset |
| TP-02 | Wizard de criação filtra apenas talentos `ORIGIN` | Vitest (`character-wizard.test.tsx`) | Apenas talentos `ORIGIN` renderizados |
| TP-03 | Criptografia e descriptografia `RSA-OAEP` no login/registro e emissão de cookie `HttpOnly` | Unit & MockMvc (`RsaOaepPasswordCipherAdapterTest`, `AuthControllerTest`) | Senha cifrada aceita, cookie `HttpOnly` emitido |
| TP-04 | `JwtHeaderFilter` autentica via cookie `HttpOnly` e injeta `X-Internal-Secret` | Unit/MockMvc (`JwtHeaderFilterTest`) | Requisição roteada com `X-User-Id` e `X-Internal-Secret` |
| TP-05 | Dono original atualiza seu próprio conteúdo `HOMEBREW` | Unit (`HomebrewServiceTest`) | Atualização permitida (200 OK) |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Usuário B tenta editar conteúdo `HOMEBREW` do Usuário A | Unit/MockMvc (`HomebrewServiceTest`, `ReferenceDataControllerTest`) | `403 Forbidden` (`ProblemDetail`) |
| TE-02 | Chamada a `POST /api/auth/users/{id}/roles` sem autenticação ou para outro usuário | MockMvc (`AuthControllerTest`, `JwtHeaderFilterTest`) | `401 Unauthorized` / `403 Forbidden` |
| TE-03 | Requisição direta ao `player-character-service` sem `X-Internal-Secret` | MockMvc (`InternalGatewaySecretFilterTest`) | `403 Forbidden` |
| TE-04 | Exceção genérica `RuntimeException` com detalhe de SQL no `GlobalExceptionHandler` | Unit (`GlobalExceptionHandlerTest`) | `500 Internal Server Error` sem vazar `ex.getMessage()` |
| TE-05 | Excesso de tentativas em `/api/auth/login` ou `/api/auth/register` | Unit (`JwtHeaderFilterTest`) | `429 Too Many Requests` |
| TE-06 | Middleware Next.js recebe cookie com JWT expirado ou malformado em `/profile` | Vitest (`middleware.test.ts`) | Redirecionamento 307 para `/login` |

### Test Commands
- **Backend Unit & API Tests:** `mvn clean test` (em `player-character-service/backend`, `auth-service/backend`, `api-gateway/backend`)
- **Backend Coverage Check:** `mvn clean verify` (JaCoCo ≥ 80% em `domain` e `application`)
- **Frontend Unit Tests:** `npx vitest run` (em `dungeoneer-frontend`)

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Quebra de testes existentes que enviam senha em texto plano ou sem `X-Internal-Secret` | Med | Med | Suportar fallback controlado ou atualizar fixtures de teste e configurar `X-Internal-Secret` de teste nos slices MockMvc |
| Cookies `HttpOnly` em ambiente local HTTP (`localhost`) | High | Med | Definir flag `Secure` condicionalmente por ambiente (`AUTH_COOKIE_SECURE=false` em dev local HTTP, `true` em produção HTTPS) |
| Imagem de produção desatualizada para endpoints de subclasse/progressão | High | High | Atualizar ponteiro de submódulo após os testes e publicar imagem |

---

## Specification Changes & Delta Stubs

- `openspec/changes/fix-feat-charges-wizard-reference-endpoints/specs/auth-domain.md` (New canonical domain spec stub for `auth-service`)
- `openspec/changes/fix-feat-charges-wizard-reference-endpoints/specs/system-architecture-delta.md` (Delta for `openspec/specs/system-architecture.md`)
- `openspec/changes/fix-feat-charges-wizard-reference-endpoints/specs/player-character-domain-delta.md` (Delta for `openspec/specs/player-character-domain.md`)
- `openspec/changes/fix-feat-charges-wizard-reference-endpoints/specs/frontend-web-app-delta.md` (Delta for `openspec/specs/frontend-web-app.md`)

---

## References

- `.github/workflows/docker-publish.yml`
- `api-gateway/backend/src/main/java/com/dungeoneer/api_gateway/security/JwtHeaderFilter.java`
- `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/controller/AuthController.java`
- `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/HomebrewService.java`
