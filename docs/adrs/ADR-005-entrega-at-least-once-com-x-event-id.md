# ADR-005 — Garantia de entrega at-least-once com deduplicação por X-Event-Id

- **Status:** Aceito
- **Decisores:** Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead), Sofia (Eng. de Segurança), Marcos (PM)
- **Origem:** `TRANSCRICAO.md` [09:24]–[09:26]

## Contexto

Com outbox + worker + retry, existem cenários em que o mesmo evento pode ser entregue mais de uma vez — por exemplo, o worker envia, o cliente recebe, mas a confirmação falha antes de o evento ser marcado como entregue; ou um replay manual da DLQ reenvia um evento já processado. É preciso definir qual garantia de entrega a plataforma oferece e como o cliente lida com duplicatas.

## Decisão

([09:26] Larissa: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão.")

1. **A plataforma garante entrega at-least-once:** o cliente pode receber o mesmo evento mais de uma vez e tem que estar preparado ([09:24] Diego).
2. **Cada evento carrega um `event_id` (UUID)** gerado quando entra na outbox, enviado no header **`X-Event-Id`**. É único por evento; se o cliente receber duas vezes, deduplica pelo `event_id` do lado dele ([09:25] Diego).
3. A responsabilidade de deduplicação fica **documentada com destaque no portal de desenvolvedor** para os clientes ([09:26] Marcos).

## Alternativas Consideradas

### 1. Garantia exactly-once — descartada

Exigiria coordenação entre os dois lados (plataforma e cliente) e tornaria o sistema muito mais complexo. At-least-once com `event_id` "resolve 99% dos casos" e é o padrão de mercado — Stripe e GitHub fazem assim ([09:25] Diego).

### 2. At-most-once (enviar uma única vez, sem retry) — incompatível

Nunca esteve na mesa como opção séria: sem reenvio, qualquer falha transitória do cliente perderia o evento definitivamente, violando o motivo de existir da feature (clientes deixarem de fazer polling porque confiam na notificação — [09:00] Marcos).

## Consequências

### Positivas

- Modelo simples e alinhado ao padrão de mercado, com o qual os times de integração dos clientes já estão acostumados ([09:25] Diego).
- Compatível com retry e com replay manual da DLQ sem lógica adicional do lado da plataforma.
- O UUID nasce junto com o evento na outbox, dentro da transação — identidade estável do evento por todo o ciclo de vida.

### Negativas

- **Empurra responsabilidade para o cliente**, que precisa implementar deduplicação por `X-Event-Id` ([09:25] Sofia apontou; aceito com a mitigação de documentação destacada no portal — [09:26] Marcos).
- Cliente que ignorar a documentação pode processar eventos em duplicidade (ex.: disparar duas faturas).

## Referências

- [ADR-001](ADR-001-outbox-transacional-no-mysql.md) — o UUID é gerado na inserção do evento na outbox
- [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) — retry e replay, as fontes de duplicação
- [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) — headers que acompanham `X-Event-Id` na entrega
