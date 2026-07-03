# ADR-001 — Padrão Outbox transacional no MySQL

- **Status:** Aceito
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Origem:** `TRANSCRICAO.md` [09:03]–[09:08] e [09:40]–[09:41]

## Contexto

O OMS precisa notificar clientes B2B externos quando o status de um pedido muda. A mudança de status hoje acontece dentro de uma única transação SQL no método `changeStatus` de `src/modules/orders/order.service.ts`: atualização da linha em `orders`, inserção do registro de auditoria em `order_status_history` e ajuste de `stockQuantity` dos produtos ([09:04] Bruno). Qualquer mecanismo de notificação precisa garantir consistência com essa transação: se o status mudou, o evento **tem** que ser emitido; se houve rollback, o evento **não pode** existir ([09:40] Bruno, [09:41] Diego).

O time é pequeno e a infraestrutura atual se resume a Node.js + MySQL (via Prisma), sem broker de mensagens ([09:07] Diego).

## Decisão

Adotar o **padrão Transactional Outbox sobre o MySQL já existente** ([09:08] Larissa: "Tá decidido então: outbox em MySQL"):

- Na mesma transação SQL que atualiza `orders` e insere em `order_status_history`, o sistema insere uma linha em uma tabela de outbox (`webhook_outbox`) com o evento a ser entregue ([09:06] Diego).
- Um worker separado lê essa tabela e dispara as chamadas HTTP ([09:06] Diego; detalhado no [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).
- A tabela terá índice no campo de status do evento (pendente, processando, falhou, entregue) e em `created_at`; o worker lê apenas pendentes em batch pequeno ([09:08] Diego).
- Se a inserção na outbox falhar, a transação inteira sofre rollback — não existe estado em que o status mudou e o evento não foi registrado ([09:40] Bruno).

## Alternativas Consideradas

### 1. Disparo síncrono dentro do service de orders — descartada

Fazer a chamada HTTP ao cliente dentro do próprio `changeStatus`. Descartada porque a transação já é pesada e um cliente HTTP lento travaria mudanças de status de outros pedidos ([09:04] Bruno); além disso, se o cliente estiver fora do ar não há resposta razoável — dar rollback na mudança de status por falha de notificação é inaceitável ([09:04] Bruno; [09:06] Diego: "Síncrono está fora de questão").

### 2. Fila externa (Redis Streams ou similar) — descartada

Publicar os eventos em infraestrutura de fila dedicada. Descartada pelo custo operacional: exigiria subir e operar mais infraestrutura, o que é overengineering para um time pequeno quando o MySQL existente resolve ([09:07] Larissa e Diego).

## Consequências

### Positivas

- **Consistência garantida por construção:** evento e mudança de status commitam ou falham juntos; "não tem inconsistência possível" ([09:06] Diego).
- **Zero infraestrutura nova:** reusa o MySQL e o Prisma já operados pelo time ([09:07]).
- **Auditável:** a outbox é uma tabela consultável, com histórico natural dos eventos gerados.

### Negativas

- **Latência mínima imposta pelo polling:** o evento só sai quando o worker varre a tabela (aceito em [09:10], ver ADR-002).
- **Crescimento da tabela:** linhas entregues se acumulam; o arquivamento (proposto: após ~30 dias) ficou explicitamente **fora do escopo** desta feature ([09:08] Diego).
- **Acoplamento ao MySQL:** a garantia depende de tudo estar no mesmo banco; uma futura migração de storage de eventos exigiria revisitar esta decisão.

## Referências

- `src/modules/orders/order.service.ts` — transação do `changeStatus` (linhas 131–179) onde a inserção na outbox será acoplada
- `prisma/schema.prisma` — modelos `Order` e `OrderStatusHistory` que participam da mesma transação
- [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) — worker que consome a outbox
- [ADR-007](ADR-007-payload-snapshot-na-insercao-da-outbox.md) — conteúdo persistido na outbox
