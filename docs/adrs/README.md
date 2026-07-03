# Architectural Decision Records

ADRs da feature **Sistema de Webhooks de Notificação de Pedidos**, no formato MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências), cada um com a origem rastreável em `TRANSCRICAO.md`.

| ADR | Decisão | Origem principal |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-transacional-no-mysql.md) | Padrão Outbox transacional no MySQL | [09:06]–[09:08] |
| [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado com polling de 2s | [09:09]–[09:13] |
| [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ | [09:14]–[09:18] |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação | [09:19]–[09:23] |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com X-Event-Id | [09:24]–[09:26] |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso máximo dos padrões existentes do projeto | [09:27]–[09:30] |
| [ADR-007](ADR-007-payload-snapshot-na-insercao-da-outbox.md) | Payload persistido como snapshot na inserção | [09:51]–[09:52] |

Ordem de leitura sugerida: 001 → 002 → 003 (o pipeline de entrega), depois 004 → 005 (segurança e semântica), 006 → 007 (integração e detalhe).
