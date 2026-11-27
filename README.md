# Da Reunião ao Documento: Design Docs do Sistema de Webhooks

> Entrega do desafio do módulo **Design Docs com IA** do MBA. O enunciado original está no repositório base: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript + Prisma/MySQL) decidiu, numa reunião de ~55 minutos, construir um **Sistema de Webhooks de Notificação de Pedidos**. Estavam lá a tech lead, o PM, dois engenheiros e a engenheira de segurança. Ninguém registrou nada além da transcrição literal da call (`TRANSCRICAO.md`). O desafio era transformar essa conversa e o código existente num pacote de design docs acionável (PRD, RFC, FDD, ADRs e um tracker de rastreabilidade), usando IA como ferramenta principal de produção.

O ponto central não era "gerar documentos", e sim **filtrar**. A reunião tem decisões fechadas, alternativas descartadas, itens adiados, uma informação corrigida no meio da conversa (o `customer_id` *não* vem do JWT) e detalhes que ficaram ambíguos. Cada afirmação dos documentos precisava apontar para um timestamp da transcrição ou para um arquivo real do código, sem nenhum requisito inventado.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude (Opus 5.5) no claude.ai, em modo agente** | Ferramenta principal. Clonou o repositório base, leu a transcrição e o código (`order.service.ts`, `schema.prisma`, middlewares, erros, logger, testes), redigiu ADRs, RFC, FDD, PRD e README e fez a revisão crítica de cada documento contra a transcrição |
| **Script Python de validação** (escrito pela IA, rodado no ambiente do agente) | Checagem automática anti-alucinação: confere se cada `[hh:mm] Falante` citado existe na transcrição, se cada caminho de código existe no repo (ou está marcado como `(novo)`), se todos os IDs dos documentos estão no tracker e se a proporção TRANSCRICAO/CODIGO bate com os critérios |

## Workflow adotado

Segui a ordem sugerida no enunciado, com uma etapa de mapeamento antes e uma de validação automática depois:

1. **Exploração.** Leitura integral da `TRANSCRICAO.md` e dos arquivos centrais do código. A saída foi um mapa com as 6 decisões principais e seus timestamps, a lista do que foi descartado, adiado ou deixado em aberto, e as "armadilhas" (fala corrigida, ambiguidades, conflito de convenção de nome de ADR).
2. **ADRs primeiro (7).** Uma decisão por arquivo no formato MADR, com as alternativas *realmente* faladas na reunião e o trade-off de cada uma.
3. **RFC**, consolidando os ADRs em nível de arquitetura. As alternativas descartadas e as questões em aberto entraram em tabelas com ID e timestamp, e o RFC ficou propositalmente curto.
4. **FDD**, com modelo de dados Prisma, fluxos, contratos HTTP com exemplos, matriz `WEBHOOK_*`, resiliência, observabilidade e a seção de integração apontando arquivo por arquivo.
5. **PRD por último**, como consolidação de produto. Cuidei para não repetir o detalhe técnico do FDD.
6. **Tracker** gerado a partir de uma lista estruturada (ID, documento, tipo, resumo, fonte, localização), com validação automática.
7. **Revisão crítica e iteração.** O validador e uma leitura linha a linha contra a transcrição apontaram problemas, corrigidos em ciclos (ver abaixo).

Para não duplicar conteúdo, fixei uma regra para cada documento: o **RFC** fala em decisão ("o que e por quê"), o **FDD** em implementação ("como"), o **PRD** em produto, sem nomes de tabelas nem headers, e os **ADRs** trazem uma decisão cada. Quando um detalhe técnico aparecia no PRD ou no RFC, eu o trocava por um link para o FDD.

## Prompts customizados

**Prompt 1: filtragem dirigida da transcrição** (usado antes de escrever qualquer documento)

```text
Leia TRANSCRICAO.md inteira. NÃO gere documentos ainda. Produza 5 listas, cada item com
timestamp [hh:mm] e falante exatos:
1. DECISÕES FECHADAS: só o que alguém declarou como decidido ("Decidido", "Anotado",
   "Tá decidido") ou que o resumo final de [09:48] confirma.
2. ALTERNATIVAS DESCARTADAS: a alternativa + o argumento que a derrubou, com o timestamp
   do argumento.
3. ADIADO / FORA DE ESCOPO: separe "adiado para próxima fase" de "descartado".
4. AMBIGUIDADES: pontos onde a fala admite duas leituras ou onde alguém corrigiu outra
   pessoa. Para cada um, cite as duas falas.
5. GANCHOS NO CÓDIGO: cada arquivo citado na reunião, e confira no repositório se ele
   existe e se o que foi dito sobre ele bate (ex.: o que changeStatus faz hoje).
Regra: se um item não tem timestamp, ele não entra.
```

**Prompt 2: geração de um ADR com restrição de origem**

```text
Escreva docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md no formato MADR com as seções
Status, Contexto, Decisão, Alternativas Consideradas, Consequências (positivas e negativas).
Restrições:
- Toda frase factual termina com a origem: ([hh:mm] Falante) ou o caminho do arquivo.
- Alternativas: só as que aparecem na transcrição (retry indefinido, 3 tentativas,
  "failed" na própria outbox). Para cada uma, o trade-off que motivou o descarte.
- Não invente números. Se algum número da reunião for inconsistente (ex.: "5 tentativas"
  com 5 intervalos somando ~14h36 = "quase 15 horas"), explicite a leitura adotada e
  marque como ponto de confirmação no RFC em vez de escolher em silêncio.
- Consequências negativas devem incluir pelo menos um efeito colateral que a reunião NÃO
  discutiu mas que decorre da decisão (ex.: impacto no ordering), sinalizado como tal.
```

**Prompt 3: revisão crítica contra o código** (rodado sobre o FDD já escrito)

```text
Revise docs/FDD.md como um engenheiro sênior do time de Pedidos que conhece o código.
Para cada afirmação sobre o código existente, abra o arquivo e confirme. Procure:
(a) comportamentos atribuídos a arquivos que não fazem aquilo;
(b) códigos de erro que o pipeline atual não consegue emitir como descrito
    (verifique validate.middleware.ts e http-errors.ts);
(c) regras que conflitam entre si (ex.: um "erro" que, aplicado dentro da transação,
    daria rollback na mudança de status);
(d) números e defaults sem origem na transcrição.
Liste os problemas com a correção proposta. Não reescreva o documento inteiro.
```

## Iterações e ajustes

Foram **4 ciclos principais** de geração → revisão → correção. Os ajustes mais relevantes:

1. **Atribuições inventadas no PRD.** A primeira versão dos cenários dizia que *"o endpoint da Nova Cargo fica 2h fora do ar"* e que *"a MaxDistribuição vaza a secret"*. A transcrição só diz que *"já tinha cliente nosso"* com esses problemas ([09:16] e [09:22] Diego), sem nomear ninguém. Troquei por "um cliente".
2. **Números sem origem no FDD.** O primeiro FDD trazia um truncamento de `responseBody` em 2KB e um lote padrão de 20 eventos como se fossem fatos. Ao montar o tracker, nenhuma das duas linhas tinha "Localização". O truncamento foi removido e virou ponto de revisão no risco FDD-RSK-07. O lote ficou explicitamente como "padrão sugerido, a validar", apoiado no "batch pequeno" de [09:08] Diego.
3. **Limite de 64KB no lugar errado.** A primeira versão validava o tamanho do payload na inserção da outbox. Como a inserção roda dentro da transação de `changeStatus`, um erro ali daria **rollback na mudança de status do pedido**, travando o negócio por causa de uma notificação. A validação foi para o worker: o evento vai direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem retry.
4. **Código de erro que o pipeline não emite.** O FDD previa uma resposta `WEBHOOK_INVALID_URL` direta. Lendo `src/middlewares/validate.middleware.ts`, vi que qualquer `ZodError` vira `VALIDATION_ERROR`, e a Sofia pediu que o https fosse validado *no schema Zod* ([09:23]). A matriz agora documenta que o código vai em `details[].message`. Na mesma revisão apareceu que `NotFoundError` fixa o código `NOT_FOUND`, então os 404 do módulo estendem `AppError` diretamente.
5. **Ambiguidade "5 tentativas".** Somando os intervalos citados (1m+5m+30m+2h+12h ≈ 14h36), a frase "quase 15 horas entre primeira falha e última tentativa" só fecha com *1 envio + 5 retentativas*. Em vez de escolher em silêncio, adotei essa leitura e registrei a confirmação como **RFC-QA-06**.
6. **Armadilha do `customer_id`.** O Marcos diz "customer_id implícito do JWT" ([09:31]), mas logo depois a Larissa corrige: vai no body ou no path ([09:32]). Os documentos seguem a correção, e a escolha entre body e path, que ninguém fez, virou a questão em aberto RFC-QA-05.
7. **Validador automático.** Ele apontou `src/worker.ts` citado no RFC sem a marcação `(novo)`, o que contrariaria o critério "nenhum arquivo mencionado é inexistente". Todas as menções a arquivos ainda não criados agora levam `(novo)`.
8. **Convenção de nome de ADR.** `docs/adrs/README.md` sugeria `0001-titulo.md`, e o enunciado exige `ADR-NNN-titulo.md`. Segui o enunciado e atualizei o README da pasta como índice.

## Como navegar a entrega

Ordem sugerida de leitura:

| # | Documento | Para quê |
|---|---|---|
| 1 | [`docs/PRD.md`](docs/PRD.md) | Por que e o quê: problema, público, métricas, escopo e fora de escopo |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Proposta técnica, alternativas descartadas e **questões em aberto** |
| 3 | [`docs/adrs/`](docs/adrs/README.md) | As 7 decisões, uma por arquivo (ADR-001 a ADR-007) |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como construir: modelo de dados, fluxos, contratos, erros e integração com o código |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada item (timestamp da reunião ou arquivo do código) |

```
.
├── README.md                 ← este arquivo (processo)
├── TRANSCRICAO.md            ← fonte (inalterada)
└── docs/
    ├── PRD.md
    ├── RFC.md
    ├── FDD.md
    ├── TRACKER.md
    └── adrs/
        ├── README.md         ← índice
        ├── ADR-001-outbox-no-mysql.md
        ├── ADR-002-worker-separado-em-polling.md
        ├── ADR-003-retry-backoff-exponencial-e-dlq.md
        ├── ADR-004-hmac-sha256-com-secret-por-endpoint.md
        ├── ADR-005-at-least-once-com-x-event-id.md
        ├── ADR-006-reuso-dos-padroes-existentes.md
        └── ADR-007-payload-snapshot-na-insercao.md
```

Nenhum arquivo de `src/`, `prisma/`, `tests/` ou de configuração foi alterado. A entrega é só documental.
