# ADR-002 — Worker em processo separado, lendo a outbox por polling de 2 segundos

- **Data da decisão:** reunião técnica de quinta-feira (registrada em [09:10] e [09:11])
- **Decisores:** Larissa, Diego, Bruno; Marcos validou a latência
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Status

Aceito

## Contexto

Com a outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), algum componente precisa ler os eventos pendentes e fazer as chamadas HTTP. Duas questões ficam em aberto:

1. **Como o consumidor descobre que há eventos novos.** O requisito de produto é entregar em menos de 10 segundos, que para os clientes já é "tempo real" ([09:02] Marcos).
2. **Onde o consumidor roda.** Hoje o projeto tem um único entry-point, `src/server.ts`, que sobe o Express via `buildApp` (`src/app.ts`) e abre o `PrismaClient` de `src/config/database.ts`.

## Decisão

- **Polling em loop a cada 2 segundos.** A cada ciclo o worker busca, em lote pequeno, os eventos `PENDING` mais antigos (ordenados por `created_at`), processa e marca o resultado ([09:09] Diego, [09:08] Diego). Fica aceita uma latência de pelo menos ~2s ([09:10] Larissa, [09:10] Marcos).
- **Processo separado da API.** Novo entry-point `src/worker.ts` (novo), espelhando `src/server.ts`, e novo script `npm run worker` em `package.json` ([09:11] Larissa). Se a API reiniciar, o worker não cai junto ([09:11] Diego).
- **Mesma stack e mesmo banco:** mesma `DATABASE_URL` (validada por `src/config/env.ts`), mas uma **instância própria de `PrismaClient`**, porque o client é por processo ([09:30] Bruno).
- A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.processor.ts` (novo). `src/worker.ts` só faz bootstrap e shutdown ([09:28] Bruno).
- **Um único worker** nesta fase. Com isso a ordem de entrega fica garantida apenas por `order_id`, e só enquanto houver um worker. Isso é uma **limitação conhecida** ([09:12] Diego, [09:13] Larissa).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Trigger no banco / mecanismo reativo** | O MySQL não tem `NOTIFY/LISTEN` como o Postgres. Trigger só executa SQL e não avisa processo externo. Avisar o worker exigiria gambiarras, como escrever em arquivo ou chamar um endpoint ([09:09] Bruno, [09:09] Diego). |
| **Worker dentro do processo da API** | Um restart da API derruba o worker ([09:11] Diego). |
| **Múltiplos workers em paralelo** | Perde a ordem por pedido. Particionar por `order_id` ou usar lock pessimista ficou para o futuro ([09:13] Diego). |

## Consequências

**Positivas**
- É simples: um `setInterval`/loop e uma query indexada (`status`, `created_at`).
- 2s de polling cabe com folga no limite de 10s ([09:09] Diego).
- Isolamento de falhas entre API e worker, sem infraestrutura nova.

**Negativas / trade-offs**
- Há consultas ao banco a cada 2s mesmo sem eventos. Aceitamos esse custo em troca de não ter infra reativa.
- Latência mínima de ~2s no pior caso ([09:10] Larissa).
- Escala horizontal bloqueada: o worker único é ponto único de processamento. Se ele parar, os eventos acumulam (sem perda) na outbox.
- Ordem só por `order_id` e só com um worker. Um evento em retry pode ser ultrapassado por um evento posterior do mesmo pedido (ver questão em aberto no [RFC](../RFC.md)).
- Passam a existir dois processos para operar (`npm run dev`/`start` e `npm run worker`).
