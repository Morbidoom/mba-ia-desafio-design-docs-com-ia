# ADR-002 — Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM)
- **Origem:** `TRANSCRICAO.md` [09:08]–[09:13] e [09:29]–[09:30]

## Contexto

Com a outbox decidida ([ADR-001](ADR-001-outbox-transacional-no-mysql.md)), é preciso definir **como** os eventos pendentes são consumidos e entregues. Os clientes B2B consideram "tempo real" qualquer notificação abaixo de 10 segundos ([09:02] Marcos). O MySQL não possui mecanismo nativo de notificação a processos externos equivalente ao NOTIFY/LISTEN do Postgres ([09:09] Diego).

## Decisão

1. **Consumo por polling em loop:** a cada 2 segundos o worker busca os eventos pendentes mais antigos, processa em batch pequeno e marca como entregues ([09:09] Diego; [09:10] Larissa: "Worker em polling, 2s"). A latência mínima de 2 segundos no pior caso foi aceita explicitamente ([09:10] Larissa) e atende com folga o requisito de menos de 10 segundos ([09:09] Diego, [09:10] Marcos).
2. **Processo separado da API:** o worker roda como entry-point própria (`src/worker.ts`, com script `npm run worker`), nos moldes do `src/server.ts` existente ([09:11] Larissa). Motivo: se rodasse dentro da instância da API, um restart da API derrubaria o worker ([09:11] Diego).
3. **Mesmo banco, mesma stack, `PrismaClient` próprio:** o worker conecta na mesma `DATABASE_URL`, mas instancia um `PrismaClient` novo, porque o client é por processo Node ([09:11] Bruno e Diego; [09:30] Bruno).
4. **Single-worker com ordering por pedido:** com um único worker processando em ordem de `created_at`, o cliente recebe os eventos de um mesmo pedido na ordem correta. **Não há garantia de ordering global**, e a garantia por `order_id` vale apenas enquanto houver um único worker — registrado como limitação conhecida ([09:12]–[09:13] Diego e Larissa). Os clientes não pediram ordering global ([09:14] Marcos).

## Alternativas Consideradas

### 1. Trigger no banco para reagir a inserções — descartada

Trigger MySQL executa apenas SQL e não notifica processos externos; para avisar o worker seria preciso improvisar (escrever em arquivo, bater em endpoint), o que "fica esquisito". Polling de 2s atende o requisito com muito menos complexidade ([09:09] Diego, respondendo a Bruno).

### 2. Worker embutido no processo da API — descartada

Eliminaria a entry-point extra, mas acopla o ciclo de vida da entrega ao ciclo de vida da API: restart ou crash da API interrompe a entrega de webhooks ([09:11] Diego).

### 3. Múltiplos workers em paralelo — adiada

Escalar horizontalmente exigiria particionamento por `order_id` ou lock pessimista para preservar ordering. Classificado como "problema do futuro" ([09:13] Diego).

## Consequências

### Positivas

- Entrega desacoplada da API: deploys e falhas da API não afetam o worker, e vice-versa.
- Latência previsível (≤ 2s até o worker enxergar o evento), dentro do SLO de 10s.
- Sem dependência de mecanismos exóticos do banco; funciona em qualquer MySQL.

### Negativas

- **Ponto único de consumo:** se o worker cair, eventos acumulam na outbox até ele voltar (mitigado por observabilidade — ver FDD).
- **Carga constante de polling** no banco, mesmo sem eventos (batch pequeno e índice por status minimizam o custo — [09:08] Diego).
- **Teto de vazão de um único processo:** escalar exige resolver o particionamento adiado ([09:13]).

## Referências

- `src/server.ts` — entry-point existente que serve de modelo para `src/worker.ts` ([09:11] Larissa)
- `src/config/database.ts` — `createPrismaClient()`, fábrica que o worker reutiliza para abrir seu próprio client
- [ADR-001](ADR-001-outbox-transacional-no-mysql.md) — a outbox consumida pelo worker
- [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) — o que o worker faz quando a entrega falha
