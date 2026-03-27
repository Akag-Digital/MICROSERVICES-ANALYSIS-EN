# Análise Técnica: Débitos, Gargalos de Escala e Caso para Microserviços

## 0. Contexto: Por que o Refactor é Necessário

O sistema foi projetado originalmente para seguir a **mesma lógica de container do Rove LAB** — uma abordagem simplificada, pensada para operar com **no máximo 1.000 SKUs**. Nesse contexto, um monolito Remix era suficiente e adequado para a complexidade existente.

Com o tempo, novas funcionalidades e regras de negócio foram adicionadas — especialmente as features de **sincronização de bundles pai-filho, validação de ETA, importação/exportação em massa e processamento de pedidos via webhook** — sem uma revisão da arquitetura base. Cada nova feature foi encaixada na estrutura existente, que não foi concebida para suportá-las em escala.

O resultado é que o **impacto dessas mudanças atingiu o sistema na sua fundação**: a lógica de sincronização, que deveria ser um processo isolado e assíncrono, hoje roda no mesmo processo que serve a UI e responde webhooks. O que era uma base sólida para 1.000 SKUs se torna um gargalo estrutural para 12.000.

Não se trata de má implementação — o sistema cumpriu bem o papel para o qual foi desenhado. Trata-se de **evolução de escopo**: as novas regras de negócio exigem uma arquitetura diferente, e esse refactor é a resposta natural a esse crescimento.

### O Schema Prisma Não Acompanhou a Evolução

A estrutura do banco de dados reflete a lógica original simplificada e acumulou limitações que hoje impactam diretamente o fluxo de sincronização e escala:

**1. `ETA`, `compressionDate` e `completedAt` são `String` no modelo `Container`**
```prisma
model Container {
  ETA             String   // deveria ser DateTime
  compressionDate String?  // deveria ser DateTime?
  completedAt     String?  // deveria ser DateTime?
}
```
Comparações de data acontecem em código de aplicação, não no banco. É impossível fazer `WHERE ETA < NOW()` com índice — o cron de validação de ETA precisa carregar todos os containers para memória e comparar em JS.

**2. `Variant` e `ProductsWithoutContainers` são modelos paralelos com campos duplicados**
```prisma
// Variant tem:          preOrderStock, preOrderSales, status, startDate, endDate, templateId, stockManagement...
// ProductsWithoutContainers tem: preOrderStock, preOrderSales, status, startDate, endDate, templateId, stockManagement...
```
São duas tabelas separadas representando conceitos similares, sem base comum. Toda lógica de sincronização precisa ser escrita duas vezes — uma para cada modelo — e qualquer nova feature de pré-venda precisa ser implementada em ambos.

**3. `ProductChildren` usa strings Shopify como referência em vez de FK**
```prisma
model ProductChildren {
  shopifyChildId  String?  // string solta, sem FK para Variant
  shopifyParentId String?  // string solta, sem FK para ProductsWithoutContainers
  parentId        Int?     // FK para ProductsWithoutContainers (nullable)
  childId         Int?     // campo sem FK definida
}
```
A relação entre um filho e sua `Variant` real é resolvida por matching de string em código, não por integridade referencial. Isso significa que um `shopifyChildId` desatualizado nunca é detectado pelo banco — vira dado silenciosamente inconsistente.

**4. `Variant.containerId` é obrigatório (non-null)**
```prisma
model Variant {
  containerId Int  // obrigatório — toda variante DEVE ter container
}
```
No fluxo de bundles, filhos de `ProductsWithoutContainers` existem sem container direto. Isso força workarounds no código para criar containers artificiais ou manter dois caminhos de código completamente separados para o mesmo conceito de "variante em pré-venda".

**5. `preOrderStock` e `preOrderSales` como campos diretos na linha**
```prisma
preOrderStock Int @default(0)
preOrderSales Int @default(0)
```
Múltiplos webhooks de pedido processados em paralelo fazem `UPDATE variant SET preOrderSales = preOrderSales + qty` na mesma linha simultaneamente. Sem controle de concorrência explícito, isso é uma race condition em escala — dois webhooks processando ao mesmo tempo podem sobrescrever um ao outro.

**6. Sem campo `shop` em `Variant`, `Container` e `OrderItem`**
```prisma
model Variant   { /* sem shop */ }
model Container { /* sem shop */ }
model OrderItem { /* sem shop */ }
```
Para filtrar dados por loja, é necessário fazer join: `Variant → Product → shop`. Em queries de sincronização com 12k produtos, esse join em cadeia aumenta significativamente o custo das queries e impede índices simples por shop.

**7. `WebhookIdempotency` e `Webhooks` são tabelas separadas com sobreposição**
```prisma
model Webhooks           { orderId BigInt @unique; data Json? }  // payload
model WebhookIdempotency { orderId BigInt; webhookType String }  // deduplicação
```
O controle de idempotência cresceu como tabela separada em vez de ser consolidado. Com filas assíncronas (necessárias para 12k produtos), essa lógica precisaria ser repensada de qualquer forma.

---

Esses pontos do schema não bloqueiam o funcionamento atual, mas **multiplicam o custo de cada operação de sincronização** e tornam qualquer extração para microserviços mais complexa do que precisaria ser. O redesign do schema faz parte do refactor arquitetural.

---

## 1. Estado Atual: Monolito Remix com Múltiplos Papéis

O projeto é um **Shopify Admin App** que roda tudo num único processo Node.js/Remix:
- Serve a UI administrativa (React/Polaris)
- Processa webhooks em tempo real
- Executa importações/exportações CSV bloqueantes
- Faz chamadas síncronas para a Shopify API
- Roda scripts de cron job externos que usam o mesmo DB

Isso funciona para dezenas de produtos. Para **12.000 produtos**, cada um desses papéis vai colidir com os outros.

---

## 2. Débitos Técnicos Identificados

### 2.1 Dead Code e Duplicação Severa

O `refactoring-plan.md` já catalogou ~1.155 linhas eliminíveis, mas o problema vai além da cosmética:

| Problema | Arquivo(s) | Impacto |
|---|---|---|
| Dois `orderTagManager` quasi-idênticos | `.js` (~1.389 linhas) + `.server.js` (~1.595 linhas) | Manutenção duplicada, risco de bugs divergentes |
| 8 hooks de seleção com código idêntico | `useAddProductsSelection`, `useAddProductChildrenSelection`, etc. | Qualquer bug precisa ser corrigido 8 vezes |
| 4 implementações de `searchProducts` | `products-sku`, `products-by-container`, `products`, `products-parents` | Comportamentos podem divergir silenciosamente |
| `normalizeToVariantGid()` em 4 lugares diferentes | `orderTagManager`, `separateItemsByContainer`, `syncParentShopifyInventoryNoPreorder`, `csvUpload` | Cada um pode normalizar diferente e causar ID mismatch |
| `escapeCSV`, `ensureExportsDir`, `streamFinish`, `getUniqueExportFileName` duplicados | `containers/csvExport.js` e `products-parents/csvExport.js` | Qualquer melhoria no CSV precisa ser feita em ambos |

### 2.2 Webhooks Registrados Manualmente (Decisão Arquitetural)

Os webhooks críticos estão **ativos em produção**, registrados **manualmente** via Shopify API — intencionalmente fora do `shopify.app.toml`. Essa é uma decisão arquitetural para ter controle total sobre quando e como os webhooks são registrados, independente do ciclo de deploy do app.

```
orders/create           ← ativo (registro manual)
orders/updated          ← ativo (registro manual)
orders/cancelled        ← ativo (registro manual)
inventory_levels/update ← ativo (registro manual)
products/update         ← ativo (registro manual)
```

**Esse padrão deve ser mantido.** O registro manual garante que mudanças no `shopify.app.toml` ou execuções de `npm run deploy` não interfiram nos webhooks em produção.

### 2.3 Arquivos Monolíticos sem Separação de Responsabilidade

| Arquivo | Tamanho | Problema |
|---|---|---|
| `orderTagManager.server.js` | ~1.595 linhas / 48KB | Geração de tags, processamento de pedidos, lógica de inventário tudo junto |
| `containerActions.js` | 857 linhas | Criar, atualizar, arquivar e calcular estoque de container num arquivo só |
| `products-container-id.action.server.js` | 812 linhas | Múltiplos `intent` de formulário numa função gigante |
| `parentPreOrderAndContainers.js` | 766 linhas | Análise de status + busca de containers + cálculo de inventário |
| `ContainersTable.jsx` | 833 linhas | Componente UI com lógica de negócio complexa embutida |

### 2.4 Exportação CSV para Disco Local

```javascript
// app/utils/containers/csvExport.js
// Gera arquivo em: public/exports/export_container_data.csv
```

Os arquivos CSV são gravados em `public/exports/` no disco do servidor. Num ambiente com múltiplas réplicas (necessário para 12k produtos), **cada réplica teria seus próprios arquivos**, e a réplica que serviu o download pode não ser a que gerou o CSV.

### 2.5 Três Padrões de Error Handling Coexistindo

```javascript
// Padrão A — correto (com logger)
} catch (error) { logError("contexto", error); throw error; }

// Padrão B — silencia o contexto
} catch (error) { throw error; }

// Padrão C — sem try/catch (erro borbulha não capturado)
const result = await operacaoArriscada();
```

Em produção com 12k produtos, um erro não logado num webhook pode fazer o Shopify retentar o mesmo webhook centenas de vezes sem você saber o que aconteceu.

---

## 3. Por que 12.000 Produtos Vai Travar — Gargalos de Escala Concretos

### 3.1 Validação de ETA: Loop Serial por Produto

```javascript
// app/utils/etaValidationManager.js — linha 30
for (const parentProduct of parentProducts) {            // loop por TODOS os pais
  const childVariants = await prisma.variant.findMany({  // 1 query por pai
    where: { shopifyId: { in: childShopifyIds } }
  });
  // ... mais queries, mais chamadas Shopify API
}
```

**O problema:** O cron job de validação de ETA faz **pelo menos 1 query de banco por produto pai**. Com 12.000 produtos (hipotético: 3.000 pais × 4 filhos cada), isso são **3.000 queries sequenciais** só para carregar os dados — sem contar as chamadas à Shopify API que vêm depois para atualizar as policies.

A execução do cron vai bloquear o banco durante todo esse tempo, deixando a UI lenta para todos os usuários.

### 3.2 Importação CSV: Query por Linha

```javascript
// app/utils/containers/csvUpload.js
async function getOrCreateTemplate(templateName, shop) {
  let template = await prisma.template.findFirst({ ... }); // query por linha CSV
  if (!template) {
    template = await prisma.template.create({ ... });       // create por linha CSV
  }
  return template;
}
```

Um CSV de 12.000 produtos pode ter 12.000+ chamadas `getOrCreateTemplate`, todas síncronas no processo principal do Remix, bloqueando **todas as outras requisições** da UI durante esse tempo.

Além disso, a importação de Products Parents faz chamadas à Shopify GraphQL por bloco de 100 variantes:

```javascript
// app/utils/products-parents/csvUpload.js
const CHUNK = 100;
for (let i = 0; i < unique.length; i += CHUNK) {
  const response = await admin.graphql(query, { variables: { ids: chunk } });
}
```

12.000 produtos = 120 chamadas GraphQL sequenciais, cada uma ~300ms = **~36 segundos só de I/O de API**, durante os quais o processo Remix está completamente bloqueado.

### 3.3 Sincronização de Bundle: N+1 Queries

```javascript
// app/utils/updateChildrenInContainer.js
for (const item of itemsWithContainers) {     // loop por item do pedido
  for (const child of item.productChildren) { // loop por filho
    rawOps.push({ childId, containerId, qty });
  }
}
// Depois: 1 findMany + N updates (ainda assim, N transações)
```

Cada pedido com 10 itens × 5 filhos cada = 50 operações de DB dentro do handler de webhook. Quando o webhook de `orders/create` for ativado, **a Shopify espera resposta em 5 segundos** antes de considerar falha e retentar. Se o banco estiver lento (porque o cron está rodando um loop de 12k produtos), o webhook vai falhar e ser retentado, criando duplicações.

### 3.4 API Pública Sem Cache

```javascript
// /api/get-product — chamado pelo JavaScript do tema para CADA pageview
// Calcula stockCalc = preOrderStock - preOrderSales em tempo real
// Sem cache, sem Redis, sem CDN
```

Uma loja com 12.000 produtos em pré-venda e 1.000 visitantes simultâneos = 1.000 queries simultâneas ao MySQL, todas passando pelo mesmo processo Remix que também está servindo a UI admin e processando webhooks.

### 3.5 Cron Jobs Externos Compartilham o Mesmo Banco

```bash
# package.json
npm run validate-eta:all       # scripts/cronjob-validate-eta.js
npm run sync:products-parents  # scripts/sync-products-parents.js
```

Esses scripts rodam fora do processo Remix mas usam o **mesmo Prisma apontando para o mesmo MySQL**. Quando rodam, disputam conexões com a UI e os webhooks. Com 12k produtos, o cron vai segurar o connection pool por minutos.

---

## 4. Por que Microserviços São a Solução Correta

A questão não é só performance — é **isolamento de falhas** e **escalabilidade independente**. Atualmente, um CSV pesado trava a UI. Um cron lento engole a API pública. Um bug de webhook derruba tudo.

### Arquitetura Proposta

```
                    ┌─────────────────────┐
                    │   Shopify Webhooks   │
                    └──────────┬──────────┘
                               │ HTTP (5s timeout)
                               ▼
              ┌────────────────────────────────┐
              │     Webhook Ingestor           │  ← recebe e ACK imediato
              │   (Node.js leve, sem lógica)   │
              └──────────────┬─────────────────┘
                             │ publica em fila
                             ▼
              ┌─────────────────────────────────┐
              │          Message Queue          │
              │     (BullMQ + Redis  ou  SQS)   │
              └──┬──────────┬──────────┬────────┘
                 │          │          │
     ┌───────────┘   ┌──────┘   ┌─────┘
     ▼               ▼          ▼
┌─────────┐  ┌──────────┐  ┌──────────────┐
│  Order  │  │Inventory │  │   Product    │
│ Worker  │  │  Worker  │  │   Worker     │
│(pedidos)│  │(estoque) │  │  (updates)   │
└────┬────┘  └────┬─────┘  └──────┬───────┘
     │             │               │
     └──────────┬──┘               │
                ▼                  ▼
     ┌─────────────────┐   ┌───────────────────┐
     │   MySQL (RDS)   │   │  Shopify Admin API │
     └─────────────────┘   └───────────────────┘

┌─────────────────────────────────────────────────────┐
│               Serviços Independentes                │
├────────────────┬────────────────┬───────────────────┤
│  CSV Service   │  Sync Service  │  Storefront API   │
│ (import/export)│  (cron ETAs,   │  (get-product,    │
│  assíncrono    │  sync bundles) │  badges — cached) │
│  com progresso │  agendado      │  Redis/CDN        │
└────────────────┴────────────────┴───────────────────┘

┌─────────────────────────────────────────────────────┐
│              Remix App (atual)                      │
│  APENAS: UI admin + autenticação Shopify            │
│  Sem processamento pesado, sem cron, sem CSV sync   │
└─────────────────────────────────────────────────────┘
```

---

## 5. Detalhamento de Cada Microserviço

### 5.1 Webhook Ingestor — Isolamento Crítico

**Problema atual:** Webhook handler dentro do Remix faz queries de DB e chamadas Shopify API dentro do timeout de 5s da Shopify. Se falhar, gera retentativas duplicadas que o `WebhookIdempotency` precisa filtrar.

**Solução:** Serviço separado que só faz **duas coisas**: validar HMAC + publicar na fila. Responde 200 em <100ms. O processamento real acontece nos Workers com retry automático, backoff exponencial e dead-letter queue.

**Stack sugerida:** Node.js + Fastify (leve e rápido) + BullMQ + Redis

**Por que separado:**
- Pode ter 0 downtime durante deploys do Remix
- Pode escalar horizontalmente sem afetar a UI
- Falhas nos workers não atrasam o ACK para a Shopify

---

### 5.2 CSV Service — Sem Bloquear a UI

**Problema atual:** Upload de CSV de 12k linhas bloqueia o processo Remix por 60+ segundos, travando todos os outros usuários. Exportações são salvas em disco local (`public/exports/`), incompatível com múltiplas réplicas.

**Solução:** CSV Service aceita o arquivo, responde imediatamente com um `jobId`, processa em background com BullMQ. A UI faz polling no `jobId` para ver progresso (0%... 45%... 100%). Arquivos exportados vão para **Digital Ocean Spaces** (não disco local), com URL pré-assinada para download direto.

**Stack sugerida:** Node.js Worker + BullMQ + Digital Ocean Spaces + progress events via SSE ou polling

**Por que separado:**
- Pode processar arquivos grandes sem afetar latência do Remix
- Pode ter workers dedicados com mais memória (`--max-old-space-size`)
- Exportações paralelas de múltiplos shops não competem entre si
- URL do Digital Ocean Spaces funciona em qualquer réplica, sem acoplamento ao disco do servidor

---

### 5.3 Sync Service — Cron com Controle Real

**Problema atual:** `scripts/cronjob-validate-eta.js` roda sequencialmente por todos os shops, segurando conexões do banco por minutos. Não tem fila, não tem retry, não tem observabilidade. Compete com a UI e webhooks pelo mesmo connection pool MySQL.

**Solução:** Sync Service com scheduler (BullMQ Scheduler ou cron Cloud). Valida ETA processando shops em paralelo com concorrência controlada (`concurrency: 5`). Cada shop é um job independente — se um falha, os outros continuam. Usa seu próprio connection pool ao MySQL com menor prioridade.

**Stack sugerida:** BullMQ Scheduler + Redis + pool dedicado Prisma (connection limit separado)

**Por que separado:**
- O cron hoje compete com webhooks e UI pelo mesmo connection pool MySQL
- Separado, pode ter seu próprio pool com menor prioridade e não impactar a UX
- Jobs falhos entram em dead-letter queue para reprocessamento manual
- Observabilidade real: dashboard BullMQ mostra jobs ativos, falhos, tempo médio

---

### 5.4 Storefront API — Cache na Borda - Analisar a possibilidade de usar um serviço de edge (Cloudflare Workers) para os endpoints públicos chamados pelo tema Shopify, com cache Redis de 60s para aliviar a carga do MySQL. ou outro modelo.

**Problema atual:** `/api/get-product` é chamado por cada pageview do tema Shopify, sem cache, passando pelo Remix que vai ao MySQL. Com tráfego alto, satura o banco.

**Contexto importante:** esse endpoint **não é parte do Shopify Admin**. É uma API pública, sem autenticação, chamada pelo JavaScript do tema no browser do cliente final da loja — completamente separada do app Remix embarcado no admin. O app admin (Remix + Polaris) **não muda** e continua embarcado normalmente no Shopify Admin.

**Solução:** Extrair apenas os endpoints públicos (`/api/get-product`, `/api/get-checkout-badges`, `/api/validate-eta`) para um serviço de edge com cache Redis de 60s. A query é read-heavy e stateless — candidato perfeito para edge.

**Stack sugerida:** Cloudflare Workers + Redis (Upstash) + cache-control headers

**Por que separado:**
- Latência de ~10ms (edge cache) vs ~200ms (Remix → MySQL)
- Pode aguentar 100.000 requests/min sem impactar o admin
- Escala automaticamente no CDN, sem custo de instâncias extras
- Cache pode ser invalidado seletivamente quando um produto muda (via webhook)
- A Shopify já usa Cloudflare como CDN — o domínio da loja já passa pela rede deles, o Worker intercepta antes de sair

**Separação clara de responsabilidades:**

| | Remix App (não muda) | Storefront API (edge) |
|---|---|---|
| Quem usa | Lojista no Shopify Admin | Cliente final navegando na loja |
| Autenticação | Shopify OAuth obrigatório | Nenhuma — endpoint público |
| Frequência | Baixa | Muito alta (cada pageview) |
| Pode ir para edge? | Não | Sim |

**Alternativa mais simples:** se preferir manter tudo na DigitalOcean, um Redis com TTL de 60s na frente do endpoint Remix resolve o problema de carga no MySQL sem nova infraestrutura — não resolve latência geográfica, mas elimina o gargalo principal.

---

### 5.5 Backend de Endpoints para Alterações (Mutations API)

**Problema atual:** Mudanças de configuração de produto (ativar pré-venda, mudar template, alterar stock) passam pelo Remix action, que chama a Shopify API de forma **síncrona** enquanto o usuário espera a resposta na tela.

**Solução:** API Service dedicado (Express ou Fastify) que recebe as mutations, aplica no DB, e **enfileira** as atualizações na Shopify API. A UI recebe confirmação imediata; a propagação para a Shopify API acontece em background com rate limiting controlado. O `shopifyAdminRestThrottle.js` — que já existe — é movido para cá.

**Stack sugerida:** Fastify + BullMQ (fila de mutations Shopify) + Redis (rate limit state)

**Por que separado:**
- Usuário não fica esperando a Shopify API responder (~300ms por chamada)
- Rate limiting centralizado: múltiplos usuários não ultrapassam o teto da API Shopify
- Pode retentar mutations falhas sem impactar a UX
- Remix vira apenas camada de UI/auth, sem lógica de negócio pesada

---

## 6. Impacto de Cada Gargalo com a Solução

| Cenário | Monolito (12k produtos) | Com Microserviços |
|---|---|---|
| Cron de ETA rodando | UI trava 5-10 min, webhooks atrasam | Processo isolado, sem impacto na UI |
| Import CSV de 5k linhas | Todos os usuários sem resposta por ~60s | Job async, UI livre, progresso em tempo real |
| Pico de 1k pageviews na loja | API pública satura MySQL, admin fica lento | Edge cache absorve, MySQL não vê a carga |
| Shopify dispara 500 webhooks | Processo Remix pode perder ou duplicar eventos | Fila garante processamento ordenado com retry |
| Deploy de nova versão | Downtime de ~30s mata webhooks em voo | Ingestor continua up; Workers drenados antes de parar |
| Bug em CSV import | Derruba o processo principal | Apenas o CSV worker para; UI e webhooks continuam |
| Shopify API rate limit atingido | Erros 429 chegam na UI para o usuário | Fila segura, retenta com backoff, usuário não vê erro |

---

## 7. Ordem de Extração Recomendada (Menor Risco → Maior Ganho)

```
Fase 1 — Sem tocar no core do Remix
  1. Webhook Ingestor + BullMQ/Redis
     ↳ Extrai os handlers, coloca fila na frente
     ↳ Remix continua recebendo, mas delega imediatamente

  2. Storefront API no Edge
     ↳ /api/get-product → Cloudflare Worker + Redis cache
     ↳ Remove carga pesada do MySQL principal

Fase 2 — Extrair processamento pesado
  3. CSV Service assíncrono
     ↳ Aceita upload, devolve jobId, processa em worker
     ↳ Digital Ocean Spaces para arquivos exportados

  4. Sync Service para cron
     ↳ ETA validation + sync-products-parents como jobs agendados
     ↳ Pool de conexões separado ao MySQL

Fase 3 — Backend de mutations
  5. Mutations API Service
     ↳ Operações de escrita com enfileiramento para Shopify API
     ↳ Remix vira apenas camada de UI/auth

```

---

## 8. Observação Final

O projeto já tem uma estrutura que **antecipa** essa separação. A divisão entre `endpoints/`, `utils/`, `hooks/` e `components/` respeita fronteiras de domínio que mapeiam diretamente para os microserviços. O trabalho é principalmente **mover código que já existe** para processos que podem escalar e falhar independentemente.

O maior risco de não fazer essa separação não é a lentidão isolada de uma feature: é o **efeito cascata**. Um CSV lento trava o Remix, que atrasa webhooks, que fazem a Shopify retentar, que sobrecarregam mais o banco, que deixa o cron mais lento, que deixa ETAs desatualizados, que causa vendas indevidas de produtos já esgotados.

---

## 9. Aviso sobre as Sugestões deste Documento

As stacks, fases de extração, modelos de cache e decisões arquiteturais descritas aqui são **sugestões iniciais baseadas na análise do estado atual** — não um plano definitivo.

Ainda será feito um **planejamento mais detalhado e direto**, considerando restrições reais de infraestrutura, prioridades de negócio e capacidade de execução. Decisões como escolha de ferramentas (BullMQ vs SQS, Cloudflare vs Redis na DO, redesign do schema vs adaptação incremental) serão revisadas nessa etapa.

Este documento serve como diagnóstico e ponto de partida para essa conversa — não como especificação técnica final.
