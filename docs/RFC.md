# RFC-001 — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autora** | Larissa (Tech Lead), responsável pelo doc de design ([09:50] Larissa) |
| **Status** | Em revisão |
| **Data** | A definir. A reunião de origem foi numa quinta-feira, às 09:00, e a data exata não consta na transcrição |
| **Revisores** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança), Marcos (Produto) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

> Este RFC trata **o que propomos e por quê**. Contratos, schemas, fluxos passo a passo e a matriz de erros ficam no [FDD](FDD.md).

---

## 1. TL;DR

Propomos notificar clientes B2B por **webhooks outbound** quando o status de um pedido muda. O evento é gravado numa **outbox no MySQL**, dentro da mesma transação de `OrderService.changeStatus`. Um **worker em processo separado** lê essa outbox a cada **2 segundos** e envia um POST JSON **assinado com HMAC-SHA256** (secret por endpoint), com garantia **at-least-once** e dedup por `X-Event-Id`. Quando o envio falha, há **5 retentativas com backoff 1m/5m/30m/2h/12h**; esgotadas, o evento vai para uma **DLQ** com replay manual por ADMIN. Não entra infraestrutura nova: o módulo segue os padrões existentes do projeto. A estimativa é de **3 sprints**, incluindo a revisão de segurança.

## 2. Contexto e problema

- Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente para ser notificados quando o status dos pedidos muda. Hoje fazem polling em `GET /orders`, o que deixa a integração lenta e cara. A Atlas sinalizou que pode migrar para um concorrente se a entrega não sair até o fim do trimestre ([09:00] Marcos).
- Para esses clientes, "tempo real" é **menos de 10 segundos** ([09:02] Marcos).
- O fluxo é **só de saída**: os clientes recebem, não enviam ([09:02] Marcos, [09:03] Sofia).
- O OMS não tem hoje nenhum mecanismo de eventos, filas ou notificação externa. A mudança de status (`src/modules/orders/order.service.ts`) já é uma transação pesada, que atualiza `orders`, grava `order_status_history` e mexe no estoque ([09:04] Bruno).

## 3. Proposta técnica

```
 PATCH /api/v1/orders/:id/status
          │
          ▼
 OrderService.changeStatus ── $transaction ───────────────────────────────┐
   │ update orders · insert order_status_history · estoque                  │
   │ publishWebhookEvent(tx, …) → INSERT webhook_outbox (1 linha/webhook)   │
   └─────────────────────────── COMMIT (ou rollback de tudo) ──────────────┘
                                         │
             (processo separado: npm run worker, polling 2s)
                                         ▼
                         webhook.processor ── POST https://cliente
                          │   X-Signature (HMAC-SHA256) · X-Event-Id
                          │   X-Webhook-Id · X-Timestamp · timeout 10s
                ┌─────────┴─────────┐
             2xx: DELIVERED     falha: retry 1m→5m→30m→2h→12h
                                     └─ esgotou → webhook_dead_letter
                                                  └─ replay manual (ADMIN)
```

Principais elementos da proposta (cada um tem seu ADR):

1. **Captura transacional via outbox no MySQL.** O evento só existe se a mudança de status commitou. O filtro por status assinado é aplicado na inserção. → [ADR-001](adrs/ADR-001-outbox-no-mysql.md)
2. **Worker dedicado em polling de 2s**, num entry-point novo, `src/worker.ts` (novo), com `PrismaClient` próprio. Fica um único worker nesta fase. → [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)
3. **Resiliência**: timeout de 10s, 5 retentativas com backoff fixo, DLQ em tabela separada e replay manual auditado. → [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md)
4. **Autenticidade e integridade**: HMAC-SHA256 sobre o corpo, secret por endpoint gerada pela plataforma, rotação com carência de 24h e URL só `https`. → [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
5. **Semântica de entrega at-least-once**, com `X-Event-Id` estável entre retentativas. → [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)
6. **Módulo `src/modules/webhooks` no padrão da codebase**: `AppError`, códigos `WEBHOOK_*`, Pino, `errorMiddleware`, Zod e `requireRole`. → [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)
7. **Payload enxuto montado como snapshot na inserção**, com teto de 64KB. → [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md)

**Superfície de API (visão geral):** CRUD de configuração de webhooks para qualquer usuário autenticado, rotação de secret, histórico de entregas (`GET /webhooks/:id/deliveries`) e replay de DLQ restrito a `ADMIN` ([09:31]–[09:37]). Os contratos estão no [FDD §6](FDD.md#6-contratos-públicos).

## 4. Alternativas consideradas

| ID | Alternativa | Trade-off que levou ao descarte | Origem |
|---|---|---|---|
| RFC-ALT-01 | **Disparo HTTP síncrono dentro do `changeStatus`** | Prende a transação a I/O externo. Um cliente lento trava as mudanças de status de outros pedidos, e um cliente fora do ar obrigaria a dar rollback numa mudança de negócio válida. | [09:04] Bruno, [09:06] Diego |
| RFC-ALT-02 | **Fila em Redis Streams / Redis Cluster** | Resolve o desacoplamento, mas exige subir e operar infra nova. Para um time pequeno é overengineering, e a outbox no MySQL já existente resolve o problema. | [09:07] Larissa, [09:07] Diego |
| RFC-ALT-03 | **Trigger no banco para reagir sem polling** | O MySQL não tem `LISTEN/NOTIFY`. O trigger só executa SQL e não avisa processo externo, então seria preciso improvisar. O polling de 2s já atende os 10s. | [09:09] Bruno, [09:09] Diego |
| RFC-ALT-04 | **Retry indefinido** ou **só 3 tentativas** | O retry indefinido deixa eventos pendurados para sempre. Com 3 tentativas, tudo se esgota em ~30 min, menos do que manutenções reais de 2h. | [09:15] Diego, [09:16] Bruno, [09:16] Diego |
| RFC-ALT-05 | **Secret global da plataforma** | Uma única secret vazada comprometeria todos os clientes. | [09:21] Sofia |
| RFC-ALT-06 | **Exactly-once** | Exige coordenação dos dois lados e é muito mais complexo. At-least-once com `event_id` é o padrão de mercado (Stripe, GitHub). | [09:25] Diego |
| RFC-ALT-07 | **Renderizar o payload no envio** (guardar só `order_id`) | O evento poderia descrever um estado posterior à transição notificada. | [09:51] Bruno, [09:52] Larissa |

## 5. Questões em aberto

| ID | Questão | Situação | Origem |
|---|---|---|---|
| RFC-QA-01 | **Rate limiting de saída.** Se 50 pedidos de um cliente mudarem de status em um minuto, enviamos 50 chamadas? | Fora do escopo; **observar e decidir depois** com base nas métricas de volume por endpoint. | [09:38] Diego, [09:39] Larissa |
| RFC-QA-02 | **Avisar o cliente quando o webhook dele falha** (ex.: e-mail após 3 falhas seguidas) | **Adiado para a próxima fase**, depois de medir o impacto. | [09:37] Marcos, [09:37] Larissa |
| RFC-QA-03 | **Escalar para múltiplos workers** (particionar por `order_id` ou lock pessimista) | Adiado ("problema do futuro"). Até lá, ordem só por `order_id` e com worker único. | [09:13] Diego, [09:13] Larissa |
| RFC-QA-04 | **Endurecer a autorização do CRUD de webhooks** (hoje qualquer role autenticada) | "Mais pra frente a gente pode endurecer." | [09:37] Sofia |
| RFC-QA-05 | **`customer_id` no body ou no path?** A Larissa definiu que ele **não** vem do JWT, mas não escolheu entre body e path. | O FDD propõe body no `POST` de criação e path no `GET` de listagem (`/customers/:customerId/webhooks`), ficando dentro das duas opções faladas. **Precisa de confirmação.** | [09:32] Larissa |
| RFC-QA-06 | **Leitura de "5 tentativas".** As falas se contradizem. O resumo final diz "**total** 5 tentativas" ([09:48] Larissa), o que sugere 5 envios. Já os 5 intervalos citados somam ~14h36min, e "quase 15 horas entre primeira falha e última tentativa" só fecha com 1 envio inicial + 5 retentativas (6 requisições). | O FDD adota a leitura de 6 requisições, porque é a única que usa os 5 intervalos e a janela de ~15h. Com 5 envios no total, o último intervalo (12h) cai e a janela vira ~2h36min. **Precisa da confirmação de Diego e Larissa.** | [09:17] Diego, [09:17] Larissa, [09:48] Larissa |
| RFC-QA-07 | **Ordem durante o retry.** Um evento em backoff pode ser ultrapassado por um evento posterior do mesmo pedido, o que contradiz a "ordem implícita por `order_id`". | Não discutido. Opções: aceitar e documentar, ou segurar os eventos seguintes do mesmo pedido/webhook enquanto houver um em retry. | Derivado de [09:12] Diego + [09:17] Larissa |
| RFC-QA-08 | **Para a revisão da Sofia:** (a) como proteger a secret em repouso, já que ela não pode ser hash; (b) como o cliente valida durante a carência de 24h (o FDD propõe enviar duas assinaturas); (c) a `X-Timestamp` fica fora do HMAC, que só cobre o corpo, o que limita a proteção contra replay. | Pendente na revisão de segurança (2 dias úteis antes do deploy). | [09:21]/[09:22] Sofia, [09:44] Diego, [09:46] Sofia |
| RFC-QA-09 | **Criação de pedido (`PENDING`) gera evento?** A reunião tratou só de mudança de status, ou seja, do `changeStatus`. | Proposta: não gerar nesta fase. Confirmar com o Marcos. | [09:40] Bruno |
| RFC-QA-10 | **Como o ADMIN encontra o `:id` da DLQ?** Só o endpoint de replay foi definido, não uma listagem. | Nesta fase o id sai dos logs. Avaliar um `GET` admin na revisão. | [09:18] Diego, [09:35] Diego |

## 6. Impacto e riscos

**Impacto no sistema existente**
- `src/modules/orders/order.service.ts`: a transação de `changeStatus` ganha uma chamada a `publishWebhookEvent`. É a mudança mais crítica, e uma falha nela passa a dar rollback na mudança de status ([09:40] Bruno, [09:41] Diego).
- `prisma/schema.prisma`: entram 4 tabelas novas (configuração de endpoints, outbox, histórico de entregas, DLQ).
- `package.json`: novo script `worker`. A operação passa a ter dois processos ([09:11] Larissa).
- Nenhuma mudança em `errorMiddleware`, `auth.middleware` ou no logger ([09:29] Bruno).

**Riscos principais** (a matriz completa está no [PRD §10](PRD.md#10-riscos-e-mitigação) e no [FDD §12](FDD.md#12-riscos-e-mitigação))
- **Prazo:** a Atlas quer a entrega até o fim de novembro, e a estimativa é de 3 sprints com revisão ([09:45] Marcos, [09:46] Larissa).
- **Transação de status mais lenta ou mais frágil:** é preciso medir o efeito de N inserts por mudança.
- **Worker único parado** deixa os eventos acumulando. Não há perda, mas a latência sobe.
- **Clientes que não deduplicam** podem processar o mesmo evento duas vezes ([09:25] Sofia).

## 7. Decisões relacionadas

| ADR | Título |
|---|---|
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL existente |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2s |
| [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial e DLQ |
| [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint |
| [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes |
| [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md) | Payload snapshot na inserção |

## 8. Próximos passos

1. Sessão de revisão deste RFC com Bruno e Diego antes de começar a codar ([09:50] Larissa).
2. Fechar RFC-QA-05, RFC-QA-06 e RFC-QA-09 nessa sessão.
3. Agendar a revisão de segurança da Sofia (RFC-QA-08) no fim da sprint 3 ([09:47] Larissa).
