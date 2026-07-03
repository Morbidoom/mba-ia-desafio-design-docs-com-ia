# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

- **Status:** Aceito
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Sofia (Eng. de Segurança)
- **Origem:** `TRANSCRICAO.md` [09:14]–[09:18] e [09:35]–[09:36]

## Contexto

Endpoints de clientes ficam indisponíveis — houve caso real de cliente com 2 horas de indisponibilidade em manutenção planejada ([09:16] Diego). O worker precisa de uma política definida para falhas de entrega: quantas vezes tentar, com que espaçamento, e o que fazer quando esgotar as tentativas ([09:14] Larissa).

## Decisão

1. **Retry com backoff exponencial, 5 tentativas no total**, com progressão de intervalos **1m / 5m / 30m / 2h / 12h** — janela de quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego; [09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h").
2. **Após esgotar as tentativas, o evento vai para uma DLQ persistida em tabela separada** (`webhook_dead_letter`), guardando payload, motivo da falha e timestamp ([09:18] Diego). Tabela separada mantém a leitura da outbox principal limpa e serve de evidência para debug e reprocessamento ([09:18] Diego).
3. **Reprocessamento manual via endpoint admin** — `POST /admin/webhooks/dead-letter/:id/replay` recoloca o evento na outbox como pendente ([09:18] Diego). O endpoint exige **role `ADMIN`** e registra em log quem executou o replay, para auditoria ([09:36] Sofia e Larissa), reaproveitando o `requireRole` existente ([09:36] Larissa).

Timeout de 10 segundos por chamada HTTP: resposta que não chega em 10s é tratada como falha e marcada para retry ([09:42] Diego).

## Alternativas Consideradas

### 1. Apenas 3 tentativas (mais agressivo) — descartada

Proposta por Bruno ([09:16]). Descartada porque 3 tentativas seriam esgotadas em cerca de 30 minutos, matando eventos de clientes com indisponibilidade de algumas horas — cenário que já aconteceu na prática ([09:16] Diego).

### 2. Retry indefinido com backoff — descartada

Defendida por parte do mercado, mas traz o problema de eventos pendurados para sempre quando o cliente simplesmente sumiu ([09:15] Diego).

### 3. Marcar como `failed` na própria outbox (sem tabela DLQ) — descartada

Evitaria uma tabela extra, mas sujaria a leitura da outbox principal e perderia o registro estruturado (payload + motivo + timestamp) que serve de evidência para debug e reprocessamento ([09:17] Larissa levantou; [09:18] Diego descartou).

## Consequências

### Positivas

- Tolera janelas reais de indisponibilidade de clientes (até ~15h) sem perder eventos.
- Falhas permanentes ficam isoladas e documentadas na DLQ, com trilha para reprocessamento manual.
- Replay auditado: apenas `ADMIN`, com log de quem executou ([09:36] Sofia).

### Negativas

- **Cliente que caiu por mais de ~15h perde a entrega automática** — recuperação passa a depender de replay manual.
- **Intervenção humana na DLQ:** não há reprocessamento automático nem alerta ao cliente nesta fase (e-mail de aviso ficou para fase futura — [09:37] Larissa).
- Mais uma tabela e mais um fluxo administrativo para manter.

## Referências

- `src/middlewares/auth.middleware.ts` — `requireRole('ADMIN')` reaproveitado no endpoint de replay
- [ADR-001](ADR-001-outbox-transacional-no-mysql.md) — outbox de onde o retry lê e para onde o replay devolve eventos
- [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) — replay pode gerar entrega duplicada, coberta pela semântica at-least-once
