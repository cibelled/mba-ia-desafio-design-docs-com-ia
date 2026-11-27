# Da Reunião ao Documento: Design Docs do Sistema de Webhooks

> Entrega do desafio do módulo **Design Docs com IA** do MBA. O enunciado original está no repositório base: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript + Prisma/MySQL) decidiu, numa reunião de ~55 minutos, construir um **Sistema de Webhooks de Notificação de Pedidos**. Estavam lá a tech lead, o PM, dois engenheiros e a engenheira de segurança. Ninguém registrou nada além da transcrição literal da call (`TRANSCRICAO.md`). O desafio era transformar essa conversa e o código existente num pacote de design docs acionável (PRD, RFC, FDD, ADRs e um tracker de rastreabilidade), usando IA como ferramenta principal de produção.

O ponto central não era "gerar documentos", e sim **filtrar**. A reunião tem decisões fechadas, alternativas descartadas, itens adiados, uma informação corrigida no meio da conversa (o `customer_id` *não* vem do JWT) e detalhes que ficaram ambíguos. Cada afirmação dos documentos precisava apontar para um timestamp da transcrição ou para um arquivo real do código, sem nenhum requisito inventado.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude (Opus 5.5 e, no final, Sonnet 5.5) no Claude Code, em modo agente** | Ferramenta única de produção. Clonou o repositório base, leu a transcrição e o código (`order.service.ts`, `order.status.ts`, `schema.prisma`, middlewares, erros, logger, rotas, testes), escreveu ADRs, RFC, FDD, PRD, tracker e README, e fez a revisão de cada documento contra a transcrição. Também aplicou os arquivos no meu clone local e fez o commit |
| **Script Python de validação** (escrito pelo Claude durante a sessão) | Checagem anti-alucinação: confere se cada `[hh:mm] Falante` citado existe na transcrição, se cada caminho de código existe no repositório (ou está marcado como `(novo)`), se todos os IDs dos documentos estão no tracker e se a proporção TRANSCRICAO/CODIGO atende aos critérios. O script gera o `TRACKER.md` a partir de uma lista estruturada |

## Workflow adotado

Eu entreguei ao Claude o enunciado completo e pedi que ele produzisse o pacote inteiro. Ele seguiu a ordem sugerida, com uma etapa de mapeamento antes e uma de validação automática depois:

1. **Exploração.** Leitura integral da `TRANSCRICAO.md` e dos arquivos centrais do código. A saída foi um mapa com as 6 decisões principais e seus timestamps, o que foi descartado, adiado ou deixado em aberto, e as armadilhas (fala corrigida, ambiguidades, conflito de convenção de nome de ADR).
2. **ADRs primeiro (7).** Uma decisão por arquivo no formato MADR, com as alternativas realmente faladas na reunião e o trade-off de cada uma.
3. **RFC**, consolidando os ADRs em nível de arquitetura. Alternativas descartadas e questões em aberto entraram em tabelas com ID e timestamp.
4. **FDD**, com modelo de dados Prisma, fluxos, contratos HTTP com exemplos, matriz `WEBHOOK_*`, resiliência, observabilidade e a seção de integração arquivo por arquivo.
5. **PRD por último**, como consolidação de produto, sem repetir o detalhe técnico do FDD.
6. **Tracker** gerado por script a partir de uma lista (ID, documento, tipo, resumo, fonte, localização), com validação automática.
7. **Revisão e correções** a partir do validador e de uma releitura contra a transcrição (ver abaixo).

Para evitar conteúdo duplicado, a regra foi: o **RFC** fala em decisão, o **FDD** em implementação, o **PRD** em produto (sem nomes de tabela nem headers) e cada **ADR** traz uma decisão. Detalhe técnico que aparecia no PRD ou no RFC virava um link para o FDD.

## Prompts customizados

A conversa foi conduzida com o enunciado do desafio colado na íntegra, seguido de instruções curtas, como pedir que o Claude gerasse tudo no próprio ambiente (e não só prompts para eu rodar). Abaixo estão os dois prompts de direcionamento que descrevem, em forma reutilizável, as instruções que guiaram a produção. Foram formalizados depois da execução, a partir do que o Claude fez, e não digitados um a um durante a sessão.

**Prompt 1: filtragem dirigida da transcrição** (antes de escrever qualquer documento)

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

## Iterações e ajustes

O pacote foi gerado em uma primeira passada e depois passou por **3 rodadas de correção**. Os ajustes concretos:

1. **Atribuições inventadas no PRD.** Os cenários de uso citavam clientes pelo nome ("a Nova Cargo fica 2h fora do ar", "a MaxDistribuição vaza a secret"). A transcrição só diz "já tinha cliente nosso" com esses problemas ([09:16] e [09:22] Diego), sem nomear ninguém. Troquei por "um cliente".
2. **Números sem origem no FDD.** A primeira versão trazia um truncamento de `responseBody` em 2KB e um lote padrão de 20 eventos como se fossem fatos. Nenhum dos dois tinha "Localização" na transcrição. O truncamento foi removido e virou ponto de atenção no risco FDD-RSK-07. O lote ficou como "padrão sugerido, a validar", ancorado no "batch pequeno" de [09:08] Diego.
3. **Caminho de arquivo inexistente.** O validador acusou `src/worker.ts` citado no RFC e no FDD sem a marcação `(novo)`, o que violaria o critério "nenhum arquivo mencionado é inexistente". Todas as menções a arquivos que ainda serão criados passaram a levar `(novo)`.
4. **Formato do ADR.** Os ADRs começaram com o Status como item de uma lista de metadados. Para atender ao critério de ter a *seção* Status, converti para um `## Status` em todos.
5. **Questão em aberto descoberta no meio do caminho.** Ao escrever o FDD, percebi que só existe o endpoint de replay da DLQ, sem listagem para o ADMIN descobrir o `:id`. Acrescentei **RFC-QA-10** em vez de inventar um endpoint.
6. **Convenção de nome de ADR.** `docs/adrs/README.md` sugeria `0001-titulo.md`, e o enunciado exige `ADR-NNN-titulo.md`. Segui o enunciado e transformei o README da pasta em índice.

Dois pontos da transcrição foram tratados desde a primeira versão, sem retrabalho, mas valem registro porque são onde a IA mais poderia errar: o `customer_id` (o Marcos diz que vem do JWT em [09:31] e a Larissa corrige em [09:32]; a escolha entre body e path ficou como **RFC-QA-05**) e a conta das "5 tentativas" (os 5 intervalos somam ~14h36, o que só fecha com "quase 15 horas" se for 1 envio + 5 retentativas; ficou como **RFC-QA-06**).

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
