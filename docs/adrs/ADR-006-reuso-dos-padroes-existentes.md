# ADR-006 — Webhooks como módulo comum do projeto, com reuso máximo dos padrões existentes

- **Data da decisão:** reunião técnica de quinta-feira (registrada em [09:30] por Larissa)
- **Decisores:** Larissa, Bruno, Diego
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

## Status

Aceito

## Contexto

O OMS segue um padrão consistente, que o Bruno descreveu em [09:27] e que o código confirma:

| Padrão | Onde está no código |
|---|---|
| Módulo por domínio com `controller`, `service`, `repository`, `routes`, `schemas` | `src/modules/orders/`, `src/modules/customers/`, `src/modules/products/`, `src/modules/users/` |
| Montagem dos módulos e injeção de dependências | `src/app.ts` (`buildControllers`) e `src/routes/index.ts` (`buildApiRouter`, prefixo `/api/v1`) |
| Erro de domínio com `statusCode` + `errorCode` | `src/shared/errors/app-error.ts` (`AppError`) |
| Erros específicos com código em `UPPER_SNAKE_CASE` | `src/shared/errors/http-errors.ts` (`InvalidStatusTransitionError` → `INVALID_STATUS_TRANSITION`, `InsufficientStockError` → `INSUFFICIENT_STOCK`) |
| Tratamento centralizado de `AppError`, `ZodError` e erros conhecidos do Prisma | `src/middlewares/error.middleware.ts` |
| Validação de entrada com Zod | `src/middlewares/validate.middleware.ts` + `*.schemas.ts` |
| Autenticação JWT e autorização por role | `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`) |
| Log estruturado com Pino e redaction | `src/shared/logger/index.ts` |
| Chaves primárias UUID (`@db.Char(36)`) | `prisma/schema.prisma` |

Uma feature nova pode seguir esse padrão ou trazer estruturas próprias (filas, logger ou outro framework de erros).

## Decisão

**Reuso máximo do que já existe. Webhooks será um módulo igual aos outros** ([09:30] Larissa):

- Pasta `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts` e `webhook.schemas.ts` (novos), além de `webhook.processor.ts` (novo) com a lógica do worker ([09:27] Bruno, [09:28] Bruno). O FDD acrescenta arquivos de apoio (`webhook.errors.ts`, `webhook.publisher.ts`, `webhook.signature.ts` e as rotas de admin), todos dentro do mesmo módulo.
- Erros novos estendem `AppError` ou as subclasses de `src/shared/errors/http-errors.ts`, com **prefixo `WEBHOOK_`** em todos os códigos do módulo (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno, [09:29] Larissa). Observação: `WEBHOOK_INVALID_URL` nasce de um schema Zod, e o `validate.middleware.ts` converte todo `ZodError` em `VALIDATION_ERROR`, então esse código aparece só em `details[].message` da resposta (detalhe no FDD §7).
- Os erros chegam ao `errorMiddleware` **sem nenhuma alteração** no middleware ([09:29] Bruno).
- Log só com o Pino existente (`logger` de `src/shared/logger/index.ts`), **sem biblioteca nova** ([09:29] Bruno).
- O replay admin reaproveita `requireRole('ADMIN')`, como já faz `src/modules/users/user.routes.ts` ([09:36] Larissa).
- Validação de entrada com schemas Zod no padrão de `src/modules/customers/customer.schemas.ts`, incluindo a regra de URL `https` ([09:23] Sofia).
- UUID em todas as novas tabelas ([09:51] Larissa).
- Integração com `OrderService` por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o client da transação, em vez de injetar um repository inteiro ([09:41] Bruno, [09:41] Diego).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Injetar `WebhookRepository` no `OrderService`** | O Bruno levantou como opção ([09:41] Bruno). Perdeu para uma função pura que recebe o `tx`, que acopla menos e não exige mudar o construtor nem a montagem em `src/app.ts` ([09:41] Diego). |
| **Estrutura própria para o módulo** (logger, erros ou pastas próprias) | Alternativa plausível, rejeitada pela decisão explícita de reuso ([09:29] Bruno: "Não vamos botar nada novo"; [09:30] Larissa). Aumentaria a carga cognitiva e duplicaria o tratamento de erro. |

## Consequências

**Positivas**
- Curva de aprendizado quase zero para o time de Pedidos. Revisão e testes seguem o padrão de `tests/orders.test.ts`.
- O tratamento de erro e a resposta HTTP (`{ error: { code, message, details } }`) ficam consistentes com o resto da API.
- `errorMiddleware` e `auth.middleware` não mudam.

**Negativas / trade-offs**
- O padrão de módulo é pensado para a camada HTTP. O worker (`webhook.processor.ts`) não passa pelo `errorMiddleware` e precisa tratar e logar os próprios erros.
- O Pino não traz métricas nem tracing. A observabilidade desta fase vai depender de logs estruturados e campos correlacionáveis, sem ferramenta nova (ver [FDD](../FDD.md#9-observabilidade)).
- O `PrismaClient` não é compartilhado: o worker cria uma instância própria com `createPrismaClient()` de `src/config/database.ts` ([09:30] Bruno).
