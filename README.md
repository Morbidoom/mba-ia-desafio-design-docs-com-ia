# Da Reunião ao Documento: Design Docs Gerados por IA — Entrega

> Este README documenta o **processo de produção** do pacote de design docs. O enunciado original do desafio está no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

O ponto de partida é um Order Management System (Node.js + TypeScript + Prisma/MySQL) em produção e a transcrição literal de uma reunião técnica de ~55 minutos (`TRANSCRICAO.md`), em que tech lead, PM, dois engenheiros e uma engenheira de segurança decidiram como construir um **Sistema de Webhooks de Notificação de Pedidos**. Nada além da transcrição foi registrado.

A tarefa foi transformar essas duas fontes — transcrição e código — em um pacote completo de documentação de engenharia (PRD, RFC, FDD, ADRs, Tracker), acionável o suficiente para o time começar a implementar, usando IA como ferramenta principal de produção e com uma regra de integridade rígida: **nenhuma informação sem origem rastreável na transcrição ou no código**. O papel humano foi o de maestro: dirigir a exploração, formular os prompts, revisar criticamente cada saída e cortar o que não tinha origem identificável.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code (CLI) com o modelo Claude Fable 5** | Ferramenta única de produção: exploração do código, análise da transcrição, planejamento (plan mode com aprovação humana), geração dos documentos e verificação automatizada anti-alucinação |
| **Subagente Explore do Claude Code** | Mapeamento inicial da codebase em paralelo à leitura da transcrição — varredura de `src/` e `prisma/` com relatório de caminhos, padrões e pontos de integração |
| **Skills do professor ([claude-mkt-place](https://github.com/devfullcycle/claude-mkt-place))** | Usadas como **inspiração conceitual** (sem instalar): a abordagem dos plugins `adrs-management` (identificar decisões → gerar ADRs → linkar) e `project-analizer` (análise arquitetural antes de escrever) moldou a ordem do workflow e os prompts customizados abaixo |

Os materiais de aula no Notion (classificação de documentos, PRD de feature, design e arquitetura) serviram de referência para o papel e a "altura" de cada documento.

## Workflow adotado

1. **Contextualização em duas frentes paralelas** — um subagente Explore mapeou a codebase (estrutura modular, transação do `changeStatus`, classes de erro, middlewares, logger) enquanto a transcrição era lida na íntegra e classificada em: decisões fechadas, requisitos explícitos, alternativas descartadas, itens adiados e detalhes secundários.
2. **Verificação do mapa contra o código real** — antes de citar qualquer arquivo nos documentos, os arquivos-chave foram relidos diretamente (o relatório do subagente não foi tomado como verdade).
3. **Plano aprovado pelo humano** — o plano completo (quais ADRs, quais alternativas no RFC, quais endpoints no FDD, com timestamps já levantados) passou por aprovação antes de qualquer documento ser escrito.
4. **ADRs primeiro** (7 decisões → esqueleto do pacote) → **RFC** (proposta + alternativas + questões em aberto, referenciando os ADRs) → **FDD** (implementação em detalhe, com a seção de integração com o código existente) → **PRD** (consolidação de mais alto nível) → **Tracker** (varredura dos documentos prontos) → **README**.
5. **Verificação automatizada anti-alucinação** — dois scripts: um extrai todo caminho `src/...`/`prisma/...` citado nos docs e confere a existência no repositório; outro extrai todo par `[hh:mm] Falante` citado e confere que a fala existe em `TRANSCRICAO.md` com aquele falante. Rodados antes da revisão final (resultados na seção Iterações).
6. **Revisão final** contra a checklist de critérios de aceite do enunciado, item a item.

Uma convenção criada durante a produção: detalhes de implementação necessários ao FDD mas não ditos literalmente na reunião (nomes exatos de tabelas, status codes de resposta, métricas) são marcados **[derivado]** no texto e apontam no Tracker para a decisão ou arquivo que os fundamenta — em vez de serem apresentados como se tivessem sido decididos.

## Prompts customizados

Prompt de mapeamento da codebase (enviado ao subagente Explore, antes de qualquer documento):

```text
Explore o repositório (Node.js + TypeScript, OMS com Prisma/MySQL).
Preciso de um mapa do código para escrever design docs de uma futura
feature de webhooks. NÃO altere nada — só leia. Reporte:
1. Estrutura de src/ (módulos) e prisma/ (modelos e enums, especialmente
   Order e status).
2. Caminho exato e resumo do método changeStatus no serviço de pedidos
   (máquina de estados, transação, auditoria).
3. Onde estão e como funcionam: classes de erro e padrão de códigos;
   middleware requireRole; error middleware centralizado; logger Pino.
4. Como um módulo típico é organizado (controller/service/repository/
   routes/schemas), usando um módulo existente como exemplo.
5. Onde as rotas são registradas e como novos módulos são plugados.
6. Qualquer coisa relacionada a eventos/filas/webhooks (esperado: nada —
   confirme o vácuo).
7. Liste os 10+ caminhos de arquivo reais que os design docs deveriam
   referenciar.
```

Prompt de filtragem dirigida da transcrição (o oposto de "gere um PRD a partir dessa transcrição"):

```text
Leia TRANSCRICAO.md na íntegra e classifique cada item discutido em
exatamente uma categoria, sempre com timestamp e falante:
1. DECISÃO FECHADA — confirmada na reunião (busque "decidido",
   "anotado", o resumo da Larissa em [09:48] e as confirmações finais).
2. REQUISITO EXPLÍCITO — funcional ou não funcional, com o número/valor
   exato dito (tentativas, intervalos, limites, timeouts).
3. DESCARTADO — alternativas postas na mesa e rejeitadas, com o
   trade-off que motivou o descarte.
4. ADIADO / FORA DE ESCOPO — o que foi explicitamente empurrado para
   fase futura ou "observar e decidir depois".
5. GANCHO COM O CÓDIGO — toda menção a arquivo, classe, método ou
   padrão existente da codebase.
Regra: se um item não se encaixa em nenhuma categoria com timestamp
identificável, ele NÃO existe para fins de documentação. Itens da
categoria 3 e 4 NÃO podem aparecer como requisito em nenhum documento.
```

Prompt de verificação anti-alucinação (executado como script após a produção):

```bash
# Todo caminho de código citado nos docs existe no repo?
grep -rhoE '(src|prisma|tests)/[A-Za-z0-9_./-]+' docs/ | sort -u |
  while read -r p; do [ -e "$p" ] || echo "FALTANDO: $p"; done

# Todo par [hh:mm] Falante citado no tracker existe na transcrição?
grep -hoE '\[[0-9]{2}:[0-9]{2}\] (Larissa|Marcos|Bruno|Diego|Sofia)' \
  docs/TRACKER.md | sort -u |
  while read -r ref; do grep -qF "$ref:" TRANSCRICAO.md || echo "NAO CONFERE: $ref"; done
```

## Iterações e ajustes

1. **Relatório do subagente não foi tomado como verdade.** O mapa inicial da codebase veio de um subagente de exploração; antes de escrever qualquer documento, os arquivos centrais (`order.service.ts`, `order.status.ts`, `http-errors.ts`, `auth.middleware.ts`, `schema.prisma`, `app.ts` etc.) foram relidos diretamente para confirmar nomes de classes, assinaturas e números de linha (ex.: `changeStatus` nas linhas 126–179, o tipo `TxClient` na linha 24 — ambos citados no FDD). O relatório estava correto, mas a verificação era condição para citar linha e símbolo com segurança.
2. **O problema dos detalhes "necessários mas não ditos".** A primeira estruturação do FDD esbarrou num dilema: um documento acionável precisa de nomes de tabela, status codes e semânticas de erro que a reunião não fixou. Em vez de deixar a IA "completar" silenciosamente (alucinação clássica), foi criada a convenção **[derivado]**: cada detalhe desse tipo é marcado no texto e ancorado no Tracker à decisão ou arquivo que o fundamenta. Exemplo: o código `WEBHOOK_SECRET_REQUIRED` foi citado literalmente na reunião ([09:28] Bruno), mas seu cenário de uso não — a matriz de erros do FDD registra a semântica como derivada.
3. **Citação de fonte corrigida pela verificação automatizada.** O check de timestamps apontou no FDD a referência "[09:25] Stripe/GitHub" — Stripe/GitHub não são participantes; a fala é do Diego. Corrigido para atribuir ao falante certo. O mesmo check validou os 79 pares timestamp+falante do Tracker contra a transcrição, sem divergências.
4. **Nota de cobertura do Tracker recontada.** A primeira versão da seção de cobertura dizia "71 linhas, 57 TRANSCRICAO" — contagem manual errada. A contagem por script deu 92 linhas (79 TRANSCRICAO / 13 CODIGO) e a seção foi corrigida. Lição registrada: números que podem ser computados não devem ser estimados.
5. **Templates do Notion inacessíveis por fetch.** As páginas de aula são renderizadas via JavaScript e a extração automática falhou; em vez de "lembrar" o conteúdo (risco de inventar template), os documentos seguiram as seções obrigatórias do enunciado + formato MADR padrão para os ADRs.

No total, o pacote fechou em **4 ciclos principais**: exploração+plano → geração (ADRs→RFC→FDD→PRD→Tracker) → verificação automatizada com correções → revisão final contra a checklist.

## Como navegar a entrega

```
docs/
├── PRD.md          ← comece aqui: por que e o quê (produto)
├── RFC.md          ← a proposta técnica e o que foi descartado/adiado
├── adrs/           ← as 7 decisões, uma a uma (ADR-001 … ADR-007)
├── FDD.md          ← como construir, em detalhe (o mais técnico)
└── TRACKER.md      ← a origem de cada item (92 linhas rastreadas)
```

**Ordem de leitura sugerida:** `PRD.md` → `RFC.md` → `adrs/ADR-001…007` → `FDD.md` → `TRACKER.md`. O Tracker também funciona como índice reverso: qualquer afirmação dos documentos pode ser conferida contra `TRANSCRICAO.md` (formato `[hh:mm] Nome`) ou contra o arquivo de código citado.

**Nota sobre caminhos citados:** os documentos referenciam três caminhos que **ainda não existem** por serem a proposta da própria feature (`src/worker.ts`, `src/modules/webhooks/`) — estão sempre marcados como novos. Todos os demais caminhos citados existem no repositório (verificado por script).

O código da aplicação (`src/`, `prisma/`, `tests/`) e a `TRANSCRICAO.md` não foram alterados, conforme a regra do desafio.
