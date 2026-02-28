---
name: ace-orchestrator
description: Orquestra a implementação completa do sistema ACE (Agentic Context Engineering). Use quando precisar implementar o sistema de aprendizado de contexto em um projeto de agente LLM do zero, seguindo a ordem correta de dependências baseada no artigo ACE e nas lições aprendidas da implementação de referência.
---

# ACE Orchestrator: Guia Completo de Implementação (Revisado)

## Visão Geral

Este documento orchestra a implementação completa do sistema **ACE (Agentic Context Engineering)** descrito no artigo de Qizheng Zhang et al. O ACE resolve dois problemas principais:

1. **Brevity Bias**: Otimizadores de prompt criam instruções curtas que perdem detalhes específicos de domínio
2. **Context Collapse**: Resumos monolíticos perdem informações valiosas ao longo do tempo

O sistema ACE treats contextos como **playbooks evolutivos** que acumulam, refinam e organizam estratégias através de um processo modular.

---

## Arquitetura do Sistema

### Os Três Pilares (Seção 3 do Artigo)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ARQUITETURA ACE                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
│   │  GENERATOR  │────▶│  REFLECTOR  │────▶│  CURATOR   │          │
│   └─────────────┘     └─────────────┘     └─────────────┘          │
│         │                   │                   │                  │
│         │                   │                   │                  │
│         ▼                   ▼                   ▼                  │
│   Produz reasoning    Analisa execuções    Aplica deltas          │
│   trajectories        e extrai lições      incrementais          │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────┐       │
│   │              SEMANTIC EMBEDDINGS (model2vec)            │       │
│   │  • Deduplicação semântica (Seção 3.2 do artigo)        │       │
│   │  • Recuperação inteligente de entries                 │       │
│   └─────────────────────────────────────────────────────────┘       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Dois Modos de Operação

```
┌─────────────────────────────────────────────────────────────────────┐
│                     MODOS DE OPERAÇÃO                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────┐        ┌─────────────────────┐            │
│  │    OFFLINE          │        │     ONLINE          │            │
│  ├─────────────────────┤        ├─────────────────────┤            │
│  │ • Pré-sessão        │        │ • Durante sessão   │            │
│  │ • System prompt     │        │ • Memória агент     │            │
│  │ • Batch processing  │        │ • Real-time         │            │
│  │ • seed from docs    │        │ • Execution feedback│            │
│  └─────────────────────┘        └─────────────────────┘            │
│                                                                      │
│  Usar quando:                        Usar quando:                  │
│  - Novo projeto                      - Já existe histórico         │
│  - Melhorar system prompt            - Sessão em andamento          │
│  - Preparar contexto inicial         - Adaptar a erros/sucessos     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Ordem de Implementação (Revisada)

A ordem abaixo é **OBRIGATÓRIA** devido às dependências entre módulos. Cada etapa depende da anterior.

```
┌────────────────────────────────────────────────────────────────────────┐
│                     DEPENDÊNCIAS                                       │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ETAPA 1: FOUNDATION                  (base de tudo)                │
│  ├── 1.1 Schema (Zod)                  - Tipos e validação           │
│  ├── 1.2 Storage                        - Persistência JSON          │
│  └── 1.3 Base Playbook                  - Estratégias seed inicial   │
│         │                                                             │
│         ▼                                                             │
│  ETAPA 2: AGENTIC ROLES                    (núcleo do ACE)           │
│  ├── 2.1 Reflector                       - Extrai lições            │
│  ├── 2.2 Curator                         - Aplica deltas            │
│  └── 2.3 Generator                       - Recupera entries          │
│         │                                                             │
│         ▼                                                             │
│  ETAPA 3: SEMANTIC INTELLIGENCE            (embeddings)              │
│  ├── 3.1 model2vec                       - Embeddings semânticos     │
│  ├── 3.2 Python fallback                  - Script Python             │
│  └── 3.3 Levenshtein fallback            - Similaridade textual      │
│         │                                                             │
│         ▼                                                             │
│  ETAPA 4: CORE INTEGRATION                  (conecta ao sistema)     │
│  ├── 4.1 Processor hook                  - Extrai lições             │
│  ├── 4.2 Compaction hook                   - Usa playbook             │
│  └── 4.3 System prompt                     - Injeta playbook          │
│         │                                                             │
│         ▼                                                             │
│  ETAPA 5: MULTI-EPOCH REFLECTION            (refinamento iterativo) │
│  ├── 5.1 Iterative extraction             - Refina lições           │
│  ├── 5.2 Cross-validation                  - Valida lições           │
│  └── 5.3 Quality scoring                   - Score de qualidade      │
│         │                                                             │
│         ▼                                                             │
│  ETAPA 6: OFFLINE PREPARATION               (preparação pré-sessão)  │
│  ├── 6.1 Document seeding                  - Extrai de docs         │
│  ├── 6.2 Project initialization             - Setup inicial         │
│  └── 6.3 System prompt optimization          - Modo offline          │
│         │                                                             │
│         ▼                                                             │
│  ETAPA 7: EVALUATION & MONITORING            (avaliação)             │
│  ├── 7.1 Metrics collection                 - Track qualidade        │
│  ├── 7.2 A/B testing                         - Comparar abordagens    │
│  └── 7.3 Threshold tuning                     - Ajuste dinâmico       │
│         │                                                             │
│         ▼                                                             │
│  ETAPA 8: MAINTENANCE & SCALING              (robustez)              │
│  ├── 8.1 Rate Limiting                       - Previne spam          │
│  ├── 8.2 Pruner                               - Remove obsoletas      │
│  ├── 8.3 WAL                                  - Durability            │
│  └── 8.4 Locking                               - Concorrência         │
│         │                                                             │
│         ▼                                                             │
│  ETAPA 9: USER INTERFACE                     (interação)             │
│  └── 9.1 Slash Commands                      - /playbook             │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Por Que Esta Ordem Revisada

| Etapa            | Dependência      | Por Que (Revisado)                               |
| ---------------- | ---------------- | ------------------------------------------------ |
| Foundation       | Nenhuma          | Base de tudo                                     |
| Agentic Roles    | Foundation       | Precisa do schema para funcionar                 |
| Semantic         | Agentic Roles    | Usa Curator para deduplicação                    |
| Core Integration | Todas anteriores | Conecta tudo ao agente                           |
| Multi-Epoch      | Core Integration | REQUER feedback real para iterar (artigo: +3.7%) |
| Offline Prep     | Core Integration | Pode rodar antes de sessão começar               |
| Evaluation       | Multi-Epoch      | Precisa de lições para avaliar                   |
| Maintenance      | Core Integration | Precisa estar integrado para funcionar bem       |
| UI               | Maintenance      | Usa Curator/Generator existentes                 |

---

## Etapa 1: Foundation

**Por que primeiro?** Sem schema e storage, não há onde salvar as lições aprendidas.

### Conceito do Artigo (Seção 3.1)

> "Each bullet consists of: Metadata (unique identifier and counters tracking helpful/harmful) + Content (a small unit such as a reusable strategy)"

### Referência

- Skill: `ace-foundation/SKILL.md`
- Código: `packages/opencode/src/session/playbook/schema.ts`

---

## Etapa 2: Agentic Roles

**Por que segundo?** Agora que temos onde salvar, precisamos das funções que manipulam os dados.

### Conceito do Artigo (Seção 3)

> "1. **Generator**: Produces reasoning trajectories for new queries, which surface both effective strategies and recurring pitfalls. 2. **Reflector**: Critiques these traces to extract lessons, optionally refining them across multiple iterations. 3. **Curator**: Synthesizes these lessons into compact delta entries, which are merged deterministically into the existing context by lightweight, non-LLM logic."

### Fluxo de Dados

```
Tool Execution
      │
      ▼
┌─────────────┐
│  REFLECTOR  │  ← Analisa output, detecta erros/sucessos
└─────────────┘
      │  ExecutionLesson { content, tags, isPositive }
      ▼
┌─────────────┐
│   CURATOR   │  ← Cria/Atualiza/Remove entries
└─────────────┘
      │  PlaybookDelta { added[], updated[], removed[] }
      ▼
┌─────────────┐
│   STORAGE   │  ← Persiste no JSON
└─────────────┘
```

### Referência

- Skill: `ace-agentic-roles/SKILL.md`
- Código: `packages/opencode/src/session/playbook/reflector.ts`, `curator.ts`, `generator.ts`

---

## Etapa 3: Semantic Intelligence (model2vec)

**Por que terceiro?** Agora que temos o básico funcionando, adicionamos embeddings para melhorar a qualidade da deduplicação e busca.

### Por que o artigo usa embeddings (Seção 3.2)

> "A de-duplication step prunes redundancy by comparing bullets via semantic embeddings"

### Threshold Guidance (NOVO)

O threshold de deduplicação é crítico e depende do domínio:

| Domínio      | Threshold Recomendado | Por Que                          |
| ------------ | --------------------- | -------------------------------- |
| Código       | 0.80-0.85             | 相似 sintático é mais previsível |
| Documentação | 0.75-0.80             | Mais variações de linguagem      |
| Commands     | 0.85-0.90             | Comandos são muito específicos   |
| General      | 0.75                  | Safe default                     |

### Arquitetura de Fallback

```
┌─────────────────────────────────────────────────────────────┐
│              EMBEDDINGS BACKEND CHAIN                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  embedTexts(texts[])                                        │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────────┐  ← Mais rápido (~1.7x que Python)  │
│  │ 1. Rust/Native  │    Zero overhead de processo        │
│  │    (model2vec)  │    ~128MB modelo                     │
│  └────────┬─────────┘                                       │
│           │ Fallback                                        │
│           ▼                                                 │
│  ┌──────────────────┐  ← Script Python subprocess        │
│  │ 2. Python        │    Funciona sem Rust                │
│  │    (subprocess) │    pip install model2vec             │
│  └────────┬─────────┘                                       │
│           │ Fallback                                        │
│           ▼                                                 │
│  ┌──────────────────┐  ← Similaridade textual simples     │
│  3. Levenshtein    │    Sempre funciona                   │
│  (JS native)       │    fallback final                    │
│  └──────────────────┘                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Quando Usar Embeddings vs Keywords

| Cenário             | Recomendação           |
| ------------------- | ---------------------- |
| Primeiro uso        | Keywords (mais rápido) |
| +50 entries         | Adicionar embeddings   |
| Domínio técnico     | Keywords + embeddings  |
| Performance crítica | Só keywords            |

### Referência

- Skill: `ace-semantic-intelligence/SKILL.md`
- Código: `packages/opencode/src/session/playbook/embeddings/index.ts`

---

## Etapa 4: Core Integration

**Por que quarto?** Agora que o sistema de aprendizado existe, precisamos conectá-lo ao fluxo real do agente.

### Por que é crítico

Sem esta etapa, o sistema **nunca é executado**. É a "fiação" que conecta o módulo de aprendizado ao fluxo do agente.

### Conceito do Artigo (Seção 3.1 - Localization)

> "Localization: Only the relevant bullets are updated"

Isto é possível porque o Generator busca apenas as entries relevantes para o contexto atual.

### Referência

- Skill: `ace-core-integration/SKILL.md`
- Código: `packages/opencode/src/session/processor.ts`, `compaction.ts`

---

## Etapa 5: Multi-Epoch Reflection (NOVO!)

**Por que agora?** O artigo mostra que reflexão iterativa adiciona +3.7% de performance:

> "Reflector with iterative refinement and multi-epoch adaptation each contribute substantially to performance"

### O que é Multi-Epoch

```
┌─────────────────────────────────────────────────────────────────┐
│                 MULTI-EPOCH REFLECTION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Epoch 1: Extract         Epoch 2: Refine       Epoch 3: Final│
│  ┌─────────────┐         ┌─────────────┐        ┌────────────┐ │
│  │ "bash failed│         │ "ENOENT     │        │ "When      │ │
│  │ ENOENT"     │ ──────▶ │ means file  │ ──────▶│ file not   │ │
│  │             │         │ not found"  │        │ found:     │ │
│  └─────────────┘         └─────────────┘        │ check path ││
│                                                │ or create"  ││
│                                                └────────────┘ │
│                                                                  │
│  Sem multi-epoch: 55.7%    Com multi-epoch: 59.4% (+3.7%)      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementação

1. **Extração inicial**: Reflector extrai lição bruta
2. **Refinamento**: Segunda passagem generaliza a lição
3. **Validação**: Verifica se a lição é acionável

### Quando Usar

- **Sempre**: Para lições extraídas via LLM
- **Para erros críticos**: Para lições negativas importantes
- **Não usar**: Para lições simples (fire-and-forget)

### Referência

- Skill: `ace-multi-epoch-reflection/SKILL.md`

---

## Etapa 6: Offline Preparation (NOVO!)

**Por que agora?** Permite preparar playbooks antes de sessões começarem.

### Casos de Uso

1. **Novo projeto**: Criar playbook seed a partir de documentação
2. **System prompt optimization**: Preparar contexto inicial offline
3. **Batch processing**: Processar histórico de execuções passadas

### O que Implementar

1. **Document Seeding**: Extrair estratégias de README, docs, código
2. **Project Init**: Setup inicial quando novo projeto é aberto
3. **Offline Mode**: Modo batch para otimização de system prompt

### Referência

- Skill: `ace-offline-preparation/SKILL.md`

---

## Etapa 7: Evaluation & Monitoring (NOVO!)

**Por que agora?** Sem avaliação, não sabemos se o ACE está funcionando.

### Métricas Importantes

| Métrica                | Descrição                         |
| ---------------------- | --------------------------------- |
| Lesson Quality Score   | Quão acionável é a lição          |
| Retrieval Success Rate | Lições recuperadas foram úteis?   |
| Deduplication Rate     | Quantas lições foram mescladas    |
| Context Improvement    | Delta de performance com playbook |

### A/B Testing

```
┌─────────────────────────────────────────────────────────────────┐
│                      A/B TESTING                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Request A (playbook):    Request B (no playbook):             │
│  ┌─────────────┐          ┌─────────────┐                       │
│  │ + playbook  │          │ - playbook  │                       │
│  │ context     │          │ (baseline)  │                       │
│  └─────────────┘          └─────────────┘                       │
│         │                         │                              │
│         └────────────┬────────────┘                              │
│                      ▼                                           │
│              Compare success rate                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Referência

- Skill: `ace-evaluation-framework/SKILL.md`

---

## Etapa 8: Maintenance & Scaling

**Por que oitavo?** O sistema já funciona, agora precisa ser robusto para produção.

### Conceito do Artigo (Seção 3.2 - Grow-and-Refine)

> "This refinement can be performed proactively (after each delta) or lazily (only when the context window is exceeded)."

### Rate Limiting

```typescript
const RATE_LIMIT = {
  maxLessonsPerMinute: 10,
  maxSimilarLessonsPerMinute: 5,
  dedupWindowMs: 60000,
}
```

### Pruner

```typescript
// Scores para decisão de poda
function score(entry): number {
  return entry.helpfulCount * 2 + entry.retrievalCount - entry.harmfulCount * 3
}
```

### Referência

- Skill: `ace-maintenance-scaling/SKILL.md`
- Código: `packages/opencode/src/session/playbook/rate-limit.ts`, `pruner.ts`

---

## Etapa 9: User Interface

**Por que por último?** O sistema já funciona de forma autônoma. Isto é apenas para permitir interação manual.

### Comandos

| Comando                  | Descrição                                |
| ------------------------ | ---------------------------------------- |
| `/playbook list [n]`     | Lista n entries mais úteis               |
| `/playbook show <id>`    | Mostra detalhes completos                |
| `/playbook helpful <id>` | Marca como útil (+1 helpfulCount)        |
| `/playbook harmful <id]` | Marca como prejudicial (+1 harmfulCount) |
| `/playbook stats`        | Estatísticas do playbook                 |
| `/playbook clear`        | Limpa todas as entries                   |
| `/playbook eval`         | Avalia qualidade do playbook             |
| `/playbook help`         | Ajuda                                    |

### Conceito do Artigo (Seção 3.1 - Metadata)

> "Metadata: unique identifier and counters tracking how often it was marked helpful or harmful"

### Referência

- Skill: `ace-user-interface/SKILL.md`
- Código: `packages/opencode/src/session/playbook/commands.ts`

---

## Configuração Final

```json
{
  "playbook": {
    "enabled": true,
    "mode": "webdev",
    "deduplicationThreshold": 0.75,
    "maxEntries": 500,
    "useEmbeddings": true,
    "multiEpochReflection": true,
    "offlinePreparation": true
  }
}
```

### Modos Disponíveis

| Modo           | Escopo      | maxEntries | Uso                    |
| -------------- | ----------- | ---------- | ---------------------- |
| `webdev`       | Global      | 500        | Projetos web gerais    |
| `conservative` | Global      | 500        | Pouca poda             |
| `gamedev`      | Por projeto | 200        | Jogos (escopo fechado) |
| `aggressive`   | Por projeto | 200        | Máxima poda            |

---

## Contextos Onde Esta Versão É Insatisfatória

| Contexto          | Limitação Atual   | Solução Proposta                   |
| ----------------- | ----------------- | ---------------------------------- |
| Multi-tenancy     | Isolamento básico | Adicionar tenant ID                |
| Real-time collab  | Sem sync          | Adicionar CRDT ou server           |
| Very long horizon | Sem archival      | Adicionar compressão older entries |
| Privacy-sensitive | Dados locais      | Adicionar encryption               |

---

## Resumo das Novidades vs Versão Original

1. **Modo Offline/Online** - Diferencia preparação de execução
2. **Multi-Epoch Reflection** - Refinamento iterativo (+3.7%)
3. **Threshold Guidance** - Por domínio
4. **Evaluation Framework** - Métricas e A/B testing
5. **Offline Preparation** - Seed de documentação
6. **Quando usar embeddings** - Guidance de performance

---

## Referências Completas

1. **Artigo Original**: `ArtigoACE.md` - Qizheng Zhang et al., "Agentic Context Engineering"
2. **Skill: ace-foundation**: `ORQUESTRATOR/ace-foundation/SKILL.md`
3. **Skill: ace-agentic-roles**: `ORQUESTRATOR/ace-agentic-roles/SKILL.md`
4. **Skill: ace-semantic-intelligence**: `ORQUESTRATOR/ace-semantic-intelligence/SKILL.md`
5. **Skill: ace-core-integration**: `ORQUESTRATOR/ace-core-integration/SKILL.md`
6. **Skill: ace-multi-epoch-reflection**: `ORQUESTRATOR/ace-multi-epoch-reflection/SKILL.md` (NOVO)
7. **Skill: ace-offline-preparation**: `ORQUESTRATOR/ace-offline-preparation/SKILL.md` (NOVO)
8. **Skill: ace-evaluation-framework**: `ORQUESTRATOR/ace-evaluation-framework/SKILL.md` (NOVO)
9. **Skill: ace-maintenance-scaling**: `ORQUESTRATOR/ace-maintenance-scaling/SKILL.md`
10. **Skill: ace-user-interface**: `ORQUESTRATOR/ace-user-interface/SKILL.md`
11. **Código de Referência**: `packages/opencode/src/session/playbook/`
