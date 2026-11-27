# ADR-003 — Retry com backoff exponencial (1m/5m/30m/2h/12h) e DLQ em tabela separada

- **Data da decisão:** reunião técnica de quinta-feira (registrada em [09:17] e [09:19])
- **Decisores:** Larissa, Diego, Bruno; Marcos validou a janela de 15h; Sofia definiu a regra de acesso ao replay
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Status

Aceito, com um ponto de confirmação: a leitura de "5 tentativas" (RFC-QA-06, ver Decisão)

## Contexto

O endpoint do cliente pode estar fora do ar ou lento. Já houve cliente com duas horas de indisponibilidade em manutenção planejada ([09:16] Diego). Precisamos de uma política que:

- cubra indisponibilidades de horas sem descartar eventos cedo demais;
- não deixe eventos pendurados para sempre quando o cliente sumiu ([09:15] Diego);
- preserve as falhas definitivas para debug e reprocessamento.

## Decisão

**Retry**
- Backoff exponencial com a progressão fixa **1 min → 5 min → 30 min → 2 h → 12 h** ([09:17] Diego, [09:17] Larissa).
- **Teto de 5 tentativas** de retry ([09:15] Diego, [09:16] Larissa). A janela total, da primeira falha à última tentativa, é de ~14h36min ("quase 15 horas", [09:17] Diego). Marcos considerou aceitável ([09:17] Marcos).
- **Como lemos o "5 tentativas":** a soma dos 5 intervalos (1+5+30+120+720 = 876 min) só bate com "quase 15 horas entre a primeira falha e a última tentativa" se houver um envio inicial seguido de 5 retentativas, ou seja, até 6 requisições HTTP por evento. Há uma fala em sentido contrário: no resumo final, a Larissa diz "**total** 5 tentativas" ([09:48] Larissa), o que sugere 5 envios e uma janela de só ~2h36min. Adotamos a leitura de 6 requisições porque é a única que usa os 5 intervalos e a janela de ~15h aceita pelo Marcos. O conflito está registrado como ponto de confirmação no [RFC](../RFC.md#5-questões-em-aberto) (RFC-QA-06).
- Conta como falha: resposta fora de 2xx, erro de rede ou **timeout de 10s** ([09:42] Diego).

**DLQ**
- Depois da última tentativa, o evento vai para a **tabela separada `webhook_dead_letter`**, com payload, motivo da falha e timestamp ([09:18] Diego). A linha é removida da `webhook_outbox` na mesma transação, o que mantém a leitura da outbox limpa.
- **Replay manual** por `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como `PENDING` ([09:18] Diego, [09:35] Diego).
- O replay exige **role `ADMIN`**, reaproveitando o `requireRole` de `src/middlewares/auth.middleware.ts`, e **registra quem fez o replay** para auditoria ([09:36] Sofia, [09:36] Larissa).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | O evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego). |
| **3 tentativas** (mais agressivo, [09:16] Bruno) | Pouco: uma indisponibilidade de manhã esgotaria as tentativas em ~30 min, e já houve cliente com 2h de manutenção ([09:16] Diego). |
| **Marcar como `failed` na própria outbox** (sem tabela de DLQ) | Considerada na pergunta de [09:17] Larissa. Perdeu porque a tabela separada deixa a leitura da outbox principal mais limpa e serve de evidência para debug e reprocessamento ([09:18] Diego). |

## Consequências

**Positivas**
- Cobre indisponibilidades de até ~15h sem perder eventos.
- A DLQ dá visibilidade e um caminho de recuperação controlado e auditado.
- A query do worker continua simples, porque eventos mortos saem da outbox.

**Negativas / trade-offs**
- Um evento pode chegar até ~15h atrasado. Para esse caso, o "tempo real" do produto não vale ([09:17] Marcos).
- O reprocessamento é **manual** e exige um ADMIN atento. Não há alerta automático: o aviso por e-mail ao cliente ficou para a próxima fase ([09:37] Larissa).
- Durante o backoff de um evento, eventos posteriores do mesmo pedido podem ser entregues antes dele. Isso quebra a ordem implícita do [ADR-002](ADR-002-worker-separado-em-polling.md) e ficou como questão em aberto.
- O replay gera uma nova entrega do mesmo `event_id`. É coerente com at-least-once ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)).
