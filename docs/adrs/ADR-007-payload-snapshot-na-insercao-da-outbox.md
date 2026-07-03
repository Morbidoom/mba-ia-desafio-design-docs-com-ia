# ADR-007 — Payload persistido como snapshot na inserção da outbox

- **Status:** Aceito
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Origem:** `TRANSCRICAO.md` [09:51]–[09:52]

## Contexto

Ao inserir um evento na outbox, existem duas formas de guardar o conteúdo: persistir o payload já renderizado (JSON final que será enviado ao cliente) ou guardar apenas o `order_id` e montar o payload na hora do envio, consultando o estado atual do pedido. A dúvida foi levantada por Bruno no fim da reunião ([09:51]).

Entre a inserção do evento e a entrega podem se passar horas (retry de até ~15h — [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)), e nesse intervalo o pedido pode mudar de novo.

## Decisão

**O payload é renderizado e persistido no momento da inserção na outbox (snapshot)** ([09:52] Larissa: "Eu prefiro renderizado já, na hora da inserção"; [09:52] Diego: "snapshot na inserção"; [09:52] Bruno: "Beleza, snapshot. Decidido.").

Assim, se o pedido mudar depois, o evento ainda reflete o estado de quando **aquela** mudança de status aconteceu — sem "caso esquisito" de um evento antigo carregar dados novos ([09:52] Larissa).

O conteúdo do snapshot segue o formato definido em [09:43] (Diego): JSON com `event_id`, `event_type` (ex.: `order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents` — **sem `items`**, para não inflar; cliente que quiser detalhe consulta `GET /orders/:id` depois ([09:43] Diego).

## Alternativas Consideradas

### 1. Guardar só `order_id` e renderizar no envio — descartada

Economizaria espaço na tabela, mas o payload passaria a refletir o estado do pedido **no momento do envio**, não da mudança de status que originou o evento. Com retries de horas, um evento de `PAID` poderia ser entregue descrevendo um pedido já `SHIPPED` — exatamente o "caso esquisito" apontado por Larissa ([09:52]). Também acoplaria a renderização ao caminho quente do worker e a registros que podem ter sido alterados ou removidos.

## Consequências

### Positivas

- Evento imutável e fiel ao instante da transição de status — semanticamente correto para o consumidor e consistente entre tentativas de retry.
- Worker mais simples e rápido: lê a linha da outbox e envia, sem joins para montar payload.
- Retry e replay da DLQ reenviam exatamente os mesmos bytes, mantendo a assinatura HMAC estável por evento.

### Negativas

- **Linhas maiores na outbox** (payload completo em vez de uma FK), reforçando a necessidade do arquivamento futuro já mencionado em [09:08].
- Mudança no formato do payload exige cuidado com eventos antigos ainda pendentes na outbox, que carregam o formato da época da inserção.

## Referências

- `src/modules/orders/order.service.ts` — a transação onde o snapshot é montado e persistido
- [ADR-001](ADR-001-outbox-transacional-no-mysql.md) — a outbox onde o snapshot vive
- [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) — assinatura calculada sobre o payload persistido
