### PRD: Order Management System — Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-06-24
Responsável: Marcos (Product Manager)

---

### Resumo

O Order Management System é uma plataforma B2B de gestão de pedidos utilizada por clientes como Atlas Comercial, MaxDistribuição e Nova Cargo. Hoje esses clientes monitoram mudanças de status dos pedidos por polling periódico na API, o que gera latência variável, carga desnecessária e fricção operacional. Esta feature adiciona ao sistema um mecanismo de webhooks outbound: quando o status de um pedido muda, a plataforma notifica proativamente os endpoints registrados pelos clientes, com garantia de entrega, autenticação HMAC-SHA256 e suporte a reprocessamento manual de falhas. O prazo acordado é até fim de novembro de 2026, em três sprints.

---

### Contexto e problema

Público-alvo
- Clientes B2B que integram seus sistemas operacionais com o Order Management System (Atlas Comercial, MaxDistribuição, Nova Cargo)
- Equipe interna de operações e suporte que gerencia configurações de webhook e trata falhas de entrega

Cenários de uso chave
- Cliente recebe notificação automática quando um pedido muda para SHIPPED e aciona seu sistema de logística sem polling manual
- Operador configura quais status de pedido o endpoint do cliente vai monitorar, sem depender do time de engenharia
- Admin da plataforma reprocessa manualmente um evento que falhou e foi para a fila de mensagens mortas, com trilha de auditoria

Onde essa feature será implantada
- Sistema existente: Order Management System em Node.js + TypeScript + Prisma + MySQL 8.0. A feature adiciona um novo módulo (`src/modules/webhooks`), novas tabelas no banco já existente e um processo worker separado (`src/worker.ts`). Nenhuma nova infraestrutura de mensageria é introduzida.

Problemas priorizados
- Polling ineficiente por clientes B2B: clientes batem repetidamente no `GET /orders` para detectar mudanças de status, gerando carga desnecessária na API e latência variável de atualização. A Atlas Comercial sinalizou risco de cancelamento de contrato se a feature não for entregue até fim do trimestre (prioridade: alta)
- Ausência de mecanismo de notificação externa: o sistema não tem como empurrar informações para sistemas de clientes, tornando qualquer integração proativa impossível sem desenvolvimento customizado por parte do cliente (prioridade: alta)
- Sem visibilidade de falhas de entrega: quando uma integração falha hoje, não há registro estruturado do erro, dificultando diagnóstico e reprocessamento (prioridade: média)

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar clientes B2B em tempo real sobre mudanças de status de pedidos | Tempo entre commit da transação de mudança de status e entrega do webhook ao endpoint do cliente | Menos de 10 segundos em condições normais de rede |
| Eliminar dependência de polling pelos clientes | Redução de chamadas ao `GET /orders` pelos três clientes iniciais após go-live | Redução de pelo menos 80% de chamadas de polling pelos clientes que adotarem webhooks |
| Garantir resiliência na entrega em caso de indisponibilidade do cliente | Taxa de eventos que chegam à DLQ sem que o cliente estivesse disponível dentro da janela de retry | Zero eventos perdidos por indisponibilidade de até 15 horas do endpoint do cliente |
| Viabilizar auditoria e reprocessamento de falhas de entrega | Disponibilidade de histórico de tentativas e endpoint de replay para operações de suporte | 100% dos eventos com falha permanente auditáveis e reprocessáveis via endpoint ADMIN |

---

### Escopo

Incluso
- CRUD de configurações de webhook por cliente: `POST`, `PATCH`, `DELETE`, `GET /api/v1/webhooks`
- Filtro de eventos por lista de status configurada por webhook (ex: apenas SHIPPED e DELIVERED)
- Geração de secret HMAC por endpoint, com rotação e grace period de 24 horas
- Inserção atômica de eventos na tabela `webhook_outbox` dentro da transação de mudança de status de pedido
- Worker de polling a cada 2 segundos para consumo do outbox e despacho de chamadas HTTP
- Retry com backoff exponencial: 5 tentativas nos intervalos 1m / 5m / 30m / 2h / 12h
- Dead Letter Queue em tabela `webhook_dead_letter` com payload, motivo e timestamp
- Endpoint `GET /api/v1/webhooks/:id/deliveries` para histórico de tentativas de entrega
- Endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` restrito a role ADMIN, com log de auditoria
- Assinatura HMAC-SHA256 de cada payload com headers: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`
- Validação de URL HTTPS obrigatório, limite de 64KB por payload, timeout de 10 segundos

Fora de escopo
- Webhooks inbound (recepção de eventos externos pela plataforma)
- Dashboard visual para o cliente acompanhar histórico de entregas
- Notificação por email em caso de falhas consecutivas (adiado para fase seguinte)
- Rate limiting de envio de webhooks por cliente (observar em produção antes de implementar)
- Escalabilidade multi-worker com particionamento por `order_id` (limitação documentada, adiado)
- Archival automático de linhas entregues na outbox após 30 dias (adiado)

---

### Requisitos funcionais

#### RF-001 Gerenciamento de configurações de webhook
Operadores e clientes devem poder cadastrar, editar, listar e remover configurações de endpoint de webhook para receber notificações de mudança de status de pedidos.

**Fluxo principal**
- Operador envia `POST /api/v1/webhooks` com `customerId`, `url` (HTTPS) e lista de `events` (status de pedido de interesse)
- Sistema valida schema via Zod: URL obrigatoriamente HTTPS, lista de eventos não vazia
- Sistema gera secret HMAC via `crypto.randomBytes(32)` com prefixo `whsec_` e armazena na tabela `webhooks`
- Resposta `201 Created` retorna configuração completa incluindo a secret (única vez que a secret é retornada)
- Operador consulta lista com `GET /api/v1/webhooks?customerId=:id` — resposta nunca inclui o campo `secret`
- Operador edita com `PATCH /api/v1/webhooks/:id` (URL, eventos, estado ativo/inativo)
- Operador remove com `DELETE /api/v1/webhooks/:id`

**Fluxos alternativos e exceções**
- Se o cliente perde a secret, não é possível recuperá-la via `GET` — deve usar o endpoint de rotação para gerar nova
- Webhook inativo não recebe eventos: `publishWebhookEvent` filtra apenas webhooks com `active = true` na busca dentro da transação

**Erros previstos**
- `WEBHOOK_INVALID_URL` (400): URL não é HTTPS ou está malformada
- `WEBHOOK_EVENTS_REQUIRED` (400): lista de eventos vazia ou ausente
- `WEBHOOK_DUPLICATE_URL` (409): URL já cadastrada para o mesmo `customerId`
- `WEBHOOK_NOT_FOUND` (404): ID de webhook inexistente em operações de edição ou remoção
- `401 Unauthorized`: token ausente ou inválido

**Prioridade:** alta

---

#### RF-002 Entrega de eventos de mudança de status
Quando um pedido muda de status, a plataforma deve inserir um evento na fila e despachá-lo para todos os endpoints do cliente que têm aquele status na lista de eventos configurados.

**Fluxo principal**
- `OrderService.changeStatus` executa transação Prisma
- Dentro da transação, após atualizar status e estoque: `publishWebhookEvent(tx, order, fromStatus, toStatus)` busca webhooks ativos do customer com `toStatus` na lista de eventos
- Para cada webhook encontrado: insere linha em `webhook_outbox` com payload snapshot, `webhook_id`, `status = PENDING`, `attempt_count = 0`
- Transação commita; worker (processo separado, polling a cada 2s) lê eventos `PENDING` ordenados por `created_at ASC`
- Worker assina payload com HMAC-SHA256, executa `POST` para URL do cliente com timeout 10s e headers obrigatórios
- Resposta `2xx`: marca evento como `DELIVERED`, registra em `webhook_deliveries`

**Fluxos alternativos e exceções**
- Se nenhum webhook do customer tem o `toStatus` inscrito, nenhuma linha é inserida na outbox (filtro na inserção, irreversível)
- Resposta não-`2xx` ou timeout: incrementa `attempt_count`, agenda `retry_at` conforme backoff, mantém status `PENDING`
- Após 5 falhas: evento movido para `webhook_dead_letter` com payload, motivo e timestamp

**Erros previstos**
- `WEBHOOK_PAYLOAD_TOO_LARGE`: payload do evento excede 64KB; evento vai direto para DLQ sem tentativa de envio
- `WEBHOOK_DELIVERY_TIMEOUT`: cliente não responde em 10 segundos; tratado como falha e agendado para retry

**Prioridade:** alta

---

#### RF-003 Rotação de secret HMAC por endpoint
Clientes devem poder solicitar uma nova secret para um endpoint de webhook sem interromper a entrega de eventos durante a migração.

**Fluxo principal**
- Operador chama `POST /api/v1/webhooks/:id/rotate-secret`
- Sistema gera nova secret via `crypto.randomBytes(32).toString('hex')` com prefixo `whsec_`
- Secret atual é movida para `previous_secret`; nova secret é armazenada em `secret`; `previous_secret_expires_at = now + 24h`
- Resposta retorna apenas a nova secret e `previousSecretExpiresAt`
- Durante o grace period, o worker aceita verificação de assinatura gerada com qualquer uma das duas secrets

**Fluxos alternativos e exceções**
- Após 24h, `previous_secret` é considerada inválida pelo worker (verificação inline no ciclo de despacho)
- Se o cliente não migrar dentro de 24h, eventos assinados com a secret anterior passam a falhar na verificação do lado do cliente

**Erros previstos**
- `WEBHOOK_NOT_FOUND` (404): ID de webhook inexistente
- `401 Unauthorized`: token ausente ou inválido

**Prioridade:** alta

---

#### RF-004 Histórico de entregas por endpoint
Clientes e operadores devem poder consultar o histórico de tentativas de entrega de um endpoint de webhook para diagnóstico de falhas e verificação de recebimento.

**Fluxo principal**
- Operador chama `GET /api/v1/webhooks/:id/deliveries` (com paginação opcional)
- Sistema retorna lista de tentativas com: `event_id`, `event_type`, `status`, `http_status`, `attempt_count`, `duration_ms` e timestamps

**Fluxos alternativos e exceções**
- Eventos que foram para DLQ aparecem com `status = FAILED` e `attempt_count = 5` na listagem

**Erros previstos**
- `WEBHOOK_NOT_FOUND` (404): ID de webhook inexistente
- `401 Unauthorized`: token ausente ou inválido

**Prioridade:** média

---

#### RF-005 Reprocessamento manual de eventos na fila de mensagens mortas
Administradores da plataforma devem poder reprocessar manualmente eventos que falharam todas as tentativas e estão na fila de mensagens mortas, com registro de auditoria da ação.

**Fluxo principal**
- Admin autenticado (role ADMIN) chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay`
- Sistema verifica role via `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`)
- Cria nova linha em `webhook_outbox` com `status = PENDING` e `attempt_count = 0`
- Preenche `replayed_at` e `replayed_by` na linha original de `webhook_dead_letter` (registro preservado)
- Log estruturado registra `admin_id`, `dead_letter_id` e timestamp

**Fluxos alternativos e exceções**
- Se o webhook de destino foi desativado entre a falha original e o replay, o evento será inserido na outbox mas não terá webhooks ativos para despachar na inserção de `publishWebhookEvent` — o replay não garante entrega, apenas reprocessamento

**Erros previstos**
- `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404): ID não encontrado em `webhook_dead_letter`
- `WEBHOOK_FORBIDDEN` (403): usuário autenticado sem role ADMIN
- `401 Unauthorized`: token ausente ou inválido

**Prioridade:** média

---

### Requisitos não funcionais

Performance
- Latência ponta a ponta (commit da transação até entrega no endpoint do cliente) menor que 10 segundos em condições normais de rede, com latência de polling máxima de 2 segundos
- Timeout de 10 segundos por chamada HTTP outbound do worker; expiração tratada como falha

Disponibilidade
- Worker deve tolerar indisponibilidade do cliente de até aproximadamente 15 horas sem perder o evento (cobertura via janela de retry de 14h36m)
- Restart da API não deve interromper o ciclo de polling do worker (processos independentes)

Segurança e autorização
- Autenticação JWT obrigatória em todos os endpoints do módulo webhooks
- Secret HMAC gerada pela plataforma com `crypto.randomBytes(32)`, nunca exposta em `GET` nem em logs
- Secret por endpoint; vazamento de uma não compromete outras integrações
- TLS obrigatório: URL `http://` recusada com erro de validação no cadastro
- Endpoint de replay de DLQ restrito a role ADMIN via `requireRole('ADMIN')` existente
- Revisão de segurança de 2 dias úteis com engenheira de segurança antes do deploy em produção

Observabilidade
- Logs estruturados via Pino com campos: `event_id`, `webhook_id`, `attempt`, `http_status`, `duration_ms` em cada linha do worker
- Campos sensíveis nunca logados: `secret`, `previous_secret`, `authorization`, `X-Signature`
- Métricas: `webhook_outbox_pending_count` (gauge), `webhook_delivery_duration_ms` (histogram), `webhook_delivery_success_total` e `webhook_delivery_failure_total` (counters), `webhook_dead_letter_total` (counter)
- Alerta operacional: `webhook_outbox_pending_count` acima de 100 por mais de 5 minutos indica worker parado

Confiabilidade e integridade de dados
- Inserção em `webhook_outbox` deve ocorrer dentro da mesma transação Prisma de `changeStatus`; rollback da transação de pedido elimina o evento da outbox
- `event_id` (UUID) gerado na inserção do outbox e estável entre todas as tentativas de retry
- Eventos na DLQ são preservados permanentemente; nunca descartados automaticamente

Compatibilidade e portabilidade
- API REST JSON com prefixo `/api/v1`, padrão do projeto
- Worker empacotado como processo Node.js separado na mesma imagem do projeto; script `npm run worker` como entry-point
- Sem nova dependência de infraestrutura: outbox persiste no MySQL existente

Compliance
- Log de auditoria de replay de DLQ com `admin_id`, `dead_letter_id` e timestamp, persistido em `webhook_dead_letter.replayed_by` e `replayed_at`
- Secret nunca persistida em texto claro em logs ou respostas de listagem

---

### Arquitetura e abordagem

Abordagem
- Feature adicionada como módulo ao monólito Node.js existente, seguindo o padrão estabelecido de módulos (controller / service / repository / routes / schemas). O worker de despacho roda como processo Node.js separado (`src/worker.ts`) conectado ao mesmo banco MySQL, sem broker de mensagens externo. O padrão Transactional Outbox garante atomicidade entre a mudança de status do pedido e a criação do evento de notificação.

Componentes
- `src/modules/webhooks/` — módulo de CRUD de configurações de webhook (WebhookController, WebhookService, WebhookRepository, rotas, schemas Zod)
- `src/modules/webhooks/webhook.publisher.ts` — função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro da transação de `OrderService.changeStatus`
- `src/worker.ts` — entry-point do processo worker; instancia PrismaClient separado e inicia ciclo de polling a cada 2 segundos
- `src/modules/webhooks/webhook.processor.ts` — lógica de despacho: leitura da outbox, assinatura HMAC, chamada HTTP, gestão de retry e DLQ
- Tabelas MySQL: `webhooks`, `webhook_outbox`, `webhook_dead_letter`, `webhook_deliveries`

Integrações
- `src/modules/orders/order.service.ts:changeStatus` — único ponto de integração com código existente; recebe chamada a `publishWebhookEvent` dentro da transação Prisma
- Endpoints HTTPS dos clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) — destino das chamadas HTTP outbound do worker
- `src/middlewares/auth.middleware.ts` — `requireRole('ADMIN')` reutilizado no endpoint de replay de DLQ sem modificação
- `src/middlewares/error.middleware.ts` — trata erros `AppError` com prefixo `WEBHOOK_` automaticamente sem modificação

### Decisões e trade-offs

#### Decisão: Transactional Outbox em MySQL em vez de broker de mensagens externo
- **Justificativa:** Um broker externo (Redis Streams, Kafka, RabbitMQ) resolveria o problema de despacho assíncrono, mas não eliminaria a necessidade de coordenação entre o commit da transação MySQL e a publicação no broker — o problema de consistência persiste. O outbox no MySQL existente resolve com uma única transação sem nova infraestrutura, adequado para o tamanho do time.
- **Trade-off:** Sem broker, não há backpressure nativo, particionamento automático nem escalonamento horizontal de consumidores. Com crescimento de volume, a tabela `webhook_outbox` pode se tornar gargalo de leitura. Archival de linhas entregues é necessário no longo prazo (adiado para fase seguinte).

#### Decisão: Worker em polling a cada 2 segundos em vez de trigger reativo
- **Justificativa:** MySQL 8.0 não possui mecanismo equivalente ao `LISTEN/NOTIFY` do PostgreSQL. Trigger de banco não notifica processo externo; qualquer implementação reativa exigiria workarounds frágeis (escrita em arquivo, chamada HTTP a partir do trigger). Polling de 2 segundos atende o requisito de entrega abaixo de 10 segundos com latência máxima de 2 segundos.
- **Trade-off:** O worker acorda a cada 2 segundos mesmo sem eventos novos, consumindo uma conexão do pool e executando um SELECT. Com low traffic, o overhead é irrisório; com alta frequência de eventos, o batching por ciclo amortiza o custo por evento.

#### Decisão: Garantia at-least-once com `X-Event-Id` em vez de exactly-once
- **Justificativa:** Exactly-once exigiria coordenação bidirecional com cada sistema externo (two-phase commit ou acknowledgment protocol), com complexidade desproporcional para o caso de uso. Stripe e GitHub adotam at-least-once com identificador estável. `X-Event-Id` UUID gerado na inserção do outbox e estável entre retries permite deduplicação eficiente pelo cliente.
- **Trade-off:** A responsabilidade de deduplicação é transferida para o cliente. Clientes que não implementarem idempotência processarão duplicatas. Documentação no portal do desenvolvedor é obrigatória antes do go-live.

#### Decisão: Secret HMAC por endpoint em vez de secret global
- **Justificativa:** O time já vivenciou incidente em que um cliente expôs uma secret em log de aplicação. Com secret global, esse vazamento comprometeria todas as integrações simultaneamente. Secret por endpoint limita o raio de explosão a um único webhook.
- **Trade-off:** Aumenta a complexidade de gestão de chaves: geração, armazenamento e rotação por linha de webhook. Operacionalmente mais custoso do que uma secret única, mas necessário dado o histórico de incidente.

#### Decisão: Filtro de eventos na inserção do outbox em vez de no despacho
- **Justificativa:** Filtrar na inserção evita criar linhas para eventos que nenhum webhook do customer quer receber, mantendo a tabela `webhook_outbox` menor e o worker mais eficiente.
- **Trade-off:** O filtro é irreversível: se um cliente não tinha o status configurado no momento da mudança, o evento não pode ser entregue retroativamente. Clientes precisam configurar os filtros antes das mudanças ocorrerem.

---

### Dependências

#### Técnica: Integração em `OrderService.changeStatus`
Bruno (Eng. Pleno, time de Pedidos) é responsável pela modificação de `src/modules/orders/order.service.ts` para chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da transação existente. Sem essa integração, nenhum evento é gerado independentemente do módulo de webhooks estar pronto.

#### Técnica: Migração de banco para criação das novas tabelas
As tabelas `webhooks`, `webhook_outbox`, `webhook_dead_letter` e `webhook_deliveries` precisam ser criadas via Prisma Migrate antes de qualquer deploy do módulo. Dependência de ambiente de staging e produção com MySQL acessível.

#### Organizacional: Revisão de segurança com Sofia antes do deploy
Sofia (Eng. de Segurança) reservou pelo menos 2 dias úteis para revisar o código de HMAC e geração de secret antes de qualquer deploy em produção. Essa revisão precisa estar agendada no Sprint 3 antes da janela de deploy.

#### Organizacional: Documentação no portal do desenvolvedor por Marcos
Marcos (PM) se comprometeu a documentar o contrato de at-least-once com exemplos de deduplicação por `X-Event-Id` no portal do desenvolvedor antes da abertura das integrações para os três clientes iniciais. Sem essa documentação, clientes podem processar duplicatas sem saber.

#### Organizacional: Definição da ferramenta de supervisão do processo worker
A ferramenta para supervisionar o processo worker em produção (Docker restart policy, PM2 ou systemd) não foi definida na reunião. Diego e Larissa precisam decidir antes do Sprint 3 para que o `Dockerfile` e os scripts de inicialização sejam corretos.

---

### Riscos e mitigação

#### Worker para silenciosamente e eventos acumulam na outbox sem alerta
- **Probabilidade:** média
- **Impacto:** Clientes ficam sem receber notificações por tempo indeterminado sem que a equipe saiba; pode impactar os três clientes iniciais simultaneamente
- **Mitigação:**
  - Alerta ativo em `webhook_outbox_pending_count` acima de 100 por mais de 5 minutos
  - Supervisão do processo worker com restart automático (Docker restart policy ou PM2) antes do go-live
  - Log de ciclo de polling a cada iteração para rastrear últimas execuções
- **Plano de contingência:** Restart manual do worker; eventos presos em `PROCESSING` com `updated_at` antigo são resetados para `PENDING` pelo worker no início de cada ciclo

#### Cliente não implementa idempotência e processa eventos duplicados
- **Probabilidade:** alta (para clientes sem aviso prévio)
- **Impacto:** Operações duplicadas no sistema do cliente (ex: confirmar despacho de pedido duas vezes); risco de inconsistência nos dados do cliente
- **Mitigação:**
  - Documentação clara e obrigatória no portal do desenvolvedor antes do go-live, com exemplo de código de deduplicação por `X-Event-Id` (Marcos, `TRANSCRICAO.md:156`)
  - `X-Event-Id` estável entre retries é invariante verificável em testes de integração com os clientes
  - Suporte ativo na fase de onboarding dos três clientes iniciais
- **Plano de contingência:** Se duplicata for detectada em produção, equipe de suporte identifica o `event_id` duplicado via `webhook_deliveries` e coordena correção com o cliente afetado

#### Vazamento de secret HMAC por um cliente
- **Probabilidade:** baixa (incidente anterior já ocorreu com cliente diferente)
- **Impacto:** Ator malicioso pode forjar eventos para aquele cliente específico; raio de explosão limitado a um único endpoint
- **Mitigação:**
  - Secret por endpoint: isolamento de comprometimento
  - Endpoint de rotação de secret disponível imediatamente via `POST /api/v1/webhooks/:id/rotate-secret`
  - Grace period de 24h permite migração sem downtime
- **Plano de contingência:** Revogar secret imediatamente via rotação; notificar o cliente para atualizar verificação antes do fim do grace period de 24 horas

#### Crescimento descontrolado da tabela `webhook_outbox` sem archival
- **Probabilidade:** média (archival está explicitamente fora do escopo desta fase)
- **Impacto:** Degradação progressiva de performance das queries do worker ao longo do tempo em produção
- **Mitigação:**
  - Índices em `(status, retry_at)` e `order_id` na tabela `webhook_outbox`
  - Monitorar volume da tabela mensalmente após go-live
- **Plano de contingência:** Script de limpeza manual das linhas `DELIVERED` com `created_at` acima de 30 dias, executado por DBA como medida emergencial antes que o archival automático seja implementado na fase seguinte

#### Ferramenta de supervisão do worker não definida antes do deploy
- **Probabilidade:** média (ponto em aberto ao fim da reunião de inception)
- **Impacto:** Worker sobe sem restart automático em produção; primeira queda do processo não é recuperada automaticamente
- **Mitigação:**
  - Decisão deve ser tomada por Diego e Larissa antes do início do Sprint 3
  - Bloquear deploy em produção até supervisão estar configurada e testada
- **Plano de contingência:** Se o deploy avançar sem decisão, usar Docker restart policy como default mínimo por ser a opção mais simples e disponível em qualquer ambiente containerizado

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Mudança de status de pedido resulta em evento entregue ao endpoint do cliente em menos de 10 segundos em condições normais de rede
- Rollback da transação de `changeStatus` não cria nenhuma linha em `webhook_outbox` (atomicidade garantida)
- Se nenhum webhook do customer tem o `toStatus` inscrito, nenhuma linha é inserida na outbox
- Evento com resposta `2xx` do cliente é marcado como `DELIVERED` e não é reenviado
- Evento com falha segue o backoff exato: 1m / 5m / 30m / 2h / 12h
- Após 5 falhas consecutivas, evento está em `webhook_dead_letter` com payload, motivo e timestamp corretos
- Replay de DLQ por usuário ADMIN cria nova linha em `webhook_outbox` com `attempt_count = 0`; registro original em `webhook_dead_letter` é preservado com `replayed_at` e `replayed_by` preenchidos
- Log de cada replay contém `admin_id`, `dead_letter_id` e timestamp
- `X-Event-Id` do mesmo evento é idêntico em todas as tentativas de retry
- Endpoint com URL `http://` é recusado com erro `WEBHOOK_INVALID_URL` no cadastro
- Payload com mais de 64KB não é enviado; evento vai direto para DLQ com motivo `WEBHOOK_PAYLOAD_TOO_LARGE`
- Campo `secret` nunca aparece em resposta de `GET /api/v1/webhooks` nem em nenhum log
- Durante grace period de 24h após rotação, assinatura gerada com `previous_secret` é aceita pelo worker
- Usuário com role OPERATOR recebe 403 ao chamar o endpoint de replay de DLQ
- Worker e API são processos independentes: restart da API não interrompe o ciclo de polling do worker
- Revisão de segurança com Sofia concluída e aprovada antes do deploy em produção

---

### Testes e validação

Tipos de teste obrigatórios
- Testes de integração para os fluxos principais: criação de webhook, mudança de status gerando evento, entrega bem-sucedida, ciclo completo de retry até DLQ, replay de DLQ
- Testes de integração para atomicidade: rollback de transação de `changeStatus` não deve deixar linha na outbox
- Testes de segurança de autorização: endpoint de replay deve retornar 403 para role OPERATOR e 401 sem token
- Testes de validação de contrato: URL `http://` recusada, eventos vazios recusados, payload > 64KB vai para DLQ
- Testes de assinatura HMAC: verificação correta com secret atual durante grace period e rejeição após expiração de `previous_secret`
- Revisão manual de segurança por Sofia cobrindo geração de secret, armazenamento e rotação (2 dias úteis no Sprint 3)

Estratégia de validação
- Testes de integração contra MySQL real (sem mocks), seguindo o padrão estabelecido em `tests/` com Vitest + Supertest
- Validação end-to-end manual com endpoint de teste controlado antes do go-live: simular indisponibilidade do cliente, verificar retry, verificar chegada na DLQ e replay
- Onboarding guiado com os três clientes iniciais (Atlas Comercial, MaxDistribuição, Nova Cargo) usando ambiente de staging antes de produção
