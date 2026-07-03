# ADR-006 — Reuso máximo dos padrões existentes do projeto

- **Status:** Aceito
- **Decisores:** Bruno (Eng. Pleno, Pedidos), Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma)
- **Origem:** `TRANSCRICAO.md` [09:27]–[09:30], [09:36] e [09:51]

## Contexto

A codebase do OMS tem padrões consolidados e consistentes entre os módulos existentes (`auth`, `users`, `customers`, `products`, `orders`). Introduzir a feature de webhooks com convenções próprias criaria uma ilha de estilo dentro do projeto e aumentaria o custo de manutenção. Bruno abriu o bloco da reunião afirmando: "A gente tem um padrão claro na codebase. [...] Webhook vai seguir igual" ([09:27]).

## Decisão

([09:30] Larissa: "Decisão: reuso máximo do que já existe.") O módulo de webhooks reutiliza, sem introduzir nada novo:

1. **Estrutura modular** — novo módulo em `src/modules/webhooks/` com controller, service, repository, routes e schemas, espelhando os módulos existentes ([09:27] Bruno). Exemplo do padrão: `src/modules/orders/order.controller.ts`, `order.service.ts`, `order.repository.ts`, `order.routes.ts`, `order.schemas.ts`. A lógica do worker fica dentro do módulo (ex.: `src/modules/webhooks/webhook.processor.ts`), com entry-point separada em `src/worker.ts` ([09:28] Bruno).
2. **Hierarquia de erros** — novas classes estendem `AppError` (`src/shared/errors/app-error.ts`), seguindo o modelo das específicas existentes como `InsufficientStockError` e `InvalidStatusTransitionError` (`src/shared/errors/http-errors.ts`). Códigos de erro com **prefixo `WEBHOOK_`** — ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno; [09:29] Larissa: "Prefixo WEBHOOK_ pra tudo do módulo").
3. **Error middleware centralizado** — `src/middlewares/error.middleware.ts` já trata `AppError`, `ZodError` e erros do Prisma; os erros de webhook são capturados sem nenhuma mudança nele ([09:29] Bruno).
4. **Logger Pino** — `src/shared/logger/index.ts` já está no projeto inteiro; nada novo de logging ([09:29] Bruno).
5. **Validação Zod** — schemas do módulo seguem o padrão de `src/modules/orders/order.schemas.ts` com o middleware `src/middlewares/validate.middleware.ts` ([09:30] Larissa).
6. **Autenticação e autorização existentes** — endpoints protegidos com `authenticate`; o replay de DLQ reaproveita `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts` ([09:36] Larissa).
7. **Chaves primárias UUID** — as novas tabelas usam UUID, seguindo o padrão do resto do projeto ("Tudo é uuid" — [09:51] Larissa).

## Alternativas Consideradas

### 1. Introduzir stack própria para o módulo (novo logger, novo formato de erro, libs de fila) — descartada

Cada dependência ou convenção nova aumenta a superfície de manutenção de um time pequeno e quebra a uniformidade que hoje permite a qualquer engenheiro navegar todos os módulos. A reunião foi explícita: "Não vamos botar nada novo" ([09:29] Bruno).

### 2. Webhooks como serviço separado (repositório/deploy próprio) — não considerada viável

Isolaria a feature, mas quebraria a garantia transacional da outbox (que exige participar da mesma transação SQL do `changeStatus` — ver [ADR-001](ADR-001-outbox-transacional-no-mysql.md)) e adicionaria custo de operação sem benefício para o escopo atual.

## Consequências

### Positivas

- Curva de aprendizado zero para o time: webhooks se parece com qualquer outro módulo do projeto.
- O error middleware, a autenticação e o logger funcionam para o módulo novo **sem alteração** ([09:29] Bruno).
- Revisão de código mais eficiente: desvios de padrão ficam evidentes.

### Negativas

- As convenções atuais viram restrição: se o padrão de módulos evoluir, webhooks acompanha o custo da migração como todos os demais.
- O worker (`src/worker.ts`) é o primeiro processo não-HTTP do projeto — o padrão existente cobre módulos de API, e o bootstrap do worker cria um precedente novo que precisa de desenho cuidadoso no FDD.

## Referências

- `src/app.ts` — `buildControllers`/`buildApp`, onde o módulo novo é registrado
- `src/routes/index.ts` — `buildApiRouter`, onde o router `/webhooks` é montado
- [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) — entry-point `src/worker.ts`
