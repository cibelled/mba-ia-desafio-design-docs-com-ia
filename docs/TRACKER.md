# TRACKER — Rastreabilidade da documentação

Mapa de cada item registrado em PRD, RFC, FDD e ADRs até sua origem: a transcrição da reunião (`TRANSCRICAO.md`, no formato `[hh:mm] Falante`) ou o código-fonte (caminho do arquivo). Os IDs correspondem aos que aparecem nos próprios documentos. Linhas com sufixo (`b`, `c`, `-D2`…) desdobram um item com mais de uma origem relevante.

**Totais:** 259 linhas · 220 com Fonte = TRANSCRICAO (85%) · 39 com Fonte = CODIGO.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Outbox no MySQL existente, inserida na mesma transação do changeStatus | TRANSCRICAO | [09:06] Diego |
| ADR-001-D2 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Decisão registrada: outbox em MySQL | TRANSCRICAO | [09:08] Larissa |
| ADR-001-D3 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Requisito Não Funcional | Índices em status e created_at na outbox | TRANSCRICAO | [09:08] Diego |
| ADR-001-D4 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Filtro de eventos aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-001-CTX | `docs/adrs/ADR-001-outbox-no-mysql.md` | Contexto | changeStatus já usa $transaction com estoque e order_status_history | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-001-ALT-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | Disparo síncrono descartado: trava transação e exigiria rollback | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | Redis Streams/Cluster descartado: infra nova, overengineering | TRANSCRICAO | [09:07] Diego |
| ADR-001-TO-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Trade-off | Arquivamento de linhas entregues (~30 dias) fora do escopo | TRANSCRICAO | [09:08] Diego |
| ADR-002 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Worker lê a outbox por polling a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| ADR-002-D2 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Latência mínima de ~2s aceita e registrada como decisão | TRANSCRICAO | [09:10] Larissa |
| ADR-002-D3 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-D4 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Novo entry-point src/worker.ts e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-D5 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | PrismaClient próprio por processo, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-CTX | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Contexto | Entry-point atual da API que o worker espelha | CODIGO | `src/server.ts` |
| ADR-002-ALT-01 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Alternativa descartada | Trigger/LISTEN descartado: MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-002-TO-01 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Trade-off | Ordering só por order_id e com single-worker (limitação conhecida) | TRANSCRICAO | [09:13] Larissa |
| ADR-002-TO-02 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Trade-off | Escala com múltiplos workers adiada (particionar/lock) | TRANSCRICAO | [09:13] Diego |
| ADR-003 | `docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md` | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-D2 | `docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md` | Decisão | Progressão de backoff e janela de quase 15h | TRANSCRICAO | [09:17] Diego |
| ADR-003-D3 | `docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md` | Decisão | DLQ em tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-003-D4 | `docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md` | Decisão | Replay manual via POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| ADR-003-D5 | `docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md` | Decisão | Replay exige role ADMIN e loga quem fez | TRANSCRICAO | [09:36] Sofia |
| ADR-003-ALT-01 | `docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md` | Alternativa descartada | Retry indefinido descartado: evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | `docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md` | Alternativa descartada | 3 tentativas descartadas: insuficiente p/ manutenção de 2h | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-03 | `docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md` | Alternativa descartada | Marcar failed na própria outbox (sem DLQ) preterido | TRANSCRICAO | [09:17] Larissa |
| ADR-003-TO-01 | `docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md` | Trade-off | Atraso de até ~15h aceitável pelo produto | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-D2 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | Assinatura enviada em header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-D3 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | Secret única por endpoint de webhook | TRANSCRICAO | [09:21] Sofia |
| ADR-004-D4 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | Tabela de configuração com url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-D5 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Restrição | URL https obrigatória via schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-ALT-01 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Alternativa descartada | Secret global descartada: vazou uma, vaza tudo | TRANSCRICAO | [09:21] Sofia |
| ADR-004-TO-01 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Trade-off | Secret não pode ser hash como passwordHash (precisa assinar) | CODIGO | `prisma/schema.prisma` |
| ADR-004-TO-02 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Trade-off | Histórico de vazamento de secret em log de cliente | TRANSCRICAO | [09:22] Diego |
| ADR-005 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Decisão | Garantia at-least-once; cliente deve suportar duplicatas | TRANSCRICAO | [09:24] Diego |
| ADR-005-D2 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Decisão | X-Event-Id com UUID gerado na entrada da outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-D3 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Decisão | Decisão registrada: at-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT-01 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Alternativa descartada | Exactly-once descartado: coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005-TO-01 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Trade-off | Responsabilidade de dedup transferida ao cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-TO-02 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Trade-off | Comportamento destacado no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Reuso máximo dos padrões existentes; webhook como módulo comum | TRANSCRICAO | [09:30] Larissa |
| ADR-006-D2 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Módulo src/modules/webhooks com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-D3 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Códigos de erro com prefixo WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| ADR-006-D4 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Pino e error middleware reaproveitados sem mudança | TRANSCRICAO | [09:29] Bruno |
| ADR-006-D5 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | publishWebhookEvent(tx, ...) em vez de injetar repository | TRANSCRICAO | [09:41] Diego |
| ADR-006-CTX-01 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Contexto | Classe AppError com statusCode e errorCode | CODIGO | `src/shared/errors/app-error.ts` |
| ADR-006-CTX-02 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Contexto | InvalidStatusTransitionError / InsufficientStockError como modelo | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-CTX-03 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Contexto | Tratamento centralizado de AppError, Zod e Prisma | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-CTX-04 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Contexto | Logger Pino com redaction | CODIGO | `src/shared/logger/index.ts` |
| ADR-006-CTX-05 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Contexto | requireRole existente usado com ADMIN em users | CODIGO | `src/modules/users/user.routes.ts` |
| ADR-006-CTX-06 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Contexto | Montagem de módulos e prefixo /api/v1 | CODIGO | `src/routes/index.ts` |
| ADR-006-ALT-01 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Alternativa descartada | Injetar WebhookRepository no OrderService preterido | TRANSCRICAO | [09:41] Bruno |
| ADR-007 | `docs/adrs/ADR-007-payload-snapshot-na-insercao.md` | Decisão | Payload renderizado (snapshot) na inserção da outbox | TRANSCRICAO | [09:52] Larissa |
| ADR-007-D2 | `docs/adrs/ADR-007-payload-snapshot-na-insercao.md` | Decisão | Snapshot confirmado como decisão | TRANSCRICAO | [09:52] Bruno |
| ADR-007-D3 | `docs/adrs/ADR-007-payload-snapshot-na-insercao.md` | Decisão | Formato enxuto: event_id, event_type, timestamp, ids, status, total_cents; sem items | TRANSCRICAO | [09:43] Diego |
| ADR-007-D4 | `docs/adrs/ADR-007-payload-snapshot-na-insercao.md` | Requisito Não Funcional | Teto de 64KB com erro (sem truncar) | TRANSCRICAO | [09:24] Larissa |
| ADR-007-D5 | `docs/adrs/ADR-007-payload-snapshot-na-insercao.md` | Decisão | IDs UUID seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-007-ALT-01 | `docs/adrs/ADR-007-payload-snapshot-na-insercao.md` | Alternativa descartada | Renderizar no envio (guardar só order_id) descartado | TRANSCRICAO | [09:51] Bruno |
| RFC-META-01 | `docs/RFC.md` | Contexto | Larissa responsável pelo doc de design e revisão com Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| RFC-CTX-01 | `docs/RFC.md` | Contexto | Pedido formal de Atlas, MaxDistribuição e Nova Cargo; polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | `docs/RFC.md` | Requisito Não Funcional | "Tempo real" = abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | `docs/RFC.md` | Restrição | Webhook apenas outbound | TRANSCRICAO | [09:03] Sofia |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa descartada | Disparo síncrono no changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa descartada | Redis Streams / Redis Cluster | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa descartada | Trigger no banco / LISTEN-NOTIFY | TRANSCRICAO | [09:09] Bruno |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa descartada | Retry indefinido ou só 3 tentativas | TRANSCRICAO | [09:16] Bruno |
| RFC-ALT-05 | `docs/RFC.md` | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-06 | `docs/RFC.md` | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | `docs/RFC.md` | Alternativa descartada | Renderizar payload no envio | TRANSCRICAO | [09:51] Bruno |
| RFC-QA-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-QA-02 | `docs/RFC.md` | Questão em aberto | Aviso por e-mail ao cliente adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-QA-03 | `docs/RFC.md` | Questão em aberto | Escalar para múltiplos workers adiado | TRANSCRICAO | [09:13] Diego |
| RFC-QA-04 | `docs/RFC.md` | Questão em aberto | Endurecer autorização do CRUD mais pra frente | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-05 | `docs/RFC.md` | Questão em aberto | customer_id no body ou no path (não decidido) | TRANSCRICAO | [09:32] Larissa |
| RFC-QA-06 | `docs/RFC.md` | Questão em aberto | Leitura de "5 tentativas" vs. "quase 15 horas" | TRANSCRICAO | [09:17] Diego |
| RFC-QA-07 | `docs/RFC.md` | Questão em aberto | Ordering durante retry (derivado de single-worker + backoff) | TRANSCRICAO | [09:12] Diego |
| RFC-QA-08 | `docs/RFC.md` | Questão em aberto | Pontos para revisão de segurança (secret em repouso, grace, X-Timestamp) | TRANSCRICAO | [09:46] Sofia |
| RFC-QA-09 | `docs/RFC.md` | Questão em aberto | Criação de pedido gera evento? Discussão só cobriu changeStatus | TRANSCRICAO | [09:40] Bruno |
| RFC-QA-10 | `docs/RFC.md` | Questão em aberto | Listagem da DLQ não definida; só o replay | TRANSCRICAO | [09:35] Diego |
| RFC-IMP-01 | `docs/RFC.md` | Risco | Prazo fim de novembro vs. 3 sprints | TRANSCRICAO | [09:46] Larissa |
| RFC-IMP-02 | `docs/RFC.md` | Integração | changeStatus ganha chamada a publishWebhookEvent | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-IMP-03 | `docs/RFC.md` | Integração | Novo script worker no package.json | CODIGO | `package.json` |
| FDD-OBJ-01 | `docs/FDD.md` | Requisito Não Funcional | Latência < 10s entre commit e primeira tentativa | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-02 | `docs/FDD.md` | Requisito Não Funcional | Zero transições sem evento / sem eventos de rollback | TRANSCRICAO | [09:06] Diego |
| FDD-OBJ-03 | `docs/FDD.md` | Restrição | Nenhuma chamada HTTP dentro da transação | TRANSCRICAO | [09:04] Bruno |
| FDD-OBJ-04 | `docs/FDD.md` | Requisito Não Funcional | At-least-once com event_id estável | TRANSCRICAO | [09:24] Diego |
| FDD-OBJ-05 | `docs/FDD.md` | Restrição | Sem infra nova; sem mudança em middlewares | TRANSCRICAO | [09:29] Bruno |
| FDD-EXC-01 | `docs/FDD.md` | Fora de escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| FDD-EXC-02 | `docs/FDD.md` | Fora de escopo | E-mail de aviso de falha | TRANSCRICAO | [09:37] Larissa |
| FDD-EXC-03 | `docs/FDD.md` | Fora de escopo | Rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| FDD-EXC-04 | `docs/FDD.md` | Fora de escopo | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| FDD-EXC-05 | `docs/FDD.md` | Fora de escopo | Arquivamento após ~30 dias | TRANSCRICAO | [09:08] Diego |
| FDD-EXC-06 | `docs/FDD.md` | Fora de escopo | Múltiplos workers / ordering global | TRANSCRICAO | [09:13] Diego |
| FDD-EXC-07 | `docs/FDD.md` | Fora de escopo | Items do pedido no payload | TRANSCRICAO | [09:43] Diego |
| FDD-MODEL-01 | `docs/FDD.md` | Decisão | Model WebhookEndpoint (url, secret, customerId, ativo, events) | TRANSCRICAO | [09:21] Bruno |
| FDD-MODEL-02 | `docs/FDD.md` | Decisão | Model WebhookOutbox com status PENDING/PROCESSING/FAILED/DELIVERED | TRANSCRICAO | [09:08] Diego |
| FDD-MODEL-03 | `docs/FDD.md` | Decisão | Model WebhookDelivery para histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-MODEL-04 | `docs/FDD.md` | Decisão | Model WebhookDeadLetter com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-MODEL-05 | `docs/FDD.md` | Decisão | Padrão de IDs uuid @db.Char(36) nos models | CODIGO | `prisma/schema.prisma` |
| FDD-FLX-01 | `docs/FDD.md` | Fluxo | Criação do evento na outbox dentro do changeStatus; rollback se falhar | TRANSCRICAO | [09:40] Bruno |
| FDD-FLX-02 | `docs/FDD.md` | Fluxo | Ciclo do worker: batch de pendentes por created_at, processa, marca | TRANSCRICAO | [09:08] Diego |
| FDD-FLX-03 | `docs/FDD.md` | Fluxo | Retry com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| FDD-FLX-04 | `docs/FDD.md` | Fluxo | Movimento para DLQ após esgotar tentativas | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-05 | `docs/FDD.md` | Fluxo | Replay recoloca na outbox como pendente; ADMIN; auditado | TRANSCRICAO | [09:36] Larissa |
| FDD-FLX-06 | `docs/FDD.md` | Fluxo | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | POST /webhooks: url, events; secret gerada e devolvida | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | GET /webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | GET /webhooks/:id/deliveries (últimos 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Headers outbound X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-08b | `docs/FDD.md` | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-09 | `docs/FDD.md` | Contrato | X-Signature = HMAC-SHA256 do corpo | TRANSCRICAO | [09:20] Sofia |
| FDD-CONTRATO-10 | `docs/FDD.md` | Contrato | Envelope de erro { error: { code, message, details } } | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-CONTRATO-11 | `docs/FDD.md` | Contrato | Paginação data/pagination no GET de listagem | CODIGO | `src/shared/http/response.ts` |
| FDD-CONTRATO-12 | `docs/FDD.md` | Restrição | CRUD de configuração liberado a qualquer role autenticada | TRANSCRICAO | [09:37] Sofia |
| FDD-ERR-01 | `docs/FDD.md` | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | `docs/FDD.md` | Erro | WEBHOOK_INVALID_URL (via Zod) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-02b | `docs/FDD.md` | Erro | validate middleware converte ZodError em VALIDATION_ERROR | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-ERR-03 | `docs/FDD.md` | Erro | WEBHOOK_CUSTOMER_NOT_FOUND | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-04 | `docs/FDD.md` | Erro | WEBHOOK_SECRET_ROTATION_IN_PROGRESS (proposta p/ preservar grace 24h) | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-05 | `docs/FDD.md` | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-06 | `docs/FDD.md` | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED | TRANSCRICAO | [09:36] Sofia |
| FDD-ERR-07 | `docs/FDD.md` | Erro | WEBHOOK_ENDPOINT_INACTIVE (endpoint ativo/inativo) | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-08 | `docs/FDD.md` | Erro | WEBHOOK_DELIVERY_TIMEOUT (10s) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-09 | `docs/FDD.md` | Erro | WEBHOOK_DELIVERY_HTTP_ERROR | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-10 | `docs/FDD.md` | Erro | WEBHOOK_DELIVERY_NETWORK_ERROR (cliente offline) | TRANSCRICAO | [09:14] Larissa |
| FDD-ERR-11 | `docs/FDD.md` | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (64KB) | TRANSCRICAO | [09:24] Diego |
| FDD-ERR-12 | `docs/FDD.md` | Erro | WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-13 | `docs/FDD.md` | Erro | WEBHOOK_ENDPOINT_INACTIVE no worker | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-14 | `docs/FDD.md` | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-15 | `docs/FDD.md` | Erro | NotFoundError fixa código NOT_FOUND; 404 do módulo estendem AppError | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-RES-01 | `docs/FDD.md` | Resiliência | Timeout HTTP 10s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | `docs/FDD.md` | Resiliência | Backoff exponencial fixo | TRANSCRICAO | [09:17] Diego |
| FDD-RES-03 | `docs/FDD.md` | Resiliência | Teto de tentativas e DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-RES-04 | `docs/FDD.md` | Resiliência | Fallback = replay manual; e-mail adiado | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-05 | `docs/FDD.md` | Resiliência | Isolamento: worker em processo separado | TRANSCRICAO | [09:11] Diego |
| FDD-RES-06 | `docs/FDD.md` | Resiliência | Durabilidade via outbox transacional | TRANSCRICAO | [09:06] Diego |
| FDD-RES-07 | `docs/FDD.md` | Resiliência | Shutdown gracioso espelhando SIGINT/SIGTERM do server | CODIGO | `src/server.ts` |
| FDD-RES-08 | `docs/FDD.md` | Resiliência | Limite de payload 64KB | TRANSCRICAO | [09:24] Larissa |
| FDD-RES-09 | `docs/FDD.md` | Resiliência | Batch pequeno configurável | TRANSCRICAO | [09:08] Diego |
| FDD-RES-10 | `docs/FDD.md` | Integração | Novas chaves WEBHOOK_* no envSchema | CODIGO | `src/config/env.ts` |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Logs estruturados via Pino existente, sem lib nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-01b | `docs/FDD.md` | Observabilidade | Redaction de secret motivada por vazamento em log | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-01c | `docs/FDD.md` | Observabilidade | Log de auditoria do replay com adminUserId | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade | Métrica de lag p95 < 10s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-02b | `docs/FDD.md` | Observabilidade | Métrica de volume por endpoint para decidir rate limiting | TRANSCRICAO | [09:38] Diego |
| FDD-OBS-03 | `docs/FDD.md` | Observabilidade | Correlação via X-Request-Id existente | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-INT-01 | `docs/FDD.md` | Integração | changeStatus chama publishWebhookEvent(tx, order, from, to) | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-01b | `docs/FDD.md` | Integração | Assinatura publishWebhookEvent proposta por Bruno | TRANSCRICAO | [09:41] Bruno |
| FDD-INT-02 | `docs/FDD.md` | Integração | canTransition inalterado; events validados contra OrderStatus | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | webhook.errors.ts estende AppError/ConflictError/UnprocessableEntityError | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | errorMiddleware sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | authenticate + requireRole(ADMIN) no router admin | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-05b | `docs/FDD.md` | Integração | Reaproveitar requireRole existente | TRANSCRICAO | [09:36] Larissa |
| FDD-INT-06 | `docs/FDD.md` | Integração | Schemas aplicados via validate({ body, params, query }) | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | Registro de routers /webhooks e /admin/webhooks | CODIGO | `src/routes/index.ts` |
| FDD-INT-07b | `docs/FDD.md` | Integração | buildControllers instancia WebhookRepository/Service/Controller | CODIGO | `src/app.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | Worker cria PrismaClient com createPrismaClient() | CODIGO | `src/config/database.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração | DATABASE_URL compartilhada validada no env | CODIGO | `src/config/env.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração | Novos redactPaths para secret | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração | 4 models + enum + relação inversa em Customer | CODIGO | `prisma/schema.prisma` |
| FDD-INT-12 | `docs/FDD.md` | Integração | Scripts worker/worker:dev; engines node >= 20 (fetch nativo) | CODIGO | `package.json` |
| FDD-INT-13 | `docs/FDD.md` | Integração | beforeEach limpa tabelas de webhook antes de customer | CODIGO | `tests/setup.ts` |
| FDD-INT-13b | `docs/FDD.md` | Integração | Factory createTestWebhook | CODIGO | `tests/helpers/factories.ts` |
| FDD-INT-14 | `docs/FDD.md` | Integração | customerId em query segue listOrdersQuerySchema | CODIGO | `src/modules/orders/order.schemas.ts` |
| FDD-RSK-01 | `docs/FDD.md` | Risco | Transação de changeStatus mais pesada | TRANSCRICAO | [09:04] Bruno |
| FDD-RSK-02 | `docs/FDD.md` | Risco | Bug no publish bloqueia mudanças de status (rollback) | TRANSCRICAO | [09:40] Bruno |
| FDD-RSK-03 | `docs/FDD.md` | Risco | Worker parado acumula eventos | TRANSCRICAO | [09:11] Diego |
| FDD-RSK-04 | `docs/FDD.md` | Risco | Ordering quebrada durante retry | TRANSCRICAO | [09:12] Diego |
| FDD-RSK-05 | `docs/FDD.md` | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| FDD-RSK-06 | `docs/FDD.md` | Risco | Cliente não deduplica | TRANSCRICAO | [09:25] Sofia |
| FDD-RSK-07 | `docs/FDD.md` | Risco | Crescimento das tabelas | TRANSCRICAO | [09:08] Diego |
| FDD-RSK-08 | `docs/FDD.md` | Risco | Rajada de chamadas para um cliente | TRANSCRICAO | [09:38] Diego |
| FDD-AC-01 | `docs/FDD.md` | Critério de aceite | 1 linha PENDING por webhook interessado, mesma transação | TRANSCRICAO | [09:06] Diego |
| FDD-AC-02 | `docs/FDD.md` | Critério de aceite | Status não assinado não gera linha | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-03 | `docs/FDD.md` | Critério de aceite | Falha na outbox faz rollback do status | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-04 | `docs/FDD.md` | Critério de aceite | Headers e HMAC conferem no endpoint de teste | TRANSCRICAO | [09:44] Diego |
| FDD-AC-05 | `docs/FDD.md` | Critério de aceite | Agendamentos de backoff e DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-AC-06 | `docs/FDD.md` | Critério de aceite | Timeout de 10s gera retry | TRANSCRICAO | [09:42] Diego |
| FDD-AC-07 | `docs/FDD.md` | Critério de aceite | Payload > 64KB vai à DLQ sem envio | TRANSCRICAO | [09:24] Larissa |
| FDD-AC-08 | `docs/FDD.md` | Critério de aceite | http recusado; replay 403 para OPERATOR, 202 para ADMIN | TRANSCRICAO | [09:36] Larissa |
| FDD-AC-09 | `docs/FDD.md` | Critério de aceite | Duas assinaturas durante grace de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-10 | `docs/FDD.md` | Critério de aceite | Deliveries limitado a 100 itens | TRANSCRICAO | [09:34] Marcos |
| FDD-AC-11 | `docs/FDD.md` | Critério de aceite | Recuperação de PROCESSING ao reiniciar worker | TRANSCRICAO | [09:11] Diego |
| FDD-AC-12 | `docs/FDD.md` | Critério de aceite | Testes existentes continuam passando | CODIGO | `tests/orders.test.ts` |
| FDD-DEP-01 | `docs/FDD.md` | Dependência | Revisão de segurança 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-DEP-02 | `docs/FDD.md` | Dependência | MySQL 8.0 da stack atual | CODIGO | `docker-compose.yml` |
| PRD-CTX-01 | `docs/PRD.md` | Contexto | Ciclo de vida PENDING→PAID→PROCESSING→SHIPPED→DELIVERED/CANCELLED | CODIGO | `src/modules/orders/order.status.ts` |
| PRD-PROB-01 | `docs/PRD.md` | Contexto | Clientes fazem polling em GET /orders; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | `docs/PRD.md` | Risco | Atlas pode migrar para concorrente | TRANSCRICAO | [09:00] Marcos |
| PRD-CEN-01 | `docs/PRD.md` | Cenário | Cliente assina só SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | `docs/PRD.md` | Cenário | Indisponibilidade de 2h coberta pelo retry | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | `docs/PRD.md` | Cenário | Rotação de secret após vazamento | TRANSCRICAO | [09:21] Sofia |
| PRD-CEN-04 | `docs/PRD.md` | Cenário | Consulta ao histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-05 | `docs/PRD.md` | Cenário | Reprocessamento manual por ADMIN | TRANSCRICAO | [09:18] Diego |
| PRD-MET-01 | `docs/PRD.md` | Métrica | p95 < 10s até a primeira tentativa | TRANSCRICAO | [09:02] Marcos |
| PRD-MET-02 | `docs/PRD.md` | Métrica | 3 de 3 clientes ativos até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-MET-03 | `docs/PRD.md` | Métrica | Zero transições sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-MET-04 | `docs/PRD.md` | Métrica | Janela de ~15h de retentativas | TRANSCRICAO | [09:17] Diego |
| PRD-MET-05 | `docs/PRD.md` | Métrica | Redução do polling em GET /orders (sem meta numérica) | TRANSCRICAO | [09:00] Marcos |
| PRD-OOS-01 | `docs/PRD.md` | Fora de escopo | E-mail ao cliente em caso de falha | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | `docs/PRD.md` | Fora de escopo | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | `docs/PRD.md` | Fora de escopo | Rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | `docs/PRD.md` | Fora de escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | `docs/PRD.md` | Fora de escopo | Ordering global | TRANSCRICAO | [09:14] Marcos |
| PRD-OOS-06 | `docs/PRD.md` | Fora de escopo | Exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-07 | `docs/PRD.md` | Fora de escopo | Arquivamento de entregues | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-08 | `docs/PRD.md` | Fora de escopo | Items no payload | TRANSCRICAO | [09:43] Diego |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Notificar a cada mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Cadastrar webhook com url e status; secret gerada | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | customer_id no body/path, não no JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | PATCH, DELETE e GET de webhooks | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Filtro por status assinados | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Rotação de secret com 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Verificação de origem/integridade por assinatura | TRANSCRICAO | [09:19] Sofia |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Identificador único por evento | TRANSCRICAO | [09:25] Diego |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Retry automático com intervalos crescentes | TRANSCRICAO | [09:15] Diego |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | DLQ com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Reprocessamento por ADMIN com auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-13 | `docs/PRD.md` | Requisito Funcional | Payload resumido, sem items | TRANSCRICAO | [09:43] Diego |
| PRD-FR-14 | `docs/PRD.md` | Requisito Funcional | Snapshot do pedido no momento da mudança | TRANSCRICAO | [09:52] Larissa |
| PRD-FR-15 | `docs/PRD.md` | Requisito Funcional | Identificação do webhook de origem | TRANSCRICAO | [09:44] Sofia |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Latência < 10s; mínimo ~2s | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Consistência status/evento | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Cliente lento não afeta mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Somente https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | Secret por webhook | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | Limite 64KB com erro | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | Timeout 10s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | At-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-09 | `docs/PRD.md` | Requisito Não Funcional | Ordering só por pedido e single-worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-10 | `docs/PRD.md` | Requisito Não Funcional | Sem infra nova | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-11 | `docs/PRD.md` | Requisito Não Funcional | Auditoria do replay | TRANSCRICAO | [09:36] Sofia |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Revisão de segurança da Sofia | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Portal do desenvolvedor | TRANSCRICAO | [09:40] Marcos |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Confirmação de prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-04 | `docs/PRD.md` | Dependência | Usuários que representam o cliente | TRANSCRICAO | [09:32] Marcos |
| PRD-DEP-05 | `docs/PRD.md` | Dependência | Sessão de revisão do design | TRANSCRICAO | [09:50] Larissa |
| PRD-RSK-01 | `docs/PRD.md` | Risco | Perder a Atlas por atraso | TRANSCRICAO | [09:00] Marcos |
| PRD-RSK-02 | `docs/PRD.md` | Risco | Processamento duplicado pelo cliente | TRANSCRICAO | [09:25] Sofia |
| PRD-RSK-03 | `docs/PRD.md` | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| PRD-RSK-04 | `docs/PRD.md` | Risco | Cliente fora do ar > 15h | TRANSCRICAO | [09:17] Marcos |
| PRD-RSK-05 | `docs/PRD.md` | Risco | Rajada de notificações | TRANSCRICAO | [09:38] Diego |
| PRD-RSK-06 | `docs/PRD.md` | Risco | Atraso na revisão de segurança | TRANSCRICAO | [09:46] Sofia |
| PRD-AC-01 | `docs/PRD.md` | Critério de aceite | Entrega < 10s para status assinado | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-02 | `docs/PRD.md` | Critério de aceite | Sem notificação para status não assinado | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-03 | `docs/PRD.md` | Critério de aceite | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-04 | `docs/PRD.md` | Critério de aceite | Assinatura validável com a secret | TRANSCRICAO | [09:20] Sofia |
| PRD-AC-05 | `docs/PRD.md` | Critério de aceite | Grace de 24h na rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-06 | `docs/PRD.md` | Critério de aceite | 5 retentativas em ~15h e DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-AC-07 | `docs/PRD.md` | Critério de aceite | Somente ADMIN reprocessa; autor registrado | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-08 | `docs/PRD.md` | Critério de aceite | Histórico de 100 tentativas | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-09 | `docs/PRD.md` | Critério de aceite | Falha no registro desfaz mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-TST-01 | `docs/PRD.md` | Estratégia de teste | Testes de integração no padrão Vitest + Supertest | CODIGO | `tests/orders.test.ts` |
| PRD-TST-02 | `docs/PRD.md` | Estratégia de teste | Integração no order.service e testes ponta a ponta (meia sprint) | TRANSCRICAO | [09:46] Larissa |
