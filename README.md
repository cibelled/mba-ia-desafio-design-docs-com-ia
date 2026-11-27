# Da Reunião ao Documento: Design Docs do Sistema de Webhooks

> Entrega do desafio do módulo **Design Docs com IA** do MBA. O enunciado original está no repositório base: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript + Prisma/MySQL) decidiu, numa reunião de ~55 minutos, construir um **Sistema de Webhooks de Notificação de Pedidos**. Estavam lá a tech lead, o PM, dois engenheiros e a engenheira de segurança. Ninguém registrou nada além da transcrição literal da call (`TRANSCRICAO.md`). O desafio era transformar essa conversa e o código existente num pacote de design docs acionável (PRD, RFC, FDD, ADRs e um tracker de rastreabilidade), usando IA como ferramenta principal de produção.

O ponto central não era "gerar documentos", e sim **filtrar**. A reunião tem decisões fechadas, alternativas descartadas, itens adiados, uma informação corrigida no meio da conversa (o `customer_id` *não* vem do JWT) e detalhes que ficaram ambíguos. Cada afirmação dos documentos precisava apontar para um timestamp da transcrição ou para um arquivo real do código, sem nenhum requisito inventado.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude (Opus 5.5 e, no final, Sonnet 5.5) no Claude Code, em modo agente** | Ferramenta única de produção. Clonou o repositório base, leu a transcrição e o código (`order.service.ts`, `order.status.ts`, `schema.prisma`, middlewares, erros, logger, rotas, testes), escreveu ADRs, RFC, FDD, PRD, tracker e README, e fez a revisão de cada documento contra a transcrição. Também aplicou os arquivos no meu clone local e fez o commit |
| **Dois subagentes revisores do Claude Code** | Revisão independente, sem editar arquivos. Um **revisor adversarial** leu transcrição, código e todos os documentos procurando contradições, itens inventados e erros técnicos. Um **classificador** leu só a transcrição, sem ver os documentos, e eu comparei o resultado dele com o que tinha escrito |
| **Script Python de validação** (escrito pelo Claude durante a sessão) | Checagem anti-alucinação: confere se cada `[hh:mm] Falante` citado existe na transcrição, se cada caminho de código existe no repositório (ou está marcado como `(novo)`), se todos os IDs dos documentos estão no tracker e se a proporção TRANSCRICAO/CODIGO atende aos critérios. O script gera o `TRACKER.md` a partir de uma lista estruturada |

## Workflow adotado

Eu entreguei ao Claude o enunciado completo e pedi que ele produzisse o pacote inteiro. Ele seguiu a ordem sugerida, com uma etapa de mapeamento antes e uma de validação automática depois:

1. **Exploração.** Leitura integral da `TRANSCRICAO.md` e dos arquivos centrais do código. A saída foi um mapa com as 6 decisões principais e seus timestamps, o que foi descartado, adiado ou deixado em aberto, e as armadilhas (fala corrigida, ambiguidades, conflito de convenção de nome de ADR).
2. **ADRs primeiro (7).** Uma decisão por arquivo no formato MADR, com as alternativas realmente faladas na reunião e o trade-off de cada uma.
3. **RFC**, consolidando os ADRs em nível de arquitetura. Alternativas descartadas e questões em aberto entraram em tabelas com ID e timestamp.
4. **FDD**, com modelo de dados Prisma, fluxos, contratos HTTP com exemplos, matriz `WEBHOOK_*`, resiliência, observabilidade e a seção de integração arquivo por arquivo.
5. **PRD por último**, como consolidação de produto, sem repetir o detalhe técnico do FDD.
6. **Tracker** gerado por script a partir de uma lista (ID, documento, tipo, resumo, fonte, localização), com validação automática.
7. **Revisão e correções** a partir do validador e de uma releitura contra a transcrição.
8. **Revisão independente** com dois subagentes (um adversarial, um classificador), seguida de uma nova rodada de correções (ver abaixo).

Para evitar conteúdo duplicado, a regra foi: o **RFC** fala em decisão, o **FDD** em implementação, o **PRD** em produto (sem nomes de tabela, de arquivo nem de ferramenta de teste) e cada **ADR** traz uma decisão. Detalhe técnico que aparecia no PRD ou no RFC virava um link para o FDD.

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
- Alternativas: priorize as que aparecem na transcrição (retry indefinido, 3 tentativas,
  "failed" na própria outbox). Se incluir uma alternativa plausível que NÃO foi discutida,
  rotule como "não discutida na reunião" e não cite timestamp para ela. Para cada uma,
  o trade-off que motivou o descarte.
- Não invente números. Se algum número da reunião for inconsistente (ex.: "5 tentativas"
  com 5 intervalos somando ~14h36 = "quase 15 horas"), explicite a leitura adotada e
  marque como ponto de confirmação no RFC em vez de escolher em silêncio.
- Consequências negativas devem incluir pelo menos um efeito colateral que a reunião NÃO
  discutiu mas que decorre da decisão (ex.: impacto no ordering), sinalizado como tal.
```

## Iterações e ajustes

O pacote passou por **6 rodadas**: a geração inicial, 3 rodadas de autocorreção, 1 rodada de revisão final contra os critérios de aceite (pedida por mim ao Claude) e 1 rodada de **revisão independente** por dois subagentes, que gerou 22 achados e mais de uma dezena de correções. Os ajustes concretos:

**Rodadas 1 a 3 (autocorreção)**

1. **Atribuições inventadas no PRD.** Os cenários de uso citavam clientes pelo nome ("a Nova Cargo fica 2h fora do ar", "a MaxDistribuição vaza a secret"). A transcrição só diz "já tinha cliente nosso" ([09:16] e [09:22] Diego). Troquei por "um cliente".
2. **Números sem origem no FDD.** A primeira versão trazia truncamento de `responseBody` em 2KB e lote padrão de 20 como se fossem fatos. O truncamento foi removido. O lote ficou como sugestão a validar.
3. **Caminho de arquivo inexistente.** O validador acusou `src/worker.ts` sem a marcação `(novo)`, o que violaria o critério "nenhum arquivo mencionado é inexistente".
4. **Formato do ADR.** Os ADRs começaram com o Status como item de uma lista de metadados. Converti para uma seção `## Status` em todos.
5. **Questão em aberto descoberta no FDD.** Só existe o endpoint de replay da DLQ, sem listagem para o ADMIN achar o `:id`. Acrescentei RFC-QA-10 em vez de inventar um endpoint.
6. **Convenção de nome de ADR.** `docs/adrs/README.md` sugeria `0001-titulo.md`, e o enunciado exige `ADR-NNN-titulo.md`. Segui o enunciado.

**Rodada 4 (revisão final contra os critérios de aceite)**

7. **Revisão final contra os critérios de aceite.** Pedi ao Claude que validasse o pacote inteiro contra o enunciado. Ele apontou três problemas, que foram corrigidos. (a) O FDD listava webhooks com `GET /webhooks?customerId=…`, só que a query não é nenhuma das opções faladas em [09:32] ("body ou path"). A rota virou `GET /customers/:customerId/webhooks`. (b) O RFC-QA-06 ignorava a fala "total 5 tentativas" do resumo de [09:48], que contradiz a leitura de 6 envios. O conflito passou a constar no RFC, no ADR-003, no FDD e no PRD. (c) Só um endpoint tinha exemplo completo de request e response. Todos ganharam bloco `http` com método, caminho e headers, e o PATCH ganhou a resposta em JSON. A data inventada no RFC também saiu.

**Rodada 5 (revisão independente)**

Um revisor adversarial (somente leitura) e um classificador que só lia a transcrição. Os problemas mais importantes e o que mudou:

8. **Bug de validação de URL (alta).** O schema Zod do FDD fazia `new URL()` dentro de um `refine`. No Zod 3, o `refine` roda mesmo quando `.url()` falha, e uma URL malformada lançaria `TypeError`, que o `validate.middleware.ts` não captura, virando 500 em vez de 400. Reproduzi o erro rodando o Zod e troquei por `superRefine` com `try/catch`.
9. **Worker sequencial reintroduzia o problema da reunião (alta).** O FDD processava o lote em série, e um cliente lento seguraria o ciclo de todos, que é o argumento de [09:04] Bruno contra o envio síncrono. O FDD agora agrupa por endpoint, com uma cadeia independente por endpoint e um único processo. Ficou como questão em aberto RFC-QA-11, com risco e critério de aceite próprios.
10. **"5 tentativas" tratado como decidido em alguns documentos e como pendente em outros (alta).** O ADR-003 e o PRD diziam "5 retentativas", enquanto o RFC-QA-06 admitia a dúvida. Padronizei: todos citam "5 tentativas" e remetem à leitura em confirmação.
11. **Afirmação de ordem que o próprio design quebrava (média).** O FDD dizia que o processamento "preserva a ordem por order_id", mas um evento em retry deixa o seguinte ultrapassar. Reescrevi o trecho e deixei a limitação explícita em RFC-QA-07.
12. **`DELETE` com cascade descartava eventos em silêncio (média).** Agora os eventos pendentes são movidos para a DLQ antes de apagar o endpoint.
13. **Recuperação e shutdown do worker (média).** O FDD só recuperava `PROCESSING` no boot e prometia um shutdown que o `server.ts` atual não faz. Acrescentei tratamento de erro por evento e documentei o shutdown como comportamento novo.
14. **Mudança de assinatura sem origem (média).** O FDD propunha passar `requestId` para `changeStatus`, alterando `order.controller.ts` e `order.service.ts` além do previsto. Removi a proposta.
15. **Autorização das rotas de rotação e histórico (média).** A reunião liberou só o CRUD a qualquer role. `rotate-secret` e `deliveries` não foram discutidas e hoje permitem acesso aos dados de qualquer customer. Registrei no FDD §6.10 e nas questões RFC-QA-04 e RFC-QA-08 para a revisão da Sofia.
16. **Citações que não sustentavam o item no tracker (média).** Vários códigos de erro `WEBHOOK_*` apontavam para falas que não os mencionam. Corrigi as localizações e marquei como "(derivado)" os itens que são deduções do documento. O cabeçalho do tracker passou a informar essa contagem.
17. **Valores sem origem (média).** Criei a tabela do FDD §8.1, com todas as propostas que não vieram de uma fala (lote de 20, prefixo `sha256=`, códigos 202/409/422, `limit` de 100, tabela `webhook_deliveries`, nome do arquivo do worker, entre outras).
18. **Alternativas não discutidas com timestamp (média).** Dois ADRs citavam falas de reunião para sustentar alternativas que ninguém discutiu. Removi as citações e rotulei as alternativas como "não discutidas".
19. **Itens menores.** Troquei o domínio de um cliente real por `example.com`, unifiquei `previousSecretExpiresAt`, registrei que "os últimos 100" foi só um exemplo do Marcos, tirei detalhes técnicos do PRD, registrei no ADR-007 que Marcos e Sofia já tinham saído da call quando UUID e snapshot foram decididos, e acrescentei o risco da estimativa de sprints (a soma das partes passa de 3).

O classificador independente concordou com os documentos em decisões, descartes e adiamentos, e ajudou a pegar os pontos 16 e 18.

**Ponto que eu rejeitei.** Pensei em tornar o worker paralelo entre pedidos. Não fiz, porque a reunião decidiu um único worker ([09:12] e [09:13]). A solução adotada mantém um único processo e só isola endpoints dentro dele.

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
