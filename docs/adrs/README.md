# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante fica em um arquivo individual, no formato MADR, nomeado
`ADR-NNN-titulo-em-kebab-case.md`.

## Feature: Sistema de Webhooks de Notificação de Pedidos

| ADR | Decisão | Status |
|---|---|---|
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL existente | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2s | Aceito |
| [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md) | Retry 1m/5m/30m/2h/12h e DLQ em tabela separada | Aceito |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação com 24h | Aceito (revisão de segurança pendente) |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso máximo dos padrões do projeto | Aceito |
| [ADR-007](ADR-007-payload-snapshot-na-insercao.md) | Payload snapshot na inserção, formato enxuto | Aceito |
