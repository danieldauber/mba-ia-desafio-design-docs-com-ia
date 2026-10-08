# Sistema de Webhooks de Notificação de Pedidos — Processo de Produção

---

## Sobre o desafio

O desafio consiste em transformar uma transcrição bruta de reunião técnica em um pacote completo de design docs — PRD, RFC, FDD, ADRs e Tracker — usando IA como ferramenta principal de produção. A única fonte de verdade era a gravação da call (`TRANSCRICAO.md`) e o código existente de um Order Management System em Node.js + TypeScript + Prisma + MySQL.

O aspecto mais exigente não foi gerar conteúdo, mas filtrar o que não deveria entrar: a transcrição mistura decisões fechadas, itens descartados, deferidos e comentários técnicos secundários. Identificar o que NÃO é requisito foi tão importante quanto identificar o que é. Toda linha dos documentos precisou ter rastreabilidade verificável na transcrição ou no código — sem esse critério, a IA tende a extrapolar e inventar restrições que não existem.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code (claude-sonnet-4-6)** | Orquestrador central de todo o processo. Leu a transcrição, filtrou decisões, gerou a tabela de classificação, escreveu o `decisions.md`, coordenou os agentes de ADR e produziu os demais documentos via prompts dirigidos. |
| **Plugin `adrs-management` (devfullcycle/fullcycle-claude-marketplace)** | Pipeline de 3 fases para geração de ADRs: Phase 1 (mapeamento do codebase → `mapping.md`), Phase 2 (identificação de ADRs candidatos por módulo em paralelo → `potential-adrs/`), Phase 3 (geração formal em pt-BR com análise de tier e marcadores de input necessário). |
| **Agentes paralelos (adr-analyzer / adr-generator)** | Subagentes especializados lançados em paralelo para analisar módulos independentes (WEBHOOKS, ORDERS, INFRA, DATA, SHARED) e gerar cada ADR formal de forma isolada, sem interferência de contexto entre eles. |

---

## Workflow adotado

O processo seguiu a ordem sugerida pelo enunciado, com uma etapa de pré-processamento adicionada antes dos documentos principais.

### Etapa 0 — Filtragem da transcrição (pré-requisito de tudo)

Antes de gerar qualquer documento, foi feita uma análise dirigida da transcrição para classificar cada item em três categorias: **aplicado**, **adiado** ou **descartado**. O resultado foi salvo em `decisions.md` e serviu como referência única para todos os documentos subsequentes.

Esse passo foi fundamental para evitar que a IA incluísse itens descartados (como webhook síncrono ou Redis Streams) como requisitos nos documentos.

### Etapa 1 — ADRs (esqueleto das decisões)

**Phase 1**: o agente `adr-analyzer` mapeou o codebase completo (39 arquivos fonte) e gerou `docs/adrs/mapping.md` com 10 módulos identificados, pilha tecnológica e padrões arquiteturais.

**Phase 2**: 5 agentes foram lançados em paralelo — um por módulo (WEBHOOKS, ORDERS, INFRA, DATA, SHARED) — para identificar ADRs candidatos com sistema de scoring (0–150 pontos). WEBHOOKS gerou 5 must-document + 2 consider; os demais módulos contribuíram com candidatos de infra e dados.

**Phase 3**: 5 agentes `adr-generator` rodaram em paralelo, um por arquivo candidato do módulo WEBHOOKS, gerando ADRs formais em pt-BR. Após geração, os arquivos foram renomeados com numeração sequencial (ADR-001 a ADR-005) e movidos para `docs/adrs/`.

### Etapa 2 — RFC

Gerado com prompt dirigido referenciando os 5 ADRs, `decisions.md` e `TRANSCRICAO.md`. O prompt explicitou a fronteira RFC/FDD (RFC documenta decisões e alternativas; FDD documenta contratos e implementação) para evitar que o RFC reproduzisse tabelas de headers e campos de payload que pertencem ao FDD.

### Etapa 3 — FDD

Gerado com prompt estruturado de 10 seções obrigatórias (Contexto, Objetivos, Escopo, Fluxos, Contratos públicos, Erros, Observabilidade, Dependências/Integração, Critérios de aceite, Riscos). Passou por ciclo de revisão crítica com identificação de 8 problemas concretos e regravação parcial. Ver Iterações 6–8 abaixo.

### Etapa 4 — PRD

Gerado com prompt de entrevista estruturada de 12 etapas (Contexto, Problema, Objetivos, Escopo, RF, RNF, Arquitetura, Decisões, Dependências, Riscos, Critérios de aceite, Testes). Como todo o contexto já estava disponível na conversa, o PRD foi gerado diretamente sem entrevista interativa, usando `decisions.md`, `TRANSCRICAO.md`, `RFC.md` e `FDD.md` como fontes.

### Etapa 5 — TRACKER

Gerado com mapeamento manual de cada decisão, requisito e restrição à sua fonte primária (timestamp na transcrição ou arquivo de código existente). 71 itens rastreados, 85% com fonte na transcrição.

### Etapa 6 — README (este arquivo)

Produzido por último, quando o processo estava completo e documentável com precisão.

---

## Prompts customizados

### Prompt 1 — Filtragem da transcrição com classificação tripartite

```
leia o arquivo transcricao.md. esse arquivo é uma transcrição de uma reunião
para definir itens técnicos para a aplicação desse repo. A transcrição inclui
decisões fechadas, requisitos funcionais explícitos, restrições, ganchos com o
código existente, pontos descartados ou adiados para fases futuras e detalhes
técnicos secundários. Nem tudo que foi mencionado vira requisito. Algumas coisas
foram explicitamente descartadas, outras foram adiadas. Identificar o que NÃO
entra é tão importante quanto identificar o que entra. Use a IA com prompts
dirigidos para fazer essa filtragem, não pedidos genéricos.

Identifique quais são as decisões que serão aplicadas, quais foram descartadas
e quais foram adiadas, coloque isso em forma de uma tabela. Vamos precisar dessas
informações para criar design docs, como ADR, PRD, RFC e FDD.
```

Esse prompt foi o mais importante de todo o processo. Ao exigir explicitamente que itens descartados e adiados fossem identificados como categoria separada, evitou que a IA tratasse menções descartadas da reunião (Redis Streams, webhook síncrono, exactly-once, DLQ na própria outbox) como requisitos válidos. A instrução "não pedidos genéricos" forçou a IA a trabalhar com inferência específica, não resumo livre.

---

### Prompt 2 — Geração de ADRs com cobertura mínima obrigatória

```
usando o plugin de ADRs, eu preciso agora fazer isso:

Produza entre 5 e 8 ADRs em arquivos separados dentro de docs/adrs/,
nomeados no formato ADR-NNN-titulo-em-kebab-case.md.

Cada ADR deve seguir o formato MADR com no mínimo as seções: Status, Contexto,
Decisão, Alternativas Consideradas (pelo menos 1 alternativa real discutida ou
plausível), Consequências (positivas e negativas, com trade-off explícito).

Pelo menos 1 ADR deve referenciar explicitamente arquivos, módulos ou padrões
do código existente.

O conjunto de ADRs deve cobrir, no mínimo, 5 das 6 decisões principais discutidas
na reunião:
- Padrão Outbox no MySQL
- Política de retry com backoff e DLQ
- Autenticação HMAC-SHA256 com secret por endpoint
- Garantia at-least-once com X-Event-Id
- Worker em processo separado em polling
- Reuso dos padrões existentes do projeto
```

Esse prompt operou como especificação de contrato para o plugin de ADRs: definiu quantidade, formato, cobertura mínima e rastreabilidade ao código existente. A lista explícita das 6 decisões principais serviu como checklist interno para o agente de identificação priorizar corretamente os candidatos do módulo WEBHOOKS.

---

### Prompt 3 — Geração paralela de ADRs por módulo (Phase 2)

```
Identify potential ADRs for the WEBHOOKS module
Identify potential ADRs for the ORDERS module
Identify potential ADRs for the INFRA module
Identify potential ADRs for the SHARED module
Identify potential ADRs for the DATA module
```

Cinco prompts idênticos em estrutura, lançados em paralelo para agentes independentes. A chave aqui foi a isolação: cada agente operou sem contexto dos outros, garantindo que os scores de relevância fossem calculados de forma independente por módulo. O WEBHOOKS foi o único módulo a gerar candidatos must-document para as 6 decisões alvo.

---

### Prompt 4 — FDD com estrutura de 10 seções obrigatórias e critérios verificáveis

```
Gere docs/FDD.md seguindo a estrutura de 10 seções obrigatórias:
1. Contexto e motivação técnica
2. Objetivos técnicos
3. Escopo e exclusões
4. Fluxos detalhados e diagramas
5. Contratos públicos (endpoints, headers, exemplos de request/response, status codes)
6. Erros, exceções e fallback
7. Observabilidade (métricas, logs, tracing)
8. Dependências e compatibilidade (inclui "Integração com o sistema existente"
   com pelo menos 4 caminhos de arquivo reais)
9. Critérios de aceite técnicos
10. Riscos e mitigação

Critérios obrigatórios:
- Seção 5 deve ter pelo menos 4 endpoints HTTP com payload de exemplo
  (request e response) e status codes
- Matriz de erros usa códigos com prefixo WEBHOOK_
- Seção 8 referencia pelo menos 4 caminhos de arquivo reais do código base
- Seção 7 cita métricas nomeadas, logs estruturados e spans de tracing
- Toda afirmação técnica deve ter rastreabilidade à TRANSCRICAO.md ou ao código
- Não inventar decisões que não foram tomadas na reunião
```

Esse prompt operou como especificação de contrato para o FDD: definiu a estrutura, os critérios quantitativos mínimos (4 endpoints, 4 arquivos reais, prefixo WEBHOOK_) e o critério de rastreabilidade que impediu a IA de extrapolar. A restrição "não inventar decisões" foi adicionada após a primeira geração do RFC ter incluído uma afirmação de probabilidade de duplicatas sem base na transcrição.

---

### Prompt 5 — PRD com entrevista estruturada de 12 etapas

```
Usando o prompt de entrevista para PRD (12 etapas: Contexto, Problema, Objetivos,
Escopo, RF, RNF, Arquitetura, Decisões, Dependências, Riscos, Critérios de aceite,
Testes), gere docs/PRD.md.

Como todo o contexto já está disponível em TRANSCRICAO.md, decisions.md, RFC.md
e FDD.md, preencha diretamente sem entrevista interativa. Cada seção deve:
- Usar dados da transcrição com timestamps como fonte primária
- Referenciar caminhos de arquivo reais para decisões de implementação
- Separar explicitamente o que está incluído do que está fora de escopo
- Incluir pelo menos 4 decisões com justificativa e trade-off
- Incluir pelo menos 4 riscos com probabilidade, impacto, mitigação (subitens)
  e plano de contingência
- Critérios de aceitação devem ser verificáveis objetivamente (sem "funciona bem")
```

O diferencial aqui foi substituir o fluxo de entrevista por preenchimento direto com fonte explícita. Em vez de responder perguntas uma a uma, o prompt especificou o nível de detalhe esperado por seção — o que produziu um PRD com rastreabilidade sem depender de iteração interativa.

---

## Iterações e ajustes

### Iteração 1 — Problema de numeração dos ADRs gerados

O plugin `adr-generator` usa placeholder `XXX` para numeração e espera renumeração manual após a geração de todos os arquivos. Um dos agentes (Worker Process) auto-numerou como `ADR-001` antes dos demais terminarem, criando conflito de numeração. Foi necessário renomear manualmente todos os arquivos após a conclusão paralela dos 5 agentes, aplicando a sequência lógica (001-Outbox, 002-Retry/DLQ, 003-Worker, 004-HMAC, 005-At-least-once).

**Lição**: em geração paralela com placeholder, nunca confiar em auto-numeração parcial. Sempre aguardar todos os agentes e renumerar em lote.

### Iteração 2 — SHARED module sem ADRs: decisão correta, não falha

O agente do módulo SHARED retornou 0 ADRs candidatos (AppError, Pino e PaginatedResponse ficaram abaixo do threshold de 75/150). A primeira reação foi questionar se havia falha. Após análise do raciocínio do agente, a decisão estava correta: esses padrões têm peso arquitetural insuficiente para ADR isolado — pertencem como contexto em ADRs de INFRA e WEBHOOKS, onde são citados. Nenhuma correção foi necessária; o resultado foi validado como correto.

### Iteração 3 — ADRs classificados como `needs-input`

Dois ADRs (Worker Process e HMAC-SHA256) foram classificados como Tier 2 pelo `adr-generator`, com marcadores `[NECESSITA INFORMAÇÃO]` para pontos não decididos na reunião: SLA formal, ferramenta de supervisão de processo (systemd/PM2/Docker), política de armazenamento de secrets em repouso e requisitos regulatórios (SOC 2 / PCI-DSS). Esses marcadores foram mantidos intencionalmente — refletem gaps reais que a equipe precisará resolver antes da implementação, não falhas de geração.

### Iteração 4 — Contexto da transcrição no mapeamento de codebase

Na Phase 1 do plugin de ADRs, o parâmetro `--context-dir` foi passado apontando para `decisions.md` em vez de um diretório de docs de arquitetura. O agente aceitou o arquivo individual como contexto e integrou corretamente as 24 decisões aplicadas, 4 adiadas e 8 descartadas no mapeamento. Isso enriqueceu o `mapping.md` com as intenções de design antes de qualquer código existir para o módulo WEBHOOKS.

### Iteração 5 — Revisão crítica dos ADRs gerados e correção de 3 problemas

Após a geração e linking dos ADRs, foi feita uma leitura completa de todos os 5 arquivos para avaliar superficialidade. Foram identificados e corrigidos 3 problemas concretos:

1. **Marcadores `[NECESSITA INFORMAÇÃO]` com respostas na transcrição**: o plugin gerou 8 marcadores no total. Vários tinham resposta explícita na reunião (política de retorno do secret via GET, rotação de secret com grace period, contrato de entrega at-least-once, archival de 30 dias como item adiado). Esses marcadores foram substituídos pelo conteúdo correto com referência à transcrição. Apenas os marcadores sobre decisões genuinamente não tomadas na reunião (ferramenta de supervisão de processo, estratégia de monitoramento do worker) foram mantidos.

2. **Referência a Kubernetes sem base na transcrição**: o ADR-003 incluía "manifests Kubernetes" na lista de artefatos de deployment. Kubernetes não foi mencionado em nenhum momento na reunião — era extrapolação da IA. Removido e substituído por "scripts de inicialização", termo neutro e sem pressuposição de stack.

3. **Progressão do backoff ausente no ADR-002**: a sequência `1m / 5m / 30m / 2h / 12h` estava no `decisions.md` mas não aparecia explicitamente nas consequências do ADR. Um desenvolvedor lendo só o ADR não encontrava os valores concretos. Adicionada como tabela na seção de consequências com tempo total calculado (14h36m).

### Iteração 6 — Primeira geração do FDD e revisão crítica com 8 problemas identificados

O FDD foi gerado seguindo o prompt de 10 seções. Uma revisão crítica com agente independente identificou 8 problemas concretos na primeira versão:

1. **Fluxo do worker descrevia filtro de webhooks no despacho**: a transcrição ([09:34] Bruno/Diego) é explícita que o filtro ocorre na inserção, não no worker. O worker recebe uma linha já endereçada a um `webhook_id` específico. Corrigido com referência à transcrição.

2. **"secret UUID" na rotação**: o FDD dizia que a secret era um UUID. A decisão correta é `crypto.randomBytes(32).toString('hex')` com prefixo `whsec_` — entropia de 32 bytes aleatórios, não um UUID v4. Corrigido.

3. **Ordenação `created_at ASC` sem justificativa de ordering por pedido**: o fluxo listava a ordenação como dado arbitrário. A razão ([09:12-09:13] Diego) é que ela garante sequência de eventos do mesmo pedido com single worker — uma propriedade que se perde com multi-worker. Adicionada explicação e referência à limitação conhecida.

4. **Fluxo do worker não mostrava que `webhook_id` já vem na linha do outbox**: o fluxo descrevia "recuperar configuração do webhook de destino" sem explicar como o worker sabe qual webhook buscar. A linha do outbox contém `webhook_id` desde a inserção — detalhe crítico que tornava o fluxo logicamente incompleto.

5. **`publishWebhookEvent` sem origem de importação**: o FDD descrevia a função como "pura" mas não dizia de onde vem. Adicionada referência a `src/modules/webhooks/webhook.publisher.ts` como módulo de origem.

6. **Taxa de amostragem de 10% inventada**: o FDD afirmava "10% para entregas bem-sucedidas" como taxa de tracing. Essa taxa nunca foi discutida na reunião. Removida; substituída por "a definir pela equipe de plataforma".

7. **Schema das tabelas ausente**: o FDD descrevia fluxos que pressupunham campos de tabela sem nunca defini-los. Adicionada subseção com schema de `webhooks`, `webhook_outbox`, `webhook_dead_letter` e `webhook_deliveries`.

8. **Critério de aceite do grace period ausente**: a seção 9 não tinha nenhum critério verificável sobre rotação de secret com `previous_secret` durante o grace period — o caso mais propenso a bugs de implementação. Adicionado como critério explícito.

**Lição**: a primeira versão de um FDD gerado por IA tende a acertar a estrutura e errar nos detalhes de fluxo que envolvem decisões de timing (quando o filtro ocorre, quem resolve o endereçamento). Esses erros não são visíveis sem leitura linha a linha comparada à transcrição.

### Iteração 7 — Auditoria de consistência entre documentos e código

Após todos os documentos estarem gerados, foi executada uma auditoria de consistência verificando:

- Todos os 12 arquivos de código mencionados nos documentos como existentes foram confirmados no repositório (`src/modules/orders/order.service.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/config/database.ts`, `src/shared/errors/app-error.ts`, `src/shared/logger/index.ts`, `src/modules/orders/order.status.ts`, `prisma/schema.prisma`, `tests/setup.ts`, entre outros).

- Funções específicas referenciadas nos documentos foram verificadas no código: `canTransition`, `shouldDebitStock`, `shouldReplenishStock` em `order.status.ts`; `$transaction` e `changeStatus` na linha 126 de `order.service.ts`.

- **Inconsistência encontrada**: `npm run worker` é referenciado nos documentos como entry-point do worker, mas o script não existe em `package.json` — é parte do que será implementado, não do que existe hoje. Os documentos que descrevem esse script como passo de execução (ADR-003, FDD §9) deixam claro que é planejado. O `package.json` atual tem apenas os scripts do sistema existente: `dev`, `build`, `start`, `test`, `db:migrate`, `db:seed`.

**Lição**: verificar a diferença entre "arquivo existente referenciado" e "artefato planejado referenciado" é uma checagem que a IA não faz automaticamente — requer instrução explícita na auditoria.

---

## Como navegar a entrega

```
.
├── README.md                          ← este arquivo (processo de produção)
├── TRANSCRICAO.md                     ← fonte primária: transcrição da reunião
├── decisions.md                       ← classificação tripartite das decisões (pré-processamento)
├── docs/
│   ├── PRD.md                         ← o quê e por quê (visão de produto, 10 RF, métricas, riscos)
│   ├── RFC.md                         ← proposta técnica para revisão (visão de arquitetura)
│   ├── FDD.md                         ← como construir em detalhe (fluxos, contratos, schema, integração)
│   ├── TRACKER.md                     ← rastreabilidade de 71 itens à fonte (transcrição ou código)
│   └── adrs/
│       ├── mapping.md                 ← mapa do codebase (gerado pelo plugin, Phase 1)
│       ├── ADR-001-transactional-outbox-para-despacho-de-eventos-webhook.md
│       ├── ADR-002-exponential-backoff-retry-com-dead-letter-queue.md
│       ├── ADR-003-processo-worker-separado-com-polling-para-consumo-do-outbox.md
│       ├── ADR-004-hmac-sha256-assinatura-payload-por-endpoint-com-rotacao-de-segredo.md
│       └── ADR-005-entrega-pelo-menos-uma-vez-com-idempotencia-x-event-id.md
```

### Ordem de leitura sugerida

1. `TRANSCRICAO.md` — fonte primária; entender o contexto da reunião
2. `decisions.md` — filtro das decisões: o que entrou, o que foi descartado, o que foi adiado
3. `docs/adrs/ADR-001` a `ADR-005` — as decisões arquiteturais individuais, da mais estrutural à mais operacional
4. `docs/RFC.md` — proposta técnica consolidada com alternativas e questões em aberto
5. `docs/FDD.md` — especificação de implementação acionável (fluxos, contratos, schema de tabelas, integração com código existente)
6. `docs/PRD.md` — visão de produto, métricas de sucesso e critérios de aceite
7. `docs/TRACKER.md` — rastreabilidade cruzada de todos os 68 itens

---

*Enunciado original do desafio: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia)*
