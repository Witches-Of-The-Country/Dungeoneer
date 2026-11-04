# Walkthrough Cumulativo — `auth-jwt-backend`

Este documento registra a implementação completa da change `auth-jwt-backend`, cobrindo tanto o backend em Clean Architecture quanto a integração com o frontend.

---

## 1. Backend (`auth-service/backend`)

### Camada de Domínio (`domain/model/` e `domain/exception/`)
- [`User.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/domain/model/User.java): Entidade pura Java com validação de formato de email, tamanho de nickname (`3..50`), data de nascimento no passado e hash de senha.
- [`Role.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/domain/model/Role.java): Papéis de usuário (`PLAYER`, `DUNGEON_MASTER`).
- [`DuplicateUserException.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/domain/exception/DuplicateUserException.java), [`InvalidCredentialsException.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/domain/exception/InvalidCredentialsException.java), [`UserValidationException.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/domain/exception/UserValidationException.java).
- Testes de unidade em [`UserTest.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/test/java/com/dungeoneer/auth/domain/model/UserTest.java) cobrindo regras de negócio e validações.

### Camada de Aplicação (`application/`)
- Inbound Ports: [`RegisterUserUseCase.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/in/RegisterUserUseCase.java), [`LoginUserUseCase.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/in/LoginUserUseCase.java).
- Outbound Ports: [`UserRepositoryPort.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/out/UserRepositoryPort.java), [`PasswordEncoderPort.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/out/PasswordEncoderPort.java), [`TokenProviderPort.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/out/TokenProviderPort.java).
- Serviço: [`AuthService.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/application/service/AuthService.java).
- Testes de unidade com Mockito em [`AuthServiceTest.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/test/java/com/dungeoneer/auth/application/service/AuthServiceTest.java) cobrindo registro, login, duplicação e credenciais inválidas.

### Camada de Adaptadores (`adapter/` e `core/`)
- Persistência: [`UserJpaEntity.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/out/persistence/UserJpaEntity.java), [`UserRepository.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/out/persistence/UserRepository.java), [`UserPersistenceAdapter.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/out/persistence/UserPersistenceAdapter.java), [`UserMapper.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/out/persistence/UserMapper.java).
- Segurança: [`BCryptPasswordEncoderAdapter.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/out/security/BCryptPasswordEncoderAdapter.java), [`JwtTokenProviderAdapter.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/out/security/JwtTokenProviderAdapter.java) (assinador HMAC-SHA256 via `${JWT_SECRET}` com validação de tamanho mínimo de 256 bits), [`SecurityConfig.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/core/config/SecurityConfig.java) (stateless, endpoints `/api/auth/**` liberados).
- Web Controller & Erros: [`AuthController.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/controller/AuthController.java) (`POST /api/auth/register`, `POST /api/auth/login`) e [`GlobalExceptionHandler.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/main/java/com/dungeoneer/auth/core/web/GlobalExceptionHandler.java) emitindo RFC 7807 `ProblemDetail` para `401`, `409` e `422`.
- Testes MockMvc: [`AuthControllerTest.java`](file:///D:/Projects/Dungeoneer/auth-service/backend/src/test/java/com/dungeoneer/auth/adapter/in/web/controller/AuthControllerTest.java) cobrindo todos os códigos HTTP de sucesso e falha.

---

## 2. Frontend (`dungeoneer-frontend`)

### Conexão e Fluxo de Autenticação
- [`lib/api/auth.ts`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/lib/api/auth.ts): Funções `loginUser`, `registerUser`, `getStoredToken` e persistência do token JWT no `localStorage`.
- [`components/login-form.tsx`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/components/login-form.tsx): Conectado à API com tratamento de erros `ProblemDetail` e redirecionamento para `/characters`.
- [`components/register-form.tsx`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/components/register-form.tsx): Conectado à API com validação Zod e redirecionamento para `/characters`.
- [`app/characters/page.tsx`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/app/characters/page.tsx): Nova tela "Em Construção" que confirma a autenticação bem-sucedida e disponibiliza botão de acesso direto ao protótipo público de criação de personagem (`/prototype/character-creation`).
- O protótipo em [`/prototype/character-creation`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/app/prototype/character-creation/page.tsx) permanece 100% público e acessível.

---

## 3. Verificação de Build & Gates
- Backend Maven: `mvn -f auth-service/backend/pom.xml test verify -DskipTests` -> **BUILD SUCCESS** com checagem JaCoCo satisfeita.
- Frontend: `npm run lint` e `npm run build` executados com sucesso (gerando rota estática `/characters`).
