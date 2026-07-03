# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

- **Status:** Aceito
- **Decisores:** Sofia (Eng. de Segurança), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Origem:** `TRANSCRICAO.md` [09:19]–[09:23]

## Contexto

Os webhooks expõem eventos com dados de pedidos para endpoints fora da infraestrutura da empresa. O cliente precisa conseguir validar duas coisas: que a requisição veio realmente da plataforma e que o payload não foi adulterado no caminho ([09:19] Sofia). Já houve caso real de cliente que vazou uma secret em log da aplicação dele ([09:22] Diego), então o raio de dano de um vazamento e o processo de troca de secret precisam ser considerados desde o início.

## Decisão

([09:22] Sofia: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h.")

1. **Assinatura HMAC-SHA256 sobre o corpo do request**, enviada no header `X-Signature`. O cliente recalcula e compara do lado dele ([09:20] Sofia). SHA-256 é o padrão de mercado, com biblioteca disponível em qualquer stack séria ([09:20] Sofia).
2. **Secret única por endpoint de webhook**, não uma secret global da plataforma — "senão se vaza uma, vaza tudo" ([09:21] Sofia). A tabela de configuração armazena url + secret + customer_id + estado ativo ([09:21] Bruno/Sofia).
3. **Secret gerada pela plataforma** e devolvida ao cliente no momento do cadastro ([09:31] Marcos).
4. **Rotação de secret via API**, com **grace period de 24 horas** em que a secret antiga continua válida em paralelo, dando tempo de o cliente migrar seus sistemas; depois disso a antiga expira ([09:21] Sofia).
5. **TLS obrigatório:** a URL do webhook tem que ser `https`; cadastro com `http` é recusado com erro de validação no schema Zod ([09:23] Sofia).

## Alternativas Consideradas

### 1. Secret global da plataforma — descartada

Uma única secret compartilhada por todos os clientes simplificaria a gestão, mas um único vazamento comprometeria a autenticidade dos webhooks de **todos** os clientes ([09:21] Sofia). O caso real de vazamento em log de cliente ([09:22] Diego) torna o cenário concreto, não hipotético.

### 2. Sem assinatura (confiar apenas no TLS) — não aceitável

TLS protege o canal, mas não permite ao cliente verificar a **origem** (qualquer um que conheça a URL poderia forjar um POST) nem detectar adulteração antes do endpoint dele. A verificação de autenticidade foi colocada por Sofia como requisito de partida ([09:19]).

## Consequências

### Positivas

- Cliente valida origem e integridade com criptografia padrão de mercado, sem infraestrutura extra (PKI, mTLS).
- Vazamento de uma secret compromete somente um endpoint, e a rotação com grace period permite troca sem janela de indisponibilidade.
- `X-Timestamp` no envio permite ao cliente detectar replay attack se quiser ([09:44] Diego).

### Negativas

- **Gestão de ciclo de vida de secrets:** geração, armazenamento, rotação e expiração da secret antiga após 24h são responsabilidade da plataforma.
- **Janela dupla durante o grace period:** por até 24h duas secrets assinam validamente — o worker precisa assinar com a atual, e a verificação de rotação precisa de cuidado extra na implementação.
- Secrets não podem aparecer em logs: exige atenção à redaction já configurada no logger (`src/shared/logger/index.ts` censura `*.token`, `*.password` etc.; o campo de secret do webhook precisa entrar nessa lista).

## Referências

- `src/shared/logger/index.ts` — `redactPaths` do Pino, onde o campo da secret deve ser incluído
- `src/middlewares/validate.middleware.ts` — validação Zod onde a regra de `https` obrigatório se encaixa ([09:23] Sofia: "é só uma validação no schema Zod")
- [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) — demais headers enviados na entrega
