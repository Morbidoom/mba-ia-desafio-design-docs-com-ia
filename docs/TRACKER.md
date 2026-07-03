# Tracker de Rastreabilidade

Cada item registrado no pacote de documentação (PRD, RFC, FDD, ADRs) mapeado à sua origem: a transcrição da reunião (`TRANSCRICAO.md`, formato `[hh:mm] Nome`) ou o código-fonte da aplicação (caminho de arquivo real). Itens marcados como **[derivado]** nos documentos são detalhes de implementação decorrentes de uma decisão registrada ou de um padrão do código — a linha aponta para essa origem.

## Requisitos funcionais (PRD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RF-01 | `docs/PRD.md` | Requisito Funcional | Cadastro de webhook via POST; secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-RF-01b | `docs/PRD.md` | Restrição | `customer_id` vai no body/path, não vem do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-RF-02 | `docs/PRD.md` | Requisito Funcional | Edição de webhook via PATCH | TRANSCRICAO | [09:32] Bruno |
| PRD-RF-03 | `docs/PRD.md` | Requisito Funcional | Remoção de webhook via DELETE | TRANSCRICAO | [09:32] Bruno |
| PRD-RF-04 | `docs/PRD.md` | Requisito Funcional | Listagem dos webhooks de um customer via GET | TRANSCRICAO | [09:32] Bruno |
| PRD-RF-05 | `docs/PRD.md` | Requisito Funcional | Filtro por status escolhido por webhook, aplicado na inserção na outbox | TRANSCRICAO | [09:33] Marcos / [09:34] Bruno |
| PRD-RF-06 | `docs/PRD.md` | Requisito Funcional | Histórico de entregas (sucesso/falha, payload, response, tempo) via GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-07 | `docs/PRD.md` | Requisito Funcional | Replay de DLQ via endpoint admin, role ADMIN, com log de quem executou | TRANSCRICAO | [09:18] Diego / [09:36] Sofia |
| PRD-RF-08 | `docs/PRD.md` | Requisito Funcional | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-RF-09 | `docs/PRD.md` | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp e X-Webhook-Id em toda entrega | TRANSCRICAO | [09:44] Diego / [09:44] Sofia |
| PRD-RF-10 | `docs/PRD.md` | Requisito Funcional | Payload JSON com event_id, event_type, timestamp ISO, order_id, order_number, from/to_status, customer_id, total_cents; sem items | TRANSCRICAO | [09:43] Diego |
| PRD-RF-11 | `docs/PRD.md` | Requisito Funcional | URL obrigatoriamente https; http recusado na validação | TRANSCRICAO | [09:23] Sofia |
| PRD-RF-12 | `docs/PRD.md` | Requisito Funcional | Evento com 5 falhas vai para DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:17] Diego / [09:18] Diego |

## Requisitos não funcionais (PRD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RNF-01 | `docs/PRD.md` | Requisito Não Funcional | Latência de notificação < 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-RNF-02 | `docs/PRD.md` | Requisito Não Funcional | Sem chamada HTTP dentro da transação de mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-RNF-03 | `docs/PRD.md` | Requisito Não Funcional | Consistência: evento na mesma transação, rollback conjunto | TRANSCRICAO | [09:40] Bruno |
| PRD-RNF-04 | `docs/PRD.md` | Requisito Não Funcional | Garantia at-least-once; dedup pelo cliente | TRANSCRICAO | [09:24] Diego |
| PRD-RNF-05 | `docs/PRD.md` | Requisito Não Funcional | Timeout de 10s por tentativa de entrega | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-06 | `docs/PRD.md` | Requisito Não Funcional | Limite de payload de 64KB; erro (não trunca) | TRANSCRICAO | [09:23] Sofia / [09:24] Diego |
| PRD-RNF-07 | `docs/PRD.md` | Requisito Não Funcional | Autenticidade/integridade via HMAC-SHA256 com secret por endpoint | TRANSCRICAO | [09:20] Sofia / [09:21] Sofia |
| PRD-RNF-08 | `docs/PRD.md` | Restrição | Ordering só por pedido e só com worker único (limitação documentada) | TRANSCRICAO | [09:13] Larissa |
| PRD-RNF-09 | `docs/PRD.md` | Requisito Não Funcional | Worker sobrevive a restart da API (processo separado) | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-10 | `docs/PRD.md` | Requisito Não Funcional | Secrets nunca em logs (redaction do Pino) | CODIGO | `src/shared/logger/index.ts` |

## Objetivos, escopo e riscos (PRD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo/Métrica | Entrega < 10s; "tempo real" na percepção do cliente | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo/Métrica | 3 clientes B2B demandantes integrados até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo/Métrica | Entrega em 3 sprints, revisão de segurança incluída | TRANSCRICAO | [09:46] Larissa |
| PRD-ESC-01 | `docs/PRD.md` | Fora de Escopo | E-mail de alerta de webhook falhando — próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-ESC-02 | `docs/PRD.md` | Fora de Escopo | Dashboard visual — projeto separado do frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-ESC-03 | `docs/PRD.md` | Fora de Escopo | Rate limiting de envio — observar e decidir depois | TRANSCRICAO | [09:39] Diego |
| PRD-ESC-04 | `docs/PRD.md` | Fora de Escopo | Arquivamento da outbox (~30 dias) fora do escopo | TRANSCRICAO | [09:08] Diego |
| PRD-ESC-05 | `docs/PRD.md` | Fora de Escopo | Múltiplos workers — "problema do futuro" | TRANSCRICAO | [09:13] Diego |
| PRD-ESC-06 | `docs/PRD.md` | Fora de Escopo | Webhooks inbound — fluxo é só outbound | TRANSCRICAO | [09:02] Marcos |
| PRD-RISK-01 | `docs/PRD.md` | Risco | Atraso além do trimestre → churn da Atlas | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | `docs/PRD.md` | Risco | Indisponibilidade prolongada de cliente (caso real de 2h) | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-03 | `docs/PRD.md` | Risco | Vazamento de secret (caso real em log de cliente) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | `docs/PRD.md` | Risco | Cliente sem dedup processa duplicata | TRANSCRICAO | [09:25] Sofia |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Revisão de segurança da Sofia: ≥2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Documentação de integração no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |

## RFC — proposta, alternativas e questões em aberto

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-PROP-01 | `docs/RFC.md` | Decisão | Arquitetura geral: outbox + worker + HMAC + retry/DLQ + at-least-once (resumo confirmado por todos) | TRANSCRICAO | [09:48] Larissa |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa Descartada | Disparo síncrono no service — transação pesada, cliente lento trava, rollback inaceitável | TRANSCRICAO | [09:04] Bruno / [09:06] Diego |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa Descartada | Redis Streams / fila externa — mais infra, overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa Descartada | Trigger MySQL — não notifica processo externo; polling atende | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa Descartada | 3 tentativas de retry (pouco) e retry indefinido (evento pendurado) | TRANSCRICAO | [09:15] Diego / [09:16] Diego |
| RFC-ALT-05 | `docs/RFC.md` | Alternativa Descartada | Exactly-once — coordenação dos dois lados, complexidade alta | TRANSCRICAO | [09:25] Diego |
| RFC-QA-01 | `docs/RFC.md` | Questão em Aberto | Rate limiting de saída — observar e decidir | TRANSCRICAO | [09:38] Diego |
| RFC-QA-02 | `docs/RFC.md` | Questão em Aberto | E-mail de alerta ao cliente — próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-QA-03 | `docs/RFC.md` | Questão em Aberto | Escala para múltiplos workers (particionamento/lock) | TRANSCRICAO | [09:13] Diego |
| RFC-QA-04 | `docs/RFC.md` | Questão em Aberto | Arquivamento das linhas entregues da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-IMP-01 | `docs/RFC.md` | Impacto | Alteração no código existente concentrada no changeStatus | CODIGO | `src/modules/orders/order.service.ts` |

## ADRs — decisões

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Decisão | Outbox no MySQL, inserção na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego / [09:08] Larissa |
| ADR-001b | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Decisão | Índices em status e created_at; worker lê pendentes em batch pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-002 | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Decisão | Worker em processo separado (src/worker.ts), polling de 2s | TRANSCRICAO | [09:09] Diego / [09:11] Larissa |
| ADR-002b | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Decisão | Mesmo banco, PrismaClient próprio por processo | TRANSCRICAO | [09:30] Bruno |
| ADR-003 | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa / [09:18] Diego |
| ADR-004 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Decisão | At-least-once com X-Event-Id (UUID gerado na inserção na outbox) | TRANSCRICAO | [09:25] Diego / [09:26] Larissa |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md` | Decisão | Reuso máximo: módulo padrão, AppError, Pino, error middleware, Zod, prefixo WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| ADR-006b | `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md` | Decisão | Estrutura de módulo espelhada nos existentes (controller/service/repository/routes/schemas) | CODIGO | `src/modules/orders/order.service.ts` (padrão) |
| ADR-006c | `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md` | Decisão | PKs UUID seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-007 | `docs/adrs/ADR-007-payload-snapshot-na-insercao-da-outbox.md` | Decisão | Payload renderizado (snapshot) na inserção na outbox | TRANSCRICAO | [09:52] Larissa |

## FDD — fluxos, modelo de dados e integração

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-FLUXO-01 | `docs/FDD.md` | Especificação | publishWebhookEvent(tx, order, fromStatus, toStatus) recebendo o tx da transação corrente | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | `docs/FDD.md` | Especificação | Loop do worker: a cada 2s busca pendentes mais antigos, processa, marca | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | `docs/FDD.md` | Especificação | Ordem por created_at, single-worker → ordering por pedido | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-04 | `docs/FDD.md` | Especificação | Replay: POST /admin/webhooks/dead-letter/:id/replay recoloca na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-DATA-01 | `docs/FDD.md` | Especificação | Tabela de configuração: url + secret + customer_id + estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DATA-02 | `docs/FDD.md` | Especificação | Outbox com status pendente/processando/falhou/entregue | TRANSCRICAO | [09:08] Diego |
| FDD-DATA-03 | `docs/FDD.md` | Especificação | Dead letter com payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DATA-04 | `docs/FDD.md` | Especificação | Deliveries com sucesso/falha, payload, response, tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | POST /webhooks (url, statuses, customerId; secret devolvida) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | GET /webhooks (listagem por customer) | TRANSCRICAO | [09:32] Bruno |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:32] Bruno |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | DELETE /webhooks/:id | TRANSCRICAO | [09:32] Bruno |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | GET /webhooks/:id/deliveries (últimos 100 envios) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | POST rotate-secret (nova secret; antiga válida 24h) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | POST /admin/webhooks/dead-letter/:id/replay (ADMIN) | TRANSCRICAO | [09:18] Diego / [09:36] Sofia |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Entrega ao cliente: payload JSON enxuto + 4 headers | TRANSCRICAO | [09:43] Diego / [09:44] Diego |
| FDD-CONTRATO-09 | `docs/FDD.md` | Contrato | CRUD com qualquer role autenticada; endurecer depois se preciso | TRANSCRICAO | [09:37] Sofia |
| FDD-ERR-01 | `docs/FDD.md` | Especificação | Códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED no padrão existente | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | `docs/FDD.md` | Especificação | Prefixo WEBHOOK_ para tudo do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-03 | `docs/FDD.md` | Especificação | Payload > 64KB: erro, não trunca | TRANSCRICAO | [09:23] Sofia / [09:24] Larissa |
| FDD-ERR-04 | `docs/FDD.md` | Especificação | Timeout de 10s tratado como falha, marca para retry | TRANSCRICAO | [09:42] Diego |
| FDD-OBS-01 | `docs/FDD.md` | Especificação | Log de auditoria no replay: registrar quem executou | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-02 | `docs/FDD.md` | Especificação | Logs estruturados e redaction seguem o logger existente; métricas/tracing derivados | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-01 | `docs/FDD.md` | Integração | Extensão da transação do changeStatus (linhas 131–179) com insert na outbox | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | Máquina de estados define eventos possíveis e valores válidos do filtro | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | Novos modelos seguem padrões do schema (uuid Char(36), @@map, índices) | CODIGO | `prisma/schema.prisma` |
| FDD-INT-04 | `docs/FDD.md` | Integração | Erros estendem AppError no modelo das classes específicas existentes | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | authenticate + requireRole('ADMIN') protegem as rotas | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | Error middleware captura os novos erros sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | Registro do módulo em buildControllers/buildApiRouter | CODIGO | `src/app.ts` / `src/routes/index.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | Worker segue bootstrap do server.ts e cria PrismaClient próprio | CODIGO | `src/server.ts` / `src/config/database.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração | Validação Zod (inclusive https-only) no padrão do middleware existente | CODIGO | `src/middlewares/validate.middleware.ts` |

## Cobertura

- **Total de linhas:** 92 — cobrindo os requisitos, decisões, restrições, contratos, alternativas, questões em aberto e riscos identificáveis nos 10 documentos do pacote (cobertura > 80%).
- **Fonte TRANSCRICAO:** 79 linhas (~86%), todas com timestamp no formato `[hh:mm] Nome` verificado contra `TRANSCRICAO.md`.
- **Fonte CODIGO:** 13 linhas, todas com caminho de arquivo real existente no repositório.
- Itens marcados **[derivado]** nos documentos (ex.: nomes exatos de tabelas, status codes HTTP das respostas, métricas propostas) apontam aqui para a decisão ou arquivo que os fundamenta.
