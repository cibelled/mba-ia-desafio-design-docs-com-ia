# ADR-004 — Assinatura HMAC-SHA256 com secret única por endpoint e rotação com carência de 24h

- **Data da decisão:** reunião técnica de quinta-feira (registrada em [09:22] por Sofia)
- **Decisores:** Sofia (Segurança), Larissa, Diego, Bruno
- **Relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Status

Aceito (implementação sujeita à revisão de segurança da Sofia antes do deploy — [09:46] Sofia)

## Contexto

Os webhooks levam dados de pedidos para um endpoint fora da nossa infraestrutura. O cliente precisa conferir duas coisas: que a requisição veio de nós e que o payload não foi adulterado no caminho ([09:19] Sofia). Os webhooks são só de saída (outbound), então não precisamos validar requisições de entrada ([09:02] Marcos, [09:03] Sofia).

Já houve caso de cliente que vazou uma secret no log da própria aplicação ([09:22] Diego). Por isso o vazamento de uma secret precisa ter impacto limitado e ser recuperável.

## Decisão

- **HMAC-SHA256 sobre o corpo do request**, enviado no header `X-Signature` ([09:20] Sofia, [09:22] Sofia). A assinatura usa `node:crypto`, sem dependência nova.
- **Secret única por endpoint de webhook**, nunca global da plataforma ([09:21] Sofia).
- A **secret é gerada pela plataforma** e devolvida na criação do webhook ([09:31] Marcos).
- A tabela de configuração `webhook_endpoints` guarda `url`, `secret`, `customer_id` e o estado ativo ([09:21] Bruno, [09:21] Sofia).
- **Rotação pela API:** um endpoint permite ao cliente pedir uma nova secret. A secret anterior continua **válida em paralelo por 24h** e depois é invalidada ([09:21] Sofia).
- **TLS obrigatório:** só URLs `https://`. Uma URL `http://` é recusada com erro de validação no schema Zod. A Sofia deixou claro que isso é validação, não decisão arquitetural ([09:23] Sofia).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Secret global da plataforma** | Se vaza uma, vaza tudo ([09:21] Sofia). |
| **Rotação sem período de carência** (troca imediata) | *Alternativa não discutida na reunião; incluída por ser plausível.* Foi descartada porque o cliente precisa de tempo para migrar os próprios sistemas, daí as 24h em paralelo ([09:21] Sofia). |

## Consequências

**Positivas**
- HMAC-SHA256 é padrão de mercado e todo cliente sério tem biblioteca para validá-lo ([09:20] Sofia).
- O vazamento de uma secret afeta só um endpoint e se resolve com rotação, sem downtime para o cliente.

**Negativas / trade-offs**
- Diferente de uma senha de usuário, a secret **não pode ser guardada como hash** (bcrypt, como em `passwordHash` de `prisma/schema.prisma`), porque precisamos dela em texto para assinar. Como protegê-la em repouso não foi discutido e fica como ponto para a revisão da Sofia (ver [RFC](../RFC.md#5-questões-em-aberto)).
- A carência de 24h exige guardar duas secrets por algum tempo e definir **como o cliente valida durante a janela**. A proposta do [FDD](../FDD.md) é enviar as duas assinaturas, e ela também depende da revisão.
- A assinatura cobre só o corpo, como decidido em [09:22]. O `X-Timestamp` ([09:44] Diego) **não** entra no HMAC, então a proteção contra replay pelo timestamp é limitada. Registrado como questão em aberto.
- Entram na conta dois dias úteis de revisão de segurança antes do deploy ([09:46] Sofia).
