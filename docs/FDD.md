# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Webhooks outbound de mudança de status de pedido |
| **Status** | Em revisão, junto com o RFC. Pendências para fechar antes de codar: RFC-QA-05, 06, 07, 09 e 10 |
| **Proposta de origem** | [RFC-001](RFC.md) |
| **Decisões** | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md) |
| **Rastreabilidade** | [TRACKER.md](TRACKER.md). Os IDs entre colchetes, como `FDD-FLX-01`, aparecem lá. |

> **Convenção de caminhos.** Arquivos marcados com **(novo)** ainda não existem e serão criados na implementação. Todos os demais caminhos existem hoje no repositório.

---

## 1. Contexto e motivação técnica

Três clientes B2B fazem polling em `GET /api/v1/orders` para descobrir mudanças de status ([09:00] Marcos). O OMS não tem nenhum mecanismo de evento ou notificação. A única fonte de verdade das transições é `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que já roda numa `prisma.$transaction` com quatro passos: validação de transição (`canTransition` em `src/modules/orders/order.status.ts`), débito/reposição de estoque, `update` em `orders` e `insert` em `order_status_history`.

A feature precisa **emitir um evento por transição** sem alongar essa transação com I/O externo ([09:04] Bruno) e sem que exista transição sem evento ([09:40] Bruno).

## 2. Objetivos técnicos

| ID | Objetivo | Origem |
|---|---|---|
| FDD-OBJ-01 | Latência entre o commit da transição e a primeira tentativa de entrega **< 10s** (alvo de projeto: ≤ ~2s + tempo de envio) | [09:02] Marcos, [09:10] Larissa |
| FDD-OBJ-02 | **Zero transições sem evento** e zero eventos de transições que sofreram rollback | [09:06] Diego, [09:40] Bruno |
| FDD-OBJ-03 | Nenhuma chamada HTTP externa dentro da transação de `changeStatus` | [09:04] Bruno |
| FDD-OBJ-04 | Entrega at-least-once, com `event_id` estável entre tentativas | [09:24] Diego |
| FDD-OBJ-05 | Nenhuma infraestrutura nova e nenhuma alteração em `errorMiddleware`/`auth.middleware` | [09:07] Diego, [09:29] Bruno |

## 3. Escopo e exclusões

**Incluso:** modelagem (endpoints, outbox, deliveries, DLQ), `publishWebhookEvent` no `changeStatus`, worker em `src/worker.ts` (novo), assinatura HMAC, retry/DLQ, CRUD de configuração, rotação de secret, histórico de entregas, replay admin.

**Excluído nesta fase:**

| ID | Exclusão | Origem |
|---|---|---|
| FDD-EXC-01 | Webhooks de entrada (inbound) | [09:02] Marcos |
| FDD-EXC-02 | Aviso por e-mail ao cliente sobre falhas | [09:37] Larissa |
| FDD-EXC-03 | Rate limiting de saída (só observar) | [09:39] Larissa |
| FDD-EXC-04 | Dashboard visual para o cliente | [09:40] Larissa |
| FDD-EXC-05 | Arquivamento/limpeza de linhas entregues (~30 dias) | [09:08] Diego |
| FDD-EXC-06 | Múltiplos workers / ordenação global | [09:13] Diego |
| FDD-EXC-07 | Itens do pedido no payload | [09:43] Diego |

---

## 4. Modelo de dados

Alterações em `prisma/schema.prisma`, gerando uma nova migration em `prisma/migrations/`. Todos os IDs seguem o padrão do projeto: `String @id @default(uuid()) @db.Char(36)` ([09:51] Larissa).

```prisma
enum WebhookOutboxStatus {
  PENDING      // aguardando primeira tentativa ou replay
  PROCESSING   // reservado pelo worker no ciclo atual
  FAILED       // última tentativa falhou; aguarda next_attempt_at
  DELIVERED    // cliente respondeu 2xx
}

model WebhookEndpoint {                                  // [09:21] Bruno/Sofia
  id                       String        @id @default(uuid()) @db.Char(36)
  customerId               String        @db.Char(36)
  url                      String        @db.VarChar(2048)
  secret                   String        @db.VarChar(128)
  previousSecret           String?       @db.VarChar(128) // carência de rotação
  previousSecretExpiresAt  DateTime?                      // now() + 24h
  events                   Json          // OrderStatus[] assinados  [09:33] Marcos
  active                   Boolean       @default(true)
  createdAt                DateTime      @default(now())
  updatedAt                DateTime      @updatedAt

  customer   Customer            @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {                                    // [09:06] Diego
  id             String              @id @default(uuid()) @db.Char(36) // = event_id
  webhookId      String              @db.Char(36)
  orderId        String              @db.Char(36)
  eventType      String              @db.VarChar(64)   // "order.status_changed"
  payload        Json                                   // snapshot [09:52]
  status         WebhookOutboxStatus @default(PENDING)
  attempts       Int                 @default(0)
  nextAttemptAt  DateTime            @default(now())
  lastError      String?             @db.VarChar(500)
  deliveredAt    DateTime?
  createdAt      DateTime            @default(now())
  updatedAt      DateTime            @updatedAt

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([status])                     // [09:08] Diego
  @@index([createdAt])                  // [09:08] Diego
  @@index([status, nextAttemptAt])      // seleção do ciclo do worker
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDelivery {                                  // PROPOSTA: a reunião pediu o histórico [09:34] Marcos, mas não definiu tabela
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  eventId        String   @db.Char(36)
  attemptNumber  Int
  success        Boolean
  requestPayload Json
  responseStatus Int?
  responseBody   String?  @db.Text   // "response" [09:34] Marcos
  durationMs     Int?
  errorCode      String?  @db.VarChar(64)
  createdAt      DateTime @default(now())

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@index([eventId])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {                                // [09:18] Diego
  id            String    @id @default(uuid()) @db.Char(36)
  eventId       String    @db.Char(36)
  webhookId     String    @db.Char(36)   // sem FK: a linha sobrevive à remoção do endpoint (ver §6.4)
  orderId       String    @db.Char(36)
  payload       Json
  failureReason String    @db.VarChar(500)
  attempts      Int
  failedAt      DateTime  @default(now())
  replayedAt    DateTime?
  replayedById  String?   @db.Char(36)   // auditoria [09:36] Sofia

  @@index([failedAt])
  @@index([eventId])
  @@map("webhook_dead_letter")
}
```

No model `Customer` existente entra só a relação inversa `webhookEndpoints WebhookEndpoint[]`. Nenhuma coluna existente muda.

---

## 5. Fluxos detalhados

### 5.1 [FDD-FLX-01] Criação do evento na outbox (dentro de `changeStatus`)

```mermaid
sequenceDiagram
  participant C as OrderController
  participant S as OrderService.changeStatus
  participant P as publishWebhookEvent
  participant DB as MySQL (tx)
  C->>S: changeStatus(id, {toStatus, reason}, userId)
  S->>DB: BEGIN; findUnique(order + items)
  S->>S: canTransition / estoque (inalterado)
  S->>DB: update orders; insert order_status_history
  S->>P: publishWebhookEvent(tx, order, from, to)
  P->>DB: SELECT webhook_endpoints WHERE customerId AND active
  P->>P: filtra endpoints cujo events contém `to`
  alt nenhum endpoint interessado
    P-->>S: return (nenhuma linha) [09:34]
  else N endpoints
    P->>DB: INSERT webhook_outbox × N (payload snapshot)
  end
  S->>DB: COMMIT (qualquer erro acima → ROLLBACK de tudo) [09:40]
```

1. `publishWebhookEvent` é chamada **depois** do `insert` em `order_status_history` e **antes** do `findUnique` final (`refreshed`), sempre com o mesmo `tx` ([09:41] Bruno).
2. Os endpoints são lidos com `tx.webhookEndpoint.findMany({ where: { customerId: order.customerId, active: true } })`.
3. O filtro é feito em memória: `(endpoint.events as OrderStatus[]).includes(to)`. O cast é necessário porque a coluna é `Json`, e o conteúdo já foi validado por `z.nativeEnum(OrderStatus)` na gravação ([09:33] Marcos, [09:34] Bruno).
4. Para cada endpoint interessado é gerado um `id = uuidv4()`, usando o pacote `uuid` que já está no `package.json`. Esse `id` é o `event_id`, montado no payload (§6.8). A linha entra com `status = PENDING` e `nextAttemptAt = now()`.
5. As inserções usam `tx.webhookOutbox.createMany`. Se falhar, a exceção sobe e o Prisma faz rollback da transação inteira. O cliente da API recebe 500 `INTERNAL_SERVER_ERROR` do `errorMiddleware`, e o status do pedido **não muda** ([09:40] Bruno, [09:41] Diego).

> Só `changeStatus` emite evento. A criação do pedido (`create`, `fromStatus = null`) **não** emite nesta fase (RFC-QA-09).

### 5.2 [FDD-FLX-02] Processamento pelo worker (ciclo de polling)

Executado por `src/modules/webhooks/webhook.processor.ts` (novo), disparado por `src/worker.ts` (novo) a cada `WEBHOOK_POLL_INTERVAL_MS` (padrão **2000**, [09:09] Diego). O Bruno deixou em aberto o nome do arquivo (`webhook.worker.ts` ou `webhook.processor.ts`, [09:28] Bruno); este FDD adota `webhook.processor.ts`.

1. **Recuperação no startup:** linhas em `PROCESSING` voltam para `PENDING`. Como há um único worker ([09:12] Diego), qualquer `PROCESSING` no boot é resto de um processo que morreu no meio de um ciclo.
2. **Seleção do lote:** busca linhas com `status IN (PENDING, FAILED)` e `nextAttemptAt <= now()`, ordenadas por `createdAt ASC`, com `take = WEBHOOK_BATCH_SIZE` ("batch pequeno", [09:08] Diego; o valor 20 é sugestão, ver §8.1).
3. **Reserva:** `updateMany` para `PROCESSING` usando os ids do lote.
4. **Agrupamento por endpoint (proposta, ver RFC-QA-11).** O lote é agrupado por `webhookId`. Cada grupo roda como uma cadeia assíncrona própria, **em ordem de `createdAt`**. O worker guarda o conjunto de `webhookId` com cadeia em andamento e **não inicia outra cadeia para o mesmo endpoint**. Assim, um cliente lento ou fora do ar prende só a própria cadeia, e os eventos dos outros clientes seguem no próximo ciclo de 2s. O ciclo **não espera** as cadeias terminarem. Isso preserva o argumento de [09:04] Bruno (cliente lento não trava os outros) dentro do worker e continua sendo um único processo, como decidido ([09:12] Diego).
5. **Para cada evento da cadeia**, dentro de um `try/catch` próprio:
   1. carrega o `WebhookEndpoint`. Se está `active=false` ou foi removido, a linha vai para a DLQ com o motivo `WEBHOOK_ENDPOINT_INACTIVE`, sem enviar;
   2. serializa o `payload` com `JSON.stringify`. Se `Buffer.byteLength > 65536`, vai para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, **sem retry e sem envio** ([09:23] Sofia, [09:24] Diego);
   3. calcula `X-Signature` (§6.9) e monta os headers;
   4. faz o `POST` com `fetch` nativo do Node ≥ 20 (`engines` em `package.json`) e `signal: AbortSignal.timeout(WEBHOOK_HTTP_TIMEOUT_MS)`, padrão 10.000 ms ([09:42] Diego);
   5. grava uma linha em `webhook_deliveries` com o resultado da tentativa;
   6. **se o status for 2xx**, passa para `DELIVERED`, `deliveredAt = now()` e `attempts += 1`;
   7. **em qualquer outro resultado**, aplica a política de retry (§5.3);
   8. **se uma exceção inesperada escapar** (por exemplo, falha ao gravar `webhook_deliveries`), o `catch` devolve a linha a `FAILED` com `lastError = INTERNAL_ERROR` e `nextAttemptAt = now()`, e registra o log `webhook_event_error`. Nenhuma linha fica presa em `PROCESSING` fora de um crash do processo.
6. **Ordem de entrega.** Dentro de um endpoint, os eventos saem na ordem de `createdAt` enquanto não houver falha. Um evento em `FAILED` espera o seu `nextAttemptAt`, e um evento posterior do mesmo pedido pode ser entregue antes dele. Isso **não** está resolvido na reunião e é a questão RFC-QA-07.
7. O log `webhook_worker_cycle` (§9) é emitido a cada ciclo. O próximo ciclo começa sempre `WEBHOOK_POLL_INTERVAL_MS` depois do anterior, sem sobreposição de seleção de lote, porque o ciclo só seleciona, reserva e dispara cadeias.

### 5.3 [FDD-FLX-03] Retry com backoff

| Tentativa que falhou | `attempts` após a falha | Próxima tentativa em |
|---|---|---|
| Envio inicial | 1 | +1 min |
| Retentativa 1 | 2 | +5 min |
| Retentativa 2 | 3 | +30 min |
| Retentativa 3 | 4 | +2 h |
| Retentativa 4 | 5 | +12 h |
| Retentativa 5 | 6 | → DLQ (§5.4) |

Fonte: [09:17] Diego e [09:17] Larissa. A transcrição é ambígua aqui: o resumo final fala em "**total** 5 tentativas" ([09:48] Larissa), mas os 5 intervalos citados somam ~14h36min ("quase 15 horas entre primeira falha e última tentativa", [09:17] Diego), o que só fecha com envio inicial + 5 retentativas. Este FDD adota a segunda leitura, pendente de confirmação em RFC-QA-06. Se a confirmação for "5 envios no total", basta tirar a última linha da tabela, ou seja, trocar a constante `MAX_ATTEMPTS` de 6 para 5.

Progressão: backoff exponencial com intervalos fixos. Implementação: `const BACKOFF_SCHEDULE_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]`. Na falha, `status = FAILED`, `lastError = <código>`, `nextAttemptAt = now() + BACKOFF_SCHEDULE_MS[attempts - 1]`.

Contam como falha ([09:42] Diego): timeout (`WEBHOOK_DELIVERY_TIMEOUT`), erro de rede/DNS/TLS (`WEBHOOK_DELIVERY_NETWORK_ERROR`) e qualquer status fora de 2xx (`WEBHOOK_DELIVERY_HTTP_ERROR`). Redirecionamentos não são seguidos (`redirect: 'manual'`), então 3xx conta como falha.

### 5.4 [FDD-FLX-04] Movimento para a DLQ

Feito numa única `prisma.$transaction` no worker:
1. `INSERT webhook_dead_letter` com `eventId = outbox.id`, `payload`, `failureReason` (código e mensagem curta), `attempts` e `failedAt` ([09:18] Diego);
2. `DELETE webhook_outbox` da linha, o que mantém a outbox limpa ([09:18] Diego);
3. log `warn` `webhook_dead_lettered`.

### 5.5 [FDD-FLX-05] Replay manual da DLQ

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` (§6.7), só para ADMIN ([09:36] Sofia). Em uma transação:
1. carrega a linha da DLQ. Se não existir, retorna 404 `WEBHOOK_DEAD_LETTER_NOT_FOUND`. Se `replayedAt` já estiver preenchido, retorna 409 `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`;
2. verifica se o endpoint existe e está ativo. Se não, retorna 422 `WEBHOOK_ENDPOINT_INACTIVE`;
3. recoloca o evento na outbox como `PENDING`, com **o mesmo `id`/`event_id`** e o mesmo `payload`, `attempts = 0` e `nextAttemptAt = now()` ([09:18] Diego; [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md));
4. preenche `replayedAt = now()` e `replayedById = req.user.id`;
5. registra o log `info` `webhook_replayed` com `adminUserId`, `deadLetterId` e `eventId` ([09:36] Sofia).

### 5.6 [FDD-FLX-06] Rotação de secret

`POST /api/v1/webhooks/:id/rotate-secret` (§6.5) ([09:21] Sofia):
1. se `previousSecretExpiresAt > now()`, retorna 409 `WEBHOOK_SECRET_ROTATION_IN_PROGRESS`. É uma proposta: impede que uma segunda rotação dentro da janela derrube a secret antiga antes das 24h;
2. `previousSecret = secret`, `previousSecretExpiresAt = now() + 24h`, `secret = novaSecret()`;
3. a resposta devolve a nova secret **uma única vez**.

Geração da secret: `crypto.randomBytes(32).toString('hex')`, com `node:crypto` e sem dependência nova. Está sujeita à revisão da Sofia ([09:46] Sofia).

---

## 6. Contratos públicos

Todas as rotas ficam sob `/api/v1` (`src/routes/index.ts`), exigem `Authorization: Bearer <jwt>` (`authenticate`) e usam o envelope de erro padrão `{ "error": { "code", "message", "details?" } }` (`src/middlewares/error.middleware.ts`). O CRUD aceita **qualquer role autenticada** ([09:37] Sofia). O replay exige **ADMIN** ([09:36] Larissa).

O `customerId` **não** vem do JWT e é "passado no body ou no path" ([09:32] Larissa). Por isso vai no **body** do `POST` (criação) e no **path** do `GET` de listagem (`/customers/:customerId/webhooks`), já que um `GET` não tem body. A validação do parâmetro segue `orderIdParamSchema` (`z.string().uuid()`) em `src/modules/orders/order.schemas.ts`. A escolha exata de body/path por rota ainda está pendente de confirmação em RFC-QA-05.

### 6.1 [FDD-CONTRATO-01] `POST /api/v1/webhooks`: cadastrar webhook ([09:31] Marcos)

**Request**
```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "6f1c2a9e-3b7d-4c1a-9a51-0d2f6a7b8c90",
  "url": "https://hooks.example.com/oms/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```

**Response 201**
```json
{
  "id": "0b7e9c3d-1f2a-4e5b-8c6d-7a8b9c0d1e2f",
  "customerId": "6f1c2a9e-3b7d-4c1a-9a51-0d2f6a7b8c90",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "9f2c4e7a1b3d5f60718293a4b5c6d7e8f9012a3b4c5d6e7f8091a2b3c4d5e6f7",
  "createdAt": "2026-11-26T12:00:00.000Z",
  "updatedAt": "2026-11-26T12:00:00.000Z"
}
```

| Status | Quando |
|---|---|
| 201 | Webhook criado. A `secret` aparece **só** nesta resposta e na de rotação. Nas demais rotas ela fica de fora (proposta, sujeita a RFC-QA-08). |
| 400 | Falha de schema: `url` sem `https` (detalhe `WEBHOOK_INVALID_URL`), `events` vazio ou com status fora de `OrderStatus`, `customerId` que não é UUID |
| 401 | JWT ausente ou inválido |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` |

Schema (`webhook.schemas.ts`, novo):
```ts
// [09:23] Sofia. O Zod 3 executa o refine mesmo quando .url() falha, e new URL() lançaria
// TypeError (virando 500). Por isso o try/catch dentro do superRefine.
const httpsUrl = z.string().max(2048).superRefine((value, ctx) => {
  let ok = false;
  try { ok = new URL(value).protocol === 'https:'; } catch { ok = false; }
  if (!ok) ctx.addIssue({ code: z.ZodIssueCode.custom, message: 'WEBHOOK_INVALID_URL: url must be a valid https URL' });
});
export const createWebhookSchema = z.object({
  customerId: z.string().uuid(),
  url: httpsUrl,
  events: z.array(z.nativeEnum(OrderStatus)).min(1),   // [09:33] Marcos
});
```

### 6.2 [FDD-CONTRATO-02] `GET /api/v1/customers/:customerId/webhooks`: listar webhooks de um customer ([09:33] Bruno, [09:32] Larissa)

**Request**
```http
GET /api/v1/customers/6f1c2a9e-3b7d-4c1a-9a51-0d2f6a7b8c90/webhooks?page=1&pageSize=20
Authorization: Bearer eyJhbGciOi...
```
`page` e `pageSize` seguem os limites de `listOrdersQuerySchema` (`pageSize` de 1 a 100, padrão 20).

**Response 200**, com paginação de `src/shared/http/response.ts`:
```json
{
  "data": [
    {
      "id": "0b7e9c3d-1f2a-4e5b-8c6d-7a8b9c0d1e2f",
      "customerId": "6f1c2a9e-3b7d-4c1a-9a51-0d2f6a7b8c90",
      "url": "https://hooks.atlascomercial.com.br/oms/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "previousSecretExpiresAt": null,
      "createdAt": "2026-11-26T12:00:00.000Z",
      "updatedAt": "2026-11-26T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
| Status | Quando |
|---|---|
| 200 | Lista, possivelmente vazia |
| 400 | `customerId` do path que não é UUID, ou `page`/`pageSize` inválidos |
| 401 | Não autenticado |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` |

### 6.3 [FDD-CONTRATO-03] `PATCH /api/v1/webhooks/:id`: editar ([09:33] Bruno)

**Request**
```http
PATCH /api/v1/webhooks/0b7e9c3d-1f2a-4e5b-8c6d-7a8b9c0d1e2f
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```
Campos aceitos (todos opcionais, pelo menos um): `url` (https), `events` (`min(1)`), `active`. `customerId` e `secret` **não** são editáveis por aqui.

**Response 200** (mesmo formato de item do §6.2, sem `secret`)
```json
{
  "id": "0b7e9c3d-1f2a-4e5b-8c6d-7a8b9c0d1e2f",
  "customerId": "6f1c2a9e-3b7d-4c1a-9a51-0d2f6a7b8c90",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "secretRotationExpiresAt": null,
  "createdAt": "2026-11-26T12:00:00.000Z",
  "updatedAt": "2026-11-26T14:10:00.000Z"
}
```

| Status | Quando |
|---|---|
| 200 | Atualizado. A mudança vale para eventos **inseridos depois**, já que o filtro acontece na inserção ([09:34] Bruno). |
| 400 | Schema inválido ou URL `http` |
| 404 | `WEBHOOK_NOT_FOUND` |

### 6.4 [FDD-CONTRATO-04] `DELETE /api/v1/webhooks/:id`: remover ([09:33] Bruno)

**Request**
```http
DELETE /api/v1/webhooks/0b7e9c3d-1f2a-4e5b-8c6d-7a8b9c0d1e2f
Authorization: Bearer eyJhbGciOi...
```

**Response 204:** sem corpo. **Response 404:**
```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found" } }
```

| Status | Quando |
|---|---|
| 204 | Removido. Numa única transação, as linhas `PENDING`/`FAILED`/`PROCESSING` desse webhook na outbox são **movidas para a DLQ** com motivo `WEBHOOK_ENDPOINT_INACTIVE` (nenhum evento some em silêncio), e só então o endpoint é apagado. O histórico em `webhook_deliveries` sai em cascade. A DLQ guarda a evidência. |
| 404 | `WEBHOOK_NOT_FOUND` |

### 6.5 [FDD-CONTRATO-05] `POST /api/v1/webhooks/:id/rotate-secret`: rotacionar secret ([09:21] Sofia)

**Request** (sem corpo)
```http
POST /api/v1/webhooks/0b7e9c3d-1f2a-4e5b-8c6d-7a8b9c0d1e2f/rotate-secret
Authorization: Bearer eyJhbGciOi...
```

**Response 200**
```json
{
  "id": "0b7e9c3d-1f2a-4e5b-8c6d-7a8b9c0d1e2f",
  "secret": "4a6c8e0f2b4d6f8091a3c5e7f9b1d3f5a7c9e1b3d5f7a9c1e3b5d7f9a1c3e5b7",
  "previousSecretExpiresAt": "2026-11-27T12:00:00.000Z"
}
```
| Status | Quando |
|---|---|
| 200 | Nova secret gerada. A anterior continua aceita por 24h. |
| 404 | `WEBHOOK_NOT_FOUND` |
| 409 | `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` |

### 6.6 [FDD-CONTRATO-06] `GET /api/v1/webhooks/:id/deliveries`: histórico de entregas ([09:34] Marcos)

**Request**
```http
GET /api/v1/webhooks/0b7e9c3d-1f2a-4e5b-8c6d-7a8b9c0d1e2f/deliveries?limit=100
Authorization: Bearer eyJhbGciOi...
```
`limit` vai de 1 a 100 (padrão 100), com ordenação por `createdAt DESC`. O número 100 vem de um exemplo do Marcos ("tipo, os últimos 100", [09:34]) e é proposta de implementação (§8.1).

**Response 200**
```json
{
  "data": [
    {
      "id": "a1b2c3d4-0000-4000-8000-000000000001",
      "eventId": "5d1e7c2b-9a8f-4b6e-a3d2-1c0b9a8f7e6d",
      "attemptNumber": 2,
      "success": true,
      "responseStatus": 200,
      "durationMs": 184,
      "errorCode": null,
      "requestPayload": { "event_id": "5d1e7c2b-9a8f-4b6e-a3d2-1c0b9a8f7e6d", "event_type": "order.status_changed", "...": "..." },
      "responseBody": "{\"received\":true}",
      "createdAt": "2026-11-26T12:01:03.120Z"
    },
    {
      "id": "a1b2c3d4-0000-4000-8000-000000000000",
      "eventId": "5d1e7c2b-9a8f-4b6e-a3d2-1c0b9a8f7e6d",
      "attemptNumber": 1,
      "success": false,
      "responseStatus": 503,
      "durationMs": 92,
      "errorCode": "WEBHOOK_DELIVERY_HTTP_ERROR",
      "requestPayload": { "...": "..." },
      "responseBody": "Service Unavailable",
      "createdAt": "2026-11-26T12:00:02.410Z"
    }
  ]
}
```
| Status | Quando |
|---|---|
| 200 | Últimas N tentativas, com sucesso/falha, payload, resposta e tempo ([09:34] Marcos). Sem verificação de dono: ver §6.10 |
| 404 | `WEBHOOK_NOT_FOUND` |

### 6.7 [FDD-CONTRATO-07] `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: replay de DLQ ([09:18] Diego, [09:36] Sofia)

Middlewares: `authenticate`, `requireRole('ADMIN')` e `validate({ params })`.

**Request** (sem corpo; o JWT precisa ser de um usuário `ADMIN`)
```http
POST /api/v1/admin/webhooks/dead-letter/c3d4e5f6-0000-4000-8000-00000000abcd/replay
Authorization: Bearer eyJhbGciOi...
```

**Response 202**
```json
{
  "deadLetterId": "c3d4e5f6-0000-4000-8000-00000000abcd",
  "eventId": "5d1e7c2b-9a8f-4b6e-a3d2-1c0b9a8f7e6d",
  "status": "PENDING",
  "replayedAt": "2026-11-26T15:30:00.000Z",
  "replayedById": "e2f3a4b5-0000-4000-8000-0000000000ad"
}
```
| Status | Quando |
|---|---|
| 202 | Evento de volta na outbox. A entrega é assíncrona, pelo worker. |
| 401 / 403 | Não autenticado / role diferente de ADMIN (`FORBIDDEN`, vindo do `requireRole` existente) |
| 404 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` |
| 409 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` |
| 422 | `WEBHOOK_ENDPOINT_INACTIVE` |

> A **listagem** da DLQ não foi discutida na reunião. Nesta fase, o `:id` sai do log `webhook_dead_lettered` (§9). Um `GET` admin seria uma extensão natural, a propor na revisão.

### 6.8 [FDD-CONTRATO-08] Contrato outbound: requisição enviada ao cliente ([09:43]–[09:45])

```http
POST https://hooks.atlascomercial.com.br/oms/orders
Content-Type: application/json
X-Event-Id: 5d1e7c2b-9a8f-4b6e-a3d2-1c0b9a8f7e6d
X-Webhook-Id: 0b7e9c3d-1f2a-4e5b-8c6d-7a8b9c0d1e2f
X-Timestamp: 2026-11-26T12:00:02.318Z
X-Signature: sha256=3f5a9c0e7b...e41

{
  "event_id": "5d1e7c2b-9a8f-4b6e-a3d2-1c0b9a8f7e6d",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-26T12:00:00.517Z",
  "order_id": "9a8b7c6d-5e4f-4a3b-9c2d-1e0f9a8b7c6d",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1c2a9e-3b7d-4c1a-9a51-0d2f6a7b8c90",
  "total_cents": 158990
}
```
- `timestamp` (corpo) é o instante da transição, gravado no snapshot. `X-Timestamp` é o instante **deste envio**, então muda a cada tentativa ([09:44] Diego).
- Os campos do corpo vêm de `Order` em `prisma/schema.prisma` (`id`, `orderNumber`, `customerId`, `totalCents`), convertidos para snake_case. Os itens não vão ([09:43] Diego).
- **Resposta esperada do cliente:** qualquer 2xx em até 10s. O corpo da resposta não decide sucesso ou falha, mas fica guardado no histórico de entregas ([09:34] Marcos).

### 6.9 [FDD-CONTRATO-09] Assinatura `X-Signature`

```ts
const body = JSON.stringify(outbox.payload);                       // exatamente os bytes enviados
const sig = (s: string) => 'sha256=' + createHmac('sha256', s).update(body).digest('hex');
const header = [sig(endpoint.secret),
  ...(endpoint.previousSecret && endpoint.previousSecretExpiresAt! > now ? [sig(endpoint.previousSecret)] : []),
].join(',');
```
- Algoritmo HMAC-SHA256 sobre o **corpo** ([09:22] Sofia).
- O prefixo `sha256=` e o formato do header são **proposta** (a Sofia disse só "um header tipo X-Signature", [09:20] Sofia).
- Durante a carência de 24h, o header traz **duas** assinaturas separadas por vírgula, e o cliente aceita se qualquer uma bater. **É uma proposta** para a regra "a antiga fica válida em paralelo" ([09:21] Sofia) e depende da revisão (RFC-QA-08).
- Orientação ao cliente a publicar no portal: deduplicar por `X-Event-Id` ([09:26] Marcos se comprometeu a documentar a dedup) e, como boa prática proposta aqui, comparar a assinatura com `timingSafeEqual`.

### 6.10 Autorização das rotas de configuração (ponto para a revisão de segurança)

A reunião liberou "o resto do CRUD de configuração" para qualquer role autenticada, "por enquanto" ([09:36] Marcos, [09:37] Sofia). Duas rotas deste FDD vão além do CRUD e **não foram discutidas**: `rotate-secret` (§6.5), que devolve a secret em texto, e `deliveries` (§6.6), que devolve payloads e respostas. Como o `customerId` não vem do JWT ([09:32] Larissa), nenhuma rota confere se o usuário autenticado pertence ao customer do webhook. Portanto, **qualquer usuário autenticado poderia rotacionar a secret ou ler o histórico de qualquer customer**. Este FDD segue a decisão literal da reunião e registra o ponto como RFC-QA-04 e RFC-QA-08 para a revisão da Sofia.

---

## 7. Matriz de erros

As classes ficam em `src/modules/webhooks/webhook.errors.ts` (novo) e estendem as classes de `src/shared/errors/http-errors.ts`/`app-error.ts`. São exportadas pelo próprio módulo, e `src/shared/errors/index.ts` não precisa mudar. **Observação de código:** `NotFoundError` fixa o código `NOT_FOUND`, por isso os 404 do módulo estendem `AppError` diretamente, passando `statusCode = 404` e um código `WEBHOOK_*`.

### 7.1 Erros da API (respondidos pelo `errorMiddleware`)

| ID | Código | HTTP | Classe base | Quando | Rotas |
|---|---|---|---|---|---|
| FDD-ERR-01 | `WEBHOOK_NOT_FOUND` | 404 | `AppError` | `:id` de webhook inexistente | PATCH, DELETE, rotate, deliveries |
| FDD-ERR-02 | `WEBHOOK_INVALID_URL` | 400 | via Zod → `ValidationError` | URL que não é `https` ou malformada. O código vai em `details[].message`, porque `validate.middleware.ts` embrulha todo erro Zod como `VALIDATION_ERROR` | POST, PATCH |
| FDD-ERR-03 | `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `AppError` | `customerId` inexistente | POST |
| FDD-ERR-04 | `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | 409 | `ConflictError` | Nova rotação dentro das 24h | rotate |
| FDD-ERR-05 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `AppError` | `:id` da DLQ inexistente | replay |
| FDD-ERR-06 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `ConflictError` | `replayedAt` já preenchido | replay |
| FDD-ERR-07 | `WEBHOOK_ENDPOINT_INACTIVE` | 422 | `UnprocessableEntityError` | Endpoint removido ou `active=false` no replay | replay |

### 7.2 Erros de entrega (worker), gravados em `webhook_deliveries.errorCode`, `webhook_outbox.lastError` ou `webhook_dead_letter.failureReason`

| ID | Código | Retry? | Quando |
|---|---|---|---|
| FDD-ERR-08 | `WEBHOOK_DELIVERY_TIMEOUT` | Sim | Sem resposta em 10s ([09:42] Diego) |
| FDD-ERR-09 | `WEBHOOK_DELIVERY_HTTP_ERROR` | Sim | Status fora de 2xx (inclui 3xx, 4xx e 5xx) |
| FDD-ERR-10 | `WEBHOOK_DELIVERY_NETWORK_ERROR` | Sim | DNS, conexão recusada, falha de TLS |
| FDD-ERR-11 | `WEBHOOK_PAYLOAD_TOO_LARGE` | **Não**, vai direto para a DLQ | Corpo > 64KB ([09:24] Larissa) |
| FDD-ERR-12 | `WEBHOOK_SECRET_REQUIRED` | **Não**, vai direto para a DLQ | Nome citado na reunião ([09:28] Bruno), sem cenário definido. Uso proposto, apenas defensivo: o worker encontra o endpoint sem secret para assinar (a coluna é obrigatória, então só ocorreria por dado corrompido) |
| FDD-ERR-13 | `WEBHOOK_ENDPOINT_INACTIVE` | **Não**, vai direto para a DLQ | Endpoint desativado (`active=false`) entre a inserção e o envio. Na remoção, as linhas já foram movidas para a DLQ por `DELETE` (§6.4) |
| FDD-ERR-14 | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — | Retentativas esgotadas, motivo final na DLQ ([09:15] Diego) |

> Uma falha ao inserir na outbox **dentro** de `changeStatus` não tem código próprio. É um erro do Prisma que vira 500 `INTERNAL_SERVER_ERROR` no `errorMiddleware` e dá rollback na mudança de status (FDD-FLX-01, passo 5).

---

## 8. Estratégias de resiliência

| ID | Estratégia | Valor | Origem |
|---|---|---|---|
| FDD-RES-01 | Timeout por requisição | 10s (`WEBHOOK_HTTP_TIMEOUT_MS`, `AbortSignal.timeout`) | [09:42] Diego |
| FDD-RES-02 | Retry com backoff exponencial, progressão fixa | 1m, 5m, 30m, 2h, 12h | [09:17] Larissa |
| FDD-RES-03 | Teto de tentativas e depois DLQ | "5 tentativas" ([09:17], [09:48] Larissa). Leitura adotada: 1 envio + 5 retentativas, a confirmar (RFC-QA-06) → `webhook_dead_letter` | [09:15] Diego, [09:18] Diego |
| FDD-RES-04 | Fallback operacional | Replay manual via endpoint ADMIN. Sem fallback automático: o e-mail ficou para a próxima fase. | [09:18] Diego, [09:37] Larissa |
| FDD-RES-05 | Isolamento de falhas | Worker em processo separado. A API não depende dele, e ele não depende da API. | [09:11] Diego |
| FDD-RES-06 | Durabilidade | O evento é persistido na transação. Se o worker cair, os eventos acumulam sem perda. As linhas `PROCESSING` são recuperadas no boot, e exceções por evento devolvem a linha a `FAILED` (§5.2 passo 5.8). | [09:06] Diego |
| FDD-RES-07 | Shutdown gracioso | `src/worker.ts` (novo) trata `SIGINT`/`SIGTERM` como `src/server.ts`, **com um comportamento novo**: para de iniciar cadeias, aguarda as cadeias em andamento (cada uma limitada pelo timeout de 10s) e só então chama `prisma.$disconnect()`. O `server.ts` atual não aguarda nada: chama `$disconnect()` e `process.exit(0)` | `src/server.ts` |
| FDD-RES-08 | Limite de payload | 64KB, com erro e sem truncar | [09:24] Larissa |
| FDD-RES-09 | Tamanho do lote | `WEBHOOK_BATCH_SIZE`. A reunião só disse "batch pequeno" ([09:08] Diego). O valor 20 é sugestão e deve ser validado com o Diego (§8.1) | [09:08] Diego |

Configuração em `src/config/env.ts`, com novas chaves opcionais e defaults no schema Zod: `WEBHOOK_POLL_INTERVAL_MS=2000`, `WEBHOOK_HTTP_TIMEOUT_MS=10000`, `WEBHOOK_BATCH_SIZE=20`.

### 8.1 Propostas deste FDD sem origem direta na reunião

Valores e regras que o FDD precisou escolher para fechar o desenho. Nenhum veio de uma fala; todos devem ser confirmados na sessão de revisão com Bruno e Diego ([09:50] Larissa), e a revisão de segurança da Sofia cobre os de segurança.

| Item | Valor proposto | Onde aparece |
|---|---|---|
| Tamanho do lote | 20 (a reunião disse "batch pequeno", [09:08] Diego) | §5.2, §8 |
| Agrupamento por endpoint com cadeias independentes | Resposta ao risco de um cliente lento prender o ciclo (RFC-QA-11) | §5.2 |
| Prefixo `sha256=` e duas assinaturas durante a carência | Formato do header e da validação na janela de 24h | §6.9 |
| Códigos HTTP 202 (replay), 409 (rotação duplicada), 422 (endpoint inativo) | Semântica de resposta | §6 |
| `redirect: 'manual'`, com 3xx contado como falha | Evitar seguir redirecionamentos | §5.3 |
| `limit` de 1 a 100, padrão 100 | A reunião deu 100 como exemplo ([09:34] Marcos) | §6.6 |
| `VarChar(2048)` na URL, `VarChar(128)` na secret, `randomBytes(32)` | Dimensionamento e geração | §4, §5.6 |
| Payload acima de 64KB e endpoint inativo vão para a DLQ | A reunião só disse "erra" ([09:23] Sofia) | §5.2 |
| Tabela `webhook_deliveries` | A reunião pediu o histórico, sem definir tabela | §4 |
| Métrica p95 para a latência | A reunião disse "abaixo de 10 segundos" ([09:02] Marcos) | §9.2 |
| Recusa de segunda rotação dentro das 24h | Preservar a carência | §5.6 |
| Nome do arquivo `webhook.processor.ts` | O Bruno deixou entre `.worker.ts` e `.processor.ts` ([09:28]) | §5.2 |


---

## 9. Observabilidade

A reunião decidiu **não adicionar ferramentas novas** ([09:29] Bruno, [09:30] Larissa). O projeto tem só o Pino (`src/shared/logger/index.ts`) e o `X-Request-Id` do `src/middlewares/request-logger.middleware.ts`. Não há stack de métricas nem de tracing (ver `package.json`). Por isso, nesta fase, **métricas e tracing saem de logs estruturados e de consultas ao banco**.

### 9.1 Logs (Pino), [FDD-OBS-01]

| Evento (`msg`) | Nível | Processo | Campos |
|---|---|---|---|
| `webhook_event_enqueued` | info | API | `eventId`, `webhookId`, `orderId`, `fromStatus`, `toStatus` |
| `webhook_worker_started` / `webhook_worker_stopped` | info | worker | `pollIntervalMs`, `batchSize`, `recoveredProcessing` |
| `webhook_worker_cycle` | debug (info se `picked>0`) | worker | `picked`, `delivered`, `failed`, `deadLettered`, `cycleMs`, `backlog` |
| `webhook_delivery_succeeded` | info | worker | `eventId`, `webhookId`, `attempt`, `statusCode`, `durationMs`, `lagMs` |
| `webhook_delivery_failed` | warn | worker | `eventId`, `webhookId`, `attempt`, `errorCode`, `statusCode?`, `nextAttemptAt` |
| `webhook_dead_lettered` | warn | worker | `deadLetterId`, `eventId`, `webhookId`, `reason`, `attempts` |
| `webhook_replayed` | info | API | `deadLetterId`, `eventId`, `adminUserId` ([09:36] Sofia) |
| `webhook_secret_rotated` | info | API | `webhookId`, `userId`, `previousSecretExpiresAt` |

**Redaction:** incluir `'*.secret'`, `'*.previousSecret'` e `'req.headers["x-signature"]'` em `redactPaths` de `src/shared/logger/index.ts`. Motivação: já houve vazamento de secret em log, ainda que do lado do cliente ([09:22] Diego).

### 9.2 Métricas, [FDD-OBS-02]

| Métrica | Como obter | Alvo/uso |
|---|---|---|
| `webhook_delivery_lag_ms` (commit → 1ª tentativa) | `lagMs` em `webhook_delivery_succeeded` (`now - outbox.createdAt`) quando `attempt=1` | < 10.000 ms para a primeira tentativa ([09:02] Marcos); o uso do p95 é proposta (§8.1) |
| `webhook_outbox_backlog` | `backlog` em `webhook_worker_cycle`, a partir de `COUNT(*) WHERE status IN (PENDING,FAILED)` | Tendência de alta = worker parado ou lento |
| `webhook_delivery_success_rate` | `success` em `webhook_deliveries`, por endpoint/dia | Comparar endpoints |
| `webhook_dead_letter_total` | `COUNT(*)` em `webhook_dead_letter WHERE replayedAt IS NULL` | Gatilho para ação manual de ADMIN |
| `webhook_events_per_endpoint_per_minute` | `webhook_event_enqueued` agrupado por `webhookId` | Insumo para decidir rate limiting (RFC-QA-01, [09:39] Diego) |

### 9.3 Tracing, [FDD-OBS-03]

Sem biblioteca de tracing. A **correlação** é feita por:
- **`eventId`** como chave de rastreio ponta a ponta: aparece em todos os logs do evento (enfileiramento, cada tentativa, DLQ, replay), nas tabelas e no header `X-Event-Id` que o cliente recebe. O suporte consegue cruzar com os logs do cliente.
- **Não** se propõe nenhuma mudança na assinatura de `changeStatus` para propagar o `requestId` (isso exigiria alterar `order.controller.ts` e `order.service.ts` além do previsto). O `X-Request-Id` existente (`src/middlewares/request-logger.middleware.ts`) continua identificando a requisição HTTP nos logs da API, A ligação entre a requisição que mudou o status e o evento é feita por `orderId`, `toStatus` e horário no log `webhook_event_enqueued`. Uma correlação direta por `requestId` fica como melhoria futura.
- Distributed tracing (OpenTelemetry) fica fora desta fase pela decisão de não adicionar libs, e pode ser reavaliado depois.

---

## 10. Integração com o sistema existente

| ID | Arquivo existente | Como o módulo de webhooks se integra |
|---|---|---|
| FDD-INT-01 | `src/modules/orders/order.service.ts` | Em `changeStatus`, dentro do callback de `this.prisma.$transaction(async (tx) => …)`, depois de `tx.orderStatusHistory.create(...)`: `await publishWebhookEvent(tx, order, from, to)`. `order` já vem carregado pelo `findUnique` inicial e tem `customerId`, `orderNumber` e `totalCents`. Não muda o construtor (`OrderRepository`, `PrismaClient`) nem a montagem em `src/app.ts`. A função é importada de `src/modules/webhooks/webhook.publisher.ts` (novo) ([09:41] Bruno). |
| FDD-INT-02 | `src/modules/orders/order.status.ts` | Nada muda. O `canTransition` continua decidindo o que é válido. O webhook só observa o resultado: um evento é emitido **apenas** para transições que passaram por `canTransition` e commitaram. A lista `events` de um webhook é validada contra o enum `OrderStatus` (`z.nativeEnum`), como em `src/modules/orders/order.schemas.ts`. |
| FDD-INT-03 | `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | `webhook.errors.ts` (novo) estende `AppError`, `ConflictError` e `UnprocessableEntityError` com códigos `WEBHOOK_*`, como `InvalidStatusTransitionError` faz com `INVALID_STATUS_TRANSITION` ([09:28] Bruno). |
| FDD-INT-04 | `src/middlewares/error.middleware.ts` | **Sem alteração.** Os erros `WEBHOOK_*` são `AppError` e saem como `{ error: { code, message, details } }`. `ZodError` e `P2002`/`P2025` do Prisma já estão cobertos ([09:29] Bruno). |
| FDD-INT-05 | `src/middlewares/auth.middleware.ts` | O router de webhooks usa `router.use(authenticate)`, como `order.routes.ts`. O router admin acrescenta `requireRole('ADMIN')`, como `src/modules/users/user.routes.ts` ([09:36] Larissa). `req.user.id` alimenta `replayedById`. |
| FDD-INT-06 | `src/middlewares/validate.middleware.ts` | Os schemas de `webhook.schemas.ts` são aplicados com `validate({ body, params, query })`. Atenção: o middleware converte qualquer `ZodError` em `VALIDATION_ERROR` (ver FDD-ERR-02). |
| FDD-INT-07 | `src/routes/index.ts` e `src/app.ts` | Em `Controllers`, acrescentar `webhooks` e `webhooksAdmin`. Em `buildApiRouter`: `router.use('/webhooks', buildWebhookRouter(...))`, `router.use('/admin/webhooks', buildWebhookAdminRouter(...))` e `router.use('/customers/:customerId/webhooks', buildCustomerWebhookRouter(...))` (este com `Router({ mergeParams: true })`, registrado **antes** de `/customers` para não passar pelo router de customers). Em `buildControllers`: instanciar `WebhookRepository` → `WebhookService` → `WebhookController`. |
| FDD-INT-08 | `src/server.ts` e `src/config/database.ts` | `src/worker.ts` (novo) segue a estrutura do `bootstrap()` de `src/server.ts` (log de start, handlers de `SIGINT`/`SIGTERM`, `$disconnect`), mas **não** sobe o Express. Cria um `PrismaClient` próprio com `createPrismaClient()` ([09:30] Bruno). |
| FDD-INT-09 | `src/config/env.ts` | Acrescentar as chaves `WEBHOOK_*` opcionais (§8) ao `envSchema`. `DATABASE_URL` é a mesma ([09:30] Bruno). |
| FDD-INT-10 | `src/shared/logger/index.ts` | Reuso do `logger` nos dois processos. Acrescentar caminhos de secret a `redactPaths` (§9.1). |
| FDD-INT-11 | `prisma/schema.prisma` | 4 models e 1 enum novos (§4), mais a relação inversa em `Customer`, e uma migration nova em `prisma/migrations/`. |
| FDD-INT-12 | `package.json` | Scripts `"worker": "node --env-file=.env dist/worker.js"` e `"worker:dev": "tsx watch --env-file=.env src/worker.ts"`, espelhando `start`/`dev` ([09:11] Larissa). Nenhuma dependência nova: `uuid`, `node:crypto` e `fetch` já estão disponíveis. |
| FDD-INT-13 | `tests/setup.ts` e `tests/helpers/factories.ts` | O `beforeEach` precisa limpar `webhookDelivery`, `webhookOutbox`, `webhookDeadLetter` e `webhookEndpoint` **antes** de `customer.deleteMany()`, por causa da FK. As factories ganham `createTestWebhook(customerId, events)`. |

**Novos arquivos do módulo:** `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.admin.routes.ts`, `webhook.schemas.ts`, `webhook.errors.ts`, `webhook.publisher.ts`, `webhook.processor.ts`, `webhook.signature.ts`; e `src/worker.ts` (todos **novos**).

---

## 11. Dependências e compatibilidade

- **Runtime:** Node ≥ 20 (`package.json` → `engines`), necessário para `fetch` e `AbortSignal.timeout` nativos.
- **Bibliotecas:** só as já instaladas (`@prisma/client` 5.22, `zod`, `pino`, `uuid`, `express`).
- **Banco:** MySQL 8.0 (`docker-compose.yml`). A migration só adiciona tabelas e não mexe em colunas existentes.
- **Compatibilidade da API:** nenhum contrato existente muda. `PATCH /api/v1/orders/:id/status` mantém request e response. O único efeito colateral observável é um 500 se a inserção na outbox falhar.
- **Deploy:** a migration vem antes de subir a nova API e o worker. O worker pode subir depois da API, já que os eventos acumulam sem perda até lá.
- **Dependência organizacional:** revisão de segurança da Sofia, 2 dias úteis antes do deploy ([09:46] Sofia).

## 12. Riscos e mitigação

| ID | Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|---|
| FDD-RSK-01 | Transação de `changeStatus` mais lenta pela leitura de endpoints e N inserts | Média | Médio | Índice `[customerId, active]`, `createMany` e teste de carga comparando latência antes e depois |
| FDD-RSK-02 | Bug em `publishWebhookEvent` passa a bloquear mudanças de status (rollback) | Baixa | Alto | Testes de integração de `changeStatus` com e sem webhooks. A regra de rollback foi uma decisão explícita ([09:40] Bruno) |
| FDD-RSK-03 | Worker parado sem ninguém perceber | Média | Alto | `webhook_outbox_backlog` e `lagMs` nos logs. Recuperação de `PROCESSING` no boot |
| FDD-RSK-04 | Ordem quebrada durante o retry (evento posterior passa na frente) | Média | Médio | Documentar no portal que a ordem não é garantida. O cliente pode ordenar pelo `timestamp` do corpo. Ver RFC-QA-07 |
| FDD-RSK-05 | Vazamento de secret (logs, banco) | Baixa | Alto | Redaction no Pino, secret retornada só na criação e na rotação, rotação com carência, revisão da Sofia (RFC-QA-08) |
| FDD-RSK-06 | Cliente não deduplica e processa duas vezes | Média | Médio | `X-Event-Id` estável e destaque no portal ([09:26] Marcos) |
| FDD-RSK-07 | Crescimento contínuo de `webhook_outbox`/`webhook_deliveries`, que guarda payload e resposta de cada tentativa | Alta (a prazo) | Baixo | Arquivamento já previsto como trabalho futuro ([09:08] Diego). Monitorar o tamanho das tabelas. Avaliar na revisão um limite para o `responseBody` guardado |
| FDD-RSK-08 | Envio de muitas chamadas em rajada para um cliente | Média | Médio | Métrica por endpoint. Decisão adiada (RFC-QA-01) |
| FDD-RSK-09 | Um cliente lento ou fora do ar prende o processamento dos demais (o argumento de [09:04] Bruno aplicado ao worker) | Média | Alto | Cadeias independentes por endpoint, sem esperar o ciclo (§5.2 passo 4). A concorrência entre endpoints distintos é ilimitada, o que pode exigir um teto se houver muitos clientes. Validar na revisão (RFC-QA-11) |
| FDD-RSK-10 | Qualquer usuário autenticado rotaciona a secret ou lê o histórico de qualquer customer (§6.10) | Média | Alto | Seguir a decisão literal da reunião e levar o ponto à revisão da Sofia (RFC-QA-04, RFC-QA-08) |

## 13. Critérios de aceite técnicos

| ID | Critério |
|---|---|
| FDD-AC-01 | Com webhook ativo assinando `SHIPPED`, a transição `PROCESSING → SHIPPED` cria **exatamente 1** linha `PENDING` na outbox, na mesma transação |
| FDD-AC-02 | Uma transição para um status **não assinado** não cria nenhuma linha na outbox |
| FDD-AC-03 | Uma falha simulada no `insert` da outbox faz `changeStatus` retornar 500 e o pedido continua no status anterior, sem nova linha em `order_status_history` |
| FDD-AC-04 | Um endpoint de teste que responde 200 recebe o POST em ≤ 2s + envio, com `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp`, `X-Signature` e `Content-Type: application/json`. O HMAC confere com a secret |
| FDD-AC-05 | Um endpoint que responde 500 gera agendamentos em +1m, +5m, +30m, +2h e +12h (testado com relógio falso) e depois uma linha na DLQ com `WEBHOOK_MAX_ATTEMPTS_EXCEEDED`. A linha some da outbox |
| FDD-AC-06 | Um endpoint que demora mais de 10s gera `WEBHOOK_DELIVERY_TIMEOUT` e entra em retry |
| FDD-AC-07 | Um payload > 64KB vai direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem nenhuma requisição HTTP |
| FDD-AC-08 | `POST /webhooks` com `url` `http://` → 400. Replay por um OPERATOR → 403. Replay por um ADMIN → 202, com evento de volta na outbox com o **mesmo** `event_id` e log `webhook_replayed` com `adminUserId` |
| FDD-AC-09 | Depois da rotação, durante 24h, o `X-Signature` traz duas assinaturas válidas (nova e antiga). Passadas as 24h, só a nova |
| FDD-AC-10 | `GET /webhooks/:id/deliveries` retorna no máximo 100 itens, do mais recente para o mais antigo, com `success`, `responseStatus`, `durationMs`, `requestPayload` e `responseBody` |
| FDD-AC-11 | Matar o worker com linhas em `PROCESSING` e reiniciar faz essas linhas voltarem a `PENDING` e serem entregues |
| FDD-AC-12 | Os testes existentes (`tests/orders.test.ts`, `tests/auth.test.ts`) continuam passando sem alteração além do `tests/setup.ts` |
| FDD-AC-13 | Com um endpoint que sempre expira (10s) e outro que responde 200, o evento do segundo é entregue em até ~2s + envio, sem esperar o primeiro |
