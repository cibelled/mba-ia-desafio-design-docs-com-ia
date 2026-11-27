# ADR-007 — Payload renderizado (snapshot) na inserção da outbox, em formato enxuto

- **Data da decisão:** fim da reunião técnica de quinta-feira ([09:52] Bruno: "Beleza, snapshot. Decidido.")
- **Decisores:** Larissa, Diego, Bruno. Marcos e Sofia já tinham saído da call ([09:50]), então esta decisão e a de UUID ([09:51] Larissa) não passaram por eles.
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Status

Aceito

## Contexto

Entre o commit da mudança de status e o envio pelo worker passam pelo menos ~2s, e até ~15h em caso de retry ([ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)). Nesse intervalo o pedido pode mudar de novo. A outbox pode guardar o payload pronto ou guardar só o `order_id` e montar o payload no envio ([09:51] Bruno).

Também é preciso definir o formato do payload ([09:43] Marcos).

## Decisão

- **Snapshot na inserção:** o payload JSON é montado dentro da transação de `changeStatus` e gravado na coluna `payload` da `webhook_outbox`. Assim o evento reflete o estado de quando o status mudou ([09:52] Larissa, [09:52] Diego).
- **Formato enxuto** ([09:43] Diego, [09:44] Bruno): `event_id`, `event_type` (`"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`. **Os itens não vão.** Quem quiser detalhes consulta `GET /orders/:id`.
- **Teto de 64KB por payload.** Acima disso o envio falha com erro, sem truncar ([09:23] Sofia, [09:24] Diego, [09:24] Larissa). A Larissa classificou isso como requisito não funcional, e o lugar onde a regra é aplicada está detalhado no [FDD](../FDD.md).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Guardar só `order_id` e renderizar no envio** | Se o pedido mudar antes do envio, o evento descreveria um estado que não corresponde à transição notificada ("caso esquisito", [09:52] Larissa). Também obrigaria o worker a ler `orders` a cada tentativa. |
| **Payload completo com itens** | Infla o payload sem necessidade, porque o cliente já tem `GET /orders/:id` ([09:43] Diego). |

## Consequências

**Positivas**
- O payload de um evento não muda entre retentativas e replay. Isso combina com a dedup por `event_id` ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)).
- O worker não precisa consultar `orders`, só ler, assinar e enviar.
- O payload pequeno fica muito abaixo do teto de 64KB ([09:24] Diego: "Nenhum evento nosso vai chegar perto disso").

**Negativas / trade-offs**
- Gasta mais armazenamento por linha na outbox, já que o JSON fica duplicado.
- A montagem do payload passa a rodar dentro da transação de `changeStatus`. Usa dados já carregados, mas acrescenta um pouco de CPU à transação.
- Clientes que precisam dos itens fazem uma chamada extra a `GET /orders/:id`, o que traz de volta parte do tráfego que a feature quer cortar.
