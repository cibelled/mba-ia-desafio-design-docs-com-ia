# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Envolvidos** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Status** | Aprovado em reunião técnica (quinta-feira, 09:00). Detalhamento técnico em [RFC](RFC.md) e [FDD](FDD.md) |
| **Prazo-alvo** | Fim de novembro, pedido da Atlas Comercial ([09:45] Marcos). Estimativa de 3 sprints ([09:47] Larissa) |

---

## 1. Resumo e contexto da feature

O OMS vai **notificar automaticamente os sistemas dos clientes B2B sempre que o status de um pedido mudar**, enviando uma chamada HTTP (webhook) para uma URL cadastrada pelo cliente. Isso substitui a prática atual dos clientes de consultar a API de tempos em tempos para descobrir se algo mudou ([09:00] Marcos).

Hoje a plataforma não tem nenhum mecanismo de notificação externa. O ciclo de vida do pedido (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, além de `CANCELLED`) já é controlado e auditado, como mostra `src/modules/orders/order.status.ts`. O que falta é avisar quem está fora.

## 2. Problema e motivação

- **Integração lenta e cara para os clientes:** Atlas Comercial, MaxDistribuição e Nova Cargo fazem polling em `GET /orders` para detectar mudanças ([09:00] Marcos).
- **Risco comercial:** a Atlas sinalizou que pode migrar para um concorrente se não receber a funcionalidade até o fim do trimestre ([09:00] Marcos). O prazo depois foi ajustado para o fim de novembro ([09:45] Marcos).
- **Pedido formal:** os três clientes pediram formalmente a notificação na semana anterior à reunião ([09:00] Marcos).

## 3. Público-alvo e cenários de uso

**Público primário:** clientes B2B que integram seus sistemas ao OMS, começando por Atlas Comercial, MaxDistribuição e Nova Cargo ([09:00] Marcos). O cadastro é feito pela nossa API, autenticada com JWT de usuários do sistema que representam o cliente ([09:32] Marcos).

**Público secundário:** administradores do OMS (role `ADMIN`), que reprocessam notificações que falharam ([09:36] Sofia).

| ID | Cenário |
|---|---|
| PRD-CEN-01 | Um cliente cadastra uma URL `https` e escolhe receber só `SHIPPED` e `DELIVERED`. Quando o pedido é despachado, o sistema dele recebe o aviso em segundos, sem polling ([09:33] Marcos) |
| PRD-CEN-02 | O endpoint de um cliente fica 2h fora do ar por manutenção planejada, como já aconteceu com um cliente. As notificações desse período chegam quando o endpoint volta, sem intervenção ([09:16] Diego) |
| PRD-CEN-03 | Um cliente vaza a secret no próprio log, como já aconteceu antes. Ele pede uma nova secret pela API e tem 24h para atualizar seus sistemas sem perder notificações ([09:21] Sofia, [09:22] Diego) |
| PRD-CEN-04 | Um cliente quer saber se recebeu certa notificação e consulta o histórico das últimas entregas, com sucesso/falha e tempo de resposta ([09:34] Marcos) |
| PRD-CEN-05 | Uma notificação esgotou as tentativas. Um ADMIN a reprocessa manualmente depois de o cliente corrigir o endpoint ([09:18] Diego) |

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
|---|---|---|---|---|
| PRD-MET-01 | Notificar "em tempo real" na percepção do cliente | Tempo entre a mudança de status e a primeira tentativa de entrega (p95) | **< 10 segundos** | [09:02] Marcos |
| PRD-MET-02 | Atender os clientes que fizeram o pedido formal | Clientes com webhook ativo em produção | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) até o prazo de fim de novembro | [09:00] Marcos, [09:45] Marcos |
| PRD-MET-03 | Nenhuma mudança de status sem notificação | Transições de pedidos com webhook interessado que não geraram evento | **0** | [09:40] Bruno |
| PRD-MET-04 | Tolerar indisponibilidade do cliente | Janela de retentativas antes de desistir | **~15 horas** (5 retentativas) | [09:17] Diego, [09:17] Marcos |
| PRD-MET-05 | Reduzir o polling dos clientes em `GET /orders` | Requisições de listagem dos 3 clientes antes e depois | Tendência de queda. A reunião não fixou meta numérica, então a linha de base ainda precisa ser medida | [09:00] Marcos |

## 5. Escopo

### 5.1 Incluso

- Notificação **de saída** (OMS → cliente) a cada mudança de status de pedido ([09:02] Marcos).
- Cadastro, edição, listagem e remoção de webhooks por customer, com escolha dos status de interesse.
- Assinatura de segurança em cada notificação e rotação de secret.
- Retentativas automáticas e fila de falhas definitivas com reprocessamento manual por ADMIN.
- Histórico de entregas consultável via API.
- Documentação de integração no portal do desenvolvedor, a cargo do Marcos ([09:26] Marcos, [09:40] Marcos).

### 5.2 Fora de escopo

| ID | Item | Situação | Origem |
|---|---|---|---|
| PRD-OOS-01 | **Aviso por e-mail ao cliente quando o webhook falha** (ex.: 3 falhas seguidas) | **Adiado** para a próxima fase, depois de medir o impacto | [09:37] Marcos, [09:37] Larissa |
| PRD-OOS-02 | **Dashboard visual** para o cliente ver os webhooks | **Descartado** nesta fase. Fica com o time de frontend, como projeto separado | [09:39] Marcos, [09:40] Larissa |
| PRD-OOS-03 | **Rate limiting** de envio por cliente | **Não entra**. A decisão é observar e decidir depois | [09:38] Diego, [09:39] Larissa |
| PRD-OOS-04 | **Webhooks de entrada** (cliente → OMS) | **Fora**: os clientes só querem receber | [09:02] Sofia, [09:02] Marcos |
| PRD-OOS-05 | **Garantia de ordem global** entre pedidos | **Fora**. Os clientes nunca pediram | [09:14] Marcos |
| PRD-OOS-06 | **Entrega exactly-once** | **Descartada**. A garantia é at-least-once | [09:25] Diego |
| PRD-OOS-07 | **Arquivamento** do histórico de notificações entregues (~30 dias) | **Fora** desta feature | [09:08] Diego |
| PRD-OOS-08 | **Itens do pedido** dentro da notificação | **Fora**. O cliente consulta `GET /orders/:id` | [09:43] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-FR-01 | O sistema deve enviar uma notificação ao cliente sempre que o status de um pedido dele mudar | [09:00] Marcos, [09:40] Bruno |
| PRD-FR-02 | O cliente deve poder **cadastrar** um webhook informando URL e lista de status de interesse. A plataforma gera a secret e a devolve na criação | [09:31] Marcos |
| PRD-FR-03 | O customer do webhook é informado na requisição (body ou path) e **não** é deduzido do JWT do usuário | [09:32] Larissa |
| PRD-FR-04 | O cliente deve poder **editar** (PATCH), **remover** (DELETE) e **listar** (GET) os webhooks de um customer | [09:33] Bruno |
| PRD-FR-05 | Cada webhook só recebe notificações dos **status que assinou** (ex.: só `SHIPPED` e `DELIVERED`) | [09:33] Marcos, [09:34] Bruno |
| PRD-FR-06 | O cliente deve poder **solicitar uma nova secret** pela API. A anterior continua válida por **24 horas** | [09:21] Sofia |
| PRD-FR-07 | Cada notificação deve permitir ao cliente **verificar a origem e a integridade** por assinatura | [09:19] Sofia, [09:20] Sofia |
| PRD-FR-08 | Cada notificação leva um **identificador único de evento**, para o cliente descartar duplicatas | [09:25] Diego |
| PRD-FR-09 | Notificações que falham devem ser **retentadas automaticamente** com intervalos crescentes, num total de 5 retentativas | [09:15] Diego, [09:17] Larissa |
| PRD-FR-10 | Depois de esgotar as tentativas, a notificação vai para uma **fila de falhas (DLQ)**, com payload, motivo e horário | [09:18] Diego |
| PRD-FR-11 | Um usuário **ADMIN** deve poder **reprocessar** uma notificação da DLQ, e a ação fica registrada com o autor | [09:18] Diego, [09:36] Sofia |
| PRD-FR-12 | O cliente deve poder consultar o **histórico de entregas** de um webhook (últimas 100), com sucesso/falha, payload, resposta e tempo de resposta | [09:34] Marcos |
| PRD-FR-13 | A notificação deve trazer dados resumidos do pedido (identificação, status anterior e novo, cliente, total) e não os itens | [09:43] Diego |
| PRD-FR-14 | A notificação deve refletir o pedido **no momento da mudança**, mesmo que ele mude depois | [09:52] Larissa |
| PRD-FR-15 | Cada notificação informa **qual webhook cadastrado** a originou, para clientes com vários endpoints | [09:44] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-NFR-01 | **Latência:** < 10s entre a mudança e a primeira tentativa. Mínimo de ~2s aceito | [09:02] Marcos, [09:10] Larissa |
| PRD-NFR-02 | **Consistência:** se o status mudou, a notificação existe. Se a mudança foi desfeita, não existe | [09:06] Diego, [09:40] Bruno |
| PRD-NFR-03 | **Não degradar a operação:** um cliente lento ou fora do ar não pode atrasar nem impedir mudanças de status | [09:04] Bruno |
| PRD-NFR-04 | **Segurança:** só URLs `https` são aceitas | [09:23] Sofia |
| PRD-NFR-05 | **Segurança:** a secret é exclusiva de cada webhook, nunca global | [09:21] Sofia |
| PRD-NFR-06 | **Tamanho:** notificações acima de **64KB** não são enviadas e geram erro, sem truncar | [09:24] Larissa |
| PRD-NFR-07 | **Timeout:** o cliente tem até **10 segundos** para responder. Depois disso conta como falha | [09:42] Diego |
| PRD-NFR-08 | **Entrega:** at-least-once. O cliente pode receber duplicatas e deve deduplicar | [09:24] Diego |
| PRD-NFR-09 | **Ordem:** garantida só por pedido e enquanto houver um único processador (limitação conhecida) | [09:13] Larissa |
| PRD-NFR-10 | **Operação:** sem infraestrutura nova, só o banco MySQL e a stack atuais | [09:07] Diego, [09:30] Larissa |
| PRD-NFR-11 | **Auditoria:** o reprocessamento manual registra quem o executou | [09:36] Sofia |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | Detalhe |
|---|---|---|
| Notificação assíncrona via outbox no banco atual | Latência mínima de ~2s em troca de nunca travar a operação nem subir infra nova | [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em ~15h e depois DLQ | Uma notificação pode chegar com até ~15h de atraso. Depois disso, o reprocessamento é manual | [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) |
| HMAC por webhook com rotação de 24h | Mais complexidade para o cliente validar, em troca de segurança e de recuperação em caso de vazamento | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once em vez de exactly-once | O cliente precisa deduplicar, em troca de uma solução simples e padrão de mercado | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Notificação enxuta, sem itens | O cliente que precisa de detalhes faz uma chamada extra | [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md) |

## 9. Dependências

| ID | Dependência | Origem |
|---|---|---|
| PRD-DEP-01 | **Revisão de segurança da Sofia**, com pelo menos 2 dias úteis antes do deploy (HMAC e geração de secret) | [09:46] Sofia, [09:49] Sofia |
| PRD-DEP-02 | **Portal do desenvolvedor** atualizado pelo Marcos com guia de integração e destaque para a deduplicação | [09:26] Marcos, [09:40] Marcos |
| PRD-DEP-03 | **Confirmação do prazo** com a Atlas, a cargo do Marcos | [09:47] Marcos |
| PRD-DEP-04 | **Usuários do sistema que representam cada cliente**, para autenticar o cadastro via API | [09:32] Marcos |
| PRD-DEP-05 | **Sessão de revisão do design** com Bruno e Diego antes do início da codificação | [09:50] Larissa |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| PRD-RSK-01 | **Perder a Atlas** por atraso na entrega | Média | Alto | Escopo enxuto (e-mail, dashboard e rate limiting fora), estimativa de 3 sprints com revisão incluída e prazo confirmado com a Atlas ([09:37]–[09:40], [09:47]) |
| PRD-RSK-02 | **Cliente processa a mesma notificação duas vezes** por não deduplicar | Média | Médio | Identificador único por evento e destaque no portal ([09:25] Sofia, [09:26] Marcos) |
| PRD-RSK-03 | **Vazamento de secret** do cliente | Baixa | Alto | Secret por webhook e rotação com carência de 24h. Já aconteceu antes ([09:22] Diego) |
| PRD-RSK-04 | **Cliente fora do ar por mais de ~15h** perde notificações automáticas | Baixa | Médio | DLQ e reprocessamento manual por ADMIN. O aviso por e-mail fica para a próxima fase ([09:17] Marcos, [09:37] Larissa) |
| PRD-RSK-05 | **Rajada de notificações** sobrecarrega o endpoint do cliente | Média | Médio | Monitorar o volume por webhook e decidir sobre rate limiting depois ([09:39] Diego) |
| PRD-RSK-06 | **Atraso na revisão de segurança** empurra o deploy | Média | Médio | Reservar os 2 dias úteis da Sofia no planejamento da sprint 3 ([09:46] Sofia, [09:47] Larissa) |

## 11. Critérios de aceitação

| ID | Critério |
|---|---|
| PRD-AC-01 | Dado um webhook que assina `SHIPPED`, quando um pedido do customer passa para `SHIPPED`, o endpoint recebe a notificação em menos de 10s |
| PRD-AC-02 | Quando o pedido muda para um status **não** assinado, nenhuma notificação é enviada |
| PRD-AC-03 | Cadastrar um webhook com URL `http://` é recusado com erro de validação |
| PRD-AC-04 | O cliente consegue validar a assinatura de toda notificação com a secret recebida no cadastro |
| PRD-AC-05 | Depois de uma rotação, notificações assinadas continuam válidas para a secret antiga durante 24h, e só para a nova depois disso |
| PRD-AC-06 | Com o endpoint fora do ar, a notificação é retentada 5 vezes ao longo de ~15h e depois aparece na DLQ |
| PRD-AC-07 | Só um ADMIN consegue reprocessar uma notificação da DLQ, e o autor fica registrado |
| PRD-AC-08 | O histórico de entregas mostra as últimas 100 tentativas, com sucesso/falha, payload, resposta e tempo |
| PRD-AC-09 | Se o registro da notificação falhar, a mudança de status do pedido também é desfeita |

## 12. Estratégia de testes e validação

- **Testes de integração automatizados**, no padrão de `tests/orders.test.ts` (Vitest + Supertest, banco real), cobrindo CRUD, validação de URL, permissão de ADMIN no replay e inserção na outbox dentro do `changeStatus`. O detalhe por critério está no [FDD §13](FDD.md#13-critérios-de-aceite-técnicos).
- **Testes do worker com endpoint falso e relógio controlado**, para validar backoff, timeout, DLQ e limite de 64KB sem esperar 15h.
- **Teste ponta a ponta** com a integração no `order.service` (meia sprint reservada, [09:46] Larissa).
- **Revisão de segurança da Sofia** sobre assinatura e geração de secret antes do deploy ([09:46] Sofia).
- **Validação com clientes reais:** acompanhar a ativação dos 3 clientes (PRD-MET-02) e a latência p95 (PRD-MET-01) nos primeiros dias em produção.
