# ADR-001 — Padrão Outbox no MySQL existente para eventos de webhook

- **Data da decisão:** reunião técnica de quinta-feira, 09:00 (registrada em [09:08] por Larissa)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-payload-snapshot-na-insercao.md)

## Status

Aceito

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) querem ser avisados quando o status de um pedido muda. Hoje eles fazem polling no `GET /orders` ([09:00] Marcos).

A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`). O método roda dentro de `this.prisma.$transaction(...)` e, na mesma transação:

1. valida a transição com `canTransition` (`src/modules/orders/order.status.ts`);
2. debita ou repõe estoque (`shouldDebitStock` / `shouldReplenishStock`);
3. atualiza `orders.status`;
4. insere em `order_status_history`.

O Bruno apontou que essa transação já é pesada. Pôr uma chamada HTTP no meio dela deixaria a mudança de status de todos os pedidos refém de um cliente lento. Também não faz sentido dar rollback na mudança de status porque o endpoint do cliente está fora do ar ([09:04] Bruno).

Precisamos então desacoplar o envio, sem perder eventos e sem emitir evento de uma mudança que sofreu rollback.

## Decisão

Adotar o **padrão Transactional Outbox no MySQL já usado pela aplicação** ([09:06] Diego, [09:08] Larissa):

- Criar a tabela `webhook_outbox` (model Prisma `WebhookOutbox` em `prisma/schema.prisma`).
- Dentro da **mesma transação** de `changeStatus`, inserir uma linha por webhook interessado no evento. Se a inserção na outbox falhar, a mudança de status inteira sofre rollback ([09:40] Bruno).
- A tabela tem índice em `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`) e em `created_at` ([09:08] Diego).
- O processamento cabe a um worker separado ([ADR-002](ADR-002-worker-separado-em-polling.md)).
- O filtro de eventos é aplicado **na inserção**: se nenhum webhook ativo do customer assina o `to_status`, nenhuma linha é criada ([09:34] Bruno, [09:34] Diego).
- Os IDs são UUID, como no restante do schema ([09:51] Larissa).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo síncrono no `changeStatus`** | Prende a transação a I/O externo. Um cliente lento trava mudanças de status de outros pedidos, e um cliente fora do ar exigiria rollback de uma mudança de negócio legítima ([09:04] Bruno; [09:06] Diego: "Síncrono está fora de questão"). |
| **Redis Streams / Redis Cluster** | Exige subir e operar infraestrutura nova. Para um time pequeno, é overengineering ([09:07] Larissa, [09:07] Diego). Também não resolve sozinho a atomicidade com a transação MySQL. |

## Consequências

**Positivas**
- Atomicidade: se o status commitou, o evento existe; se houve rollback, o evento some junto ([09:06] Diego).
- Nenhuma infraestrutura nova, só uma tabela no banco e no Prisma que já existem.
- O filtro na inserção economiza linhas e trabalho do worker ([09:34] Bruno).

**Negativas / trade-offs**
- A transação de `changeStatus` fica um pouco maior: uma leitura dos webhooks do customer e N inserts. Aceitamos esse custo em troca da consistência.
- O MySQL passa a fazer papel de fila. Com o tempo a tabela cresce. O arquivamento de linhas entregues (cerca de 30 dias) foi deixado **fora do escopo** desta feature ([09:08] Diego).
- A latência mínima passa a depender do intervalo de polling do worker ([ADR-002](ADR-002-worker-separado-em-polling.md)).
