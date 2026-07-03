# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Webhooks outbound de notificação de mudança de status de pedidos |
| **PM** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado na reunião técnica (`TRANSCRICAO.md`) |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto

O Order Management System passará a notificar ativamente clientes B2B, via webhooks HTTP, sempre que o status de um pedido deles mudar (ciclo `PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, com cancelamentos — máquina de estados de `src/modules/orders/order.status.ts`). O cliente cadastra um endpoint HTTPS, escolhe quais status quer ouvir e recebe eventos assinados criptograficamente em menos de 10 segundos após a mudança. A feature nasce de um pedido formal de três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo ([09:00] Marcos) — e foi desenhada na reunião técnica registrada em `TRANSCRICAO.md`.

## 2. Problema e motivação

Hoje os clientes descobrem mudanças de status "batendo no GET /orders de tempos em tempos", o que deixa a integração deles **lenta e cara** ([09:00] Marcos). Não existe nenhum mecanismo de notificação externa no sistema. As consequências:

- Latência de detecção depende da frequência de polling do cliente, não do evento real.
- Carga desnecessária na API com consultas repetidas sem mudança.
- Risco comercial concreto: a Atlas sinalizou que **pode migrar para o concorrente** se a notificação não for entregue até o fim do trimestre ([09:00] Marcos).

## 3. Público-alvo e cenários de uso

**Público:** times de integração dos clientes B2B da plataforma (Atlas Comercial, MaxDistribuição e Nova Cargo como demandantes iniciais — [09:00]), além do time interno de operações (role `ADMIN`) que administra a fila de entregas.

**Cenários de uso:**
1. **Integração notificada:** o ERP da Atlas cadastra um webhook para `SHIPPED` e `DELIVERED`; quando o pedido é despachado, recebe o evento em segundos e atualiza o rastreamento sem polling ([09:33] Marcos: "só quero saber quando vira SHIPPED e DELIVERED").
2. **Verificação de autenticidade:** o cliente valida o header `X-Signature` (HMAC-SHA256) para garantir que o evento veio da plataforma e não foi adulterado ([09:19]–[09:20] Sofia).
3. **Diagnóstico pelo cliente:** o time de integração consulta o histórico das últimas entregas (sucesso/falha, payload, resposta, tempo) para depurar o lado dele ([09:34] Marcos).
4. **Recuperação operacional:** após uma indisponibilidade prolongada de um cliente, um admin reprocessa manualmente os eventos que caíram na DLQ ([09:18] Diego).
5. **Troca segura de credencial:** o cliente rotaciona a secret via API após um incidente interno, com 24h de convivência entre a antiga e a nova ([09:21] Sofia).

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar mudanças de status em "tempo real" na percepção do cliente | Tempo entre o commit da mudança de status e a entrega do webhook | **< 10 segundos** ([09:02] Marcos), com latência de detecção ≤ 2s pelo polling do worker ([09:10]) |
| Reter os clientes demandantes | Adoção pelos 3 clientes que fizeram o pedido formal; confirmação de prazo com a Atlas | 3 clientes integrados até o fim do trimestre ([09:00], [09:45]) |
| Eliminar a necessidade de polling dos clientes integrados | Redução das consultas `GET /orders` de clientes com webhook ativo | Redução observável após a adoção (baseline a medir) |
| Entregar no prazo comprometido | Sprints até produção, incluindo revisão de segurança | **3 sprints** ([09:46] Larissa) |

## 5. Escopo

### Incluso

- Cadastro, edição, remoção e listagem de webhooks por customer, com secret gerada pela plataforma ([09:31]–[09:33])
- Filtro por status: cada webhook escolhe quais transições quer receber ([09:33])
- Entrega assinada (HMAC-SHA256) com headers de identificação e deduplicação ([09:20], [09:44])
- Retry automático com backoff e DLQ com replay manual por admin ([09:15]–[09:18])
- Histórico de entregas consultável por webhook ([09:34])
- Rotação de secret com grace period de 24h ([09:21])

### Fora de escopo (descartado ou adiado na reunião)

| Item | Decisão | Origem |
| --- | --- | --- |
| E-mail avisando o cliente de webhook com falhas consecutivas | **Adiado** — "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" | [09:37] Larissa |
| Dashboard visual para o cliente acompanhar webhooks | **Descartado nesta fase** — "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend" | [09:40] Larissa |
| Rate limiting de envio para o cliente | **Adiado** — "a gente observa e implementa se virar problema" | [09:38]–[09:39] Diego/Larissa |
| Arquivamento de eventos entregues da outbox (~30 dias) | **Fora do escopo da feature** | [09:08] Diego |
| Múltiplos workers em paralelo | **Adiado** — "problema do futuro" | [09:13] Diego |
| Webhooks inbound (cliente → plataforma) | **Nunca esteve no escopo** — "Só saindo da gente pra eles" | [09:02] Marcos |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RF-01 | Cliente cadastra webhook via `POST` informando URL e lista de status desejados; a secret é gerada pela plataforma e devolvida na criação; o `customer_id` é informado no body/path (não vem do JWT) | [09:31] Marcos; [09:32] Larissa |
| RF-02 | Cliente edita webhook via `PATCH` (URL, filtro de status, ativação) | [09:32] Bruno |
| RF-03 | Cliente remove webhook via `DELETE` | [09:32] Bruno |
| RF-04 | Cliente lista os webhooks de um customer via `GET` | [09:32] Bruno |
| RF-05 | Cada webhook escolhe quais status quer receber; o filtro é aplicado na inserção na outbox (evento sem interessado não é registrado) | [09:33]–[09:34] Marcos/Bruno |
| RF-06 | Cliente consulta o histórico de entregas (últimos envios com sucesso/falha, payload, response e tempo de resposta) via `GET /webhooks/:id/deliveries` | [09:34] Marcos |
| RF-07 | Admin reprocessa item da DLQ via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente; exige role `ADMIN` e registra quem executou | [09:18] Diego; [09:36] Sofia |
| RF-08 | Cliente rotaciona a secret via API; a antiga permanece válida por 24h em paralelo | [09:21] Sofia |
| RF-09 | Toda entrega leva os headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` | [09:44] Diego/Sofia |
| RF-10 | O payload do evento é JSON com `event_id`, `event_type`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos como `total_cents` — sem `items` | [09:43] Diego |
| RF-11 | URL de webhook deve ser `https`; cadastro com `http` é recusado com erro de validação | [09:23] Sofia |
| RF-12 | Eventos que falham 5 vezes vão para a DLQ com payload, motivo da falha e timestamp | [09:17]–[09:18] |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RNF-01 | Latência de notificação < 10 segundos ("tempo real" na percepção do cliente) | [09:02] Marcos |
| RNF-02 | A emissão do evento não pode adicionar chamada HTTP nem bloqueio à transação de mudança de status | [09:04] Bruno |
| RNF-03 | Consistência absoluta: status mudou ⇔ evento registrado (mesma transação; rollback conjunto) | [09:40]–[09:41] Bruno/Diego |
| RNF-04 | Garantia de entrega at-least-once; duplicatas possíveis, dedup pelo cliente via `X-Event-Id` | [09:24]–[09:26] |
| RNF-05 | Timeout de 10s por tentativa de entrega | [09:42] Diego |
| RNF-06 | Payload limitado a 64KB; acima disso o envio falha (não trunca) | [09:23]–[09:24] |
| RNF-07 | Autenticidade e integridade verificáveis pelo cliente (HMAC-SHA256, secret por endpoint) | [09:20]–[09:21] Sofia |
| RNF-08 | Ordering garantida apenas por pedido e apenas com worker único — limitação documentada | [09:12]–[09:13] |
| RNF-09 | Worker resiliente a restart da API (processo separado) | [09:11] Diego |
| RNF-10 | Secrets nunca expostas em logs (redaction no logger Pino, padrão de `src/shared/logger/index.ts`) | Código + [09:22] |

## 8. Decisões e trade-offs principais

Resumo — o detalhamento de cada uma está no [RFC](RFC.md) e nos [ADRs](adrs/):

1. **Outbox no MySQL** em vez de fila externa ou disparo síncrono: consistência transacional sem infraestrutura nova; custo: latência mínima do polling e crescimento de tabela ([ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md)).
2. **Worker separado em polling de 2s**: simplicidade e isolamento; custo: até 2s de latência e vazão limitada a um processo ([ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)).
3. **5 tentativas com backoff 1m→12h + DLQ**: cobre indisponibilidades reais de horas; custo: cliente fora por ~15h+ depende de replay manual ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)).
4. **HMAC-SHA256 com secret por endpoint e rotação**: raio de dano de vazamento limitado; custo: gestão de ciclo de vida de secrets ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
5. **At-least-once com `X-Event-Id`**: padrão de mercado (Stripe, GitHub); custo: dedup vira responsabilidade do cliente ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).
6. **Reuso máximo dos padrões do projeto**: velocidade e uniformidade; custo: preso às convenções atuais ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)).
7. **Payload snapshot na inserção**: evento fiel ao instante da transição; custo: linhas maiores na outbox ([ADR-007](adrs/ADR-007-payload-snapshot-na-insercao-da-outbox.md)).

## 9. Dependências

- **Código existente:** transação do `changeStatus` (`src/modules/orders/order.service.ts`), máquina de estados (`src/modules/orders/order.status.ts`), autenticação/autorização (`src/middlewares/auth.middleware.ts`), erros (`src/shared/errors/`), logger (`src/shared/logger/index.ts`), schema (`prisma/schema.prisma`).
- **Times:** revisão de segurança pela Sofia — **mínimo 2 dias úteis antes do deploy**, com foco em HMAC e geração de secret ([09:46] Sofia); documentação de integração no portal de desenvolvedor pelo Marcos ([09:26], [09:40]).
- **Clientes:** precisam expor endpoint HTTPS, implementar verificação HMAC e deduplicação por `X-Event-Id` ([09:23]–[09:26]).
- **Infraestrutura:** nenhuma nova — MySQL e Node.js existentes ([09:07]).

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Atraso além do trimestre → churn da Atlas ([09:00]) | Média | Alto (perda de cliente para concorrente) | Escopo fechado em reunião com estimativa de 3 sprints já incluindo revisão de segurança ([09:46]–[09:47]); itens não essenciais explicitamente adiados (e-mail, dashboard, rate limiting) |
| Cliente com indisponibilidade prolongada perde notificações ([09:16] caso real de 2h) | Média | Médio (integração desatualizada) | Retry de ~15h em 5 tentativas + DLQ com replay manual ([09:17]–[09:18]); histórico de entregas para o cliente diagnosticar ([09:34]) |
| Vazamento de secret de webhook ([09:22] caso real) | Baixa | Alto (eventos forjáveis para aquele endpoint) | Secret por endpoint (não global), rotação via API com grace de 24h, redaction em logs ([09:21]–[09:22]) |
| Cliente não implementa dedup e processa evento duplicado | Média | Médio (efeitos duplicados no lado do cliente) | Semântica at-least-once documentada com destaque no portal + `X-Event-Id` em toda entrega ([09:25]–[09:26]) |
| Worker parado acumula eventos e fura o SLO de 10s | Baixa | Médio (notificações atrasadas em lote) | Processo separado com restart independente ([09:11]); monitoramento de lag da outbox (ver [FDD §9](FDD.md)) |

## 11. Critérios de aceitação

1. Cliente cadastra webhook com URL HTTPS e filtro de status, e recebe a secret na resposta da criação ([09:31], [09:23]).
2. Mudança de status assinada por um webhook ativo gera entrega no endpoint do cliente em menos de 10 segundos ([09:02]).
3. Mudança de status **não** assinada por nenhum webhook do customer não gera evento ([09:34]).
4. A assinatura `X-Signature` é verificável pelo cliente com a secret do endpoint (HMAC-SHA256 do corpo) ([09:20]).
5. Cliente consulta as entregas recentes com sucesso/falha, payload, response e tempo de resposta ([09:34]).
6. Endpoint indisponível recebe novas tentativas em 1m/5m/30m/2h/12h; após a 5ª falha o evento aparece na DLQ ([09:17]–[09:18]).
7. Replay da DLQ por usuário `ADMIN` reentrega o evento; usuário sem a role recebe 403; a operação registra o executor ([09:36]).
8. Rotação de secret mantém a antiga válida por 24h; após esse prazo, apenas a nova assina ([09:21]).
9. Cadastro com URL `http` é recusado com erro de validação ([09:23]).
10. Nenhum requisito descartado na reunião (e-mail, dashboard, rate limiting) está implementado.

## 12. Estratégia de testes e validação

- **Testes de integração da transação:** mudança de status com/sem webhook interessado; rollback forçado não deixa evento na outbox (garantia central — [09:40]–[09:41]). Segue o padrão da suíte existente em `tests/`.
- **Testes do worker:** entrega com sucesso (2xx), falha por status não-2xx, falha por timeout de 10s; progressão exata do backoff; movimentação para DLQ na 5ª falha; replay recolocando como pendente.
- **Testes de segurança:** assinatura HMAC verificável; rejeição de URL `http`; grace period de rotação (antiga válida por 24h, inválida depois); secret ausente de qualquer log; replay negado sem role `ADMIN`. Complementados pela **revisão manual de segurança da Sofia (≥2 dias úteis) antes do deploy** ([09:46]).
- **Teste ponta a ponta:** cenário completo criação → `PAID` → `SHIPPED` → `DELIVERED` com endpoint de teste recebendo os eventos na ordem e com payload snapshot correto — previsto na estimativa da Larissa ("Integração no order.service e testes ponta a ponta" — [09:46]).
- **Validação com clientes:** homologação com os 3 clientes demandantes e documentação no portal ([09:26], [09:47] Marcos confirma prazo com a Atlas).
