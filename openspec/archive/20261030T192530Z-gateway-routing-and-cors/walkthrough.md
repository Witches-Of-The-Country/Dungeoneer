# Walkthrough: Gateway Routing And Cors

> **Change ID:** `gateway-routing-and-cors`
> **Status:** Implementation & Verification Complete
> **Author:** Antigravity

---

## 1. Visão Geral da Mudança

Esta mudança configurou o `api-gateway` (`:9090`) como a camada unificada de entrada, autenticação perimetral, autorização e gerenciamento de CORS para a plataforma Dungeoneer:
- **Centralização de CORS**: Aceita requisições do frontend Next.js (`http://localhost:3000`) com credenciais e headers autorizados (`Authorization`, `Content-Type`, `X-User-Id`).
- **Filtro de Segurança & Propagação (`JwtHeaderFilter`)**:
  - Remoção incondicional de headers `X-User-Id` enviados por clientes externos para mitigar ameaças de spoofing de identidade e IDOR/BOLA.
  - Verificação de assinatura `HS256` e expiração (`exp`) de Bearer JWTs para as rotas protegidas (`/api/characters/**`).
  - Respostas padronizadas RFC 7807 (`ProblemDetail`) com status `401 Unauthorized` para tokens ausentes, expirados ou forjados.
  - Injeção segura e confiável do header downstream `X-User-Id: <sub-uuid>` obtido da claim `sub` do token validado.
- **Roteamento WebMVC (`GatewayRoutesConfig`)**:
  - `/api/auth/**` → `${AUTH_SERVICE_URL:http://localhost:8082}` (público)
  - `/api/reference/**` → `${CHARACTER_SERVICE_URL:http://localhost:8080}` (público)
  - `/api/characters/**` → `${CHARACTER_SERVICE_URL:http://localhost:8080}` (protegido)
- **Alinhamento de Infraestrutura (`docker-compose.yml`)**:
  - Injeção de `${JWT_SECRET}` e URLs internas no `api-gateway`.
  - Apontamento do `dungeoneer-frontend` para `NEXT_PUBLIC_API_URL=http://localhost:9090`.

---

## 2. Arquivos Implementados e Modificados

| Arquivo | Descrição |
|---|---|
| [`api-gateway/backend/pom.xml`](file:///D:/Projects/Dungeoneer/api-gateway/backend/pom.xml) | Adicionadas dependências `nimbus-jose-jwt:9.37.3`, `spring-boot-starter-test`, `spring-boot-starter-webmvc-test` e plugin JaCoCo (`0.8.12`). |
| [`api-gateway/backend/src/main/resources/application.properties`](file:///D:/Projects/Dungeoneer/api-gateway/backend/src/main/resources/application.properties) | Configuração de URLs de serviços, segredo JWT e parâmetros de CORS. |
| [`CorsConfig.java`](file:///D:/Projects/Dungeoneer/api-gateway/backend/src/main/java/com/dungeoneer/api_gateway/config/CorsConfig.java) | Configuração centralizada de CORS com `@Order(Ordered.HIGHEST_PRECEDENCE)`. |
| [`JwtHeaderFilter.java`](file:///D:/Projects/Dungeoneer/api-gateway/backend/src/main/java/com/dungeoneer/api_gateway/security/JwtHeaderFilter.java) | Filtro de segurança perimetral com descarte de `X-User-Id` forjado, verificação JWT e injeção downstream. |
| [`GatewayRoutesConfig.java`](file:///D:/Projects/Dungeoneer/api-gateway/backend/src/main/java/com/dungeoneer/api_gateway/config/GatewayRoutesConfig.java) | Definição de rotas do Spring Cloud Gateway Server WebMVC usando `GatewayRouterFunctions`. |
| [`docker-compose.yml`](file:///D:/Projects/Dungeoneer/docker-compose.yml) | Variáveis de ambiente (`JWT_SECRET`, URLs dos serviços), dependências e porta do frontend `:9090`. |
| [`JwtHeaderFilterTest.java`](file:///D:/Projects/Dungeoneer/api-gateway/backend/src/test/java/com/dungeoneer/api_gateway/security/JwtHeaderFilterTest.java) | 7 testes unitários do filtro cobrindo preflight, tokens inválidos, expirados, forjados e injeção de headers. |
| [`ApiGatewayRoutingAndSecurityTests.java`](file:///D:/Projects/Dungeoneer/api-gateway/backend/src/test/java/com/dungeoneer/api_gateway/ApiGatewayRoutingAndSecurityTests.java) | 7 testes de integração ponta a ponta com servidor HTTP downstream em tempo de teste. |

---

## 3. Resultados dos Testes Automatizados

O comando `mvn -f api-gateway/backend/pom.xml clean verify` executou com sucesso absoluto:

```
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.dungeoneer.api_gateway.ApiGatewayApplicationTests
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] Running com.dungeoneer.api_gateway.ApiGatewayRoutingAndSecurityTests
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0
[INFO] Running com.dungeoneer.api_gateway.security.JwtHeaderFilterTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 15, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] --- jacoco:0.8.12:check (check) @ api-gateway ---
[INFO] All coverage checks have been met.
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```
