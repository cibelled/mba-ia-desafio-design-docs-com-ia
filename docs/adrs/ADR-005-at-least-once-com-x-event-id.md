# ADR-005 — Garantia de entrega at-least-once com deduplicação pelo cliente via `X-Event-Id`

- **Data da decisão:** reunião técnica de quinta-feira (registrada em [09:26] por Larissa)
- **Decisores:** Diego, Larissa; Sofia e Marcos de acordo
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)

## Status

Aceito

## Contexto

Com outbox, worker e retry, um evento pode ser enviado mais de uma vez. Isso acontece quando o cliente processou a requisição mas a resposta não chegou (timeout de 10s, [09:42] Diego), quando o worker cai entre o envio e a marcação como entregue, ou quando um admin faz replay da DLQ. Precisamos dizer claramente aos clientes que garantia oferecemos.

## Decisão

- A plataforma garante entrega **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado ([09:24] Diego).
- Cada evento leva um **UUID gerado na entrada na outbox**, enviado no header **`X-Event-Id`** e repetido no campo `event_id` do payload. O ID é único por evento e não muda entre retentativas nem no replay. O cliente faz dedup por ele ([09:25] Diego, [09:26] Larissa).
- O `event_id` é o próprio `id` (UUID) da linha da outbox, o que segue o padrão "tudo é uuid" do projeto ([09:51] Larissa).
- O Marcos vai destacar esse comportamento no portal do desenvolvedor ([09:26] Marcos).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | Exige coordenação dos dois lados e fica muito mais complexo. At-least-once com `event_id` resolve quase todos os casos e é o que Stripe e GitHub fazem ([09:25] Diego). |
| **At-most-once** (enviar uma vez, sem retry) | Alternativa plausível, mas incompatível com a política de retry do [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md). Clientes fora do ar perderiam eventos, e o retry existe justamente para cobrir indisponibilidade ([09:14] Larissa, [09:15] Diego). |

## Consequências

**Positivas**
- O worker continua simples: em dúvida, reenvia.
- Os clientes já conhecem esse modelo de Stripe e GitHub.

**Negativas / trade-offs**
- A responsabilidade de deduplicar passa para o cliente ([09:25] Sofia: "Isso joga responsabilidade pro cliente").
- Um cliente que não deduplicar pode processar a mesma mudança de status duas vezes. A mitigação é documentar isso no portal ([09:26] Marcos).
- Como há uma linha na outbox por webhook interessado ([ADR-001](ADR-001-outbox-no-mysql.md)), um customer com dois endpoints assinando o mesmo status recebe um `event_id` diferente em cada um. O `X-Webhook-Id` diz qual cadastro recebeu o envio ([09:44] Sofia).
